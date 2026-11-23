---
description: Saving an outgoing event in the same database transaction as the data change, then publishing it from there afterwards.
---

The outbox pattern saves an outgoing event in the same database transaction as the data change, then publishes it to a message broker afterwards. It solves the dual-write problem, which arises when a service must update its database and also send a message, since those are two systems with no shared transaction.

Doing the two writes in sequence leaves a gap whichever order you choose. If the service commits the order and crashes before publishing, the order exists, but the warehouse never hears about it. If it publishes first and the commit then fails, the warehouse ships an order that does not exist. These bugs surface as a few records that get stuck with no error logged.

With an outbox, one database transaction covers both writes:

| Step | Who     | What happens                                        |
| ---- | ------- | --------------------------------------------------- |
| 1    | Service | Begins a database transaction                       |
| 2    | Service | Writes the business change, such as a new order row |
| 3    | Service | Inserts the event into an `outbox` table            |
| 4    | Service | Commits, so both are saved or neither is            |
| 5    | Relay   | Reads unpublished rows and sends them to the broker |
| 6    | Relay   | Marks them as published once the broker confirms    |

The relay either polls the table or uses change data capture, which reads the database's transaction log. Debezium is a common tool for the second approach.

The relay can crash between steps 5 and 6 and send the event again, so delivery is at-least-once. Consumers must therefore be idempotent, meaning a duplicate has no additional effect.

Delete or archive published rows, or the table grows without limit. Keep events for one entity in order if several relay workers run. It is often used to drive a [saga](./Saga.md) or feed [event-driven architecture](./Event-driven%20architecture.md).

_Usage:_

"A few orders each week never reach the warehouse, and there's no error."

"We commit and then publish, so a crash in between loses the event. Use the outbox pattern."
