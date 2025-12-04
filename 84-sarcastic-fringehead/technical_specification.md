# File Upload Path Pt. 2 (Sarcastic Fringehead)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://ghga.pages.hzdr.de/internal.ghga.de/main/sops/development/epic_planning/).

## Scope
### Outline:
This epic includes all work required to bring the remaining file services into line with the new file upload concept. The first portion of work for the file services was executed under [Lynx Boreal](../76-lynx-boreal/technical_specification.md), and there was also a subsequent portion of work for the GHGA Connector which was carried out according to [Hedgehog Seahorse](../80-hedgehog-seahorse/technical_specification.md). When this epic is finished, all *backend* modifications required for the new upload concept to be realized will be complete. Frontend changes are *not* included in this epic, however, so more work will be required to bring the Data Portal up to speed.
As for the work to be completed within this epic, the services affected include the File Ingest Service (FIS), Internal File Registry Service (IFRS), the Well-Known Value Service (WKVS), the Upload Controller Service (UCS), the ghga-event-schemas library, and a new service called the Data Hub File Service (DHFS). Additionally, if it is discovered during implementation that further changes need to be made to other services *beyond what is described in this epic*, then tickets will be added ad-hoc and associated with this epic.

In Lynx Boreal, the UCS was rewritten, the Upload Orchestration Service (UOS) was implemented for the first time, the Claims Repository Service (CRS) was updated to manage permissions for Research Data Upload Boxes, and the Work Package Service (WPS) was updated to manage upload-type work packages. Taken together, these changes create the operational framework for remote file upload, but only to the point of initial ingest. In order to fully realize our file upload concept, we still need to decrypt the uploaded file, verify the integrity via checksum comparison, re-encrypt the file with a new file secret (securely stored in the Encryption Key Store Service, or EKSS), and move the file to a permanent storage bucket registered with the IFRS in what we call "archival".


### Included/Required:
All work described in the Additional Implementation Details section below is required.

### Optional:
Add email notifications for important events related to archival. This could include, for example, a notification conveying that all files in a Research Data Upload Box have been successfully archived, or that there was a problem with file XYZ during interrogation. To prevent scope creep, this should *probably* be done in another epic, but we should keep that potential requirement in mind during development.

Add audit events -- can also be performed in a separate epic.

### Not included:
Data Portal updates or any upcoming metadata-related services. This is purely for file upload.

## User Journeys (optional)

All user journeys are already detailed in Lynx Boreal. The operations added in this epic will occur automatically without further action required on the part of either the user or GHGA personnel. 

## API Definitions:

### RESTful/Synchronous:

[UCS HTTP API](#ucs-http-api)  
[UOS HTTP API](#uos-http-api)  
[WKVS HTTP API](#wkvs-http-api)  
[FIS HTTP API](#fis-http-api)  

### Payload Schemas for Events:

#### ResearchDataUploadBoxState
```python
OPEN = "open"
LOCKED = "locked"
ARCHIVED = "archived"  # This used to be `CLOSED = "closed"`
```

#### ResearchDataUploadBox
```python
id: UUID4
state: ResearchDataUploadBoxState
title: str
description: str
last_changed: UTCDatetime
changed_by: UUID4
file_upload_box_id: UUID4
locked: bool
file_count: int
size: int
storage_alias: str
```

#### FileUploadBox
```python
id: UUID4
locked: bool = False  # if archived is True, then this field must also be True
archived: bool = False  # New field. If True, this box is permanently locked
file_count: int
size: int
storage_alias: str
```

#### FileUploadState
```python
INIT = "init"  # unchanged, means the file is being uploaded to the inbox
INBOX = "inbox"  # unchanged, means the file is in the inbox awaiting interrogation
FAILED = "failed"  # new state, means problem with interrogation, upload, etc.
INTERROGATED = "interrogated"  # new state, means file interrogation was valid
ARCHIVED = "archived"  # now means the file is officially in permanent storage
```

#### FileUpload
> Outbox event owned by UCS
```python
id: UUID4       # Unique identifier for the file upload
box_id: UUID4   # The ID of the FileUploadBox this FileUpload belongs to
alias: str      # The submitted alias from the metadata (unique within the box)
state: FileUploadState = "init"  # The state of the FileUpload
state_updated: UTCDatetime  # Timestamp of when state was updated
storage_alias: str  # The storage alias for the inbox bucket
# Additionally, the following fields exist but are unset until later in the process
secret_id: str | None
decrypted_sha256: str | None  # SHA-256 checksum of the entire unencrypted file content
decrypted_size: int     # The size of the unencrypted file
part_size: int  # The number of bytes in each file part (last part is likely smaller)
encrypted_parts_md5: list[str] | None     # Is None until DHFS finishes with file
encrypted_parts_sha256: list[str] | None  # Is None until DHFS finishes with file
accession: str | None   # The accession number for the file
```

#### InboxedFileUpload
> This schema represents what FIS checks for validation when consuming a `FileUpload` event with the `inbox` state
```python
id: UUID4       # Unique identifier for the file upload
state: FileUploadState = "init"  # The state of the FileUpload
state_updated: UTCDatetime  # Timestamp of when state was updated
storage_alias: str  # The storage alias for the inbox bucket
decrypted_sha256: str   # SHA-256 checksum of the entire unencrypted file content
decrypted_size: int     # The size of the unencrypted file
part_size: int  # The number of bytes in each file part (last part is likely smaller)
can_remove: bool = False  # Indicates whether file can be deleted from `interrogation` bucket
```

#### InterrogationReport
> Outbox event owned by FIS on behalf of DHFS
```python
file_id: UUID4
secret_id: str | None
storage_alias: str | None  # the interrogation bucket storage alias
interrogated_at: UTCDatetime
interrogation_result: "passed" | "failed" | "cancelled" | None
encrypted_parts_md5: list[str] | None
encrypted_parts_sha256: list[str] | None
```

#### ArchivableFileUpload
> This schema represents what IFRS checks for validation when consuming a `FileUpload` event with the `archived` state
```python
id: UUID4
box_id: UUID4  # Shouldn't actually be needed, but keeping in case it turns out useful
alias: str
storage_alias: str
secret_id: str
decrypted_sha256: str
decrypted_size: int
part_size: int
encrypted_parts_md5: list[str]
encrypted_parts_sha256: list[str]
accession: str
```

#### FileInternallyRegistered
> Persistent event owned by the IFRS
```python
# content_offset is removed because objects are stored without an envelope
# bucket_id is removed because storage_alias should already point to specific bucket
file_id: str  # for now will continue to contain the GHGA accession - will migrate & replace with object IDs (FileUpload.id) once official accession provider is implemented
object_id: UUID4  # unchanged -- see note above
upload_date: UTCDatetime  # unchanged
storage_alias: str  # renamed from s3_endpoint_alias
secret_id: str      # renamed from decryption_secret_id
decrypted_size: int     # unchanged
encrypted_size: int     # new field
decrypted_sha256: str   # unchanged
part_size: int  # renamed from encrypted_part_size
encrypted_parts_md5: list[str]    # unchanged
encrypted_parts_sha256: list[str] # unchanged
```

## Additional Implementation Details:

> For a comprehensive overview, please see the [Service Diagrams](#service-diagrams) section below.

### GHGA-Event-Schemas:
- Update existing schemas to match what is shown above.  
- Add a new ***stateful*** config class named `InterrogationReportEventsConfig`.
- Remove the `FileUploadReport` schema
- Remove the `FileUploadReportEventsConfig` ***stateful*** config class

### UCS:
The UCS takes on an expanded role from what was defined in Lynx Boreal. Previously, the UCS was only concerned with getting files into the `inbox` bucket, and after that it didn't care what happened. However, further consideration has resulted in the viewpoint that the UCS is actually the source of truth for files all the way up until they are copied into permanent storage. Intermediate steps that occur in other services provide subsequent information to the UCS regarding the `FileUpload`, but those services do not assume ownership of the essential file information. Not only that, but the relationship between `FileUpload` IDs and accession numbers should and will be managed by the UCS during the interim phase while official accession management is still under development. The UCS operates two instances - an HTTP API and an event consumer.

#### UCS Event Consumer
The UCS event consumer instance subscribes to `InterrogationReport` *outbox events* published by the FIS. The schema is [detailed above](#interrogationreport). When a new `InterrogationReport` arrives, UCS:
- Finds the matching `FileUpload` in its database and raises an error if it can't.
- Checks that the `FileUpload` has a state of `inbox`.
  - If it doesn't (e.g. the state is already `interrogated`, `failed`, `cancelled`, or `archived`), UCS compares `FileUpload.state_updated` and `InterrogationReport.interrogated_at`.
    - If the `InterrogationReport` timestamp is newer, UCS raises an error and the event is sent to the DLQ.
    - If the UCS timestamp is newer, the event is ignored and no further action is taken
- UCS decides what to do based on the value of `interrogation_result`:
  - If the value is `None`, no action is taken.
  - If the value is `"passed"`, UCS:
    - Sets `secret_id`, `state_updated`, `storage_alias`, `interrogation_result`, `encrypted_parts_md5`, `encrypted_parts_sha265` based on the information in the `InterrogationReport`
    - Sets `FileUpload.state` to `interrogated`
    - Deletes the file from the `inbox` bucket
  - If the value is `"failed"`, UCS:
    - Sets `FileUpload.state` to `failed` and `FileUpload.state_updated` to `InterrogationReport.interrogated_at`
    - Deletes the file from the `inbox` bucket
  - If the value is `"cancelled"`, UCS:
    - Checks that `FileUpload.state` is already set to `cancelled` and raises an error if it isn't (this indicates some kind of state inconsistency that shouldn't happen based on the current concept).
    - Tries to delete the file from the `inbox` bucket, just to be sure.
At this point, the event consumer instance is finished processing the event and waits for the next event. The updates to the `FileUpload` will be published as an outbox event.

As a final note on the UCS, the UCS is the place where `box_id` is populated for `FileUpload` objects. You can read about that process in Lynx Boreal, but the long-short is that a Data Steward manually creates a `ResearchDataUploadBox` in the UOS, which has a separate ID, and that automatically triggers the creation of a subordinate `FileUploadBox` in the UCS, which has an independent ID. Whenever a new file is added for that box, the `FileUpload` gets the `box_id` of the parent `FileUploadBox`.

#### UCS HTTP API
You can read about the existing UCS endpoints in the Lynx Boreal epic. I will only detail the updates here.

The UCS operates the following new endpoints:
`PATCH /boxes/{box_id}/accessions`: Accepts a payload which maps `FileUpload` IDs to accession numbers for a given `FileUploadBox`
- Authorized by a token signed by the UOS with the work type `"map"` and including the `box_id`
- Returns `204 NO CONTENT`
- Description:
  - The UCS finds the `FileUploadBox` in its database and raises an error if it can't
  - The UCS ensures the `FileUploadBox` is locked but not archived, raising an error otherwise.
  - Before making database writes, the UCS retrieves all specified `FileUpload` objects and:
    - Ensures the state is not `archived`.
    - Ensures the accession field is not assigned, or if it is assigned, that it matches the accession provided in the request. Otherwise, UCS raises an error and returns a `409 CONFLICT` response.
  - The updated `FileUpload` objects are published as outbox events.

`PATCH /boxes/{box_id}`: This endpoint already exists, but is augmented here to allow `FileUploadBox` *archival*.
- Authorized by a token signed by the UOS with the work type `"archive"` and including the `box_id`
- Returns `204 NO CONTENT`
- Description: 
  - The UCS finds the `FileUploadBox` in its database and raises an error if it can't.
  - The UCS verifies that the `FileUploadBox` is locked but not archived.
    - If archived, the UCS returns early as it assumes that the work is already done
  	- If not locked, the UCS raises an error
  - The UCS looks up every `FileUpload` associated with the box and verifies that each one has an assigned accession. If any are not set, the UCS raises an error and rejects the box archival.
    - This is performed by querying with the mapping `{"box_id": box_id, "accession": None}` instead of looping through all associated file IDs.
    - UCS sets each `FileUpload.state` to `archived` and likewise updates the `FileUpload.state_updated` timestamp.
    - If every `FileUpload` already has a set accession and a state of `archived`, the UCS skips to updating the `FileUploadBox`
  - The UCS publishes an outbox event for each `FileUpload`.
  - The UCS sets `FileUploadBox.archived` to True and publishes the updated object as an outbox event.

The work to provide a deletion endpoint accessible by GHGA Connector is *not* meant to be part of this epic. For now, assume all deletions/cancellations will be triggered from the Data Portal -> UOS -> UCS rather than from the GHGA Connector. Additionally, in case it wasn't clear, file deletions (or cancellations, rather) do not result in a `dao.delete()` call. The full document data remains, but the state is set to `cancelled`. When the GHGA Connector is enabled to perform deletions, the state might potentially be allowed to be set to `failed` in addition to `cancelled`. More thought is required here on the requirements for work order tokens, use cases, and alias vs file ID specifiers.

#### UCS Config
The UCS needs the following config changes:
- Add OutboxSubConfig for `InterrogationReport` events
  - ghga-event-schemas -> `InterrogationReportEventsConfig`
- Remove OutboxSubConfig for `FileUploadReport` events

#### Work to be performed for the UCS
- [ ] Get schema updates from ghga-event-schemas
- [ ] Implement new endpoints
- [ ] Remove existing subscription to `FileUploadReport` events
- [ ] Add outbox subscriber for `InterrogationReport` events
- [ ] Change existing deletion behavior to update `FileUpload.state` instead of actually deleting content.

### UOS:
The UOS remains mostly unchanged from its initial implementation in Lynx Boreal, except for gaining a new, temporary responsibility to relay **accession maps** to the UCS. Through a new HTTP API endpoint, the UOS will take in objects that map file IDs from `FileUpload` objects to an accession number. This is temporary because it fills in a functional gap in the overall system that still has to be planned out. In the future, this endpoint will be removed (or at least no longer used). The UOS operates both an HTTP API instance and an event consumer instance.

#### UOS HTTP API
> [Return to API list](#restfulsynchronous)

The UOS would get the following new endpoint:  
`PATCH /boxes/{box_id}/accessions`: Accept a file-to-accession mapping in order to relay it to the UCS
- Authorization uses auth context protocol and requires a Data Steward role
- Request body must contain a payload conforming to the `FileAccessionMap` schema (see below)
- Returns `204 NO CONTENT`
- Description:
  - UOS looks for the `ResearchDataUploadBox` in its database with an ID that matches the value in the path parameter, and returns a `404 NOT FOUND` if it doesn't find it.
  - UOS ensures the `ResearchDataUploadBox` is not already CLOSED, and raises an error if it is. This might return a `409 CONFLICT` status code.
  - UOS self-signs a `ChangeFileBoxWorkOrder` token and makes a PATCH request to the UCS's `/boxes/{box_id}/accessions` endpoint.
    - The UOS uses `ResearchDataUploadBox.file_upload_box_id` for the `box_id` parameter in both the token and UCS call.
    - The request body contains the accession map that was supplied in the original request to the UOS.

```python
class FileAccessionMap(BaseModel):
  """Model that maps file IDs to GHGA accession numbers"""
  mapping: dict[UUID4, str]  # this could instead be a list of tuples or similar object
```

#### UOS Config
The UOS shouldn't need any config changes.

#### Work to be performed for the UOS
- [ ] Get schema updates from ghga-event-schemas
- [ ] Add the new API endpoint described above
- [ ] Add the FileAccessionMap definition somewhere in the UOS (not in a library)
- [ ] Rename final box state to `archived` instead of `closed`
- [ ] Add two new options to the `ChangeFileBoxWorkOrder` for the work type literal"
      - `"map"`: represents the task of submitting the accession map
      - `"archive"`: represents final, permanent sealing of `FileUploadBox`
- [ ] Add the outbound UCS call to the UOS's `FileBoxClient` that uses the `"map"` type
- [ ] Add an outbound UCS call in the `FileBoxClient` with the same structure as `lock_file_upload_box()` and `unlock_file_upload_box()`, called `archive_file_upload_box()`
- [ ] Call `FileBoxClient.archive_file_upload_box()` from the UOS core when moving a `ResearchDataUploadBox` to `archived` state (formerly labeled `closed`).

### WKVS:
- Provides the Data Hub Crypt4GH public keys via public HTTP API

#### WKVS HTTP API
> [Return to API list](#restfulsynchronous)

The WKVS would get the following new endpoint:  
`GET /values/data_hub_public_keys`:
- No authentication required
- Returns `200 OK` and a mapping of storage alias to public key

#### Work to be performed for the WKVS
- [ ] Provide a way to retrieve Crypt4GH public keys for Data Hubs. This can be a dictionary where the keys are storage aliases and the values are the public keys.

### GHGA Connector:
The Connector performs initial file encryption and upload from the user's machine. In order to properly encrypt the file for a specific Data Hub, the Connector needs to contact the WKVS to obtain the appropriate Crypt4GH public key based on the storage alias assigned to the `ResearchDataUploadBox`/`FileUploadBox` created by the Data Steward.

#### Work to be performed for the GHGA Connector
- [ ] Fetch and use Data Hub public key for file encryption
- [ ] Keep list of SHA-256 checksums for each unencrypted file part and submit with file part size

### FIS:
The FIS straddles the border between the file services group and everything else, similar to the role played by the UOS. In the past, the FIS acted as a way to ingest file upload metadata and tell other services when a manually validated ("interrogated") file was ready for permanent storage. This had to be done as a temporary solution until the remote file upload and automatic file interrogation was implemented, which is the work proposed in this epic.

The new role of the FIS is to inform the DHFS when new files arrive in the DHFS's `inbox` bucket, as well as to inform other services of the results of DHFS's interrogation procedure by publishing `InterrogationReport` outbox events (defined [here](#interrogationreport)). To do this, the FIS operates as an event consumer in one instance, and runs an HTTP API in another instance. Both instances are described below.

#### FIS Event Consumer
The FIS subscribes to `FileUpload` *outbox events* from the UCS.

When a new `FileUpload` event arrives with the state `inbox` FIS first checks its database to see if it has a copy already stored in its database. If it does, FIS either ignores the event or raises an error depending on specific criteria (implementation detail). If the event is new, FIS stores the event using the [InboxedFileUpload schema](#inboxedfileupload) so that it can be later relayed to the DHFS. Finally, FIS creates a preliminary `InterrogationReport` with only the file ID field set, which gets published as an outbox event.

When a `FileUpload` event arrives with a state other than `init` or `inbox`, e.g. `failed`, `cancelled`, `interrogated`, or `archived`, FIS will merely update its local `InboxedFileUpload` copy (or create it if it doesn't exist) and set the `can_remove` field to True. FIS should never store any information for a `FileUpload` with the state `init`.

The FIS *also* subscribes to `FileInternallyRegistered` events from the IFRS. FIS is interested in these events because they communicate that a given file has been successfully copied from the `interrogation` bucket to the the `permanent` bucket. FIS locates the corresponding `InboxedFileUpload` and raises an error if it can't. It then sets the `can_remove` field to True.

#### FIS HTTP API
> [Return to API list](#restfulsynchronous)

The FIS operates an HTTP API with these endpoints:
1. `POST /secrets`: Accept a new file secret for deposition in the EKSS
   - Authorization requires a token signed with Data Hub-specific private key
   - Request body must contain an encrypted file secret and associated file ID
   - Returns `201 CREATED`
   - Description:
     - FIS finds the existing `InterrogationReport` in its database, raising an error if it doesn't find it.
     - FIS forwards the file secret to EKSS (still encrypted) in exchange for a secret ID.
     - FIS updates the `InterrogationReport` with the new secret ID and publishes the change as an outbox event.
2. `GET /storages/{storage_alias}/uploads`: Serve a list of new file uploads (yet to be interrogated)
   - Authorization requires a token signed with Data Hub-specific private key
   - Returns `200 OK` and a list of `FileUploads` for files awaiting interrogation
   - Description:
     - FIS gets the `InterrogationReports` which match the requested storage alias and have `interrogation_result=None`, i.e. interrogations which haven't reached a conclusion yet.
     - FIS returns the list of `FileUploads` corresponding to the `InterrogationReports` from the previous step
3. `GET /uploads/{file_id}/can_remove`: Returns a bool indicating whether a file can be removed from the `interrogation` bucket
   - Authorization requires a token signed with Data Hub-specific private key
   - Returns `200 OK` and the value of the `can_remove` field of the requested `InboxedFileUpload`
   - Description:
     - FIS finds the existing `InboxedFileUpload` in its database, raising an error if it doesn't find it (should be translated to True as an HTTP response but logged within the service as an error).
     - Returns the value of `InboxedFileUpload.can_remove`
4. `PATCH /interrogation-reports`: Accept an interrogation report
   - HTTP method is `PATCH` because the FIS already stores the object and requests to this endpoint are submitting a partial update to that information rather than a full, new object outright.
   - Authorization requires a token signed with Data Hub-specific private key
   - Request body must contain a payload conforming to the `InterrogationReport` [schema](#interrogationreport)
   - Returns `204 NO CONTENT`
   - Description:
     - FIS finds the matching `InterrogationReport` in its database based on the `file_id` (or raises an error).
     - FIS updates the `InterrogationReport` with the received information and publishes this as an *outbox event*.


#### FIS Configuration
TODO: Add config section to UCS and UOS
The FIS needs the following configuration:
- MongoDbConfig
- KafkaConfig
- ApiConfigBase
- LoggingConfig
- OpenTelemetryConfig
- EventSubConfig:
  - ghga-event-schemas -> FileInternallyRegisteredEventsConfig
- OutboxPubConfig:
  - ghga-event-schemas -> InterrogationReportEventsConfig
- OutboxSubConfig:
  - ghga-event-schemas -> FileUploadEventsConfig
- ekss_api_url
- wkvs_api_url

Upon startup, the HTTP API instance of the FIS will retrieve the list of Data Hub public keys from the WKVS.

In addition, the existing functionality and config that directly interacts with Vault should be removed so that EKSS is the sole middleman for Vault activity.

#### FIS Migrations
The FIS has a persisted events collection, `fisPersistedEvents`, that contains previously published `FileInterrogationSuccessEvents` (`type_` is `"file_interrogation_success"`). These events are essentially the same as the events that will, going forward, be published by the UCS. Therefore, we *also* need to make sure the FIS can publish to the same outbox topic as the UCS. The required migration for the persisted events collection in FIS essentially needs to convert the `payload` of the stored persisted events into outbox events with the new `FileUpload` [schema](#fileupload). This migration should be reversible. FIS will only publish to the `FileUpload` outbox topic if we *republish* events from the FIS. In other words, this is historical data that we are preserving here in FIS for continuity. The historical `FileUpload` information and the local copies of new `FileUpload` objects must use different DAOs. The former should use an outbox DAO while the latter uses a regular DAO. 
- **One alternative** is to manually exfiltrate the data to UCS.
- **A second alternative** is to simply drop the data since the files have already been archived and all relevant information is actually stored by IFRS.

There is one **problem**: IFRS has different object IDs than FIS. If we keep the event data which FIS currently has, we will need to update the information so that the file IDs in FIS are set to the file IDs (object IDs) known to IFRS, using the accession to match.

Finally, the `ingestedFiles` collection, which contains unassociated accessions, should be dropped.

#### Work to be performed for the FIS
- [ ] Ensure DLQ is enabled
- [ ] Swap persistent publisher for outbox publisher for `FileUpload` events.
- [ ] Address existing persisted event data as described above.
- [ ] Add event subscriber for `FileInternallyRegistered` events
- [ ] Add outbox subscriber for `FileUpload` events
- [ ] Add outbox publisher for `InterrogationReport` events
- [ ] Add HTTP endpoints as outlined above
- [ ] Write tests


### DHFS:
The DHFS is a new service that is operated by the Data Hubs for the purpose of performing file validation and re-encryption, and to keep file ingest in general as a federated operation. The DHFS operates two instances: a client instance, which performs the interrogation work and is always running; and a cleanup instance, which runs at some interval and deletes files from the `interrogation` bucket once they've been copied to permanent storage. One crucial thing to note here is that the DHFS is not connected to an event stream, and so has no direct knowledge of the information conveyed by the events in GHGA Central's event stream. The DHFS primarily interacts with FIS's REST API in order to get that information, which is limited to only what the DHFS needs to operate.

#### DHFS Interrogator (primary instance)
It polls the FIS's HTTP API to get a list of `FileUploads` for files that have been recently uploaded to its `inbox` bucket. The DHFS decrypts each file and re-encrypts it using a new, individually created file secret before uploading it to the Data Hub's `interrogation` bucket. Along the way, it calculates the:
- Cumulative SHA-256 checksum of the entire unencrypted file
  - > Used to verify that the decrypted file is identical to what was uploaded by the user
- MD5 checksum of each individual, re-encrypted file part
  - > Used to verify that the `interrogation` bucket content matches what DHFS intended to upload
- SHA-256 checksum of each individual, re-encrypted file part
  - > Can be used to perform periodic integrity checks
  - > If we deviated from the GA4GH DRS Object spec for download, the GHGA Connector could verify file parts as they were downloaded, retrying parts that don't match. But that is out of scope for this epic.
When the whole file has been re-encrypted and uploaded to the Data Hub's `interrogation` bucket, the DHFS compares the unencrypted content's SHA-256 checksum against the value obtained from the corresponding `FileUpload`. It also calculates an aggregate MD5 checksum using the individually calculated parts' MD5 checksums and compares that against the MD5 ETag calculated by S3 in the `interrogation` bucket.
If a checksum discrepancy is found, the DHFS rejects the upload and posts an `InterrogationReport` to the FIS's HTTP API which indicates that the file did not pass inspection (`interrogation_result="failed"`). If checksums match and there are no other errors during upload, the DHFS accepts the upload and the `InterrogationReport` sent to the FIS reflects that the file passed inspection (`interrogation_result="passed"`).

#### Interrogation Process in List Format
- [Per File]
  - Reads the first file part to sift for the Crypt4GH envelope
  - Decrypts the envelope using the configured private key (specific to the Data Hub)
  - Obtains the original file secret for decryption
  - Generates a new file secret for re-encryption
  - Sends the new file secret to the FIS's HTTP API
    - This is done right away instead of waiting to see if re-encryption is successful
    - The secret is encrypted using the GHGA public key
  - Calculates the content starting position (offset) from the envelope length
  - Initiates a multipart upload with the Data Hub's `interrogation` bucket
  - Streams the object from the Data Hub's `inbox` bucket part-by-part using the same part size as found in `FileUpload.part_size`
  - [Per File Part]
    - Decrypts the part
    - Updates the SHA-256 checksum over the unencrypted content
    - Re-encrypts the part using the newly generated file secret
    - Calculates the MD5 and SHA-256 checksums over the encrypted file part and appends each to their respective lists
    - Uploads the re-encrypted part to the Data Hub's `interrogation` bucket
  - Compares the unencrypted file's SHA-256 checksum against the one reported by the submitter during upload (found in `FileUpload.decrypted_sha256`), and the encrypted checksum against the one calculated by S3
  - Sends an `InterrogationReport` to the FIS's HTTP API

#### DHFS Cleanup Job (secondary instance)
The secondary duty of the DHFS is to clean up files from the `interrogation` bucket. Files must be removed once they have been fully copied to the permanent bucket, as well as on occasions that files are deleted from their parent box. Neither the FIS, UCS, nor IFRS can perform this action because they don't have write access to the `interrogation` bucket. Each time this DHFS instance runs, it retrieves a list of all objects (files) currently in the `interrogation` bucket. Then for each file, the DHFS makes a separate GET request to the FIS API's `GET /uploads/{file_id}/can_remove` endpoint. The DHFS will *delete* the file from the `interrogation` bucket if FIS returns True.

#### DHFS Configuration
The DHFS needs the following configuration:
- LoggingConfig
- OpenTelemetryConfig
- S3ObjectStoragesConfig
- auth_token_signing_key
  - This is the Data Hub's private key, which it uses to sign auth tokens sent to FIS
- wkvs_api_url
  - Used to contact WKVS to get the GHGA public key

### IFRS:
The role of the IFRS is to shepherd files into permanent storage, or "archival", by copying them from a Hub's `interrogation` bucket into the `permanent` bucket located at the same Data Hub. This only occurs once the Data Hub in question has completed the interrogation process, as detailed in the [DHFS section](#dhfs) above. Unlike the FIS and DHFS, the IFRS operates only as an event consumer.

The IFRS subscribes to `FileUpload` outbox events from the UCS but only acts when it encounters an event with the state `archived`. It further validates the event using the [ArchivableFileUpload schema](#archivablefileupload). If the event represents a new, valid `FileUpload`, the IFRS copies the file from the `interrogation` bucket specified by `FileUpload.storage_alias` into the same location's `permanent` bucket. Once that is successful, the IFRS issues a `FileInternallyRegistered` event. This process is already in place within the IFRS, but some small tweaks are required. 

#### A note on file IDs and file accessions in the IFRS
Currently, the IFRS generates object IDs for new uploads, but this will be removed. The reason that happens now is that files are uploaded by a manual process and stored according to their accession number. File accessions belong to the metadata camp, and the intent is to segregate operations and data management such that metadata and file data are as independent as possible. Therefore, file accessions will be moved out of the file services' data and replaced with UUID4 file IDs. (The file accession and file ID will be linked via a process/service not defined in this epic). Finally, the UUID4 file IDs generated by the recently revamped UCS during file upload are now also used as the object IDs in S3 storage. This does not have to be the case, and we can choose to generate a separate object ID if that layer of indirection is desired. At the time of writing though, this is not planned.

#### Migrating existing IFRS data
The `file_metadata` collection needs the following changes:
- Remove `content_offset` field. The encrypted files are stored without an envelope, meaning the content offset is always 0.
- Make a *new collection* to link existing file accessions and file IDs until we develop the permanent accession-ID association solution. In this temporary collection, the `_id` field should contain `file_metadata.object_id` and an `accession` field should contain `file_metadata._id`. This *must* be done before the next step.
- Replace `_id` content with `object_id`. 
- Delete `object_id`.
- Rename `object_size` to `encrypted_size`.
- Rename `decryption_secret_id` to merely `secret_id`.

The list `encrypted_parts_sha256` is not currently used, but we are going to keep it for now. Originally the idea was for it to serve as another integrity check, but currently we only use the decrypted content's SHA-256 and the encrypted content's MD5 checksums for verification. In the spirit of "better to have it and not need it", we will keep this data (and continue producing it during re-encryption) for the time being.

The `ifrsPersistedEvents` collection needs similar changes to the `payload` field:
- Replace the value in `file_id` with the value from `object_id`.
- Delete `object_id`.
- Delete `bucket_id`. This field is not used by anything.
- Rename `decryption_secret_id` to `secret_id`.
- Rename `s3_endpoint_alias` to `storage_alias`.
- Delete `content_offset`.

IFRS data migration should be moved to the init container style. Instead of executing `run_db_migrations()` as part of every entrypoint, the migrations should be run as their own command. 

#### Work to be performed for the IFRS
- [ ] Add event subscriber for `InterrogationReport` events
- [ ] Add outbox subscriber for `ResearchDataUploadBox` events
- [ ] Upon encountering a closed `ResearchDataUploadBox`, retrieve the list of relevant files from the local store of `InterrogationReport` events
- [ ] Copy each file from the `interrogation` bucket to the IFRS's permanent bucket
- [ ] Get the updated `ghga-event-schemas` version and adapt IFRS for changes to `FileInternallyRegistered`
- [ ] Migrate existing data in the `file_metadata` collection
- [ ] Create new collection to preserve existing file accession-to-file ID associations
- [ ] Migrate existing data in the `ifrsPersistedEvents` collection
- [ ] Move migrations to own CLI command so they can be run as an init container
  - [ ] Work with DevOps to get this configured in k8s


#### Work to be performed for the DINS
- [ ] Get the updated `ghga-event-schemas` version and adapt DINS for changes to `FileInternallyRegistered`. This should not require a DB migration because the fields in the DB are already named differently from their counterparts in the schema itself.

### DCS:
The DCS subscribes to `FileInternallyRegistered` events from the IFRS to learn about which files are available for download from the Archive. The changes in that event schema, which are described above in the [IFRS section](#ifrs), necessitate database migrations and code updates in the DCS.

#### Migrating existing DCS data

Similar to the IFRS, the DCS should get a new, temporary database collection to preserve the connection between file accession and file ID. This must be done before any of the following changes. Please note that "temporary" here does not mean the table should be dropped at the end of the migration, but that it will be made obsolete and removed after some future development.

The `drs_objects` collection needs the following migration applied:
- Replace the value of `_id` with the value from `object_id`.
- Delete `object_id`.
- Rename `decryption_secret_id` to `secret_id`.
- Rename `s3_endpoint_alias` to `storage_alias`.

The `dcsPersistedEvents` collection needs the following changes to the `payload` field:
- Where `type_` == `drs_object_served`:
  - Replace the value for `file_id` with the value from `target_object_id`
  - **QUESTION**: *Should we update the actual schema so `s3_endpoint_alias` becomes `storage_alias`?*
- Where `type_` == `drs_object_registered`:
  - Replace the value (presently accession) for `file_id` with the corresponding UUID4 value from the temporary table defined above.
  - `drs_uri` remains untouched...for now.
    - It's not clear how this will be constructed in the future, but the DCS should only be concerned with files and file IDs rather than file accessions from metadata.

Another note about the DCS migrations is that they should be moved to the init container style. Instead of executing `run_db_migrations()` as part of every entrypoint, the migrations should be run as their own command. 

#### Work to be completed for the DCS
- [ ] Get the updated `ghga-event-schemas` version
- [ ] Adapt code for schema updates
- [ ] Write migrations
- [ ] Move migrations to own CLI command so they can be run as an init container
  - [ ] Work with DevOps to get this configured in k8s

## Diagrams:

### Service Diagrams:
#### Service Map
![Service Map](./images/service_map.png)

#### S3 Bucket Access Permissions
![S3 Bucket Access Permissions](./images/bucket_access.png)

#### Normal Upload Sequence Diagram (Beginning as the Upload Completes)

```mermaid
sequenceDiagram
    box rgb(0, 150, 210, 0.5) Topics
        participant FileUploads
        participant InterrogationReports
        participant ResearchDataUploadBoxes
        participant FileInternallyRegistered
    end
    box rgb(255, 255, 200, .5) Services
        participant Connector
        participant UCS
        participant UOS
        participant EKSS
        participant FIS
        participant DHFS
        participant DINS
        participant IFRS
    end
    box rgb(200, 75, 35, 0.5) S3 Buckets
        participant inbox
        participant interrogation
        participant archive
    end
    
    Connector->>UCS: Finalize File Upload<br>to inbox bucket
    UCS->>FileUploads: UPSERT: FileUpload(completed: True)
    FileUploads->>FIS: UPSERT: FileUpload(completed: True)
    FIS->>FIS: Create InterrogationReport with initial data
    DHFS->>FIS: GET (polling)
    FIS-->>DHFS: 200: list[FileUpload]
    note left of DHFS: We will assume only one file is<br>returned. In reality, the list<br>returned by the FIS will contain<br>multiple files, and the DHFS will<br>process them in parallel.
    DHFS->>inbox: Fetch first file chunk to get envelope
    DHFS->>DHFS: Decrypt envelope with private key
    DHFS->>DHFS: Generate new file secret
    DHFS->>FIS: Submit new file secret (encrypted)
    FIS->>EKSS: Submit secret
    EKSS-->>FIS: Secret ID
    DHFS->>interrogation: Initiate multipart upload
    DHFS->>DHFS: Generate download URL
    rect rgb(30, 30, 30, .4)
        loop For each file chunk
            inbox->>DHFS: Stream file chunk
            DHFS->>DHFS: Decrypt with old secret
            DHFS->>DHFS: Re-encrypt with new secret
            DHFS->>DHFS: Update checksums
            DHFS->>interrogation: Upload file chunk
        end
    end
    DHFS->>DHFS: Compare checksums
    DHFS->>interrogation: Complete multipart upload
    DHFS->>FIS: POST InterrogationReport
    FIS->>InterrogationReports: Publish: InterrogationReport
    InterrogationReports->>UCS: Consume: InterrogationReport
    InterrogationReports->>IFRS: Consume: InterrogationReport
    UCS->>inbox: Delete File
    UCS->>FileUploads: UPSERT: FileUpload(state=ARCHIVED)
    Note right of UCS: UCS considers items as<br> archived once they're <br> uploaded to the <br> interrogation bucket.
    UOS->>ResearchDataUploadBoxes: UPSERT: ResearchDataUploadBox(state=CLOSED)
    ResearchDataUploadBoxes->>IFRS: UPSERT: ResearchDataUploadBox(state=CLOSED)
    IFRS->>interrogation: Copy File from interrogation to permanent bucket
    note left of IFRS: IFRS will copy each file in<br>the ResearchDataUploadBox
    interrogation-->>archive: Copy File
    IFRS->>FileInternallyRegistered: Publish FileInternallyRegistered
    FileInternallyRegistered->>FIS: Consume FileInternallyRegistered
    FIS->>FIS: Set InterrogationReport.is_archived=True
```

#### DHFS Cleanup Job Sequence Diagram
```mermaid
sequenceDiagram
    box rgb(0, 150, 210, 0.5) Topics
        participant InterrogationReports
    end
    box rgb(255, 255, 200, .5) Services
        participant FIS
        participant DINS
        participant DHFS
    end
    box rgb(200, 75, 35, 0.5) S3 Buckets
        participant interrogation
    end
    
    DHFS->>interrogation: List object IDs (polling)
    interrogation-->>DHFS:
    rect rgb(30, 30, 30, .8)
        loop For each file in bucket
            DHFS->>FIS: GET InterrogationReport
            FIS-->>DHFS: 200 or 404
            DHFS->>interrogation: Delete if conditions met
        end
    end
```

#### `FileUpload` State Diagram
```mermaid
stateDiagram-v2
    [*] --> init: UCS - Upload initiated
    init --> inbox: UCS - Upload completed
    inbox --> archived: DHFS report -> FIS (passed)
    inbox --> failed: DHFS report -> FIS (failed)
    inbox --> cancelled: UCS - file deleted
    archived --> [*]
    failed --> [*]
    cancelled --> [*]

    note left of inbox
      Source of truth:
      - State set by UCS
      - Transitions driven by InterrogationReport published by FIS
    end note
```

## Human Resource/Time Estimation:

Number of sprints required: 3

Number of developers required: 1-2
