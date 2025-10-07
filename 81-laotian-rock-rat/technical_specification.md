# Client Retry Logic Refactoring for Ratelimiting (Laotian Rock Rat)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://ghga.pages.hzdr.de/internal.ghga.de/main/sops/development/epic_planning/).

## Scope
### Outline:

This epic aims to improve upon the current implementation of retry logic for HTTP requests in CLI tools and to provide a solution that can be reused not only across CLI tools, but where necessary in the codebase.

The current implementation is based on [tenacity](https://tenacity.readthedocs.io/en/latest/) and tacked on top of existing functionality, provided as a wrapper around httpx.AsyncClient calls.
As the retry parameters are configured dynamically, the more ergonomic, decorator based retry handling cannot be easily used.
Subsequently, an (Async)Retrying object configured at startup is (rather awkwardly) passed along the call stack.

To move the responsibility away from the caller and make the retry functionality trivially reusable, a custom Transport class shall be implemented, dealing with both general retry logic and (potentially stateful) rate limited requests.

### Included/Required:

### Custom (Async)HTTPTransport
A transport is an object that resides at the interface of higher level abstractions and the lower level code dealing with the actaul details of transporting bytes over the network.

It can easily be plugged into a client, manages the underlying connection (pool), delegates requests and performs basic error handling (see 
[transports in httpx](https://github.com/encode/httpx/blob/master/httpx/_transports/default.py)).
As such, it seems a fitting base class for implementing a custom variant with more involved retry logic, adhering to rate limits.

Moving the existing tenacity based retry logic should be straight forward, converting the (Async)Retrying to a property of the transport, wrapping existing transport code where possible.
Adhering to rate limits might need some more involved logic around connection pooling to keep track of the current request rate per connection or across the entire connection pool, depending on further underlying assumptions and the strategy chosen.

## Additional Implementation Details:

### Tenacity Retry Logic

Implementation could be realized by subclassing the `httpx.(Async)HTTPTransport` or its base class, overriding methods to only deal with retry logic and wrap and delegate to the corresponding parent methods.
This would mean much of the existing code could be reused and only the error handling would need some additional care to be appropriate for the abstraction level it's being moved to.

### Responding to Rate Limiting 

Connection pools are handled by httpcore, one level below httpx, and introducing rate limiting on that level would likely get quite messy and should be avoided.
Instead, the logic should not change how connection pools work, but use them as best as possible.

There are some requirements and possible (existing) pitfalls to consider during implementation:
1) It is assumed that rate limiting happens on a per connection level
2) Each connection responds to its own 429 responses by adjusting its request frequency accordingly
3) As requests are effectively fired in batches (for parallel/concurrent transfer operations), a small amount of jitter should be introduced to space them out a bit more evenly, so they don't hit the remote endpoint at the same time
4) The logic around jitter could be reused for the 429 response if the `RetryAfter` header is treated as a baseline for all subsequent requests and not just the immediately following one, so something like `asyncio.sleep(max(self.jitter, self.retry_after))` could be applied per connection.
5) The potential issue in this approcach is throttling the request frequency too much and never recovering to a more appropriate rate. 
To combat this, the connection could forget about the `retry_after` after a specified amount of requests and go back to just using the jitter.
6) There's no easily apparant way to track separate connections/influence handout on the httpx/transport level. 
To guarantee that connections are reused and the limit is respected on a per connection level, the transport would need to keep track of n 1 sized connection pools, instead of one n sized connection pool. 
This idea has to be tested in practice first, to see if there are inherent shortcomings compared to using a normal transport backed by one connection pool

### Discrepancies

The goals of the last two subsections are not fully aligned, yet.
Subclassing the `(Async)HTTPTransport` class and managing multiple connection pools seem mutually exclusive, so some code duplication from the `(Async)HTTPTransport`
class and deriving from its base class instead might be required to make things work.

### Where to implement

An initial implementation could be tested in the recent S3 part size benchmarking repo, while the completed implementation should be placed in a repository from which it can easily be imported into different parts in the code base, so ghga-service-commons is most likely the appropriate place.
Then, as a first step and use case, this implemetation could be used both in the datasteward kit and the connector.

## Human Resource/Time Estimation:

Number of sprints required: 1

Number of developers required: 1
