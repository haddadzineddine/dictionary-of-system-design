---
description: A messaging pattern where publishers send to a named topic and every subscriber to that topic receives its own copy.
---

Publish-subscribe, or pub/sub, is a messaging pattern in which publishers send messages to a named topic and every subscriber to that topic receives its own copy. The publisher does not know who is listening. When an order is placed, the order service publishes one `order.placed` message, and the email, analytics and warehouse services each receive it.

The difference from a [message queue](./Message%20queue.md) is who gets each message:

| Concern           | Message queue            | Publish-subscribe                 |
| ----------------- | ------------------------ | --------------------------------- |
| Delivery          | One consumer per message | Every subscriber gets a copy      |
| Purpose           | Share out work           | Broadcast that something happened |
| Adding a consumer | Splits the existing work | Adds another independent copy     |
| Typical message   | "Resize this image"      | "An image was uploaded"           |

The main benefit is decoupling. A new service can react to orders by subscribing, with no change to the order service. This makes pub/sub a common building block of [event-driven architecture](./Event-driven%20architecture.md). The two patterns are often combined, with each subscriber having its own queue so that it can process its copy at its own pace and retry on failure.

Durability varies by product, and this catches people out. With Redis Pub/Sub, a subscriber that is disconnected when a message is published misses it permanently. A service restarts during a deploy, and the orders published in those seconds never reach it, with no error anywhere. Systems such as Google Cloud Pub/Sub retain messages for each subscription until they are acknowledged or a retention period ends. If missing a message matters, choose a durable option.

Decoupling also hides dependencies. A publisher can change a field in its message and break a consumer its team did not know existed. Keep a record of who subscribes to what, and treat message formats as a public contract.

_Usage:_

"Finance wants a copy of every refund. Do we need to change the payments service?"

"No. It already publishes refund events, so with publish-subscribe they add a subscription."
