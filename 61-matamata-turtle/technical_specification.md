# DLQ Service (Matamata Turtle)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).

## Scope
### Outline:
This epic is concerned with the implementation of a basic dead letter queue service,
which will provide the means to assess, process, and retry Kafka events that resulted
in errors during their initial processing. This service will make use of the
[Kafka DLQ](https://github.com/ghga-de/hexkit/pull/123) additions to `hexkit`.

### Included/Required:
- Implementation of the DLQ service (including its test suite)
- `hexkit`: Add exception information to DLQ event headers
- `hexkit`: Add preview functionality to `KafkaDLQSubscriber`
- `hexkit`: Add preview dry-run functionality to `KafkaDLQSubscriber`

### Optional:
- Add event ID header in `hexkit`

### Not included:
- A web-based user interface

## API Definitions:

### RESTful/Synchronous:

DLQ topics are referred to by abbreviated name of the corresponding service, not the
actual topic names. E.g. to refer to the Work Package Service's DLQ topic, presumably
"wps_dlq", the endpoint `service` parameter would be merely "wps".
- `GET /{service}`
  - *Returns the next events in the topic, not exceeding the configured maximum.*
    *This is a preview of the events and does not impact event ordering within the DLQ.*
    *Like the dry-run endpoint, this endpoint is idempotent.*
  - Auth Header: internal token
  - Response Body: array of event objects
  - Response Status:
    - `200 OK`: The preview was successful
    - `401 Unauthorized`: auth error (not authenticated)
- `GET /rpc/dry-run/{service}`
  - *Returns the result of processing the next event in the specified topic. Nothing*
    *is published and offsets are not affected. Calling the same dry-run path multiple*
    *times will return the same result -- it is idempotent.*
  - Auth Header: internal token
  - Response Body: the event that would be published, empty if event would be discarded.
  - Response Status:
    - `200 OK`: The event dry-run was successful
    - `401 Unauthorized`: auth error (not authenticated)
- `POST /{service}`
  - *Processes the next event in the topic, optionally publishing the supplied event*
  - Auth Header: internal token
  - Request Body: empty
  - Response Body: empty or JSON representation of corrected event
    - Need to consider whether checks should be added for fields like correlation ID.
  - Response Status:
    - `204 No Content`: The event has been processed
    - `401 Unauthorized`: auth error (not authenticated)
- `DELETE /{service}`
  - *Directly discards the next event in the topic*
  - Auth Header: internal token
  - Response Body: empty
  - Response Status:
    - `204 No Content`: The event has been discarded
    - `401 Unauthorized`: auth error (not authenticated)

Each endpoint will return a `404` status code if the `service` parameter does not match
an existing service.

All endpoints should require an internal auth token.

### Payload Schemas for Events:

**Outbound from DLQ Service**:  
The outbound events published to retry topics will look exactly like other Kafka events
published or consumed by the GHGA microservices, except they will feature the
`original_topic` header.

**Inbound to DLQ Service**:  
The inbound events in the DLQ topics will feature the `original_topic` plus additional
headers to communicate exception information like the message and class name. That will
require a small modification to `hexkit`.

**Previewed Events**:  
Events returned by the preview endpoint will match the format defined by the
[`ExtractedEventInfo`](https://github.com/ghga-de/hexkit/blob/e3cad2d57212d1ba897e620501d5d2b120c70338/src/hexkit/providers/akafka/provider/eventsub.py#L63) dataclass.

**Other**:  
If we decide to include the event ID header, then that will be featured in both inbound
and outbound events.


## Additional Implementation Details:

### Definitions:
- *Requeue/Republish an event*: In the context of the DLQ service, this means to publish
  the next Kafka event from a given DLQ topic to the corresponding retry topic.
- *Process an event*: Consume the event and apply validation and processing logic (if
  applicable). The event will either be republished or discarded as a result.
- *Discard an event*: To decline to requeue/republish an event to the retry topic. This
  can occur upfront via a direct command by the user, or it can occur as a result of
  the event-processing logic.
- *Resolve an event*: Process or discard an event.
- *Preview events*: Peek at the next N events without affecting the order of events in
  the DLQ topic. This is crucial functionality that enables the user to decide how to
  resolve the next event. The number of events previewed is capped by configuration.

### Event Ordering:
Dead letter queues inherently present a potential threat to system-wide event ordering.
However, ordering events by keys, the idempotent design of our services, and having
separate DLQ topics for each service should prevent most problems. Additionally, only
one partition should be configured per DLQ topic as there will be only one consumer
for each DLQ topic.

### Event identification:
Right now, events can be identified through a combination of correlation ID + event
type and topic. It might be useful to have a per-event identifier added to Kafka
events and managed/generated by the provider.
The work required would be small and the resulting ease of identifying an event could
be valuable.

The event ID would only be propagated for the DLQ cycle. At all other junctures, a new
event ID would be generated at time of publish. This differs from the correlation ID,
which is propagated across services and persisted in the database in the case of outbox
events. To illustrate the event ID lifecycle:

Normal flow
```
1. Event is published with a random event ID header
2. Event is consumed successfully and the event ID is not propagated further
```

DLQ flow:
```
1. Event is published with a random event ID header ("1a2b")
2. Event is consumed *unsuccessfully*
3. Event is published to the configured DLQ topic with the same "1a2b" event ID header
4. Event is processed and published to the retry topic with a new event ID
5. Event is consumed successfully and the event ID is not propagated further
```

Unlike the correlation ID, the event ID is never saved to the database.

### Event processing:
When we resolve the next event in a given DLQ topic, it can go one of two ways:
- We can discard the event, meaning it is not republished to a retry queue and is
  effectively ignored.
- We can process the event, where we run the event through custom logic that can be as
  extensive or specific as needed. Ultimately, the event would be requeued to the retry
  topic or discarded.

The event processing logic will be part of the DLQ service. The idea is that the event
can be routed to some bit of code based on its service of origin, topic, type, or
anything else.

Another useful piece of information for the DLQ service is the exception information.
There is currently no mechanism to provide that to the DLQ service, but the info could
be passed through event headers. The exception header would then be removed upon
republishing the event to a retry topic.

This [Java DLQ implementation](https://medium.com/nerd-for-tech/-to-re-queue-apache-kafka-dlq-messages-95941525ca77)
uses such headers (toward the bottom).

### Processing Dry Run:
It would be nice to be able to perform a dry-run of event processing as a sanity check.
The dry-run endpoint would perform any applicable processing on the next event and
return the result instead of publishing it. The DLQ service would maintain the initial
offset position, just like with the preview action. If processing would result in the
event being discarded instead of published, then it will be indicated by the return
value.


### Configuration:
The DLQ service needs to know the `dlq_topic` and `retry_topic` configuration values
used by each service. While a given configured `service_name` can change, the service it represents is known and fixed, so there should be an explicit config entry for each
service. Within that design, the configuration can take a variety of forms. Other config
includes the maximum number of events returned by the preview endpoint (`preview_limit`)
and the `token_hashes` used for authentication.

```python
preview_max: int
token_hashes: list[str]
ucs: tuple[str, str]
irs: tuple[str, str]
ifrs: tuple[str, str]
dcs: tuple[str, str]
# etc.
```

### Previewing Events:
The DLQ service has to be able to preview Kafka events so we can see what's in a given
DLQ topic. This can't be accomplished by the consumer loop we use in our standard
Kafka providers because the events must be seen once for the preview, then a second time
to resolve the event. The `KafkaDLQSubscriber` only has the ability right now to resolve
events, so it must be updated.

When a preview for N events is fetched, the DLQ service's consumer should revert its
offset to the original value. Sequential calls for previewing N events should always
return the same N events if no resolutions are executed in the meantime.

Previewed events will be formatted with the `ExtractedEventInfo` class and returned as 
JSON:
```json
{
  "topic": ...,
  "type_": ...,
  "payload": {
    "field1": ...,
    "field2": ...,
    ...
  },
  "key": ...,
  "headers": {
    "correlation_id": ...,
    "original_topic": ...,
    "exc_class": ...,
    "exc_msg": ...,
  }
}
```

### Tests:
At the very least, the DLQ service tests should cover the following:
- Test that previewing is idempotent and returns events in consistent order
- Test that endpoints are secured
- Test that API calls trigger the correct action
- Test that events are consumed from the correct DLQ topics
- Test that events are published to the correct retry topics
- Test that only one event is processed or discarded per API call
- Test that the preview endpoint returns the next N events (e.g. not off by 1)
- Test that the preview endpoint does not disturb the offset of the next-resolved event
- Test that the service doesn't break if Kafka data is lost (i.e. offsets adjust)


### Usage:
Startup: In the current form there is only one entrypoint, so the container only needs
to be started (no specific CLI command required).
Preview & resolve events: Done via HTTP API calls.

### Scaling:
The DLQ Service should not be scaled, because the manual intervention required will
be the limiting factor rather than infrastructure.

## Human Resource/Time Estimation:

Number of sprints required: 2

Number of developers required: 1
