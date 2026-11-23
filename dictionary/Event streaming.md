---
description: Recording events in a durable, ordered log that many consumers can read independently and replay from any position.
---

Event streaming records events in a durable, append-only log that many consumers read independently. Unlike a [message queue](./Message%20queue.md), reading an event does not remove it. Each consumer tracks its own position in the log, called an offset, and events stay until a retention limit is reached. Apache Kafka and Amazon Kinesis are examples.

| Concern        | Message queue              | Event stream                          |
| -------------- | -------------------------- | ------------------------------------- |
| After reading  | Message is removed         | Event stays in the log                |
| Consumer state | Broker tracks each message | Consumer tracks an offset             |
| Replay         | Generally not possible     | Rewind the offset and read again      |
| Ordering       | Often loose                | Guaranteed within a partition         |
| Typical use    | Task distribution          | Data pipelines, activity feeds, audit |

Replay is the main practical benefit. If a bug corrupts a derived table, you can fix the code, reset the offset and rebuild the table from history.

A stream is split into partitions so it can scale, and a partition key decides where each event goes. Order is only guaranteed inside a partition. If events for one order are keyed randomly, "order shipped" can be processed before "order paid", which produces failures that are hard to reproduce. Key by the entity whose order matters, such as the order ID. Avoid low-variety keys, which concentrate traffic in the same way as a [hot key](./Hot%20key.md).

Consumers share partitions through a [consumer group](./Consumer%20group.md). The health metric is consumer lag, the distance between the newest event and a consumer's offset. Growing lag means the consumer is falling behind and downstream data is getting older.

Retention sets a hard limit. If a consumer is down for longer than the retention period, the events it missed are deleted before it can read them. Delivery is typically at-least-once, so consumers must tolerate duplicates.

_Usage:_

"We shipped a bug that miscalculated revenue for the last three days."

"Event streaming saves us here. Fix the consumer, reset its offset to Monday and rebuild the table."
