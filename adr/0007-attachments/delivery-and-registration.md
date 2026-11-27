# Delivery and registration of endpoints

This document explains how events travel between a source and a consumer, what has to be agreed
out of band before the first event can be sent, and how the two options of ADR 0007
(OpenAPI webhooks and AsyncAPI) differ in practice.

The short version: **on the wire both options are the same.** In both cases the sender does an HTTP `POST`
of an `application/cloudevents+json` document to a URL the receiver registered. The options differ in the
contract format that describes this, the tooling around it, and the growth path towards brokers.

Terms: *source* is the system that produces the event (SIS, or an intermediary producing events on its behalf).
*Consumer* is the system that receives it (for example RIO or eduXchange, directly or via SURFeduhub).
An intermediary can play both roles, see [Intermediaries](#intermediaries).

## 1. What is agreed out of band

The OEAPI contract does not contain any addresses or credentials. Per consumer a *subscription* is set up.
Both sides keep their half of it.

| Who keeps it | What | Why |
| --- | --- | --- |
| Source | Webhook URL of the consumer | Where to `POST` events |
| Source | Event types the consumer wants (`eu.oeapi.<entities>.<verb>.v<n>`) and whether they are thin or thick | Filtering. The source only sends what was agreed |
| Source | Consumer key and consumer version, and the OEAPI version | Thin events: used as `Accept` header when the consumer fetches the entity. Thick events: decides which projection of the data is sent |
| Source | Token endpoint and scope of the consumer | The source authenticates itself at the consumer (OAuth 2.0 client credentials) |
| Source | Retry policy, rate limit, contact for alerts | Operations |
| Consumer | `source` identifier(s) of the source, per environment (prod, acc, test) | Accepting and deduplicating on `source` + `id` |
| Consumer | Base URL of the OEAPI of the source (for `dataref`, or to build the URL when `dataref` is absent) | Fetching thin events |
| Consumer | Feedback URL of the source | Where to `POST` feedback events |
| Consumer | Client credentials for the OEAPI and the feedback endpoint of the source | Authentication at the source |

Rules of thumb: one subscription per consumer per environment, test and production never share credentials, and the
`source` in the events always matches the environment of the subscription.

```mermaid
sequenceDiagram
    autonumber
    participant CO as Consumer owner
    participant SO as Source owner
    participant SRC as Source system
    participant CON as Consumer system

    Note over CO,SO: Out of band (portal, ticket, contract register, SURFeduhub onboarding)
    CO->>SO: Request subscription: event types, thin or thick, consumer and versions
    SO->>CO: Source identifier, base URL of the OEAPI, feedback URL, token endpoint
    CO->>SO: Webhook URL, token endpoint, client id for the source
    SO->>CO: Client id for OEAPI and feedback endpoint
    SO->>SRC: Configure subscription for this consumer
    CO->>CON: Configure sources, credentials and feedback URL

    Note over SRC,CON: First technical check
    SRC->>CON: POST test event (thin) to webhook
    CON-->>SRC: 202 Accepted
    CON->>SRC: GET dataref with consumer Accept header
    SRC-->>CON: 200 OK
    CON->>SRC: POST feedback event (processed) to feedback URL
    SRC-->>CON: 202 Accepted
```

## 2. Where the options differ: the contract

Both options describe the same messages. What differs is the artefact the consumer and source developers work with.

```mermaid
flowchart LR
    S[Shared schemas<br/>envelope, thin, thick, feedback<br/>Course, Programme, ...]

    subgraph O1[Option 1: OpenAPI with webhooks]
        direction TB
        W[webhooks section<br/>thinEvent, thickEvent, feedbackEvent]
        P[paths<br/>GET endpoints of the OEAPI]
        W --- P
    end

    subgraph O2[Option 2: AsyncAPI 3.1]
        direction TB
        A[channels and operations<br/>send events, receive feedback]
        M[servers: https now<br/>broker later]
        A --- M
    end

    S --> O1
    S --> O2
    O1 --> T1[OpenAPI tooling<br/>docs, validators, client and server stubs]
    O2 --> T2[AsyncAPI tooling<br/>docs, validators, handlers]
```

| Aspect | Option 1: OpenAPI webhooks | Option 2: AsyncAPI |
| --- | --- | --- |
| Where the contract lives | In the OEAPI OpenAPI document, next to the `GET` endpoints | Separate document that references the shared schemas |
| Request/response and events in one place | Yes | No, two documents (mitigated by sharing the schemas) |
| Subscription and endpoint registration | Out of scope, not in the spec | Out of scope, not in the spec |
| What the wire looks like over HTTPS | `POST` of a CloudEvent | The same |
| Moving to a broker later | Needs an additional specification | Change the server protocol, the messages stay |
| Edukoppeling contract register | Fits the OpenAPI (webhook) part | Fits the AsyncAPI (business data) part |

The two are not mutually exclusive: when the envelope and entity schemas are defined once and referenced by both
(as in the two example files next to the ADR), the "double specification" drawback is limited to the thin
channel/operation layer.

## 3. Runtime flow (identical for both options)

```mermaid
sequenceDiagram
    autonumber
    participant SRC as Source system
    participant CON as Consumer system

    Note over SRC: Change in SIS. Event stored with its UUID (outbox)
    SRC->>CON: POST webhook, application/cloudevents+json
    alt Receiver accepts
        CON-->>SRC: 202 Accepted
    else Timeout, 429 or 5xx
        CON-->>SRC: error
        SRC->>CON: Retry with the same id
    else Invalid event or not authorised
        CON-->>SRC: 400 or 401
        Note over SRC: Do not retry unchanged. Alert the owner
    end

    alt Thin event
        CON->>SRC: GET dataref with Accept header for consumer and version
        SRC-->>CON: 200 OK, entity for this consumer
    else Thick event
        Note over CON: Data is in the event
    end

    Note over CON: Deduplicate on source and id, process asynchronously
    CON->>SRC: POST feedback event to the registered feedback URL
    SRC-->>CON: 202 Accepted
```

The `202` only says the event was received. The processing result travels in the feedback event, see
[feedback-doc.md](feedback-doc.md).

## 4. Growth path

AsyncAPI keeps the messages and changes the server. HTTP webhooks first, brokers when needed:

```mermaid
flowchart LR
    subgraph Now[Now: HTTPS webhooks]
        direction LR
        A1[Source] -->|POST CloudEvent| B1[Consumer webhook]
    end
    subgraph Later[Later: broker, if needed]
        direction LR
        A2[Source] -->|publish CloudEvent| BR[(Broker)] -->|subscribe| B2[Consumers]
    end
    Now -.same messages and schemas.-> Later
```

With OpenAPI webhooks this step means adding an AsyncAPI (or equivalent) document at that moment. The messages themselves,
and the CloudEvents attributes, do not change in either option.

## 5. Intermediaries

When an intermediary such as SURFeduhub sits between source and consumer, the same mechanism applies per hop. Each hop is
a source for the next one and a consumer for the previous one, with its own subscription, endpoints and credentials.

```mermaid
flowchart LR
    SRC[Source system] -->|events| HUB[Intermediary]
    HUB -->|events| C1[Consumer 1]
    HUB -->|events| C2[Consumer 2]
    C1 -.feedback.-> HUB
    C2 -.feedback.-> HUB
    HUB -.feedback, mapped.-> SRC
```

Whether the intermediary only forwards (the original event and `dataref` stay intact) or produces a new event per
consumer (new `id`, own `source`) decides how feedback can be correlated, see
[feedback-doc.md](feedback-doc.md#correlation-through-an-intermediary).
