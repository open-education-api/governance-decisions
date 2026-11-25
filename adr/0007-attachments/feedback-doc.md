# Feedback Events

Feedback events are a special type of [CloudEvent](cloudevents-doc.md).

## What are Feedback Events?

When regular OEAPI CloudEvents are processed many thing may fail or cause warnings (such as inavailability of systems, wrong data format, missing keys, etc). The producer might want to know such 'feedback' as well. 

Notice that 'feedback' in this case is not the same as the HTTP status code returned when delivering an event. When delivering a (thin) event, a reponse like '200 OK' merely indicates that the event has been delivered successfully. It does not imply anything about the processing of that event, as that happens asynchronously, including fetching of the data.

## Requirements

- It must be possible to give feedback on a specific entity (e.g. a programme identified by an UUID
- It must also be possible to give feedback on a specific event as sent by the producer (by event ID). Only in this way it is possible to identify what the feedback was on the latest update on that specific entity.
- There can be multiple (0...*) feedback responses on a single event
- It must be possible for the consumer to establish to which endpoint the feedback should be sent 
- Differentiation for different feedback types: 
  * STATUS_UPDATE: a non-final state that tells something about this event being processed. These type of events are of a technical nature. 
  * REJECTED: the event was rejected as it did not comply with technical or functional requirements.
  * FAILED: the event failed to be processed, but it was not rejected. This usually happens after long unavailability of dependent systems. Retrying the same event might work. 
  * PROCESSED: The event was successfully processed. The feedback MAY contain non-critical warnings.
- The feedback MAY contain an array of messages. It MUST contain at least one message when the status is 'STATUS_UPDATE' or 'REJECTED'. In that case, the first message MUST be the major reason of the status update or the rejection.
- Each message can be of a type: CRITICAL, WARNING, TRACING. The CRITICAL message type MUST only occur in FAILED or REJECTED feedback type.
- Each message can be translated in multiple languages


## Specification



| Attribute             | Content                                                  | Required | Example                              |
| --------------------- | -------------------------------------------------------- | -------- | ------------------------------------ |
| id                    | UUID of this feedback event, unique                      | yes      |                                      |
| type                  | oeapi type suffixed with '.feedbackprovided'             | yes      | eu.oeapi.programmes.feedbackprovided |
| source                | URI of source system generating the feedback             | yes      | urn://duo.nl/rio/prod                |
| subject               | UUID of the entity this feedback is about                | yes      |                                      |
| correlationid         | ID of the (thin) event that this feedback applies to     | no (1)   |                                      |
| correlationsource     | Source of the (thin) event that this feedback applies to | yes      | https://sampleuniversity.eu/prod     |


(1) 'Feedback on event id' is not required, because the feedback does not have to apply to a specific event. For example, when a client is simply performing a GET on a programme, it may encounter an error or feedback may originate during the subsequent processing of that data. In that case there is no original event id, but feedback may still be provided for that specific entity (type+subject). Having said that, if the feedback does apply to an original (thin) event, the 'feedbackoneventid' SHOULD be included. 


## Sample message


```
{
  "specversion": "1.0",
  "id": "e96d9dd2-58b3-45e1-8d91-30bd31874bf9",                
  "type": "eu.oeapi.programmes.feedbackprovided",
  "source": "https://sampleregistry.eu/prod",                         
  "subject": "32fc6559-601f-4a49-a179-88152de39690",           
  "correlationid": "e96d9dd2-58b3-45e1-8d91-30bd31874bf9", 
  "correlationsource": "https://sampleuniversity.eu/prod", 
  "time": "2021-12-10T17:31:00.000Z",                          
  "payloadtype": "thick",
  "data": {
    "type": "rejected",
    "messages": [
		{
			"type": "critical",
			"content":[
		    {
		      "language": "en-GB",
		      "value": "This programme is missing a description"
		    },
		    {
		      "language": "nl",
		      "value": "Deze opleiding mist een omschrijving"
		    }
			],
		},
 		{
			"type": "warning",
 			"content":[
			    {
			      "language": "en-GB",
     		      "value": "First enrollment starts on 01-01-2036."
			    } 
			]
		},
     ]
  }
}
```