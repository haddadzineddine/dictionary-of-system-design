---
description: A buffer that holds messages from producers until a consumer processes them, so the two do not need to run in step.
---

A message queue is a buffer that holds messages from producers until a consumer processes them. The producer adds a task and moves on, without waiting for the work to finish. A checkout service can queue "send receipt" and respond to the customer at once, even if the email provider is slow or briefly down.

Each message is normally handled by one consumer, and several consumers can share a queue to work through it faster:

| Step | Who      | What happens                                                  |
| ---- | -------- | ------------------------------------------------------------- |
| 1    | Producer | Sends a message to the broker, the server that runs the queue |
| 2    | Broker   | Stores it                                                     |
| 3    | Consumer | Receives it, and the broker hides it from other consumers     |
| 4    | Consumer | Does the work                                                 |
| 5    | Consumer | Acknowledges it, and the broker removes it                    |
| 6    | Broker   | Redelivers it if no acknowledgement arrives in time           |

Step 6 means most queues provide at-least-once delivery, so a message can arrive twice. If a consumer finishes sending an email and crashes before acknowledging, another consumer sends it again, and the customer receives two receipts. Consumers should be idempotent, meaning that processing a message twice has the same effect as processing it once. A common method is to record each message ID and skip those already handled.

A job that runs longer than the redelivery window causes the same duplication, because the broker assumes the consumer died. Extend the window for long jobs.

Queue depth is the number to watch. A depth that keeps growing means producers are outrunning consumers, and users see work delayed by minutes or hours while every service reports healthy. Add consumers, or apply [backpressure](./Backpressure.md). Messages that fail repeatedly belong in a [dead letter queue](./Dead%20letter%20queue.md).

RabbitMQ and Amazon SQS are examples. To deliver one message to several independent consumers, use [publish-subscribe](./Publish-subscribe.md).

_Usage:_

"Customers are getting their invoice PDF two or three times."

"The job takes longer than the message queue's visibility [timeout](./Timeout.md), so it gets redelivered while still running."
