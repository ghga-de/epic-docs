# OpenTelemetry in file services (Common Kingslayer)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).

## Scope
### Outline:

Based on some previous exploration work, this epic aims to implement 1) OpenTelemetry instrumentation in the file services and 2) necessary OpenTelemetry functionality inside of hexkit and/or ghga-service-commons.
In addition, a viable setup of Jaeger and its components needs to be devised and implemented.

### Included/Required:

While OpenTelemetry supports tracing, metrics and logging, this epic will focus on enabling collection of distributed tracing data and neglect the other two aspects.

#### File Services:
File services can export tracing data using manual or autoinstrumentation or a combination of both.
Autoinstrumentation works by installing specific libraries that monkeypatch existing functionality to inject information that can be propagated across service boundaries.
While these are quite quite useful, they should be vetted for what data is attached to the traces and manual instrumentation should be used instead, if sensitive information is included.
If autoinstrumentation can be used in all/most cases, additional implementation work can be kept to a minimum.

#### Hexkit/Service Commons:
Hexkit needs some changes introducing logic around event subscribers to correctly propagate tracing context across service boundaries as the autoinstrumentation for Kafka does not open a new child span, ending the propagation at the event subscriber.

Depending on if some autoinstrumentation is replaced or enhanced by manual instrumentation, there might be a need to touch some middleware, which would be done in ghga-service-commons.

#### Jaeger Setup:

Jaeger consists of three different components: Jaeger Collector, Jager Query and Jaeger UI. 
An all-in-one deployment bundling all three components is also available, which is normally used for development, but could be a viable option for our deployment for now.
Jaeger Collector additionally needs a backing database for persistent storage of the traces it receives via OpenTelemetry, with multiple options available to choose from.

This part of the epic needs a bit more exploration on best practices for a production deployment and choices around e.g. database for Jaeger Collector might be limited by which versions are readily available for deployment in a Kubernetes cluster.

### Not Included:

Dealing with metrics and logs through OpenTelemetry.

## Additional Implementation Details:

Potentially there are some trade offs to be made both for the Jaeger deployment and when choosing autoinstrumentation vs manual instrumentation.
Depending on the answers to some of the open questions, there might be some additional implementation work included, but overall no major code changes should be needed in either case.

Implementation work will use the `otel_test` branches of hexkit and the file services backend monorepo as starting point.

## Human Resource/Time Estimation:

Number of sprints required: 1

Number of developers required: 1
