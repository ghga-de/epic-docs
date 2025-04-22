# Hexkit Documentation (Pygmy Goat)
**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).

## Scope
### Outline:
`Hexkit` currently has very little documentation regarding its use, terminology,
or architectural concepts. Some of this knowledge *is* documented in an unconsolidated
fashion, with files spread across Confluence, Google Docs, and the internal docs
site. However, developers or even hypothetical third parties working with or
contributing to `hexkit` do not currently have access to the comprehensive and
consolidated documentation that the library warrants. This epic will remedy that.
The aggregated documentation can be located within the library as a README or within
an external service, such as the internal docs site.


### Included/Required:
The following points should be covered in the documentation:
- Glossary
- Usage Examples
- Architectural Concepts (Triple Hexagonal Architecture, DLQ, Outbox, DAOs, etc.)
- Testing Tools (persistent vs clean fixtures, session-scoped containers, etc.)
- Requirements for adding new functionality, e.g. new protocols or new DAO providers

### Not included:
The work in this epic might include some API documentation or make improvements to,
e.g. function doc strings, but it is not explicitly required. The work in this epic
is of a broader and less granular nature meant to provide a cohesive knowledge base
for developers. API documentation should already be performed as a matter of course.

### Additional Implementation Details
The following is one example of a possible structure for the documentation:

- Architecture Concepts
  - Triple Hexagonal Architecture
  - Event-Driven Architecture
- Protocols
  - Existing protocols
  - Adding protocols
- Providers
  - S3
    - Overview
    - Test utilities
      - Container fixture and fixture reset/cleanup function
      - Accompanying fixtures/utilities
  - Kafka
    - Overview
    - Outbox Pattern
    - Persistent Publisher
    - When to use outbox vs persistent publisher
    - DLQ
    - Test utilities
      - Container fixture and fixture reset/cleanup function
  - MongoDB
    - Overview
    - Role in outbox pattern
    - Role in persistent publisher
    - Test utilities
      - Container fixture and fixture reset/cleanup function
- Adding providers
- Glossary
- Usage Examples
  - Basic Kafka Pub/Sub
  - Basic DAO
  - Basic S3
  - Dependency Injection (as in most of our `inject.py` modules)


## Human Resource/Time Estimation:

Number of sprints required: 1

Number of developers required: 2
