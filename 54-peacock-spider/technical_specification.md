# Dataset Information Service (Peacock Spider)
**Epic Type:** Implementation Epic

## Scope
### Outline:

This epic aims to create a new service that provides an authoritative source of truth for file information.
File information includes (for now) file size and checksums.

### Included/Required:
    
The Dataset Information Service needs to listen to events produced in the file service backend reporting successful file uploads and dataset registration information.
Corresponding deletion events should also need to be handled

The proposed event schema that should be used to extract and store file information by this service is `FileInternallyRegistered`.
This event contains all relevant information and it is guaranteed that the file in question is actually stored in permanent storage at that point in time.
Additionally, the service needs to subscribe to file deletion events and remove all data as requested.

Analogous functionality also needs to be implemented for upsertion and deletion events that deal with datasets. This means that the events captured by this service need to include those that are currently handled by the work package service, i.e. those conforming to the `MetadataDatasetOverview` and `MetadataDatasetID` schemas. 

File information should be provided through one simple REST endpoint returning all relevant information for a given file accession.
Analogously, information for all files in a dataset identified by a dataset accession should by returned by another REST endpoint.

### Not Included:
 
Return of internal only file metadata, i.e. encrypted part sizes as MD5 and SHA256 hashes and encrypted part size.

## API Definitions:

### RESTful/Synchronous:

#### File Information:
- GET `/file_information/{file_id}`: Get file size and sha256 for the unencrypted file content

Here the `file_id` is the public accession for the given file.

This should return a payload including the unencrypted file size in bytes and SHA256 checksum for the given file ID,
hiding unnecessary details for the requester.
```
{
    'size': ...,
    'sha256_hash': ...,
}
```

#### Dataset Information:
- GET `/dataset_information/{dataset_id}`: Get file size and sha256 for the unencrypted file content for all files in the given dataset

Here the `dataset_id` is the public accession for the given dataset.

This should return a list of objects with each element containing the `file_id`, unencrypted file size in bytes and SHA256 checksum (preferably sorted by accession).
```
{
    'file_information':
    [
        {
            `accession`: ...,    
            'size': ...,
            'sha256_hash': ...,
        },
        {
            `accession`: ...,    
            'size': ...,
            'sha256_hash': ...,
        },
        ...
    ]

}
```

### Payload Schemas for Events:

#### File Information:
Incoming population event schema: [FileInternallyRegistered](https://github.com/ghga-de/ghga-event-schemas/blob/faf00f361facc4195f2b9e9a0a69ec9645464bc3/src/ghga_event_schemas/pydantic_.py#L270-L273)

From this event, only the `file_id`, `decrypted_size` and `decrypted_sha256` need to be stored in the service.
These represent publicly accessible metadata that should be exposed by the service.

Incoming deletion event schema: [FileDeletionRequested](https://github.com/ghga-de/ghga-event-schemas/blob/faf00f361facc4195f2b9e9a0a69ec9645464bc3/src/ghga_event_schemas/pydantic_.py#L372-L381)

#### Dataset Information:

Incoming population/change event: [MetadataDatasetOverview](https://github.com/ghga-de/ghga-event-schemas/blob/54467290f2b61f2826de13a9aa78181ac38a08b8/src/ghga_event_schemas/pydantic_.py#L75-L89)

From this event the `accession` (of the dataset) and `files` fields need to be extracted and from the `files` field only the corresponding `accession` fields need to be saved.

Incoming deletion event schema: [MetadataDatasetID](https://github.com/ghga-de/ghga-event-schemas/blob/54467290f2b61f2826de13a9aa78181ac38a08b8/src/ghga_event_schemas/pydantic_.py#L51-L54)

## Additional Implementation Details:

Dataset information can exist in one of file states:

1) Dataset is not yet registered
2) Dataset is registered, but no file information is available
3) Dataset is registered and some file information is available
4) Dataset is registered and all file information is available
5) Dataset is deleted

File information can exist in one of three states:

1) File is not yet registered
2) File is registered
3) File is deleted

If the reason why data is not available is of no interest and if the all dataset registered states are treated the same, the following responses will be returned:

1) The file information endpoint either returns a payload or 404
2) The dataset information returns either a payload or 404. Within the payload, the fields of each file information object either are populated with the actual data or a None/null value. The API consumer can then check just one of the non accession fields to evaluate, if a file information object is populated. 

## Human Resource/Time Estimation:

Number of sprints required: 1

Number of developers required: 1
