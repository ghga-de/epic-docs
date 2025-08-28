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

#### For the upload path

The existing functionality for the upload is outdated and needs to be replaced with logic for the new concept described in [Lynx Boreal](../76-lynx-boreal/technical_specification.md).

This means

- Refactor/rewrite the WorkPackageAccessor to support tokens for the upload path.
- Rewriting the async task based file download code to also support uploads reusing the same general mechanism. Currently the code is ignorant about the actual function passed in for execution, but there's some coupling to the download around the `TaskHandler` that would make this cumbersome to reuse.
- The inplace encryption code present in the current, unused upload code path is a variant of an earlier implementation in the DS-Kit and can be recycled for the current vision of the upload process.
- Enable upload functionality analogous to what is implemented for the download, i.e. parallelized part upload using work package/work order tokens.
This should reuse the TaskHandler, if possible.
- Calls to UCS to manage `FileUpload`s and retrieve presigned part upload URLs

#### For the download path

- Decoupling different layers of the download process that are far too tightly intertwined right now, i.e. disentangle WorkPackageAccessor, FileStager and Downloader.
Those should be injectable components and should not form a more or less implicit hierarchy.
- Sort out the mixed responsibilities between the class based Downloader and the more or less standalone functions in the `api_calls` module. This goes hand in hand with making the different components properly injectable, as the current implementation has issues here due to how communicating with the WPS is implemented.
- The initial caching implementation was based on a bit of a misunderstanding after reading the RFC and needs to be revisited to improve upon the existing hotfix.
The current implementation uses the appropriate caching headers, but additionally relies on a fixed TTL for cache entries.
A local bound on the cache lifetime will likely still be needed, but should be more dynamic. This part might need some more investigation.
- Investigate if the `RetryHandler` could be implemented in an easier way.

### Optional:

- Requiring a lower bound of Python >=3.11, so the code can take advantage of [task groups](https://docs.python.org/3/library/asyncio-task.html#task-groups) for the actual async transfer code, which would make it easier to reason about what's happening and make setup/teardown a bit easier to handle
- Find a better way to support a range of Python versions long term.
The repository template based system curren-tly tends to needlessly break stuff sometimes, making it necessary to work around it every once in a while and opting out of the update mechanism for the affected files

## User Journeys:

This epic covers the following user journeys:
<TODO upload journey description, waiting on the final details of lynx boreal>

## Additional Implementation Details:

### Download

- Most of the issues making this a hard codebase to work with can be traced down to passing an instance of the `WorkPackageAccessor`, `httpx.AsyncClient` and `MessageDisplay` down the call stack and inconsistencies in who is responsible for calling functions on those instance objects.
- Use dependency injection for the initial configuration that does not need remote calls, i.e. init and/or inject MessageDisplay and the http client.
- As the Downloader is dependent on the current file_id, the builder pattern could be used to inject a base instance that is then fully specialized by providing the ID
- Download handling should have a central point that gets all other functionality injected and is responsible for managing and handling.
- The CLI layer should be thin and delegate more complex functionality to the core.

In the current implementation, the different concerns that are involved in the download process are entangled in the following way:

#### General issues

- URLs with a hardcoded part are defined inplace across different modules
- Using the RetryHandler causes code duplication in the calling code when dealing with exception sources

#### `cli.py`:

- Calls the WellKnownValue service
- Calls the WorkPackage service
- Contains dataclass definitions for DownloadParamters, UploadParameters, MessageDisplay
- Does some precondition checks for functionality further down in the call hieararchy, but those functions do additional precondition checks

#### `batch_processing.py`

- Needs httpx.AsyncClient and WorkPackageAccessor
- Contains multiple ABCs for inplace use
- Also accesses the WorkPackage service
- Calls auth service
- Waits on user input if file IDS are not present on the remote

#### `downloader.py`

- Needs httpx.AsyncClient and WorkPackageAccessor
- Calls functionality from `donwloading/api_calls.py`, passing either the client or WorkPackageAccessor along
- Contains TaskHandler class, which is not download specific

A better approach to properly separate the concerns could be achieved this way:

The trinity of `Downloader`, `WorkPackageAccessor` and `FileStager` are replaced by a central class that exposes the necessary API calls and delegates the responsibilities, flattening the call stack hierarchy.
The idea is to inject not yet finalized instances of classes that deal with specialized parts of the download lifecycle and inject the functionality that deals with API calls builder pattern style.

## Human Resource/Time Estimation:

Number of sprints required: 2

Number of developers required: 2
