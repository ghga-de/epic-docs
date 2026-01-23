# Basic Research Data Upload Box Frontend (Archaeopteryx)

**Epic Type:** Implementation Epic

## Scope

### Outline

In this epic we implement the minimum necessary frontend functionality to make Research Data Upload Boxes fully usable with the current version of the backend and to map uploaded research data files to their corresponding entries in the submitted experimental metadata.

This will be an interim solution that still needs the data steward to help with management of the research data upload boxes and mapping the files, and this solution is not yet integrated with the planned study repository service.

Research Data Upload Boxes (RDUBs) have been specified and were implemented in the [Lynx Boreal](../76-lynx-boreal/technical_specification.md) and [Sarcastic Fringehead](../84-sarcastic-fringehead/technical_specification.md) epics on the backend side already.

### Included/Required

The implementation contains three parts:

#### 1. Research Data Upload Manager

A Research Data Upload Manager shall be integrated into the frontend. Is shall be accessible by data stewards via the Admin menu and allow them to:

- list available RDUBs
- create a new RDUB for a specific user
- close, lock and unlock RDUBs

#### 2. Research Data Upload Token Creation

User should be able to see all RDUBs that have been opened for them on a new section on their account pages. For all RDUBs in the "open" state, the interface will show a button leading to page that allows users to create an upload token for that RDUB. This should function similarly to the existing functionality for creating download tokens.

#### 3. Research Data File to Experimental Metadata File Mapping

The Research Data Upload Manager should provide a sub-page that helps establishing the missing link between the uploaded files in the Research Data Upload Boxes (RDUBs) and the submitted Experimental Metadata (EM):

- RDUBs identify files via an internal address, but they also provide a unique alias
- EM identifies files via accession numbers, but files also have a field for the name of the file and an alias

The alias stored in RDUB files is the filename of the file as it was uploaded via the GHGA connector. The file name submitted with the EM usually should correspond to this filename, but that is not guaranteed. Sometimes the alias is used instead, or the filenames submitted in the EM could be slightly different. It is also not guaranteed that the number of files matches - there could be more files listed in the EM than have been uploaded, or files can have been uploaded that have no corresponding entry in the EM. The frontend should handle these cases gracefully, and support the data steward in creating a mapping between the uploaded files and the EM entries with as much automation as possible while allowing manual overrides.

All RDUBs in the locked state should have a button that opens this mapping tool. Further details of the mapping UI are described below.

### Optional

This epic should only provide the necessary functionality so that it can be shipped as soon as possible.

### Not included

The epic does not implement a full self-service solution for managing the RDUBs by the users who are uploading the files and no integration to the planned metadata services. It also does not implement any additional backend functionality.

## User Journeys

This epic covers the following user journeys:

1. A data steward creates a new Research Data Upload Box and authorizes a user to upload research data files to that box, using the Research Data Upload Manager in the GHGA data portal.
2. The user sees the newly created box on the user account page in the GHGA data portal and hits the button to create an upload token for the box.
3. The user confirms that an upload token shall be created for that box and after entering their public Crypt4GH key, will be shown the corresponding upload token once.
4. The user copies the token and continues the upload in the GHGA connector using the generated upload token (not part of this epic).
5. After uploading all research data files, the user locks the upload box on the user account page in the GHGA data portal and informs the data steward that the upload has been completed.
6. The data steward sees that the box has been locked (or otherwise locks the box).
7. The data steward verifies that the experimental metadata for the corresponding study has been submitted.
8. Now the data steward can use the Research Data Upload Manager in the GHGA data portal to establish a mapping between the names of the uploaded files in the box and the file names in the experimental metadata.
9. After submitting the mapping, the Research Data Upload Box is automatically closed.

## UI Wireframes

### Research Data Upload Manager

TBD

### RDUB section on User Account Page

TBD

### Research Data Mapping Tool

TBD


## API Endpoints used by the frontend

TBD

## Additional Implementation Details

TBD

It must be verified that the RDUB is in the same state when the mapping is submitted as it was when the data has been loaded by the mapping tool. For that purpose, the backend should provide some kind of version number that increases on every change of the RDUB (including changes in any of the associated files). This version number must be submitted together with the mapping and the backend should reject a submission when the current version of the RDUB is different.

The mapping can be stored in the browser of the data steward using local storage, so that a mapping session can easily be resumed. However, this is only important if the data steward needs to make many manual mappings, which should usually not happen.

## Human Resource/Time Estimation

Number of sprints required: 1

Number of developers required: 1
