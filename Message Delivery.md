## Message Delivery
The main function of QueTipp is to ensure messages are accepted and delivered so we have several processes:
* Producer Endpoint: Listen for new messages and store them
* Consumer Endpoint: Listen for consumers requesting/confirming messages and provide/update them from/in the queue.
* Push Service: Periodically send messages to registered consumer endpoints.

## Message Parsing
QueTipp is agnostic about content of the message payload and will neither parse, index nor validate it. There are really only a few things QueTipp stores about the message:
* The `topics[]` the producer defined (e.g. `["customer.create", "customer.changed"]`)
* The optional `object` the producer defined (e.g. `customer`) to determine the queue.
* The `mimeType` of the message (e.g. `application/json` or `text/plain`)
* The optional `timestamp` header of the message which the producer may provide indicating when the data was created in the producer
* The optional `sent` header indicating when the producer started sending the message to QueTipp.

## Limitations
A producer may pass in a single object which represents the data being passed. This has 2 advantages:
* The message is indexed to it's own dedicated ElasticSearch index which may help scalability
* QueTipp can check for duplicate messages by comparing the hash of the payload against the last message processed for this `object`+`key`.

The message can still be published to multiple `topics`.

## Type Inferrence
If the option `inferTypes` is set, QueTipp will automatically infer the object type from the root element of the *first* topic:
* `topic: ['customer.create', 'audit.log']` => `object: 'customer`

## Queues and Logs
Each message exists only once inside QueTipp i.e. is stored in only one ElasticSearch index.

## Single Deliveries
Unless otherwise configured, a message is marked as processed as soon as it is confirmed by at least one consumer.
* PULL: If two consumers fetch and then confirm the same message, the first one "wins" and the second confirmation is discarded.
* PUSH: QueTipp will only send to one consumer at a time, switching to another consumer only if the previous consumer failed to confirm delivery.

## Delivery per Topic
If configured a message can be delivered to as many consumers as it has topics. This is useful when the topics correspond to known consumers and the producer wants to target each consumer with the message. In this case QueTipp will consider a message with N topics processed when N consumers have confirmed it.
// TODO: Where to configure delivery-per-topic?

## Routing
Could we have a userland JavaScript function which looks at each message and decides how to route it.
```
function router(message, consumers) {
  let route = {
    option: 'once' | 'onceOnly`
  }
  if (message.topics.contains("foo"))
    // Delivery message to 1 consumer per topic
    route.option = 'oncePerTopic`;
  if (message.object === 'log') {
    // Direct all log messages to the log queue for once-only delivery
    route.
    route.consumers.push("clogger");
  }
  return route;
}
```
This function would ideally know which consumers are interested in which topic and assign the consumers to the message as it's queued.

## Technical Details

### Producer Endpoint
ExpressJS route `/in/:obj/:topic` for object queues or `/in/:topic` for the default queue.


Query `_search` with `"seq_no_primary_term": true` to get the `_seq_no` of each message so we can lock it.