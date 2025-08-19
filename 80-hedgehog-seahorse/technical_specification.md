# GHGA Connector refactoring/rewrite and upload path implementation (Hedgehog Seahorse)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://ghga.pages.hzdr.de/internal.ghga.de/main/sops/development/epic_planning/).


## Scope
### Outline:
This epic has two different, but related goals:
- Implementing functionality for the currently envisioned upload concept, discarding the pre-release placeholder implementation existing within the GHGA Connector
- Revisiting and rewriting existing functionality in the Connector

The first goal is rather straightforward, as the currently existing code for the upload within the connector predates the launch of GHGA and concepts for the upload changed in the meantime, so the business logic also has to change.

For the second goal, the Connector has always suffered from issues due to starting out with a small feature set and code written mostly in an imperative manner, tacking on features as needed.
While multiple rounds of refactoring managed to deal with the most egregious issues, both the vertical and horizontal separation of concerns is only partially realized and will likely make extending the functionality cumbersome in the long term.

Testing of the Connector functionality also follows a different approach than our services, which adds additional complexity.
Making testing easier would make testing functionality across different OSes easier, if we want to support CLI tooling across platforms.

In short, a (partial) rewrite of the existing functionality might be more beneficial long term than another round of refactoring, contingent on the fact of actually enforcing separation between different layers of the codebase, both horizontally and vertically.

Details on which parts exactly would benefit are described below.


### Included/Required:

### Optional:

### Not included:

## User Journeys:

This epic covers the following user journeys:

## API Definitions:

### RESTful/Synchronous:

### Payload Schemas for Events:

## Additional Implementation Details:


## Human Resource/Time Estimation:

Number of sprints required: 2

Number of developers required: 1
