# Upload Service Intermediary Revision (Lynx Boreal)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).

## Scope
### Outline:
The goal of this epic is to overhaul the Upload Controller Service (UCS) as part of the
new [File Upload concept](https://ghga.pages.hzdr.de/internal.ghga.de/feature_archconcept-file-upload/developer/architecture_concepts/ac007_file_upload/).
While AC 7 includes references to future services like the Study Repository Service, some of the machinery there is still in the planning phase. This epic will therefore rely upon some shortcuts, such as extra Data Steward involvement. When the full architectural concept is realized, the appropriate adaptations will follow in a future epic.

![UCS Diagram](./images/ucs.png)
> Future Final UCS Diagram with Study Repository Service


#### Domain Objects
The UCS owns two domain objects, which it broadcasts as outbox events via Kafka. The
first domain object is the `UploadContext`, which broadly serves to delineate
in-progress and finalized file submissions for a given study. The second domain
object is the `FileUpload`. As its name suggests, the `FileUpload` object reflects
the upload status of a single file within an `UploadContext`. Thus, there is a
hierarchical, one-to-many relationship between `UploadContext` and `FileUpload`.

We will define the Pydantic models for these two classes in `ghga-event-schemas`,
along with one stateful config class for each.

#### Inputs
The UCS only receives user input in the form of HTTP requests. It doesn't subscribe to
any Kafka events. However, we will define a slim CLI interface for the service that
exposes commands to `run-rest` and `publish-events`. These commands are commonly seen
across our services at this time.

### Outputs
There are three categories of output in the UCS: HTTP responses, published events, and
data stored in the database. HTTP responses are described below in the API Definitions
section. The published events and database storage are driven simultaneously by
Hexkit's MongoKafkaDaoPublisher, which the UCS uses to store `UploadContext` and
`FileUpload` instances. Anytime an `UploadContext` or `FileUpload` is created, modified,
or deleted, the UCS publishes a Kafka event containing the latest state. This is done
according to the Outbox Pattern (not described in further detail here).

#### Auth
Before general users (not Data Stewards) can upload files, three things must happen:
1. A Data Steward must create the `UploadContext` via the Upload Orchestration Service.
2. A Data Steward must grant the user a claim enabling them to use the `UploadContext`.
3. The user must create a Work Package via the Data Portal to obtain a Work Package Access Token (WPAT). Only one WPAT is needed for the entire series of files under normal circumstances.

The user then supplies the WPAT to the `ghga-connector` to upload files.
The `ghga-connector` obtains Work Order Tokens (WOTs) automatically by providing the WPAT to the WPS, and then makes at least two calls for each file:
1. A POST request to the UCS to create the `FileUpload`.
2. A GET request to the UCS to obtain a file part upload URL. This call is repeated for each file part.

Both the POST and GET requests will require a valid WOT to succeed.

There is an important difference here from the download path's usage of WOTs: during download, WOTs must carry the file ID being requested for download. For upload, file IDs are not used, and WOTs merely carry the Upload Context's uuid.
This is because we want the upload process to not be too restrictive nor prescriptive. This way, the user can upload the files they want without announcing them beforehand.

When the user has completed uploading all files, the user (or a Data Steward) moves the `UploadContext` to the `LOCKED` state and eventually to the `CLOSED` state by an HTTP call to the UOS. 
Once `CLOSED`, the WPS and Claims Repository will process the 'deleted' Kafka event and disable/delete any existing claims or tokens.

In summary:
- UOS:
  - The Data Steward role required to create `UploadContext`s
  - A valid WPAT or the Data Steward role is required to change the state of an `UploadContext`.
- UCS:
  - Upserting/Deleting `UploadContext` requires an internal service mesh token
  - Getting an upload url requires a valid WOT

For more information on the HTTP API, see the endpoint definitions below.

### Included/Required:
- Remove existing core logic
- Create new core class w/ outbox publisher
- Write Unit and Integration Tests

### Not included:
Archive test bed integration, Study Repository Service development, or front end work.

## User Journeys

### UploadContext Creation
# TODO: review this section after reaching consensus on UOS/context-claim creation
A Data Steward makes a call to the UOS to create a new `UploadContext`, specifying the ID(s) of the user(s) who will upload files.
The UOS calls the UCS's `POST /contexts` endpoint to create the actual `UploadContext` with the state set
to `OPEN`. The UCS issues an outbox event, which is consumed by the Work Package
Service. Afterward, the UOS makes subsequent calls to the CRS to award upload claims to
the user(s) for the given `UploadContext` (specified by ID). Without this claim, the
user cannot create a work package or upload files.

### UploadContext Update
For now, only Data Stewards are allowed to change the state of an `UploadContext`.
Therefore, a user must communicate with a Data Steward when they are ready to LOCK or CLOSE an `UploadContext`.

The Data Steward calls the `PATCH /contexts/{context_id}` endpoint on the UCS API,
authenticating the call with a WOT which also contains the context ID.
The request body should contain at least the desired context state.

The UCS updates the state of the `UploadContext` to `LOCKED`, `CLOSED`, or `OPEN`, as
specified by the request. If the `UploadContext` is already in the given state, nothing
happens and the UCS returns a successful response.
The initial state of the `UploadContext` is `OPEN`. When the user finishes uploading
files, they can ask a Data Steward to set the Context to a semi-finalized state,
`LOCKED`. It is possible that the user decides they need to make changes, such as
uploading or removing a file, and in that case a Data Steward can revert the Context to `OPEN`.
If no changes are needed, however, the user can request the Data Stewards to fully finalize the Context by setting
it to `CLOSED`, after which point no changes can be made without opening a new
`UploadContext`.
If user tries to change the status of an `UploadContext` that's already set to `CLOSED`,
they receive an error. Once the update operation is complete, the UCS publishes a Kafka
event reflecting the latest state of the `UploadContext` and returns an HTTP response
indicating the update was successful.

In a future iteration of the upload path, users will directly update context state
themselves via the Data Portal or `ghga-connector`.

![UploadContext State Diagram](./images/upload_context.png)

An `UploadContext` may only be moved from `LOCKED` to `CLOSED` if all its linked
`FileUpload`s are set to `COMPLETED`.

### File Upload Init
Using the Data Portal, the user creates an upload work package for the `UploadContext`
and receives a WPAT. The following is accomplished using the
`ghga-connector` and the WPAT:

1. The user initiates the upload process for a given single file (can be looped or batched):
   - The Connector contacts the WPS and exchanges the WPAT + context ID for a WOT.
   - The Connector calls the UCS's `POST /contexts/{context_id}/uploads/` endpoint.
The request body includes the unencrypted checksum, the file alias (or whichever naming element is used to match the file
with the metadata content), and possibly further information. The ID of the `UploadContext` is included in the WOT.
   - The UCS ensures it doesn't already have a completed `FileUpload` for the same file, then adds the `FileUpload` to the associated `UploadContext`.
   - The UCS initiates a multipart upload for the file.
   - The UCS publishes upsertion events to Kafka for both the `FileUpload` and `UploadContext` objects, and returns an HTTP response to the Connector indicating
that the file upload was successfully initiated. 
     - The response contains the UCS-generated UUID4 identifier of the new file upload.

2. The Connector uploads the file in chunks:
   - For each file part, the Connector uses a valid WOT (or gets a new one) to call the UCS's `GET /contexts/{context_id}/uploads/{file_id}/parts/{part_no}` endpoint.
   - Assuming the WOT is valid, the UCS returns a presigned, short-lived upload URL.
   - The Connector uploads the file parts until complete.




### File Upload Termination (Upload Completion)
# TODO: iron out API vocabulary
When the upload is complete, the connector automatically makes a request to the UCS
endpoint `PATCH /contexts/{context_id}/uploads/{file_id}`.
This call instructs the UCS to communicate with the S3 instance and terminate (complete) the multipart upload.
The UCS will update the `FileUpload` instance to `COMPLETED` and publish a Kafka event reflecting the new state.
Finally, the UCS will return an HTTP response indicating the operation was successful.
The Connector displays a message to the user saying the file upload is complete.
If applicable, the Connector proceeds with the next file in the upload batch.

### `FileUpload` Deletion
The user, via the `ghga-connector` and a valid WPAT, makes a request to the
`DELETE /contexts/{context_id}/uploads/{file_id}` endpoint, indicating they wish to
delete a file from the associated Upload Context. If a valid WOT is supplied with the
request, the UCS cancels the ongoing upload if it exists and deletes the `FileUpload`
object from the database. It removes the reference from the `file_uploads` field in the
`UploadContext` and publishes Kafka events reflecting the deletion of the `FileUpload` and the
new state of the Upload Context. Finally, the UCS returns an HTTP response to the user
indicating the deletion was successful.

## API Definitions:

### RESTful/Synchronous:

- `GET /contexts`: Retrieve all `UploadContexts`
  - Requires WOT
  - Data Stewards can see all `UploadContexts`, while users can only see ones they have an active claim for.
- `POST /contexts`: Create a new `UploadContext`
  - Requires WOT and only allowed for Data Stewards via the UOS.
- `GET /contexts/{context_id}`: Retrieve an `UploadContext` by ID
  - Requires WOT
  - Data Stewards can see all `UploadContexts`, while users can only see ones they have an active claim for.
- `PATCH /contexts/{context_id}`: Update an `UploadContext` to change the status
  - Currently only available to Data Stewards, as only they can obtain the required token.
  - Request body must contain the new state of the context
- `POST /contexts/{context_id}/files/`: Add a new `FileUpload` to an existing `UploadContext`
  - Requires WOT
  - Request body must contain the required file upload details
- `PATCH /contexts/{context_id}/uploads/{file_id}`: Conclude file upload in UCS
  - Requires WOT
  - Sets the `FileUpload` status to `COMPLETE` and tells S3 to close the multipart upload.
- `DELETE /contexts/{context_id}/uploads/{file_id}`: Remove a `FileUpload` from the `UploadContext`
  - Requires WOT
  - Deletes the `FileUpload` and tells S3 to cancel the multipart upload if applicable.
- `GET /contexts/{context_id}/uploads/{file_id}/parts/{part_no}`: Get pre-signed S3 upload URL for file part
  - Requires WOT

### Payload Schemas for Events:

```python
class UploadContextState(StrEnum):
    """The allowed states for an UploadContext instance"""

    OPEN = "open"
    LOCKED = "locked"
    CLOSED = "closed"

class UploadContext(BaseModel):
    """A class representing an Upload Context"""

    upload_context_id: UUID4 # unique identifier for the instance
    state: UploadContextState  # one of OPEN, LOCKED, CLOSED
    file_uploads: list[FileUpload]  # use list function for default_factory

class FileUploadState(StrEnum):
    """The allowed states for a FileUpload instance"""

    INIT = "init"
    COMPLETED = "completed"

class FileUpload(BaseModel):
    """A File Upload"""

    upload_id: UUID4
    state: FileUploadState  # one of INIT, COMPLETED
    original_path: str
    checksum: str
```


## Additional Implementation Details:


### Testing
Tests need to cover at least the following items (not exhaustive):
- Standard endpoint authentication battery
- Happy path for each endpoint
- Core error translation for HTTP API for each endpoint
- Disallow changing status of a CLOSED UploadContext
- Disallow removing a file from a CLOSED UploadContext


## Human Resource/Time Estimation:

Number of sprints required: 2

Number of developers required: 1
