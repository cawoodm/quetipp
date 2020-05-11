# QueTipp
HTTP Message Queue built on ElasticSearch.

[Key Concepts](./docs/QueueTipp_Concepts.pptx)

Dump your messages into QueTipp and allow your consumers to pull them out with lightning speed.
Alternatively, push them to registered consumers via HTTP endpoints and an optional transformation.

* Supports Message Queues (like RabbitMQ), Event Logs (like Kafka) and Pub/Sub Channels.
* Support PUSH or PULL for Consumers
* Supports guaranteed delivery

Easily configure logic or write code snippets in JavaScript to route messages to a Queue or subscriber Channel.

QueTipp is designed to:
* be simple to understand and operate
* run fast and perform well at scale
* be flexible to different messaging scenarios
* focus on REST/HTTP and JSON
* leverage ElasticSearch features

## Operation
QueTipp works out of the box with little to no configuration. Producers POST messages in and these are automatically added to the Event Log. Consumers can immediately receive these messages by polling the Log and providing a timestamp (e.g. `GET /log/fruit.sweet?ts=20200101101010`).

You can extend the functionality easily by configuring Objects - for example defining that certain objects are queued. A [Message Queue] provides guaranteed delivery to at least one consumer. To do this, simply set `queued: true` on the object (e.g. `fruit`). Now, all `fruit.*` messages will be added to the `fruit` queue which is a separate index in ElasticSearch. Consumers can now pop all messages off the queue like this: `GET /queue/fruit/`. They can also pull only specific messages. If you choose to PUSH to consumers just set `push: true` on the fruit object and QueTipp will POST messages to subscribers.

## Terminology:
* Producer: Generator/sender of messages (HTTP POST to QueueTipp)
  * Can specify OBJECT, TOPICS, expiration of message
  * Knows nothing of consumers or routing
* Consumer: Receiver/processor of messages (PUSH/PULL from QueueTipp)
  * Can PULL "anonymously" from the Message Log without subscribing


You can use QueTipp as a message queue (like RabbitMQ) or as a transaction/event log (like Kafka).
Pub/Sub is also available which let's consumers subscribe to get the data they want.

This means **Consumers** can easily pull messages off the queue or read messages from a given offset.

## Features
* A list of messages with information about delivery
* Change detection to de-duplicate messages
* Pub/sub functionality
* Queue or log mode operation
* Push/pull to/by consumers
* Custom routing function (JavaScript)
* Custom message transformation functions

## Customization
* Define your own routing rules with JavaScript
* Define message mapping functions for consumers

## Topics and Routing
Each message can be published to one or more **Topics** (e.g. `customer.update`). QueTipp will take the root of the topic (in this case `customer`) and create a **Queue** called `customer` for all such messages.

Consumers subscribe to one or more topics either by:
* Pull: Polling QueTipp for new messages in topics
* Push: Being called by QueTipp with new messages

## Message Lifecycle
A new message may go through the following state changes:
* State `0`: New, unprocessed message
* State `P`: Message in process
* State `1`: Message delivery failed once
* State `2`: Message delivery failed twice (etc.)
* State `X`: Message was successfully delivered

There are a couple more states which are useful:
* State `U`: QueTipp determined the message was unchanged. Compacting and de-duplication is discussed below.
* State `N`: The message was rejected by the consumer as non-relevant. The message is considered processed but the consumer does not want it.


## Message De-duplication
An unhelpful producer may produce multiple copies of the same message. We do not want to pass the same message to consumers again and again if nothing has changed. Sometimes neither the producer nor the consumer is able to handle this so QueTipp compares the message (by `object` and `key`) to see if anything has changed.

TODO: We should have a policy on how far back we consider a message duplicated.