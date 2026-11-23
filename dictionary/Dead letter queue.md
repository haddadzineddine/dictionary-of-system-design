---
description: A holding queue for messages that failed processing too many times, so they stop blocking other work and can be inspected.
---

A dead letter queue is a holding queue for messages that could not be processed after a set number of attempts. It removes the failing message from the main flow while keeping it for inspection.

The usual reason for needing one is a poison message: a message that always fails, such as one with a malformed field or a reference to a deleted record. In a [message queue](./Message%20queue.md) with automatic redelivery, the consumer picks it up, crashes, restarts and picks it up again. Logs fill with one repeated stack trace, and in an ordered system the messages behind it stop moving.

| Step | Who      | What happens                                                |
| ---- | -------- | ----------------------------------------------------------- |
| 1    | Consumer | Fails to process a message                                  |
| 2    | Broker   | Redelivers it and counts the attempt                        |
| 3    | Broker   | Moves it to the dead letter queue once the limit is reached |
| 4    | Alerting | Notifies the team that the queue is no longer empty         |
| 5    | Engineer | Finds the cause and fixes the code or the data              |
| 6    | Engineer | Redrives the message to the main queue, or discards it      |

An attempt limit that is too low sends thousands of good messages to the dead letter queue during a brief dependency outage. A consumer that recognises a permanent error, such as a validation failure, can send the message there at once, without further [retries](./Retry%20with%20backoff.md).

The common mistake is creating one and never watching it. Messages accumulate unnoticed until retention deletes them. Alert on its depth, and store the error reason and original metadata with each message.

Check ordering before redriving, because a replayed old update can overwrite newer data. Amazon SQS and RabbitMQ support this pattern directly. With Apache Kafka it is usually implemented in the consumer or its framework.

_Usage:_

"The worker has been crash-looping on the same order for an hour."

"That's a poison message. Set a max receive count so it lands in the dead letter queue and the rest can flow."
