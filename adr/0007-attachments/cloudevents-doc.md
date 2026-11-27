# CloudEvents

For asynchronous operations in OEAPI, we use the CloudEvents standard to define the metadata of the event. CloudEvents is a protocol-agnostic standard that defines the metadata (context attributes) of events (or messages) and a place for the event data. It does not define the content of the event.

Besides the metadata, we also use the standards that define how CloudEvents can be expressed in JSON and HTTP messages.

For more information, see:
* [The CloudEvents website](https://cloudevents.io/)
* [JSON formatting of Cloud Events](https://github.com/cloudevents/spec/blob/main/cloudevents/formats/json-format.md)
* [HTTP protocol binding of Cloud Events](https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/http-protocol-binding.md)

For HTTP binding, there are two modes: binary content mode and structured content mode. We use structured content mode (`Content-Type: application/cloudevents+json`), as this allows for direct forwarding of events without the need to read and translate HTTP headers.

For the OEAPI, we distinguish three types of events:
1. Thin events
2. Thick events
3. Feedback events

How events are delivered, and which endpoints and credentials are agreed out of band, is described in [delivery-and-registration.md](delivery-and-registration.md).

# Attributes common to all events

| Attribute       | Description | Required? |
| --------------- | ----------- | --------- |
| specversion     | Always `1.0` | Yes |
| id              | UUID for this event only. Created once per event and stored, so a redelivery carries the same `id`. Unique per `source` | Yes |
| source          | Identifier of the source system, including its environment (format still under debate), for example `https://example-university.fr/sis/prod` | Yes |
| type            | See below for the three families | Yes |
| subject         | The UUID of the entity the event is about | Yes |
| time            | Moment of occurrence, RFC 3339 in UTC with milliseconds, for example `2026-10-06T13:00:00.123Z` | Yes |
| payloadtype     | `thin`, `thick`. Feedback events always use `thick` | Yes |
| datacontenttype | Media type of `data`. `application/json` when omitted | No |

`payloadtype` is an OEAPI extension attribute. Extension attribute names are lowercase a-z/0-9 and at most 20 characters.

A receiver deduplicates on `source` + `id`.

# Thin Events

Thin events do not carry the payload of the data that has changed. They only 'signal' to the client that a certain entity has changed, which SHOULD lead to a query to retrieve the latest state of that entity.

An example of the CloudEvents metadata for a thin event is as follows:

```json
{
  "specversion": "1.0",
  "id": "0f8fad5b-d9cb-469f-a165-70867728950e",
  "source": "https://example-university.fr/sis/prod",
  "type": "eu.oeapi.programmes.updated.v1",
  "subject": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "time": "2026-10-06T13:00:00.123Z",
  "payloadtype": "thin",
  "dataref": "https://sis.example-university.fr/oeapi/v6/programmes/7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

We see the following attributes:

| Attribute   | Description | Required? |
| ----------- | ----------- | --------- |
| type        | Type of entity that was altered. Always starts with `eu.oeapi.`, followed by the (plural) entity that was updated or deleted, then either `.updated` or `.deleted`, followed by a version number (`.v1`). The version number will only be updated in case of breaking changes. | Yes |
| payloadtype | This MUST be `thin` | Yes |
| dataref     | URL at which the data can be retrieved. This MAY be provided by the source system. If this is not provided, the assumption is that the client can construct the URL to retrieve the data by other means (e.g. configuration files). | No |
| data        | MUST NOT be present | N/A |

## Fetching the entity

The consumer and the OEAPI version are **not** part of the `dataref`. They follow from the subscription and are sent by the consumer as the OEAPI `Accept` header when fetching, for example:

```
GET /oeapi/v6/programmes/7c9e6679-7425-40de-944b-e07fc1f90ae7
Accept: application/vnd.oeapi+json;version=6.1;consumer=rio;consumer-version=1.0
```

This keeps one thin event valid for every consumer, and lets the source apply its access rules per consumer at fetch time.

# Thick Events

Thick events contain the data of the changed entity. They are quite similar to thin events, with the following exceptions:

| Attribute   | Description | Required? |
| ----------- | ----------- | --------- |
| payloadtype | This MUST be `thick` | Yes |
| dataref     | MUST NOT be provided for thick events | N/A |
| data        | The nested data of the changed entity. `data` is the CloudEvents attribute for the event payload; in JSON format it is a member of the same JSON object | Yes |

A thick event may look like this:

```json
{
  "specversion": "1.0",
  "id": "e3b0c442-98fc-1c14-9bd5-02b2c3d479bc",
  "source": "https://example-university.fr/sis/prod",
  "type": "eu.oeapi.programmes.updated.v1",
  "subject": "123e4567-e89b-12d3-a456-426614174000",
  "time": "2026-10-06T13:00:00.123Z",
  "datacontenttype": "application/json",
  "payloadtype": "thick",
  "data": {
    "programmeId": "123e4567-e89b-12d3-a456-426614174000",
    "programmeType": "programme",
    "primaryCode": { "codeType": "identifier", "code": "1234qwe12" },
    "name": [ { "language": "en-GB", "value": "Biology" } ]
  }
}
```

`subject` equals the identifier inside `data`.

# Feedback Events

Feedback events are a special type of thick events. They communicate any type of 'feedback' (status messages, functional errors or technical errors) that are the result of an event. Feedback messages, their life cycle and what the source does with them are described separately in [this document](feedback-doc.md). Their `type` is the entity type suffixed with `.feedbackprovided` and a version, for example `eu.oeapi.programmes.feedbackprovided.v1`.

# Delivery

An event is delivered with an HTTP `POST` to the webhook the receiver registered. The receiver answers `202 Accepted` (or `200 OK`) when it has accepted the event, `400` for an invalid event, `401` for failed authentication, `413` when the event is too large (keep events below 64 KB) and `429` when the sender must slow down. The sender retries on timeouts, `429` and `5xx` using the same `id` (at-least-once delivery). Authentication is OAuth 2.0 client credentials over TLS.
