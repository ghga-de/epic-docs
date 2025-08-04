# Upload Service Intermediary Revision (Lynx Boreal)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).

## Scope
### Outline:
The goal of this epic is to overhaul the Upload Controller Service (UCS) as part of the
new [File Upload concept](https://ghga.pages.hzdr.de/internal.ghga.de/feature_archconcept-file-upload/developer/architecture_concepts/ac007_file_upload/).
While AC 7 includes references to future services like the Study Repository Service,
some of the machinery there is still in the planning phase. This epic will therefore
rely upon some shortcuts, such as extra Data Steward involvement. When the full
architectural concept is realized, the appropriate adaptations will follow in a future
epic.

### Included/Required:
- Implement new Upload Orchestrator Service as described below
- Revamp existing UCS logic
- Adapt WPS for "upload" work packages and tokens
- Adapt CRS to also manage claims for upload contexts
- Add new schemas to `ghga-event-schemas`

### Not included:
- Archive test bed integration
- Subsequent FIS or IFRS updates for the upload path
- Any front-end work that may be needed
- GHGA-Connector update

### Upload Controller Service (UCS)

The UCS is responsible for two things:
1. Cataloging individual files for a given study without knowing
anything about studies or other business concepts.
2. Facilitating the actual upload of said files by:
   - Opening a new multipart S3 upload
   - Dispensing pre-signed part upload URLs
   - Completing/cancelling the S3 upload

In the UCS, a study is represented by a minimalistic abstraction called an
`UploadContext`, and a file is represented as a `FileUpload`. The former offers a way to
group file uploads and verify when changes are allowed. The latter offers specific
file information and a way for other services to be aware of that file's existence.

Each `UploadContext` is labeled by a random UUID and has a boolean that determines
whether any changes can be made to it or a given associated `FileUpload`, as well as
fields for file count and total file size.
Each `FileUpload` has a random UUID, an alias, a file size, a checksum, a state
(`INIT` or `CLOSED`), and a field containing its parent `UploadContext` ID.
There is a 1:many relationship between `UploadContext` and `FileUpload`.

When there is a new study, a Data Steward triggers a new `UploadContext` via the UOS,
which calls the UCS. Then the user can create a work package for the new `UploadContext`
and use the `ghga-connector` to upload the files for their study. That's the mountaintop
view, see the **User Journeys** for more detailed information.

Both `UploadContext` and `FileUpload` changes are emitted as outbox events, but the
UCS does not consume any events or store any other data. `UploadContext` events are
consumed by the WPS and UOS while the `FileUploads` are consumed by other file services.

The UCS has a REST API, but requests only come from two places: `ghga-connector` and the
UOS. All endpoints are secured by Work Order Tokens (WOTs) signed by either the UOS (see
below) or the Work Package Service (WPS). All requests from the Connector use WOTs from
the WPS, while all requests from the UOS use WOTs signed by the UOS itself.
In this arrangement, the UCS avoids coupling with the auth service or business logic,
and the service as a whole remains fairly lightweight.

For more information on WOTs or API definitions, see below.


### Upload Orchestration Service (UOS)

The UOS is the service in charge of applying business and authentication logic for the
upload path, and it does not perform any of the S3 operations found in the UCS.
The UOS maintains richer information about each ongoing research data upload effort and
produces audit records on who changes that information and when.
While the UCS is tasked with facilitating the actual upload processes, the UOS stores an
enriched version of the `UploadContext`, called a `DataUploadCase`. It also
talks to the Claims Repository Service to control access to the `DataUploadCase`.
Users/Data Stewards can access the UOS through the Data Portal, and requests are
routed through the auth adapter so the UOS can inspect user IDs and roles.

The other responsibility of the UOS is to relieve the UCS from handling browsing needs.
When anyone wants to view existing `DataUploadCase`s, they talk to the UOS, not the UCS.
This is a happy arrangement because viewing this content inherently involves auth:
users should only be able to see a `DataUploadCase` if they actually have access to it.

How the UOS sits in relation to the UCS:  
When there is a new study, a Data Steward logs on to the Data Portal and creates a new
case, called a `DataUploadCase`, for uploading the research data. The Data Steward will
enter at least a title and description so they can later identify the case. This info
will flow from the Data Portal to the UOS, which will tell the UCS to create a new
`UploadContext`. The UOS will save the ID of the `UploadContext` with the other info
in the `DataUploadCase`. Then the Data Steward will grant the relevant user an upload
access claim tied to the `UploadContext` or `DataUploadCase` by calling the UOS, which
in turn talks to the CRS.

Each `DataUploadCase` has a random UUID, a title, a description, a state (`OPEN`,
`LOCKED`, `CLOSED`), and the associated `UploadContext` from the UCS.
Every change to a `DataUploadCase` triggers an outbox event, as well as a persistent
event for auditing with the user ID, timestamp, and info about what action occurred.
The UOS subscribes to `UploadContext` events but not `FileUpload` events.
The UOS offers a REST API for users and Data Stewards, but these endpoints are covered
by the auth protocol rather than WOTs. When a user interacts with the UCS via the UOS,
the UOS signs its own WOTs (assuming everything else is copacetic) authorizing the
specific action. This epic introduces changes and augmentations to GHGA's WOTs, and you
can find more information in the **Auth** section below.

### Work Package Service
The WPS is hardcoded to set up an error when creating a work package if the work type
is anything other than "download". The logic is all download-centric, so the WPS needs
to be updated to accommodate "upload" work packages. To this end, these are the main
points to address:
1. Update `WorkPackageRepository` logic to handle CRUD-ing "upload" work packages
2. Revamp Work Order Tokens (see Additional Implementation Details)
3. Listen for outbox events carrying `UploadContext` data, and store the context IDs
   - Context deletion is not allowed in this version of the upload path. We can add it later.
4. Change `AccessCheckConfig.download_access_url` to `AccessCheckConfig.access_url` to work for both up- and download
5. Augment the `AccessCheckAdapter` so it can call `/upload-access/users/{user_id}/contexts/{context_id}`
   to check if a user has access to a given `UploadContext`
6. Provide a way to distribute WOTs, either by modifying the
   `/work-packages/{work_package_id}/files/{file_id}/work-order-tokens` endpoint or
   replacing it with one or more endpoints that allow passing the type and
   additional token content


### Claims Repository
1. Extend the access API (not the claims API, which is not active at the moment) to also
support upload grants. The new endpoints here will be used by the UOS and the WPS for
granting and verifying access.

Please see the API Definitions for details on the endpoints for extending the
access API.

2. Create a new visa type. The currently-used visas for download,
ControlledAccessGrants, imply download access and should therefore not be used as upload
visas. Instead, we should create a new visa:

type: `GHGA_UPLOAD = "https://www.ghga.de/GA4GH/VisaTypes/Upload/v1.0"`
value: `https://ghga.de/uploads/{context_id}`

We will have to extend the utilities surrounding visa handling to accommodate this
new visa type in the core `claims` module because all the logic there is download-
centric.

### Auth
#### Tokens
The UCS secures its endpoints through WOT authentication. While the Download Controller
Service requires just one flavor of WOT to operate, the UCS requires more:
| Token                          | Issuer | Who                  | Action Authorized                                |
|--------------------------------|--------|----------------------|--------------------------------------------------|
| `CreateUploadContextWorkOrder` | UOS    | Data Stewards        | Create a new `UploadContext`                     |
| `ChangeUploadContextWorkOrder` | UOS    | All                  | Update an existing `UploadContext`               |
| `ListFilesWorkOrder`           | UOS    | All                  | View list of files for an `UploadContext`        |
| `CreateFileWorkOrder`          | WPS    | Users                | Initialize a `FileUpload` and obtain new file ID |
| `UploadFileWorkOrder`          | WPS    | Users                | Obtain a part upload URL for a given file ID. Can also be used to delete the file. |

These tokens authorize users to perform the labeled action within the UCS, and they only
carry information necessary for the given action, such as `file_id` or `context_id` in
addition to the work type.

#### For Enabling User Access to a New Upload Procedure
Before general users (not Data Stewards) can upload files, three things must happen:
1. A Data Steward must create the `DataUploadCase`/`UploadContext` via the Data Portal.
   - The UOS signs a `CreateUploadContextWorkOrder` token and contacts the UCS.
2. A Data Steward must grant the user a claim enabling them to use the `DataUploadCase`.
   - No token is required here, the UOS just verifies the Data Steward role exists
     before it contacts the CRS.
3. The user must create a Work Package via the Data Portal to obtain a Work Package
Access Token (WPAT). Only one WPAT is needed for the entire series of files under normal
circumstances.

#### For File Upload
The user then supplies the WPAT to the `ghga-connector` to upload files.
The `ghga-connector` obtains Work Order Tokens (WOTs) automatically by providing the
WPAT to the WPS, and then makes at least three calls for each file:
1. A POST request to the UCS to create the `FileUpload`.
   - This requires a `CreateFileWorkOrder` token from the WPS.
   - The user supplies the file `alias` in this request. 
   - The user receives the `file_id` of the newly created `FileUpload`
2. A GET request to the UCS to obtain a file part upload URL. This call is repeated for
each file part.
   - This requires an `UploadFileWorkOrder` token of type "upload" from the WPS.
   - The user supplies the `file_id` to get the above token.
   - The token is only valid for a file with the matching `file_id`.
3. A final PATCH request to the UCS to complete the file upload.
   - This uses a `UploadFileWorkOrder` token of type "close".
4. *Optional*: The user desires to delete a file:
   - The user obtains a `UploadFileWorkOrder` token of type "delete" from the WPS
   - The user performs a DELETE request via the `ghga-connector`.


#### For Altering an Existing `DataUploadCase`
- Modifying the details of an existing `DataUploadCase`, such as the title or description
  requires the user to have the Data Steward role.
- Changing the state of a `DataUploadCase` from `OPEN` to `LOCKED` requires the user to
  have either a valid claim for the `DataUploadCase` or have the Data Steward role.
- Changing the state of a `DataUploadCase` otherwise requires the Data Steward role.
- When a `DataUploadCase` is set to `LOCKED` or `CLOSED`, the UOS signs a
  `ChangeUploadContextWorkOrder` and tells the UCS to make the associated `UploadContext`
  immutable.
- Claims and work packages for closed `DataUploadCase`s remain valid in the CRS 
  - The UCS and UOS are responsible for screening requests based on the state of a given
    `DataUploadCase`, `UploadContext`, or `FileUpload`, as applicable.

#### For Viewing/Accessing `DataUploadCase`s
Data Stewards can see all `DataUploadCase`s, while other users can only see what belongs
to them. The UOS checks the auth context for the Data Steward role to distinguish between
the two categories of users. In the case of a regular user, the UOS additionally
consults the CRS to obtain a list of `DataUploadCase` IDs that the user may access.

#### In summary
- UOS:
  - Inspects auth context details to discern between Data Stewards and regular users
  - Communicates with the CRS to create or consult claims
  - Sends `DataUploadCase`-related requests to the UCS using self-signed WOTs
  - The Data Steward role is required to create `UploadContext`s and grant upload claims
  - Data Steward role is required for all `DataUploadCase` changes except `OPEN` -> `LOCKED`
  - Viewing `DataUploadCase`s does not involve any WOTs
- UCS:
  - Knows nothing about users, claims, studies, `DataUploadCase`s, etc.
  - Endpoints require protected by WOTs that come from either the WPS or UOS

For more information on the HTTP API, see the endpoint definitions below.

## User Journeys

### `DataUploadCase` Creation
A Data Steward uses the Data Portal to create a new `DataUploadCase`. The Data Portal
makes a `POST` call to the UOS, which verifies the Data Steward's access and subsequently
makes a call to the UCS to create a new `UploadContext`. The UCS returns the `UploadContext`
ID to the UOS, and the UOS stores this ID and the title/description supplied by the
Data Steward as a `DataUploadCase`, with its state set to `OPEN`. Both the UCS and the
UOS issue outbox events for their `UploadContext` and `DataUploadCase` insertions,
respectively.

The UOS returns the `DataUploadCase` ID to the Data Steward. The Data Steward then uses
the Data Portal to create a claim for the user. The Data Portal sends a `POST` request
to the UOS, which carries the user ID, IVA ID, and the `DataUploadCase` ID. The UOS
verifies the Data Steward's access, then makes a request to the CRS. The CRS creates the
claim for the user for the given `DataUploadCase` and IVA. The IVA is important because
if the user has not completed the IVA verification process, they cannot receive upload
access. Given that the IVA is valid, the CRS and UOS return a successful response. The
Data Steward can then inform the user that they make proceed with work package creation
and file upload.

### `DataUploadCase` Retrieval
In the case of a Data Steward:  
1. A Data Steward uses the Data Portal to make a `GET` request to the UOS.
2. The UOS sees that the request comes from a user with the Data Steward role and
   returns any/all `DataUploadCase`s, according to any filtering and pagination applicable.

In the case of a non-Data Steward:  
1. The user uses the Data Portal to view a list of `DataUploadCase`s available to them.
2. The Data Portal makes a `GET` request to the UOS.
3. The UOS sees that the request from a user that is NOT a Data Steward.
4. The UOS sends a request to the CRS to see which `DataUploadCase`s the user may access.
5. The CRS returns a list of IDs to the UOS.
6. The UOS returns any/all `DataUploadCase`s, according to any filtering and pagination applicable.

### `DataUploadCase` Info Update
> Requires that the user journey "`DataUploadCase` Creation" has been completed.

The Data Steward calls the `PATCH /contexts/{context_id}` endpoint on the UOS API
from a page on the Data Portal. The request body should contain the updated 
`DataUploadCase` or, alternatively, indicate which fields to change and how. This could
be a change to the title or description, for example.
The UOS verifies their Data Steward role is in the auth context and validates the
request parameters. If there are no errors, the UOS updates its copy of the
`DataUploadCase` and emits an outbox event and returns a successful response.

### Work Package Creation
> The user journey "`DataUploadCase` Creation` must have been completed.

The user uses the Data Portal to see which `DataUploadCase`s they have access to, then
creates a work package for the new `DataUploadCase`. The WPS validates the request,
creates the work package, and returns the Work Package Access Token (WPAT) to the user.
The user can then use this WPAT with the `ghga-connector` to upload files.

### File Upload Init
> Requires that the user journey "Work Package Creation" has been completed.

The following is accomplished using the `ghga-connector` and the WPAT:

1. The user initiates the upload process for a given single file (can be looped or batched):
   - The Connector contacts the WPS and exchanges the WPAT for a `CreateFileWorkOrder` token.
   - The Connector calls the UCS's `POST /contexts/{context_id}/uploads/` endpoint.
The request body includes the unencrypted checksum, the file alias, and possibly further
information. The WOT carries the context ID and file alias.
   - The UCS ensures the `UploadContext` is currently open and doesn't already have a
     completed `FileUpload` for the same file alias, then adds the `FileUpload` to the
     associated `UploadContext`.
   - The UCS initiates a multipart upload for the file.
   - The UCS publishes upsertion events for both the `FileUpload` and `UploadContext`
     objects, and returns an HTTP response to the Connector indicating that the file
     upload was successfully initiated. 
     - The response contains the UCS-generated file id (UUID4) of the new file upload.

### File Upload
> Requires that the user journey "File Upload Init" has been completed.

The following is accomplished using the `ghga-connector` for each file:

1. The Connector uploads the file in chunks:
   - The Connector sends the WPAT and file ID to the WPS in order to obtain a `UploadFileWorkOrder`.
   - The Connector uses the `UploadFileWorkOrder` token to call the UCS's
     `GET /contexts/{context_id}/uploads/{file_id}/parts/{part_no}` endpoint.
   - The WOT carries the file ID and context ID, which are cross-checked with the URL.
   - Assuming the WOT is valid, the UCS returns a presigned, short-lived upload URL.
   - The Connector uploads the file part by using the upload URL.
   - The Connector repeats this process for every file part until all parts are uploaded.
2. The Connector makes a call to `PATCH /contexts/{context_id}/uploads/{file_id}` to
   inform the UCS that the file upload is complete. This request requires a
   `UploadFileWorkOrder` token. The UCS sets the `FileUpload` with the matching ID to
   `COMPLETE` and instructs S3 to complete the multipart upload. The Connector displays
   a message to the user indicating that the file upload was successful.

### `FileUpload` Deletion
The user, via the `ghga-connector` with a valid WPAT, makes a request to the UCS's
`DELETE /contexts/{context_id}/uploads/{file_id}` endpoint, indicating they wish to
delete a file from the associated `UploadContext`. If a valid `UploadFileWorkOrder` token
of type "delete" is supplied with the request, the UCS cancels the ongoing upload if it
exists and deletes the `FileUpload` object from the database. It modifies the linked
`UploadContext` so that the total file size field is reduced by the size of the deleted
file, and decrements the file count. The UCS then emits an outbox event for the
`FileUpload` deletion and another outbox event for the change to the `UploadContext`.
Finally, the UCS returns an HTTP response to the user indicating the deletion was successful.

### `DataUploadCase` State Change
> Requires that the user journey "File Upload" has been completed.

The user uses the Data Portal to *LOCK* the `DataUploadCase`. The Data Portal sends this
request to the UOS, which checks with the CRS that the user may access this `DataUploadCase`.
As long as the user is authorized and the `DataUploadCase`'s current state is `OPEN`,
the UOS signs a `ChangeUploadContextWorkOrder` token and calls the UCS's
`PATCH /contexts/{context_id}` endpoint. The UCS checks that there are no `FileUpload`s
for the `UploadContext` still in the `INIT` state. If all files are `COMPLETE`, the UCS
changes the `UploadContext`'s mutability to `False`, emits a new outbox event for the
upsertion, and sends a successful response to the UOS. The UOS updates the
`DataUploadCase` state to `LOCKED`.

Users are only allowed to make the initial change from `OPEN` to `LOCKED`. Only Data
Stewards may move an `DataUploadCase` from `LOCKED` to `CLOSED`, `LOCKED` to `OPEN`, or
from `CLOSED` to `OPEN`. These other state changes follow a similar path to the one
described above. In the case of a Data Steward, the UOS does not make the CRS call.

## API Definitions:

### RESTful/Synchronous:

#### Upload Controller Service:
- `POST /contexts`: Create a new `UploadContext`
  - Requires `CreateUploadWorkOrder` token and only allowed for Data Stewards via the UOS.
  - Returns the `context_id` of the newly created `UploadContext`
- `PATCH /contexts/{context_id}`: Update an `UploadContext` (to make [im]mutable)
  - Requires `ChangeUploadContextWorkOrder` token issued by UOS
  - Currently only available to Data Stewards
  - Path arg and token must agree on context ID
  - Request body must contain the new state of the context
- `GET /contexts/{context_id}/uploads`: Retrieve list of file IDs for context
  - Requires a `ListFilesWorkOrder` token signed by the UOS
  - Path arg and token must agree on context ID
- `POST /contexts/{context_id}/uploads`: Add a new `FileUpload` to an existing `UploadContext`
  - Requires `CreateFileWorkOrder` token from WPS
  - Request body must contain the required file upload details
  - Path arg and token must agree on context ID, and alias must match between body and token
- `GET /contexts/{context_id}/uploads/{file_id}/parts/{part_no}`: Get pre-signed S3 upload URL for file part
  - Requires `UploadFileWorkOrder` token of type "upload" from WPS
  - Path args and token must agree on context ID and file ID
- `PATCH /contexts/{context_id}/uploads/{file_id}`: Conclude file upload in UCS
  - Requires `UploadFileWorkOrder` token of type "close" from WPS
  - Sets the `FileUpload` status to `COMPLETE` and tells S3 to close the multipart upload.
  - Path args and token must agree on context ID and file ID
- `DELETE /contexts/{context_id}/uploads/{file_id}`: Remove a `FileUpload` from the `UploadContext`
  - Requires `UploadFileWorkOrder` token of type "delete" from WPS
  - Deletes the `FileUpload` and tells S3 to cancel the multipart upload if applicable.
  - Path args and token must agree on context ID and file ID

#### Upload Orchestration Service:
- `GET /cases/{case_id}`: Retrieve a `DataUploadCase` by ID
- `POST /cases`: Create a new `DataUploadCase`
  - Requires Data Steward Role
  - Signs a `CreateUploadContextWorkOrder` token and sends request to the UCS
  - Returns the ID of the newly created `DataUploadCase`
- `PATCH /cases/{case_id}`: Update the state of a `DataUploadCase`
  - Requires Data Steward Role *or* valid claim to the `DataUploadCase`
    - Only Data Stewards can do `LOCKED` -> `CLOSED` or `LOCKED` -> `OPEN`
    - Users are allowed to do `OPEN` -> `LOCKED`
  - Request body must include the properties to update. Empty body has no effect.
  - Signs an `ChangeUploadContextWorkOrder` token and calls the matching UCS endpoint
    if the request involves toggling the `UploadContext`'s mutability
- `POST /access`: Grant user access to an `UploadContext`
  - Requires Data Steward Role
  - Instructs the CRS to create a new claim for the specified user
  - Request body must contain:
    - `UploadContext` ID
    - User ID
    - IVA ID
    - Any other pertinent information, such as access expiration date.
  - Browsing for and revoking claims can be done through the upcoming Claims Browser
- `GET /cases/{case_id}/uploads`: Retrieve list of file IDs for `DataUploadCase`
  - Signs a `ListFilesWorkOrder` token and calls matching UCS endpoint

#### Work Package Service:
- `GET /users/{user_id}/contexts`: List all `UploadContext` IDs available to the user
- `POST /work-packages/{work_package_id}/contexts/{context_id}/work-order-tokens`: Create a WOT for uploading files
  - Requires a Work Package Access Token, so the user must have already created a Work Package
  - The request body must contain the work type and file alias or ID, depending on the
    work type

#### Claims Repository Service:
- CRS Authentication for upload endpoints should match existing download counterparts
- `GET /upload-access/users/{user_id}/contexts`: lists which `UploadContext`s a user can access
- `GET /upload-access/users/{user_id}/contexts/{context_id}`: check if a user has access to a certain upload context
- `POST /upload-access/users/{user_id}/ivas/{iva_id}/contexts/{context_id}`: grant upload access
  - This is called by the UOS when the Data Steward grants a user upload access
- `DELETE /upload-access/users/{user_id}/contexts/{context_id}`: revoke upload access

### Payload Schemas for Events:

```python
class UploadContext(BaseModel):
  """A class representing a context that links files belonging to a single dataset"""

  context_id: UUID4  # unique identifier for the instance
  mutable: bool  # Whether or not changes to the files in the context are allowed
  file_count: int  # The number of files in the context
  size: int  # The total size of all files in the context

class DataUploadCaseState(StrEnum):
    """The allowed states for an DataUploadCase instance"""

    OPEN = "open"
    LOCKED = "locked"
    CLOSED = "closed"

class DataUploadCase(BaseModel):
    """A class representing a DataUploadCase"""

    case_id: UUID4 # unique identifier for the instance
    state: DataUploadCaseState  # one of OPEN, LOCKED, CLOSED
    title: str  # short meaningful name for the case
    description: str  # describes the upload case in more detail
    upload_context: UploadContext

class FileUploadState(StrEnum):
    """The allowed states for a FileUpload instance"""

    INIT = "init"
    COMPLETE = "complete"

class FileUpload(BaseModel):
    """A File Upload"""

    upload_id: UUID4
    state: FileUploadState  # one of INIT, COMPLETE
    alias: str  # the submitted alias from the metadata
    checksum: str
    size: int
```


## Additional Implementation Details:

### WOT Modifications in WPS
The WPS needs to be able to authorize work in a compartmentalized fashion.
Instead of adapting the existing WOT schema to work for multiple services,
there should be specific schemas dedicated to a given action (work order).
The existing WorkOrderToken model must be modified & renamed.
The following new WOT schemas should be created in the WPS:

```python
DownloadFileWorkOrder:  # replaces current `WorkOrderToken` model
   type: "download"
   file_id: str  # note: user information is no longer included

CreateUploadContextWorkOrder:
  type: "create"
  name: str

ChangeUploadContextWorkOrder:
   type: "lock" | "close" | "open"
   context_id: str

ListFilesWorkOrder:
   type: "view"
   context_id: str

CreateFileWorkOrder:
  type: "create"
  alias: str
  context_id: str

UploadFileWorkOrder:
   type: "upload" | "close" | "delete"
   file_id: str
   context_id: str
```


### Download Controller WOT Dependency
The Download Controller Service (DCS) uses WOTs to authenticate download URL requests.
It maintains a model in a core module that defines the WOT structure. Depending on the
exact changes to the WOT model in the WPS, the DCS may or may not have to be updated too.

### Testing
Tests need to cover at least the following items (not exhaustive):
- Standard endpoint authentication battery
- Happy path for each endpoint
- Core error translation for HTTP API for each endpoint
- Disallow adding/uploading/deleting files in immutable `UploadContext`s
- Make sure only Data Stewards can create, close, or reopen a `DataUploadCase`
- Users can only see `DataUploadCase`s that they have a valid claim for
- Data Stewards can see all `DataUploadCase`s
- UCS rejects http requests for immutable `UploadContext`s even with a valid WOT
  - Exception being to re-open the `UploadContext`
- UOS rejects requests for locked `DataUploadCase`s, except to move state to `OPEN` or `CLOSED`


## Human Resource/Time Estimation:

Number of sprints required: 3

Number of developers required: 2
