---
title: Protocol for async CloudEvent messages
adr: 0007
status: Draft
date: 2026-11-23
decision-makers:
  - Technical Working Group
consulted:
informed:
---


# Which protocol will be used to communicate CloudEvent OEAPI messages?
  

## Status: Draft

## Context

The Open Education API is initially designed as an OpenAPI (REST) protocol, using synchronous requests from clients to server to retrieve information from an educational institution. However, there are also valid usecases for asynchrounous communication, where changes in data are actively pushed towards the client / consumer.
  
It has been decided that these messages will be communicated by use of CloudEvents. The specifications of these CloudEvents messages [is documented separately](007-attachments/cloudevents-doc.md).

CloudEvents are an abstract definition of the metadata of an event, they need to be implemented in a particular protocol. This ADR is made to establish in which protocol the CloudEvents will be expressed in the context of OEAPI. 

Further requirements are:
* Although we do not support publish/subscribe patterns yet, we may want to grow towards such a model in the future. The protocol should support this or provide an easy migration path.
* Providing the endpoint at the side of the consumer/client where the message can be sent to, is considered to be out of scope. This should be configured through other means.
* The specification we choose should support easy code generation and verification

## Considered options

### Option 1: OpenAPI With Webhooks
OpenAPI already defines the synchronous REST contract for OEAPI.  Webhooks are an OpenAPI extension that allow a server to *POST* a CloudEvent to a consumer‑provided URL when a specific event occurs.

Because CloudEvents already describes the type of entity and the type of operation (upsert/delete), there are only two needed webhooks: one for thick and one for thin events. Existing definitions of content type (e.g. course or programme) can be reused.

**Advantages:**

* **Single source of truth** – The entire API surface (both request/response
  and event contracts) is described in one OpenAPI document, simplifying
  documentation, versioning and governance.
* **Tooling support** – Existing OpenAPI generators (e.g. `openapi-generator`,
  `swagger-codegen`) already understand the `webhooks` keyword and can
  produce client and server stubs for both the REST endpoints and the
  webhook callbacks.
* **Smooth migration path** – Consumers that only need synchronous calls can
  ignore the webhook definitions, while future publish/subscribe features
  can be added by extending the same document.
* **Validation** – The OpenAPI schema can be used to validate the payload
  of the CloudEvent (`data`) as it is sent, ensuring consistency with the
  existing resource models.

**Disadvantages:**

* **Limited async semantics** – Webhooks are a push‑only mechanism; they do
  not natively support subscription management, message filtering or
  durable queues. Implementations must handle retries and dead‑letter
  handling manually.
* **Consumer readiness** – The consumer must expose an HTTP endpoint that is
  publicly reachable, which may not be feasible for all clients (e.g.
  mobile apps behind NAT). Additional deployment concerns (TLS, auth) are
  required.
* **No standard Pub/Sub integration** – While OpenAPI can describe the
  webhook, it does not provide a native abstraction for broker‑based
  publish/subscribe systems (Kafka, MQTT, etc.).

### Option 2: AsyncAPI

AsyncAPI is a specification purpose‑built for event‑driven architectures.
It describes how CloudEvents are transmitted over a transport protocol
(HTTP, AMQP, MQTT, …) and includes rich metadata for topics,
message bindings and security.

**Advantages:**

* **Designed for async** – Provides first‑class concepts such as
  `servers`, `channels`, `publish`/`subscribe` and message bindings, making
  it straightforward to model both thin and thick events.
* **Pub/Sub ready** – Native support for broker‑based transports enables a
  future migration to a full publish/subscribe model without changing the
  specification.
* **Code generation** – Tools like `asyncapi-generator` can produce
  server stubs, client libraries and documentation for many languages and
  protocols.
* **Validation & tooling** – AsyncAPI includes a JSON Schema for the
  payload and a validator that checks conformance of the CloudEvent
  envelope.

**Disadvantages:**

* **Double specification** – The existing OEAPI REST contract remains in
  OpenAPI; adopting AsyncAPI adds a second specification to maintain,
  potentially causing divergence if changes are not kept in sync.
* **Tooling maturity** – While AsyncAPI tooling is improving, it is still
  less mature and less widely adopted in the Go ecosystem compared to
  OpenAPI generators.
* **Learning curve** – Teams need to become familiar with a new spec
  format and its tooling, which may increase onboarding time.


## Decision

Not yet taken.

<!-- 

AI suggestion:

### Rationale
* The OEAPI ecosystem already relies heavily on OpenAPI for all REST
  interactions. Extending this single document with the `webhooks` section
  preserves a single source of truth and leverages the existing tooling
  chain (code generation, validation, CI checks).
* Immediate needs are limited to thin events that simply notify a consumer
  that data is available via the existing synchronous API. Webhooks cover
  this use‑case without introducing a parallel specification.
* Although AsyncAPI provides richer async‑native features, the added
  complexity and maintenance burden of a second spec outweigh the benefits
  for the current scope. Should a full Pub/Sub model become required,
  the OpenAPI document can be evolved to reference an external AsyncAPI
  definition, providing a migration path. -->

## Consequences
<!-- * The OpenAPI definition for OEAPI will be extended with a `webhooks`
  component describing each CloudEvent type (e.g. `persons.updated`,
  `feedback.*`). This will be version‑controlled alongside the existing
  REST paths.
* Generated server stubs will include webhook handlers that forward the
  CloudEvent payload (thin or thick) to the configured consumer endpoint.
* Existing CI pipelines that validate OpenAPI documents will automatically
  validate the webhook definitions as well.
* Documentation tooling (e.g. ReDoc, Swagger UI) will display webhook
  information next to the REST endpoints, giving consumers a single place
  to discover both synchronous and asynchronous capabilities. -->