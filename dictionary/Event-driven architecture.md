---
description: A design where components announce what has happened through events, and other components react without being called directly.
---

Event-driven architecture is a design in which components announce what has happened by emitting events, and other components react to those events without being called directly. An event is a record of a past fact, such as `OrderPlaced` or `PaymentFailed`. The sender does not know or care who consumes it.

Events differ from commands, and confusing them leads to tightly coupled designs:

| Aspect       | Command                  | Event                   |
| ------------ | ------------------------ | ----------------------- |
| Meaning      | "Do this"                | "This happened"         |
| Tense        | Imperative: `ChargeCard` | Past: `CardCharged`     |
| Recipients   | One specific handler     | Zero, one or many       |
| Sender knows | Who will do the work     | Nothing about consumers |
| Can refuse?  | Yes                      | No, it already happened |

Events usually travel through [publish-subscribe](./Publish-subscribe.md) or [event streaming](./Event%20streaming.md). The benefit is loose coupling. To send a welcome gift when a user signs up, add a consumer of `UserSignedUp` and leave the signup service alone. If a consumer is down, the rest of the system keeps running and it can catch up later.

The costs show up in understanding and debugging. There is no single place to read the flow of a business process, so "what happens after an order is placed?" requires tracing several consumers. When a step silently does not happen, there may be no error anywhere. Attach a correlation ID (one identifier carried through every related event) and use tracing to reconstruct what occurred.

Results are also eventually consistent, meaning different parts of the system catch up at different times. A customer places an order and their order history does not show it for a few seconds. The interface has to allow for that.

Consumers must handle duplicate and out-of-order events. Publish reliably with the [outbox pattern](./Outbox%20pattern.md), and treat event schemas as contracts, because you may not know everyone who depends on them.

_Usage:_

"Do we have to touch checkout to add loyalty points?"

"No, that flow uses event-driven architecture. Loyalty can consume `OrderPaid` without checkout knowing it exists."
