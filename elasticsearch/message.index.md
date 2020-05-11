# Message Queue Index
Each message queue corresponds to an index in ElasticSearch.
In particular, the name of the index corresponds to the root of the topic so all messages posted to any `fruit.*` topic are stored in the `fruit` queue i.e. the `fruit` index.

If multiple consumers must consume a message, don't use a Queue but a Channel. A channel is created for each consumer who subscribes to one or more topics.
* Consumers of Queues are called Workers
* Consumers of Channels are called Subscribers
* Consumers of the Event Log are called Spectators

## Index Mapping
The index stores messages and we can consider the information provided by the producer as the "payload" and other fields used to manage messages in the queue as "metadata".

### Payload Fields
* `message`: The true body or payload of the message received and to be processed.
* `timestamp`: Optional producer timestamp representing the time the producer produced the message.
  * A producer may itself queue (or delay) the messages it produces. // TODO: Do we care?
* `mimeType`: The mime format (e.g. `application/json`) of the `message`. Used e.g. when pushing the message to HTTP consumers.
* `topics[]`: One or more topic keywords (e.g. `fruit.sweet`)
  * Messages with multiple different root topics (e.g. `fruit.sour` and `food.healthy`) will be duplicated out to different queues (`fruit` and `food`)
  * The topics field uses an edge n-gram analyzer to support fast prefix searches like `fruit.*`.
  * Other than the first `.` the rest of the topic name has no special meaning (`<queue>.whatever you like`).

### MetaData Fields
* `state`: The state of the message:
  * 'L': Logged: message was received and saved to Event Log
  * 'U': Unchanged: message was detected as unchanged based on object+key compared to Object History
  * `Q`: Queued: message delivered to the queue corresponding to the object (or the default queue)
  * `F`: Fetched/Forwarded: Message was pushed to or pulled by the consumer (but not yet confirmed)
  * `S`: Success: Message was confirmed as successfully processed by the consumer
  * `K`: Kicked: Message was sent to consumer but we will not wait for confirmation (at most once delivery, fire and forget)
  * `R`: Retry: No confirmation by consumer, message will be retried/requeued
  * 'E': Exasperation: Unable to deliver to consumer, no confirmation, will not retry (dead letter)
  * 'I': Invalid: Consumer replied that the message is invalud
  * 'N': Not-Interested: Consumer replied that the message is not ready/relevant
* Boolean flags:
  * `queued`: Message is in the queue.
  * `delivered`: Message was delivered/confirmed by consumer.
  * `success`: Message was successfully delivered.
  * `archived`: Message is no longer available to consumers and is considered archived.
* Timestamps:
* `created`: The UTC Timestamp the message was received
* `modified`: The UTC Timestamp the message was modified
* `consumed`: The UTC Timestamp the message was first sent to a consumer or fetched by a consumer
* `processed`: The UTC Timestamp the message was first confirmed by a consumer (success/fail)
  * The difference between processed and consumed represents the wait time of a message.
  * When multiple consumers consume a message the actual wait time(s) are more complicated.
* `completed`: The UTC Timestamp the message was successfully confirmed by all required consumers
  * Each queue may define how many consumers must successfully process a message before the message is completed and the status can be set to 'X' (COMPLETED).
  * By default a message is complete when at least one consumer confirms it.
* `failed`: The UTC Timestamp when the message was finally abandoned as failed (possibly after 1 to 9 retries).
* `topicCount`: The number of topics a message contains


## Example Messages
A message successfully delivered to all of it's consumers:
```
{
  "topic": ["customer.create", "audit.log"],
  "object": "customer",
  "key": "XYZ123",
  "state": "X",
  "queued": false,
  "archived": true,
  "confirmed" : true,
  "topicCount": 2,
  "confirmedCount": 2,
  "consumed": [
    {
      "consumerId": "c1",
      "fetchedTS": "20200507T010100",
      "confirmedTS": "20200507T010200",
      "durationMS": 60000
    },
    {
        "consumerId": "c2",
        "fetchedTS": "20200506T000000",
        "confirmedTS": "20200506T000002",
        "durationMS": 2000
    }
  ]
}
```
A message not delivered because it was marked as a duplicate (`U`=Unchanged):
```
{
  "topic": ["customer.create", "audit.log"],
  "object": "customer",
  "key": "XYZ123",
  "state": "U",
  "queued": false,
  "archived": true,
  "confirmed" : false,
  ...
```