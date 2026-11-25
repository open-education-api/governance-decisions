
# CloudEvents

For asynchronous operations in OEAPI, we use the CloudEvents standard to define the metadata of the event. CloudEvents is a protocol-agnostic standard that (only) defines metadata of events (or messages), it does not define the content of the event. 

Besides the metadata, we also use the standards that define how CloudEvents can be expressed in JSON and HTTP messages. 

For more information, see:
* [The CloudEvents website](https://cloudevents.io/)
* [JSON formatting of Cloud Events](https://github.com/cloudevents/spec/blob/main/cloudevents/formats/json-format.md)
* [HTTP protocol binding of Cloud Events](https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/http-protocol-binding.md)

For HTTP binding, there are two modes: binary content mode and structured content mode. We use structured content mode, as this allows for direct forwarding of events without the need to read and translate HTTP headers.

For the OEAPI, we distinguish three types of events:
1. Thin events
2. Thick events
3. Feedback events

# Thin Events

Thin events do not carry the payload of the data that has changed. They only 'signal' to the client that a certain entity has changed, which SHOULD lead to a query to retrieve the latest state of that entity.

An example of the CloudEvents metadata for a thin event is as follows:

```
{​
	"specversion": "1.0",​
	"id": "e3b0c442-98fc-1c14-9bd5-02b2c3d479bc",​
	"source": "https://example-university.fr/sis/prod",​
	"type": "eu.oeapi.programmes.updated.v1",​
	"subject": "7c9e6679-7425-40de-944b-e07fc1f90ae7",​
	"time": "2026-10-06T13:00:00.123Z",
	"payloadtype": "thin",​
	"dataref": "https://sis.example-university.fr/OEAPI/v6.1/programmes/7c9e6679-7425-40de-944b-e07fc1f90ae7"​
}
```

We see the following attributes: 

| Attribute   | Description                                                                                                                                                                                                                                                           | Required? |
| ----------- | --------------------------------------------------------------------------------------------------- | --------- |
| specversion | Always 1.0                                                                                                                                                                                                                                                            | Yes       |
| id          | UUID for this event only                                                                                                                                                                                                                                              | Yes       |
| source      | Identifier of the source system (format still under debate)                                                                                                                                                                                                           | Yes       |
| type        | Type of entity that was altered. Always starts with "eu.oeapi.", followed by the entity that was updated or deleted, then either ".updated" or ".deleted", followed by a version number (".v1"). The version number will only be updated in case of breaking changes. | Yes       |
| subject     | The UUID of the entity that was altered                                                                                                                                                                                                                               | Yes       |
| time        | The time (in milliseconds)                                                                                                                                                                                                                                            | Yes       |
| payloadtype | This MUST be 'thin'                                                                                                                                                                                                                                                   | Yes       |
| dataref     | URL at which the data can be retrieved. This MAY be provided by the source system. If this is not provided, the assumption is that the client can construct the URL to retrieve the data by other means (e.g. configuration files).                                   | No        |
## Thick Events

Thick events contain the data of the changed entity. They are quite similar to thin events, with the following exceptions:

| Attribute   | Description                                                                         | Required? |
| ----------- | ----------------------------------------------------------------------------------- | --------- |
| payloadtype | This MUST be 'thick'                                                                                                                                             | Yes       |
| dataref     | This may NOT be provided for thick events                                                                                                                        | N/A       |
| data        | This contains the nested data of the changed entity. This is actually not a part of the CloudEvents specification, but will be included in the structured event. | Yes       |

A thick event may look like this:

```
{​
  "specversion": "1.0",​
  "id": "e3b0c442-98fc-1c14-9bd5-02b2c3d479bc",​
  "source": "https://example-university.fr/sis/prod",​
  "type": "eu.oeapi.programmes.updated.v1",​
  "subject": "7c9e6679-7425-40de-944b-e07fc1f90ae7",​
  "time": "2026-10-06T13:00:00.123Z",
  "payloadtype": "thick",​
  "data": {
     { "programmeId": "123e4567-e89b-12d3-a456-426614174000", 
            ...
     }
  }
}
```

## Feedback Events

Feedback events are special type of thick events. They communicate any type of 'feedback' (status messages, functional errors or technical errors) that are the result of an event. Feedback messages are described separately in [this document](feedback-doc.md).





