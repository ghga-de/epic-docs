# Initial Version of a Study Registry Service (Apollo)

**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://ghga.pages.hzdr.de/internal.ghga.de/main/sops/development/epic_planning/).

## Scope

### Outline

The Study Registry Service (SRS) is the core GHGA registry service responsible for ingesting and archiving metadata from data submitters.

Its function is currently implemented by the [GHGA Data Steward Kit](https://github.com/ghga-de/ghga-datasteward-kit) in conjunction with the submission store and accession store managed via the [metldata](https://github.com/ghga-de/metldata) library.

More precisely, the service replaces the Submission Registry and the Accession Registry managed via metldata and stored on the file system of a virtual machine operated by a GHGA central data steward. The submissions are now stored in a database owned by the SRS.

The second crucial change introduced with the SRS is the separation of Experimental Metadata from Persistent Administrative Metadata, Dynamic Administrative Metadata, and Datasets, which were previously bundled in an all-in-one [GHGA metadata schema](https://github.com/ghga-de/ghga-metadata-schema), and the move from the former dataset-centric metadata concept to a study-centric one.

The Experimental Metadata should be managed in a generic way, without semantic interpretation of its content besides generating an accession number for each entity.

As a reminder, Experimental Metadata (EM) is data describing how the research data (that we store in “files”) was generated. Persistent Administrative Metadata (PAM) is non-experimental study metadata (e.g. authors, description), and Dynamic Administrative Metadata (DAM) is business metadata (e.g. authorized users, DAC contacts and policies). As the names indicate, PAM underlies the same archival immutability as the EM, while DAM is auditable.

An Experimental Metadata Ingress Model (EMIM) shall be used for validation. EMIMs shall be stored as schemapacks, and the EM shall be stored as datapacks. Both structures are provided by the [schemapack](https://github.com/ghga-de/schemapack) library.

All resources archived in GHGA that are subject to archival immutability can be retrieved by an accession number, a persistent identifier (PID) that always resolves to the exact same state of a resource.

Another change that needs to be considered in the implementation of the SRS is the new authorization concept that assumes all metadata is non-public by default. Public access to metadata or access restricted to certain users (usually the submitter) needs to be explicitly granted on the study level.

### Included/Required

- First implementation of the Study Registry Service
- REST API as detailed below
- Event publisher as detailed below

### Not included

The following features are not included in the first version of the study registry:

- Change requests to existing studies. In the future, it should be possible to create a modified copy of an existing study.
- New PID schema.
- Support for multiple or versioned EMIMs, integration of an EMIM registry.
- Migration of the existing submission store to the Study Registry Service.
- Frontend to ingest submissions and manage dynamic administrative metadata and lookups.
- Populating and updating lookup tables from existing dictionaries and ontologies.
- Implementation of the EM validation service.
- Audit logging.

## Implementation Details

### Managed Entities

#### Study

In the new study-centric metadata concept, the Study serves as the central container for Dataset, Publication, and alternative accessions for each experimental metadata entity. It has a small number of attributes but plays a critical role as a container, as its lifecycle is linked to many other entities. It is immutable and receives a permanent accession number upon creation.

Attributes:

- `id: str` - the PID of the study (primary key)
- `title: str` - comprehensive title for the Study
- `description: str` - detailed description (abstract) describing the goals of the Study
- `types: list[StudyType]` - the type(s) of this Study
- `affiliations: list[str]` - the affiliation(s) associated with this Study
- `status: StudyStatus` - the current status of the study (see below)
- `users: list[UUID] | None` - user(s) who can access the study (None means publicly accessible)
- `created: Date` - when the entry was created
- `created_by: UUID` - the id of the user who uploaded the study
- `approved_by: UUID | None` - the id of the user who approved the study

Note: Later, the Study should also have an attribute referencing the EMIM used in the submission. The experimental metadata itself is kept in a separate entity model described below.

#### ExperimentalMetadata

The ExperimentalMetadata entity stores the actual experimental metadata belonging to a study as a generic JSON object.

Attributes:

- `id: str` - the PID of the study to which the experimental metadata belongs
- `metadata: JsonObject` - the submitted experimental metadata as submitted
- `submitted: Date` - when the experimental metadata was submitted

This entity model has been separated from the Study entity model because the metadata can be large (might require GridFS) and is usually accessed separately from the study after transformation.

#### Publication

The Publication is the citation reference for the study. It is immutable and receives a permanent accession number upon creation.

Attributes:

- `id: str` - the PID of the Publication (primary key)
- `title: str` - the title for the Publication
- `abstract: str | None` - the abstract of the Publication
- `authors: list[str]` - the author(s) of the Publication
- `year: int` - the year of the Publication
- `journal: str | None` - the name of the journal
- `doi: str | None` - the DOI identifier of the publication
- `study_id: str` - the PID of the study associated with this publication
- `created: Date` - when the entry was created

New Publication entity instances should only be created after verification that the corresponding Study entity instance exists.

#### DataAccessCommittee

The DataAccessCommittee entity describes a Data Access Committee (DAC). It is managed independently of any study and is mutable.

Attributes:

- `id: str` - the code of the DAC (short, uppercase) (primary key)
- `name: str` - human readable name of the DAC
- `email: EmailStr` - the contact email address of the DAC
- `institute: str` - the institute the DAC belongs to
- `created: Date` - when the DAC entry was created
- `changed: Date` - when the DAC entry was last changed
- `active: bool` - whether the DAC is still active

Note: The `name` should correspond to the `alias` in the old model. The `id` should be derived from the name (shortened, converted to upper case). We do not assign a citable accession number to DACs anymore.

#### DataAccessPolicy

The DataAccessPolicy entity describes a policy for data access (DAP). It is managed independently of a study, is mutable, and belongs to exactly one DataAccessCommittee.

Attributes:

- `id: str` - the code of the DAP (short, uppercase) (primary key)
- `name: str` - human readable name of the DAP
- `description: str` - a longer description of the DAP
- `text: str` - the complete text for the DAP
- `url: Url | None` - if available, the URL for the DAP
- `duo_permission_id: DuoPermission` - the DUO id of the corresponding Data Use Permission
- `duo_modifier_ids: list[DuoModifier]` - the DUO id(s) of the corresponding Data Use Modifiers
- `dac_id: str` - the code of the DAC linked to this DAP
- `created: Date` - when the DAP entry was created
- `changed: Date` - when the DAP entry was last changed
- `active: bool` - whether the DAP is still active

The `id` should correspond to the `alias` in the old model, `text` and `url` correspond to `policy_text` and `policy_url`. We do not assign a citable accession number to DAPs anymore.

New DataAccessPolicy entity instances should only be created after verification that the corresponding DataAccessCommittee entity instance exists.

#### Dataset

The Dataset entity describes a set of files and represents the smallest unit for which a data access request can be formulated. All attributes except `dap_id` are immutable, and Dataset entity instances cannot be deleted. However, new Dataset entity instances can be created after study creation and assigned to a study without violating its immutability. An immutable accession number is assigned upon creation. Every Dataset belongs to exactly one Study. The DataAccessPolicy assignment is mutable.

Attributes:

- `id: str` - the PID of the Dataset (primary key)
- `title: str` - comprehensive title for the Dataset
- `description: str` - detailed description summarizing this Dataset
- `types: list[DatasetType]` - the type(s) of this Dataset
- `study_id: str` - the PID of the study associated with this Dataset
- `dap_id: str` - the code of the DAP for this Dataset
- `files: list[str]` - the corresponding IDs (aliases) as specified in EM
- `created: Date` - when the Dataset was created
- `changed: Date` - when the DAP for the Dataset was last changed

New Dataset entity instances should only be created after verification that the corresponding Study and DataAccessPolicy entity instances exist, and that all specified files exist in EM and are specified only once.

#### ResourceType

The ResourceType entity holds the possible types for studies and datasets (could be also used for other resources).

Attributes:

- `id: UUID` - internal ID (primary key)
- `code: str` - the code of the resource type (short, uppercase)
- `resource: TypedResource` - the kind of resource this type belongs to
- `name: str` - the human readable name of the resource type
- `description: str | None` - optional definition text or help text
- `created: Date` - when the resource type was created
- `changed: Date` - when the resource type was last changed
- `active: bool` - whether the resource type is still active

The corresponding collection should be created with a composite index on `code` and `resource`.

The corresponding collection can be populated with the study types as defined in the existing GHGA metadata schema. The existing dataset types need to be extracted from the current submission store, as they are not defined in the existing schema.

#### Accession

The Accession entity stores all existing primary accessions. See also the sections on the Accession Registry below.

Attributes:

- `id: str` - the primary accession number (PID)
- `type: AccessionType` - the entity type referenced by this accession
- `created: Date` - when the accession was created
- `superseded_by: str | None` - if deprecated, a new primary accession

Note: When we start introducing versioned accession numbers, we can consider splitting this into two entity models, one for holding the base accession numbers, and another one for holding the versioned ones.

#### AltAccession

The AltAccession entity stores all existing alternative accessions with a reference to the corresponding primary accession. See also the sections on the Accession Registry below.

Attributes:

- `id: str` - the alternative accession number (primary key)
- `pid: str` - the corresponding primary accession number (foreign key)
- `type: AltAccessionType` - the type of alternative accession
- `created: Date` - when the alternative accession was created

#### EmAccessionMap

The EmAccessionMap entity stores mappings from submitted IDs to primary accessions.

Attributes:

- `id: str` - the PID of the study to which the experimental metadata belongs
- `maps: dict[str, dict[str, str]]` - per-resource-type accession maps

The `maps` attribute holds, for every resource in the experimental metadata, a mapping from the identifier used in the original submission (the "alias" field in the current metadata schema) to the primary accession generated by the Study Registry Service and stored as an Accession in the database.

Example:

```json
{
  "experiments: {
    "EXP_1": "GHGAX12345678901234",
    "EXP_2": "GHGAX12345678901235"
    ...
  },
  "experiment_methods": {
    "EXP_METHOD_1": "GHGAQ12345678901234",
    "EXP_METHOD_2": "GHGAQ12345678901235"
    ...
  },
  "samples": {
    "SAMPLE_1": "GHGAN12345678901234",
    "SAMPLE_2": "GHGAN12345678901235",
    ...
  },
  ...
}
```

These maps are automatically generated by the service after EM has been submitted.

This entity model has been separated from the Study entity model because the accession maps can be large (might require GridFS) and the original accessions are rarely needed after transformation.

### Enums

#### AccessionType

The AccessionType enum holds the different types of primary accessions:

- for Experimental Metadata:
  - `ANALYSIS`
  - `ANALYSIS_METHOD`
  - `FILE`
  - `EXPERIMENT`
  - `EXPERIMENT_METHOD`
  - `INDIVIDUAL`
  - `SAMPLE`
- for Administrative Metadata:
  - `DATASET`
  - `PUBLICATION`
  - `STUDY`

#### AltAccessionType

The AltAccessionType enum holds the different kinds of alternative accessions:

- `EGA` - EGA accession
- `FILE_ID` - internal file id
- `GHGA_LEGACY` - legacy GHGA accession (after switching to new PID schema)

#### DatasetType

The DatasetType enum lists all possible Dataset types. It is populated at service start with the codes of the ResourceType instances belonging to the `DATASET` resource. The service therefore needs to be restarted in order to make new entries available.

#### DuoModifier

The DuoModifier enum lists all existing [DUO](https://www.ga4gh.org/product/data-use-ontology-duo/) modifiers. These are descendants of 'DUO:0000017: data use modifier' (e.g. 'DUO:0000043').

 The existing DUO ids, their shorthands, labels and descriptions are available a [CSV file](https://github.com/EBISPOT/DUO/blob/master/duo.csv), it contains currently 20 modifiers. The name of the enum should be the shorthand, the value should be the identifier:

```python
class Modifier(StrEnum):
    RS = "DUO:0000012"
    HMB = "DUO:0000006"
    ...
```

#### DuoPermission

The DuoPermission enum lists all existing [DUO](https://www.ga4gh.org/product/data-use-ontology-duo/) permissions. These are descendants of 'DUO:0000001: data use permission' (e.g. 'DUO:0000004').

 The existing DUO ids, their shorthands, labels and descriptions are available a [CSV file](https://github.com/EBISPOT/DUO/blob/master/duo.csv), it contains currently 5 permissions. The name of the enum should be the shorthand, the value should be the identifier:

```python
class DuoPermission(StrEnum):
    NPOA = "DUO:0000004"
    NMDS = "DUO:0000015"
    ...
```

#### StudyType

The StudyType enum lists all possible Study types. It is populated at service start with the codes of the ResourceType instances belonging to the `STUDY` resource. The service therefore needs to be restarted in order to make new entries available.

#### TypedResource

The TypedResource enum lists all resources whose types are managed by this service:

- `DATASET`: Dataset
- `STUDY`: Study

#### StudyStatus

The StudyStatus enum describes all possible states of a Study:

- `PENDING`
- `FROZEN`
- `APPROVED`
- `PERSISTED`

In the initial implementation we will only use the status values `PENDING` and `PERSISTED`.

### Core functionality

The Study Registry Service can be accessed by a REST API in order to submit and query DAM, PAM, EM and lookup values (ResourceType entries). It also allows updating DAM and lookup values. The REST API is described further below.

For now, the accession numbers should be created in the same way as before, assuming that the existing accessions have been imported into the database already, so that no duplicate accessions will be created.

A newly submitted study shall always be created with the status `PENDING`.

When the status is updated with the `PATCH /studies` endpoint or the `/rpc/publish` endpoint is called, the service should validate the submission as detailed below. If the validation fails, the status update shall be rejected and the status of the submission shall no be changed.

When the `/rpc/publish` endpoint is called and the submission has been successfully validated, the service will create new accession numbers for all resources contained in the EM and store these in the database as Accession, AltAccession, and EmAccessionMap. If the study had been already published before, accessions for resources that have been removed in the submission shall be deleted.

The service will then create an AnnotatedEMPack and publish it as an event, as described further below.

As another functionality, the service shall support the mapping of uploaded files to their experimental metadata entries. See the `POST /file_names` endpoint below.

### Validation

On ingress, submissions must be validated and rejected if there are any validation errors. Validation happens on several levels:

- ingested DAM and PAM (e.g., study or dataset) are validated as DTOs using Pydantic
- the submission must be complete (particularly, it must have a Publication and ExperimentalMetadata)
- all referenced resources in the ingested DAM and PAM must exist and not be deprecated
- ingested EM is validated by a separate EM validation service

In this epic, we assume that the EM validation service exists and provides a simple REST API for validating EM against a given EMIM (currently we only have one).

### Accession registry

The first implementation of the Study Registry Service shall also contain an initial implementation of an accession registry. Later, this can be outsourced to a separate, dedicated service.

Resources archived in GHGA are immutable and should get accession numbers that are globally unique, persistent, and long-term resolvable. To emphasize these qualities, we also call these accession numbers persistent identifiers (PIDs). A PID must always resolve to the exact same state of a resource.

Particularly, the following entity types get PIDs:

- studies
- all research data files
- datasets (sets of research data files)
- entity instances that are part of the experimental metadata

Currently, the accession numbers used by GHGA have a format that starts with the uppercase letters "GHGA", followed by another uppercase letter indicating the resource type (e.g. "S" for study, "D" for Dataset, "U" for publication), followed by a random 14-digit number. The prefixes and number of digits are defined in the [metadata configuration](https://github.com/ghga-de/metadata-config/blob/main/configuration/metadata_config.yaml).

We are referring to these accession numbers as "legacy accessions", since we plan to replace them with a new PID schema that should reflect the study-centric metadata concept better and should also contain the year and version number.

We will continue to use the legacy accession numbers until the new PID schema is fully defined, which is not covered in this epic.

In addition to GHGA accession numbers, which we also refer to as our primary (canonical) accessions, the accession registry should also keep track of alternative accessions (alt accessions) and resolve them to the corresponding primary accessions. For instance, the accession number under which a resource is archived in EGA should be stored as alternative accession. After switching to the new PID schema, the legacy accession numbers should also be stored as alternative accessions.

The accession store shall also contain a mapping from file accessions to internal file IDs, using the same mechanism of alt accession references, with internal file IDs being a special type of alt accession.

### Authorization

With the introduction of study-centric metadata processing, we also introduce authorization for metadata. Instead of assuming all metadata is public, we now require access grants for metadata, similarly to the access grants for downloading datasets and uploading files to research data upload boxes.

In the first implementation, the authorization will not yet be managed by actual grants via the claims repository, but handled via the `users` field of the Study instances. The field should store the IDs of all users who are allowed to access the study and its corresponding metadata. This field can also be set to `None`, indicating that the metadata is publicly accessible.

If study submissions contain non-public metadata, this metadata must be provided as files contained in a dataset that needs to be requested like other datasets.

## User Journeys (optional)

This epic covers the following user journeys:

\<Images and descriptions of user journeys go here. Images are deposited in the `./images` sub-directory.\>

![\<Example Image\>](./images/data_upload.jpg)

TODO (see AC: Metadata Services)

## API Definitions

### RESTful/Synchronous

The service provides an API that is accessible via the Data Portal. Through this API, data stewards can modify resources within the immutability constraints outlined in this document. Some read-only endpoints are also exposed publicly in the first implementation. This might be tightened up later when the corresponding information will be available elsewhere like via the API of the upcoming resource registry service, or we could make this configurable.

In order to support the migration of the existing dataset-centric metadata that already has been imported into GHGA, we will need to extend the API with additional parameters or end-points that allow taking over existing accession numbers or do bulk imports without the data portal. Similar APIs might be added later to ingest metadata directly from other sources.

#### Study API

##### `POST /studies`

- Auth: internal auth token with data steward role
- Request Body: `Study` (without `id`, `status`, user and date related fields)
- Response Body: `Study`
- Returns: 201 or error code

The PID will be automatically created.

After this request, the new Study will have the status `PENDING` and the `users` list should contain only the user who submitted the study.

##### `GET /studies`

- Auth: internal auth token (optional)
- Query parameters (all optional):
  - `status: StudyStatus` - filter for specific status
  - `type: StudyType` - filter for specific type
  - `text: str` - text filter (partial match in any plain text field)
  - `skip: int`, `limit: int`: used for pagination
- Response Body: `list[Study]`
- Returns: 200 or error code

Only returns studies that are either public or accessible to the user (i.e. the user must be a data steward of access must have been granted to the user).

The user related fields should only be returned if the request is made by a data steward.

##### `GET /studies/{id}`

- Auth: internal auth token (optional)
- Response Body: `Study`
- Returns: 200 or error code

If the study is not public, the auth token is required. In this case, if the user is not a data steward and the user is not granted access to the study, returns error code 403.

The user related fields should only be returned if the request is made by a data steward.

##### `PATCH /studies/{id}`

- Auth: internal auth token with data steward role
- Request Body: new `status` and `users`
- Returns: 204 or error code

The `status` can only be moved from `PENDING` to `PERSISTED`, otherwise returns error code 409. The `users` field can only be set to `None` when the status is `PERSISTED`.

When the status is changed, also validates the study similar to the `/rpc/publish` endpoint, and returns error code 409 if the validation fails, without changing the status.

##### `DELETE /studies/{id}`

- Auth: internal auth token with data steward role
- Returns: 200 or error code

The study must have the status `PENDING`, otherwise returns error code 409.

Will also delete the corresponding experimental metadata, publications and dataset, and all corresponding accessions.

#### ExperimentalMetadata API

##### `POST /metadata`

- Auth: internal auth token with data steward role
- Request Body: `ExperimentalMetadata` (without `submitted`)
- Returns: 204 or error code

Will upsert a corresponding ExperimentalMetadata instance.

The corresponding study must have the status `PENDING`, otherwise returns error code 409.

##### `GET /metadata/{id}`

- Auth: internal auth token with data steward role
- Response Body: `ExperimentalMetadata`
- Returns: 200 or error code

The `id` must be the PID of the corresponding study.

Gets the corresponding submitted, unprocessed ExperimentalMetadata instance.

##### `DELETE /metadata/{id}`

- Auth: internal auth token with data steward role
- Returns: 200 or error code

The `id` must be the PID of the corresponding study.

The study must have the status `PENDING`, otherwise returns error code 409.

#### Publication API

##### `POST /publications`

- Auth: internal auth token with data steward role
- Request Body: `Publication` (without `id`, `created`)
- Response Body: `Publication`
- Returns: 201 or error code

Will upsert a corresponding Publication instance.

The PID will be automatically created.

The corresponding study must have the status `PENDING`, otherwise returns error code 409.

##### `GET /publications`

- Auth: internal auth token (optional)
- Query parameters (all optional):
  - `year: int` - filter for specific year
  - `text: str` - text filter (partial match in any plain text field)
  - `skip: int`, `limit: int`: used for pagination
- Response Body: `list[Publication]`
- Returns: 200 or error code

Only returns publications whose studies are either public or accessible to the user (i.e. the user must be a data steward of access must have been granted to the user).

##### `GET /publications/{id}`

- Auth: internal auth token (optional)
- Response Body: `Publication`
- Returns: 200 or error code

If the corresponding study is not public, the auth token is required. In this case, if the user is not a data steward and the user is not granted access to the study, returns error code 403.

##### `DELETE /publications/{id}`

- Auth: internal auth token with data steward role
- Returns: 200 or error code

The corresponding study must have the status `PENDING`, otherwise returns error code 409.

Will also delete the accession number for the publication.

#### DataAccessCommittee API

##### `POST /dacs`

- Auth: internal auth token with data steward role
- Request Body: `DataAccessCommittee` (without `created` and `changed`)
- Returns: 204 or error code

Creates a new DataAccessCommittee instance.

##### `GET /dacs`

- Auth: None
- Response Body: `list[DataAccessCommittee]`
- Returns: 200 or error code

Gets all DataAccessCommittee instances.

##### `GET /dacs/{id}`

- Auth: None
- Response Body: DataAccessCommittee
- Returns: 200 or error code

Gets the DataAccessCommittee instance with the given `id`.

##### `PATCH /dacs/{id}`

- Auth: internal auth token with data steward role
- Request Body: partial `DataAccessCommittee`(without `id`, `created` and `changed`)
- Returns: 204 or error code

Updates one or more attributes of an existing DataAccessCommittee instance.

##### `DELETE /dacs/{id}`

- Auth: internal auth token with data steward role
- Returns: 200 or error code

If the corresponding DAC is referenced by any DAP, returns error code 409.

Deletes a DataAccessCommittee instance.

#### DataAccessPolicy API

##### `POST /daps`

- Auth: internal auth token with data steward role
- Request Body: `DataAccessPolicy` (without `created` and `changed`)
- Returns: 204 or error code

Creates a new DataAccessCommittee instance.

##### `GET /daps`

- Auth: None
- Response Body: list[DataAccessPolicy]
- Returns: 200 or error code

Gets all DataAccessPolicy instances.

##### `GET /daps/{id}`

- Auth: None
- Response Body: DataAccessPolicy
- Returns: 200 or error code

Gets the DataAccessPolicy instance with the given `id`.

##### `PATCH /daps/{id}`

- Auth: internal auth token with data steward role
- Request Body: partial `DataAccessPolicy`(without `id`, `created` and `changed`)
- Returns: 204 or error code

Updates one or more attributes of an existing DataAccessPolicy instance.

##### `DELETE /daps/{id}`

- Auth: internal auth token with data steward role
- Returns: 200 or error code

If the corresponding DAP is referenced by any study, returns error code 409.

Deletes a DataAccessPolicy instance.

#### Dataset API

##### `POST /datasets`

- Auth: internal auth token with data steward role
- Request Body: `Publication` (without `id`, `created`, `changed`)
- Response Body: `Dataset`
- Returns: 201 or error code

Will upsert a corresponding Dataset instance.

The PID will be automatically created.

The corresponding study must have the status `PENDING`, otherwise returns error code 409.

##### `GET /datasets`

- Auth: internal auth token (optional)
- Query parameters (all optional):
  - `type: DatasetType` - filter for specific type
  - `study_id: str` - filter for specific study
  - `text: str` - text filter (partial match in any plain text field)
  - `skip: int`, `limit: int`: used for pagination
- Response Body: `list[Dataset]`
- Returns: 200 or error code

Only returns publications whose studies are either public or accessible to the user (i.e. the user must be a data steward of access must have been granted to the user).

##### `GET /datasets/{id}`

- Auth: internal auth token (optional)
- Response Body: `Dataset`
- Returns: 200 or error code

If the corresponding study is not public, the auth token is required. In this case, if the user is not a data steward and the user is not granted access to the study, returns error code 403.

##### `PATCH /datasets/{id}`

- Auth: internal auth token with data steward role
- Request Body: new `dap_id`
- Returns: 204 or error code

Changing the `dap_id` will be allowed even when the study is already in status `PERSISTED`.

##### `DELETE /datasets/{id}`

- Auth: internal auth token with data steward role
- Returns: 200 or error code

The corresponding study must have the status `PENDING`, otherwise returns error code 409.

Will also delete the accession number for the dataset.

#### ResourceType API

##### `POST /resource_types`

- Auth: internal auth token with data steward role
- Request Body: `ResourceType` (without `id`, `created`, `changed`)
- Response Body: `ResourceType`
- Returns: 201 or error code

Will create a new ResourceType instance.

##### `GET /resource_types`

- Auth: None
- Query parameters (all optional):
  - `resource: TypedResource` - filter for specific type
  - `text: str` - text filter (partial match in any plain text field)
  - `skip: int`, `limit: int`: used for pagination
- Response Body: `list[ResourceType]`
- Returns: 200 or error code

##### `GET /resource_types/{id}`

- Auth: None
- Response Body: `ResourceType`
- Returns: 200 or error code

If the corresponding study is not public, the auth token is required. In this case, if the user is not a data steward and the user is not granted access to the study, returns error code 403.

##### `PATCH /resource_types/{id}`

- Auth: internal auth token with data steward role
- Request Body: partial `ResourceType`(without `id`, `created` and `changed`)
- Returns: 204 or error code

##### `DELETE /resource_types/{id}`

- Auth: internal auth token with data steward role
- Returns: 200 or error code

If the resource type is still referenced by a corresponding resource, returns error code 409.

Deletes a ResourceType instance.

#### Accession API

##### `GET /accession/{id}`

- Auth: None
- Returns: 200 or error code 404 if accession doesn't exist

Gets the Accession instance with the given primary accession number.

##### `GET /accession/{id}?type={type}`

- Auth: None
- Returns: 200 or error code 404 if accession doesn't exist

Gets the AltAccession instance with the given alternative accession number and type.

The type `FILE_ID` is not allowed here to not expose internal numbers.

#### File names

##### `GET /file_names/{id}`

- Auth: internal auth token with data steward role
- Response Body: map from file accessions to objects with `name` and `alias` properties
- Returns: 200 or error code

Returns a mapping from all file accessions for the study with the given PID to the corresponding file names and aliases as they are submitted in the EM.

This endpoint is called by the frontend file mapping tool at the end of the upload process.

##### `POST /file_names/{id}`

- Auth: work order token for file mapping from UOS
- Request Body: map from file accessions to internal file IDs
- Returns: 204 or error code

This endpoint may only be called from UOS.

Should check whether the specified file accessions exist and all belong to the study with the specified study PID.

Should then create an `AltAccession` instance with type `FILE_ID` for all entries in the passed map, where `pid` should be the key and `id` should be the value in the map.

Should hen also republish the passed map for consumption by DINS and WPS.

### RPC Style/Synchronous

#### Publication

##### `POST /rpc/publish/{id}` 

- Auth: internal auth token with data steward role
- Returns: 204 or error code

Triggers the publication of the study with the specified PID after verification that the necessary data is complete and valid. Returns error code 409 otherwise.

This endpoint may be called even before the study is persisted in order to pass the submitted study data to downstream services and make it visible in the data portal for preview.

### Payload Schemas for Events

The service publishes events that communicate the state of its entity instances.

In particular, it publishes Annotated Experimental Metadata (AEM). This consists of the original, unmodified experimental metadata published along with additional annotations. These annotations contain all other study-related information and can be consumed by the experimental metadata transformation service and integrated into downstream transformed AEM.

The published AEM events shall have the following schema:

- `metadata: JSONObject` - the actual experimental metadata from ExperimentalMetadata
- `accessions: JSONObject` - the corresponding maps from EmAccessionMap
- `study: Study` - the corresponding study with nested publication
- `datasets: list[Dataset]` - the associated datasets with nested DAP and DAC

The services also re-publishes file name mappings received from the UOS via the REST-API. The payload should be the exact same mapping, the study PID is not needed.

## Human Resource/Time Estimation

Number of sprints required: 3

Number of developers required: 2
