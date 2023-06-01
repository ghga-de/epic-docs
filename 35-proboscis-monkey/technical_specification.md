# Upload Metadata Ingest and Unique S3 ID for Permanent Storage (Proboscis Monkey)
**Epic Type:** Implementation Epic

## Scope

This epic has two different goals.
First of all, internal ID handling needs to be changed to use a unique ID for S3 objects instead of directly using the file ID.
For now, a unique internal S3 ID is only required once the object is moved into permanent storage.
The second task is the implementation of an ingestion service that processes metadata files produced by the file upload script.

### Outline:

#### Ingestion Service:

The ingestion service will provide a RESTful endpoint to process the output files of the upload script, one at a time.
This endpoint expects the assymetrically encrypted output metadata as body and an authentication token hash in the header.
To simplify the process, the public key of the encryption keypair is provided as a configuration option to the service, as is the token hash for authentication.
The actual token from which the hash is derived and the private key remain with the submitter.

The service needs to take care of
 1) firing an event to propagate information to the IFRS/DCS databases using existing mechanisms
 2) communication with the vault to store the encryption/decryption secret

In addition, the data steward scripts need to be extended to include a script interacting with this endpoint.

#### Unique S3 ID:

Currently only one ID is used across all file services, however, a different, unique ID should be used to identify the corresponding objects in S3.
This will allow to better separate internal only concerns from outward facing ones.
This ID will be in the form of a UUID4.
It should suffice to only create the unique ID when copying the incoming file from the staging bucket to permanent storage, as once the file data reaches the permanent storage, it will no longer be available in the inbox and staging buckets (to be implemented separately, at a later point in time).

This taks necessitates changes to the event schemas, internal file registry and download controller.

### Included/Required:

### Optional:

### Not included:

- Changes to upload related services, inbox and staging bucket interaction

## API Definitions:

### RESTful/Synchronous:

### Payload Schemas for Events:

## Human Resource/Time Estimation:

Number of sprints required: 1

Number of developers required: 2