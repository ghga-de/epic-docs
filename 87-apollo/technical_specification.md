# Initial Version of a Study Registry Service (Apollo)

**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://ghga.pages.hzdr.de/internal.ghga.de/main/sops/development/epic_planning/).

## Scope

### Outline

The Study Registry Service (SRS) is the core GHGA registry service responsible for ingesting and archiving metadata from data submitters.

Its function is currently implemented by the [GHGA Data Steward Kit](https://github.com/ghga-de/ghga-datasteward-kit) in conjunction with the submission store and accession store managed via the [metldata](https://github.com/ghga-de/metldata) library.

More precisely, the service replaces the Submission Registry and the Accession Registry managed via metldata and stored on the file system of a virtual machine operated by a GHGA central data steward. The submissions are now stored in a database owned by the SRS.

The second crucial change made with the introduction of the SRS is the separation of Experimental Metadata from Persistent Administrative Metadata, Dynamic Administrative Metadata and Datasets which were bundled in an all-in-one [GHGA metadata schema](https://github.com/ghga-de/ghga-metadata-schema) until now, and moving from the former dataset-centric metadata concept to the new one that is study-centric.

The Experimental Metadata should be managed in a generic way, without semantic interpretation of its content besides generating an accession number for each entity.

As a reminder, Experimental Metadata (EM) is data describing how the research data (that we store in “files”) was generated. Persistent Administrative Metadata (PAM) is non-experimental study metadata (e.g. authors, description), and Dynamic Administrative Metadata (DAM) is business metadata (e.g. authorized users, DAC contacts and policies). As the names indicate, PAM underlies the same archival immutability as the EM, while DAM is auditable.

An Experimental Metadata Ingress Model (EMIM) shall be used for validation. EMIMs shall be stored as schemapacks, and the EM shall be stored as datapacks. Both structures are provided by the [schemapack](https://github.com/ghga-de/schemapack) library.

All resources archived in GHGA that are subject to archival immutability can be retrieved by an accession number, a persistent identifier (PID) that always resolves to the exact same state of a resource.

TODO: Mention that

### Included/Required

- REST API as detailed below
- Event publisher as detailed below

### Optional

TODO

### Not included

- data life cycle (change requests to existing studies, producing a modified copy)
- multiple EMIMs
- transformation
- migration

TODO

## Details

### Managed Entities

#### Study

In the new study-centric metadata concept, the Study serves as the central container for Dataset, Publication, and alternative accessions for each experimental metadata entity. It has minimal properties but plays a critical role as a container entity, as its lifecycle is linked to many other entities. It is immutable and receives a permanent accession number upon creation.

Properties:

- `id: str` - the PID of the study (primary key)
- `title: str` - comprehensive title for the Study
- `description: str` - detailed description (abstract) describing the goals of the Study
- `types: list[str]` - the type(s) of this Study (as list of their Term ids)
- `affiliations: list[str]` - the affiliation(s) associated with this Study
- `status: StudyStatus` - the current status of the study (see below)
- `users: list[UUID] | None` - user(s) who can access the study (None means publicly accessible)

This class should have a validator that checks that the specified types are not empty and exist as Term with source `GHGA_STUDY_TYPE`.

#### Publication

The Publication is the citation reference for the study. It is immutable and receives a permanent accession number upon creation.

Properties:

- `id: str` - the PID of the Publication
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

Properties: all properties of Term (see below), plus:

- `email: str` - the contact email address of the DAC
- `institute: str` - the institute the DAC belongs to

A human-readable identifier would be useful for management and assignment purposes, but no citable accession number is required.

#### DataAccessPolicy

The DataAccessPolicy entity describes a policy for data access (DAP). It is managed independently of a study, is mutable, and belongs to exactly one DataAccessCommittee.

Properties: all properties of Term (see below), plus:

- `text: str` - the complete text for the Data Access Policy
- `duo_permission: str` - the DUO id of the corresponding Data Use Permission
- `duo_modifiers: list[str]` - the DUO id(s) of the corresponding Data Use Modifiers
- `dac_id: str` - the DAC linked to this DAP

This class should have a validator that checks that the specified permissions and modifiers exist as Term with source `GA4GH_DUO` and that the DAC linked to in `dac_id` also exists.

#### Dataset

The Dataset entity describes a set of files and represents the smallest unit for which a data access request can be formulated. The attributes Title, Description, Types, and Files are immutable, and the entity itself cannot be deleted. However, new datasets can be created after study creation and assigned to a study without violating its immutability. An immutable accession number is assigned upon creation. Every dataset belongs to exactly one Study. The DataAccessPolicy assignment is mutable.

Properties:

- `id: str` - the PID of the Dataset
- `title: str` - comprehensive title for the Dataset
- `description: str` - detailed description summarizing this Dataset
- `types: list[str]` - the type(s) of this Dataset (as list of their term ids)
- `study_id: str` - the PID of the study associated with this Dataset

This class should have a validator that checks that the specified types are not empty, that they exist as Term with source `GHGA_DATASET_TYPE`, and that the study linked to in `study_id` also exists.

#### Term

A term entity holds lookup information for terms coming from controlled vocabularies and ontologies used in the administrative metadata. This could later be expanded into a full-fledged service that allows looking up any term used in the metadata, including the EM. Such a service will be particularly useful for the frontend.

Properties:

- `id: str` - unique, canonical identifier
- `label: str` - human-readable name for the term
- `description: str | None` - optional definition text or help text
- `url: Url | None` - optional URL for the term
- `source: TermSource` - the source for the term (CV/ontology)

The class should have a validator that guarantees that the format of the `id` field corresponds to the specified `source`.

Note: This should be later expanded to also keep track of version numbers or release dates of the term sources.

#### Accession

TODO

Describe:

- PIDs and their format
- accession store
- legacy accessions
- alternative accessions (particularly EGA)
- file accession to file id mapping

### Enums

#### StudyStatus

The StudyStatus enum describes all possible states of a Study:

- `PENDING`
- `FROZEN`
- `APPROVED`
- `PERSISTED`

#### TermSource

The TermSource enum lists all supported term sources:

- `GHGA_STUDY_TYPE`: a study type supported by GHGA (e.g. `RARE_DISEASE`)
- `GHGA_DATASET_TYPE`: a dataset type supported by GHGA (e.g. `WHOLE_GENOME_SEQUENCING`)
- `GHGA_DAC`: a DataAccessCommittee as described above
- `GHGA_DAP`: a DataAccessPolicy as described above
- `GA4GH_DUO`: the GA4GH Data Use Ontology

## Core functionality

TODO (derive from REST API)

## User Journeys (optional)

This epic covers the following user journeys:

\<Images and descriptions of user journeys go here. Images are deposited in the `./images` sub-directory.\>

![\<Example Image\>](./images/data_upload.jpg)

TODO

## Authorization

TODO

## API Definitions

TODO

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

The service publishes events that communicate the state of its entities. In particular, it publishes Annotated Experimental Metadata (AEM). This consists of the original, unmodified experimental metadata published side by side with additional annotations. These annotations contain all other study-related information and can be consumed by the experimental metadata transformation service and integrated into downstream transformed AEM, including:

- The study entity itself
- The generated accession map assigning an accession number to each experimental metadata entity
- The datasets
- Potentially other related information

These events are re-published whenever any underlying information changes.

TODO

\<Describe the schema of event either using example events or json schemas\>

## Additional Implementation Details

- \<List further implementation details here. (Anything that might be relevant for defining and executing tasks.)>

TODO

## Human Resource/Time Estimation

Number of sprints required: \<Insert a number.\>

Number of developers required: \<Insert a number.\>
