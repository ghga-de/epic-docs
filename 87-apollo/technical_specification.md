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

### Optional

TODO

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

In the new study-centric metadata concept, the Study serves as the central container for Dataset, Publication, and alternative accessions for each experimental metadata entity. It has a small number of attributes but plays a critical role as a container entity, as its lifecycle is linked to many other entities. It is immutable and receives a permanent accession number upon creation.

Attributes:

- `id: str` - the PID of the study (primary key)
- `title: str` - comprehensive title for the Study
- `description: str` - detailed description (abstract) describing the goals of the Study
- `types: list[str]` - the type(s) of this Study (as list of their Term ids)
- `affiliations: list[str]` - the affiliation(s) associated with this Study
- `status: StudyStatus` - the current status of the study (see below)
- `users: list[UUID] | None` - user(s) who can access the study (None means publicly accessible)

This class should have a validator that checks that the specified types are not empty and exist as Term in the namespace `GHGA_STUDY_TYPE`.

Note: Later, the Study should also have an entity referencing the EMIM used in the submission. The experimental metadata itself is kept in a separate entity described below.

#### ExperimentalMetadata

The ExperimentalMetadata entity stores the actual experimental metadata belonging to a study as a generic JSON object.

Attributes:

- `id: str` - the PID of the study to which the experimental metadata belongs
- `metadata: JsonObject` - the submitted experimental metadata as submitted

This entity has been separated from the Study entity because the metadata can be large (might require GridFS) and is usually accessed separately from the study after transformation.

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
- `study_id: str` - the study associated with this publication

This class should have a validator that checks that the study linked to in `study_id` exists.

#### DataAccessCommittee

The DataAccessCommittee entity describes a Data Access Committee (DAC). It is managed independently of any study and is mutable.

Attributes: all attributes of Term (see below), plus:

- `email: str` - the contact email address of the DAC
- `institute: str` - the institute the DAC belongs to

A human-readable identifier would be useful for management and assignment purposes, but no citable accession number is required.

#### DataAccessPolicy

The DataAccessPolicy entity describes a policy for data access (DAP). It is managed independently of a study, is mutable, and belongs to exactly one DataAccessCommittee.

Attributes: all attributes of Term (see below), plus:

- `text: str` - the complete text for the Data Access Policy
- `duo_permission: str` - the DUO id of the corresponding Data Use Permission
- `duo_modifiers: list[str]` - the DUO id(s) of the corresponding Data Use Modifiers
- `dac_id: str` - the DAC linked to this DAP

This class should have a validator that checks that the specified permissions and modifiers exist as Term in the namespace `GA4GH_DUO` and that the DAC linked to in `dac_id` also exists.

#### Dataset

The Dataset entity describes a set of files and represents the smallest unit for which a data access request can be formulated. All attributes except `dap_id` are immutable, and the entity itself cannot be deleted. However, new Datasets can be created after study creation and assigned to a study without violating its immutability. An immutable accession number is assigned upon creation. Every Dataset belongs to exactly one Study. The DataAccessPolicy assignment is mutable.

Attributes:

- `id: str` - the PID of the Dataset (primary key)
- `title: str` - comprehensive title for the Dataset
- `description: str` - detailed description summarizing this Dataset
- `types: list[str]` - the type(s) of this Dataset (as list of Term ids)
- `study_id: str` - the PID of the study associated with this Dataset
- `dap_id: str` - the PID of the data access policy for this Dataset
- `files: list[str]` - the corresponding IDs (aliases) as specified in EM

This class should have a validator that checks that the specified types are not empty, that they exist as Term in the namespace `GHGA_DATASET_TYPE`, that the study and data access policy linked to in `study_id` and `dap_id` exists, and that all specified files exist in EM and are specified only once.

#### Term

The Term entity holds lookup information for terms coming from controlled vocabularies and ontologies used in the administrative metadata. This also includes controlled vocabularies that are only used by GHGA in the administrative metadata, like study or dataset types. This could later be expanded into a full-fledged service that allows looking up any term used in the metadata, including the EM. Such a service will be particularly useful for the frontend.

Attributes:

- `id: str` - a globally unique, canonical identifier (primary identifier)
- `label: str` - human-readable name for the term
- `description: str | None` - optional definition text or help text
- `space: Namespace` - the namespace for the term
- `url: Url | None` - optional URL for the term
- `deprecated: bool` - true if this term shouldn't be used in new studies anymore

The terms are grouped into different namespaces corresponding to one of the Namespace enum values defined in this document.

For the internally managed controlled vocabularies, the id must be a UUID in string format. For standardized controlled vocabularies and ontologies, the id must start with the officially registered upper namespace prefix (like "DUO") followed by a colon.

The class should have a validator that guarantees that the format of the `id` field corresponds to the specified `space`.

#### Accession

The Accession entity stores all existing primary accessions. See also the sections on the Accession Registry below.

Attributes:

 - `id: str` - the primary accession number (PID)
 - `type: Entity` - the entity referenced by this accession
 - `created: Date` - when the accession was created
 - `superseded_by: str | None` - if deprecated, a new primary accession

Note: When we start introducing versioned accession numbers, we might later split this into two entities, one for holding the base accession numbers, and another one for holding the versioned ones. For faster lookup, we are storing them in separate collections.

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
- `maps: dict[EmEntity, dict[str, str]]` - entity specific accession maps

The `maps` attribute holds, for every entity in the experimental metadata, a mapping from the identifier used in the original submission to the primary accession generated by the Study Registry Service and stored in the Accession entity.

Note: The original ID in the EM has been historically stored in the "alias" attribute of the metadata model. We may want to change that and rename it to "id" in the new study-centric model since it's actually used as an identifier that should be unique for a specific submission and experimental metadata entity.

This entity has been separated from the Study entity because the accession maps can be large (might require GridFS) and the original accessions are rarely needed after transformation.

### Enums

#### AltAccessionType

The AltAccessionType enum holds the different kinds of alternative accessions:

- `EGA` - EGA accession
- `FILE_ID` - internal file id
- `GHGA_LEGACY` - legacy GHGA accession (after switching to new PID schema)

#### Namespace

The Namespace enum lists all supported namespaces for terms:

- `GHGA_STUDY_TYPE`: a study type supported by GHGA (e.g. `RARE_DISEASE`)
- `GHGA_DATASET_TYPE`: a dataset type supported by GHGA (e.g. `WHOLE_GENOME_SEQUENCING`)
- `GHGA_DAC`: a DataAccessCommittee as described above
- `GHGA_DAP`: a DataAccessPolicy as described above
- `GA4GH_DUO`: the GA4GH Data Use Ontology

#### StudyStatus

The StudyStatus enum describes all possible states of a Study:

- `PENDING`
- `FROZEN`
- `APPROVED`
- `PERSISTED`

In the initial implementation we will only use the status values `PENDING` and `PERSISTED`.

#### Entity

The Entity enum lists all supported entities archived by GHGA. It should be the union of all entities used in administrative metadata (AmEntity) and all data types used in experimental metadata (EmEntity).

Values of AmEntity:

- `DataAccessCommittee`
- `DataAccessPolicy`
- `Dataset`
- `Publication`
- `Study`

Values of EmEntity:

- `Analysis`
- `AnalysisMethod`
- `AnalysisMethodSupportingFile`
- `Experiment`
- `ExperimentMethod`
- `ExperimentMethodSupportingFile`
- `Individual`
- `IndividualSupportingFile`
- `ProcessDataFile`
- `ResearchDataFile`
- `Sample`

### Core functionality

The Study Registry Service can be accessed by a REST API in order to submit and query DAM, PAM, EM and lookup values (the Term entity). It also allows updating DAM and lookup values. The REST API is described further below.

For now, the accession numbers should be created in the same way as before, assuming that the existing accessions have been imported into the database already, so that no duplicate accessions will be created.

A newly submitted study shall always be created with the status `PENDING`. When the status is updated to `PERSISTED`, the service should validate the submission as detailed below. If the validation fails, the status update should be rejected and the submission should keep the status `PENDING`.

After the submission has been successfully validated and moved to the status `PERSISTED`, the service will create new accession numbers for all resources contained in the EM and store these in the database as Accession, AltAccession, and EmAccessionMap entities.

The service will then create an AnnotatedEMPack and publish it as an event, as described further below.

TODO: mention file id mapping functionality as well

### Validation

On ingress, submissions must be validated and rejected if there are any validation errors. Validation happens on several levels:

- ingested DAM and PAM (e.g., study or dataset) are validated as DTOs using Pydantic
- the submission must be complete (particularly, it must have a Publication and ExperimentalMetadata)
- all terms referenced in the ingested DAM and PAM must exist and not be deprecated
- ingested EM is validated by a separate EM validation service

In this epic, we assume that the EM validation service exists and provides a simple REST API for validating EM against a given EMIM (currently we only have one).

### Accession registry

The first implementation of the Study Registry Service shall also contain an initial implementation of an accession registry. Later, this can be outsourced to a separate, dedicated service.

Resources archived in GHGA are immutable and should get accession numbers that are globally unique, persistent, and long-term resolvable. To emphasize these qualities, we also call these accession numbers persistent identifiers (PIDs). A PID must always resolve to the exact same state of a resource.

Particularly, the following entities get PIDs:

- studies
- all research data files
- datasets (sets of research data files)
- entities that are part of the experimental metadata

Currently, the accession numbers used by GHGA have a format that starts with the uppercase letters "GHGA", followed by another uppercase letter indicating the resource type (e.g. "S" for study, "D" for Dataset, "U" for publication), followed by a random 14-digit number. The prefixes and number of digits are defined in the [metadata configuration](https://github.com/ghga-de/metadata-config/blob/main/configuration/metadata_config.yaml).

We are referring to these accession numbers as "legacy accessions", since we plan to replace them with a new PID schema that should reflect the study-centric metadata concept better and should also contain the year and version number.

We will continue to use the legacy accession numbers until the new PID schema is fully defined, which is not covered in this epic.

In addition to GHGA accession numbers, which we also refer to as our primary (canonical) accessions, the accession registry should also keep track of alternative accessions (alt accessions) and resolve them to the corresponding primary accessions. For instance, the accession number under which a resource is archived in EGA should be stored as alternative accession. After switching to the new PID schema, the legacy accession numbers should also be stored as alternative accessions.

The accession store shall also contain a mapping from file accessions to internal file IDs, using the same mechanism of alt accession references, with internal file IDs being a special type of alt accession.

### Authorization

TODO

## User Journeys (optional)

This epic covers the following user journeys:

\<Images and descriptions of user journeys go here. Images are deposited in the `./images` sub-directory.\>

![\<Example Image\>](./images/data_upload.jpg)

TODO (see AC: Metadata Services)

## API Definitions

### RESTful/Synchronous

The service provides an API that is accessible via the Data Portal exclusively by Data Stewards. Through this API, they can modify resources within the immutability constraints outlined in this document.

In addition, the service may expose a public, anonymous read-only API that provides information about DataAccessCommittees and DataAccessPolicies.

Other endpoints:
- The REST API also needs an endpoint to look up terms
- An endpoint to fetch the file names and file accessions for a study (accessible if study is accessible)
- An endpoint to post file accession to file id mappings

TODO

\<List all endpoints in the following way:\>

- GET /samples/{sample_id}: Get metadata on a sample
- POST /submissions: Post a submission
- ...

### Payload Schemas for Events

The service publishes events that communicate the state of its entities.

In particular, it publishes Annotated Experimental Metadata (AEM). This consists of the original, unmodified experimental metadata published along with additional annotations. These annotations contain all other study-related information and can be consumed by the experimental metadata transformation service and integrated into downstream transformed AEM.

The published AEM events shall have the following schema:

- `metadata: JSONObject` - the actual experimental metadata from ExperimentalMetadata
- `accessions: JSONObject` - the corresponding maps from EmAccessionMap
- `study: Study` - the corresponding study with nested publication
- `datasets: list[Dataset]` - the associated datasets with nested DAP and DAC)

## Human Resource/Time Estimation

Number of sprints required: \<Insert a number.\>

Number of developers required: \<Insert a number.\>
