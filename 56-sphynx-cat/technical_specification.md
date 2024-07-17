# Kakfa Dead Letter Queues (Sphynx Cat)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).


## Scope
### Outline:
The goal of this epic is to both define and implement a mechanism to deal with Kafka
events that result in unhandled exceptions when consumed. Such errors can be caused
by an array of things, and may or may not be a problem with the event payload itself.
Events that fail cannot always be outright discarded, so there is a need for a way to
set the events aside (disengage them from the request flow) and allow for investigation.
After looking into why the event failed and performing any required intervention, the
events can be either discarded or reintroduced into the request flow as the situation
warrants. 

Currently, we have no elegant way to handle this kind of situation aside from immediately
diagnosing and patching the application code. As an example, there was a situation in the
Archive Test Bed where the `nos` consumed an event that required it to look up a non-
existent record in the database and crashed. Ideally, this would not prevent the service
from functioning. Instead, the error would be logged and the problematic event would be
dealt with in a way that allows us to examine the problem and later retry the event.

One of the most common mechanisms for achieving this capability is called a
Dead Letter Queue (DLQ). In a DLQ, failed events are sent to a separate location. In the
context of Kafka, this is usually another topic, but it could also be a database
collection or any other location that effectively preserves the event while preventing
further processing. Kafka does not come with DLQ support out-of-the-box unless used with
Kafka Connect. We are not using Kafka Connect and use AIOKafka's python producers and
consumers with our own library instead, meaning we can't take advantage of Kafka Connect
without considerable work.

> [!Note]
> It's important to note that using a dead letter queue *does not* remove the need for
manual intervention. It merely provides the means to perform diagnostics and corrective
action while allowing the service to continue processing events in the background.


### Included/Required:
- ADR Proposal
- Implementation of DLQ Providers in `hexkit`
- Implementation of DLQ logic in services
- Internal Documentation: Async Interservice Communication Architecture Concept


### Not included:
- Setup of a dedicated DLQ monitor or dashboard service (more on this below)


## Additional Implementation Details:

### Dead Letter Queues in Kafka

![DLQ Concept](./images/dlq.png)

The above illustrates the high-level concept for a Kafka DLQ. 


![Example DLQ flow](./images/dlq_flow.png)

The flow diagram above demonstrates the proposed use of a DLQ in a given service:

1. Failed events are retried a configurable number of times.
2. Upon final failure, they are published to the service-specific DLQ topic
(the original topic name is preserved), where they await review.
3. Events in the DLQ are manually reviewed. To ignore or reprocess the next event in the
DLQ, the corresponding command is sent to a dedicated DLQ consumer.
4. If the DLQ consumer is instructed to retry the event, it will publish it to a DLQ
Retry topic, to which the main consumer actively listens.
1. Upon consuming an event from the retry queue, the main consumer restores the original
topic name and proceeds with the normal request flow.

### Implementation of DLQ Providers in `hexkit`

The work will involve the creation of a class that combines the Kafka Subscriber and the
Kafka Publisher, which will be used in two ways.


**`KafkaRetrySubscriber`** *(name subject to change)*  
The primary expansion to hexkit to get the fundamental DLQ logic going. It is
a specialized event consumer that, upon catching an unhandled error during event
consumption, does the following:
1. Retries handling the event until the configured number of retries is exhausted (using
exponential backoff).
   - This can reduce the likelihood of transient errors populating the DLQ.
2. Checks auto-ignore rules to determine if the event should be dropped or published
   - This can be implemented later when it is more clear how best to do this.
3. Publishes the event to a service-specific DLQ topic.
   - The original topic should be added as a field appended to the payload.
4. Commits the topic offsets to avoid infinitely reprocessing the failed event.
5. Upon consuming an event from the service-specific retry topic, extracts/removes the
original topic from the payload and proceeds with processing as if it were any other
event (reentry).


**`KafkaDLQSubscriber`** *(name subject to change)*  
A specialized event consumer that will, upon instruction, consume one event from the
configured DLQ topic and either:
1. Ignore the event (do nothing), or
2. Publish the event to the configured retry topic. The original topic name should still
be contained in the payload unmodified.

This class is potentially not required if requeueing is done through a 3rd party tool
like Kafka UI. However, if requeueing is done through either the different services or
an in-house dedicated DLQ service, then this type of consumer will be helpful.

## Other: 

### Potential Problems:

> [!NOTE]
> The following list is by no means exhaustive: 

**#1.** *Kafka data is lost after publishing to the DLQ (the DLQ data is lost).*

**Solution**: DLQ events can be restored by republishing and reconsuming the upstream
events.

**#2:** *Kafka does not get lost, but upstream events are nevertheless republished, causing*
*the failed events to be reprocessed and placed in the DLQ (causing duplicate events in*
*the DLQ).*

**Solution:** Do nothing. Recognize that this is a corner case with low probability,
and resolving the first instance of the duplicated DLQ event should fix the remaining
copies automatically. If the situation is identified, the remaining copies can be either
ignored or placed in the retry queue, where idempotence measures will prevent side effects.

**Alternative:** Perform deduplication logic before publishing: Use an idempotence check
and store failed events in the database. If the outbox pattern is employed, then both
the service where the failure occurs and any potential observers can stay up to date on
which events have been resolved (inserting and deleting events in the service's database
as a representation of the state of the DLQ, while the retry queue would be a typical
Kafka topic).

**#3:** *A particular event type routinely results in DLQ’d events, overwhelming the*
*topic or logs.*

**Solution:** Ideally, this would be minimized by improving error handling in application
code. However, a supplementary tool could be the addition of a configuration-driven
mechanism that enables us to discard or ignore certain events as a rule. Another idea
is to implement a way to signal an automatic retry after a time.

**#4:** *A failed event is reviewed and then republished to the original topic, causing*
*all other consumers of that event to process a duplicate.*

**Solution:** This should not be a problem because services should be designed to be
idempotent. However, to protect against the possibility that idempotence is implemented
incorrectly, a service-specific "retry" topic is used when it's time to try consuming a
failed event again.

### Dedicated Dashboard

The work included with this epic will provide programmatic DLQ functionality,
but it does not include a convenience layer such as a dashboard or central DLQ
resolution service. On the surface, Kafka UI seems suited for this task. However,
sending a message in a DLQ topic to the corresponding retry topic in Kafka UI requires
some copy/paste action, which doesn't scale. If Kafka UI is used in conjunction with
service-level resolution (i.e. discarding/requeueing), then we need to do some work to
keep Kafka UI in sync with the service level consumers. This requires more digging and
discussion; maybe there is a simple solution.


## Human Resource/Time Estimation:

Number of sprints required: 2

Number of developers required: 1
