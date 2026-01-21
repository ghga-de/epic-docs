# Basic Research Data Upload Box Frontend (Archaeopteryx)

**Epic Type:** Implementation Epic

## Scope

### Outline

In this epic we implement the minimum necessary frontend functionality to make Research Data Upload Boxes already usable with the current version of the backend and to map uploaded research data files to their corresponding entries in the submitted experimental metadata.

Research Data Upload Boxes (RDUBs) have been specified and were implemented in the [Lynx Boreal](../76-lynx-boreal/technical_specification.md) and [Sarcastic Fringehead](../84-sarcastic-fringehead/technical_specification.md) epics on the backend side already.

### Included/Required

The implementation contains three parts:

#### 1. Research Data Upload Manager

This is a frontend that can be accessed by data stewards via the Admin menu and allows them to:

- list available RDUBs
- create a new RDUB for a specific user
- close, lock and unlock RDUBs

#### 2. Research Data Upload Token Creation

This is a frontend that can be access by users via their account pages. This page will get a new section that lists all RDUBs associated to the user. For all RDUBs in the "open" state, the interface will show a button leading to page that allows users to create an upload token for that RDUB. This should work similar to the existing functionality for creating download tokens.

### 3. Research Data File to Experimental Metadata File Mapping

This frontend provides the missing link between the uploaded files in the Research Data Upload Boxes (RDUBs) and the submitted Experimental Metadata (EM):

- RDUBs identify files via an internal address, but they also provide a unique alias
- EM identifies files via accession numbers, but files also have a field for the name of the file and an alias

The alias stored in RDUB files is the filename of the file when it was uploaded via the GHGA connector. The file name submitted with the EM usually should correspond to this filename, but that is not guaranteed. Sometimes the alias is used instead, or the filenames submitted in the EM could be different. It is also not guaranteed that the number of files matches - there could be more files listed in the EM than have been uploaded, or files can have been uploaded that have no corresponding entry in the EM. The frontend should handle these cases gracefully, and support the user in creating a mapping between the uploaded files and the EM entries with as much automation as possible while allowing manual overrides.

### Optional

\<List any optional features that may or may not be realized as part of this epic.\>

### Not included

\<List features that will not be addressed as part of this epic.\>

## User Journeys (optional)

This epic covers the following user journeys:

\<Images and descriptions of user journeys go here. Images are deposited in the `./image` sub-directory.\>

![\<Example Image\>](./images/data_upload.jpg)

## API Definitions

### RESTful/Synchronous

\<List all endpoints in the following way:\>

- GET /samples/{sample_id}: Get metadata on a sample
- POST /submissions: Post a submission
- ...

### Payload Schemas for Events

\<Describe the schema of event either using example events or json schemas\>

## Additional Implementation Details

- \<List further implementation details here. (Anything that might be relevant for defining and executing tasks.)>

## Human Resource/Time Estimation

Number of sprints required: \<Insert a number.\>

Number of developers required: \<Insert a number.\>
