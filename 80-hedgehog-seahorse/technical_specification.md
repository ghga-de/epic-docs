# GHGA Connector refactoring/rewrite and upload path implementation (Hedgehog Seahorse)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://ghga.pages.hzdr.de/internal.ghga.de/main/sops/development/epic_planning/).


## Scope
### Outline:
This epic has two different, but related goals:
- Implementing functionality for the current upload concept, discarding the pre-release placeholder implementation existing within the GHGA Connector
- Revisiting and rewriting existing functionality in the Connector

The first goal is rather straightforward, as the currently existing code for the upload within the connector predates the launch of GHGA and concepts for the upload changed in the meantime, so the business logic also has to change.

For the second goal, the Connector has always suffered from issues due to starting out with a small feature set and code written mostly in an imperative manner, tacking on features as needed.
While multiple rounds of refactoring managed to deal with the most egregious issues, both the vertical and horizontal separation of concerns is only partially realized and will likely make extending the functionality cumbersome in the long term.

Testing of the Connector functionality also follows a different approach than our services, which adds additional complexity.
Making testing easier would make testing functionality across different OSes easier, if we want to support CLI tooling across platforms.

In short, a (partial) rewrite of the existing functionality might be more beneficial long term than another round of refactoring, contingent on the fact of actually enforcing separation between different layers of the codebase, both horizontally and vertically.

Details on which parts exactly should be changed are described below.


### Included/Required:

- Upgrading the codebase to only support Python >=3.10, as 3.9 will be deprecated soon
- Structure code more clearly into shared functionality and path/command specific code
- Simplify code by using singletons instead of passing an invariant instance through the call stack to simplify function signatures/boundaries, similar to how loggers are instantiated. 

#### For the upload path

- 

#### For the download path
- Rewriting the async task based file download code to also support uploads reusing the same general mechanism. Currently the code is ignorant about the actual function passed in for execution, but there's coupling to the download around the `TaskHandler` that would make this cumbersome to reuse
- Decoupling different layers of the download process that are far too tightly intertwined right now, i.e. disentangle WorkPackageAccessor, FileStager and Downloader.
Those should be injectable components and should not form a more or less implicit hierarchy
- Sort out the mixed responsibilities between the class based Downloader and the more or less standalone functions in the `api_calls` module. This goes hand in hand with making the different components properly injectable, as the current implementation has issues here due to how communicating with the WPS is implemented 
- Refactor/rewrite the WorkPackageAccesor to support tokens for the upload path
- The initial caching implementation was based on a bit of a misunderstanding after reading the RFC and needs to be revisited to improve upon the existing hotfix.
The current implementation uses the appropriate caching headers, but additionally relies on a fixed TTL for cache entries.
A local bound on the cache lifetime will likely still be needed, but should be more dynamic. This part might need some more investigation
- Investigate if the `RetryHandler` could be implemented in an easier way

### Optional:

- Requiring a lower bound of Python >=3.11, so the code can take advantage of [task groups](https://docs.python.org/3/library/asyncio-task.html#task-groups) for the actual async transfer code, which would make it easier to reason about what's happening and make setup/teardown a bit easier to handle
- Find a better way to support a range of Python versions long term.
The repository template based system currently tends to needlessly break stuff sometimes, making it necessary to work around it every once in a while and opting out of the update mechanism for the affected files

### Not included:

## User Journeys:

This epic covers the following user journeys:

## API Definitions:

### RESTful/Synchronous:

### Payload Schemas for Events:

## Additional Implementation Details:

(<Add diagram for current vs future class structure>)

## Human Resource/Time Estimation:

Number of sprints required: 2

Number of developers required: 2
