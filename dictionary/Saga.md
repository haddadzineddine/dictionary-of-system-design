---
description: A multi-service operation run as a series of local transactions, with compensating steps to undo earlier work on failure.
---

A saga is a multi-service operation run as a series of local transactions, with a compensating action to undo each completed step if a later one fails. It is used when one business action spans several databases, as is common in [microservices](./Microservices.md), and a single atomic transaction is unavailable or impractical.

Placing an order might look like this:

| Step | Service   | Action            | Compensation   |
| ---- | --------- | ----------------- | -------------- |
| 1    | Orders    | Create as pending | Cancel order   |
| 2    | Inventory | Reserve stock     | Release stock  |
| 3    | Payments  | Charge card       | Refund         |
| 4    | Shipping  | Book courier      | Cancel booking |

If payment fails at step 3, the saga runs the compensations for steps 2 and 1 in reverse: release the stock, then cancel the order. If the courier booking fails at step 4, it refunds the card first. Without this logic, the familiar result is a customer who was charged for an order that stays stuck.

There are two ways to coordinate the steps:

| Style         | How it works                                | Trade-off                                  |
| ------------- | ------------------------------------------- | ------------------------------------------ |
| Orchestration | A coordinator tells each service what to do | Flow is visible, coordinator is extra code |
| Choreography  | Each service reacts to the previous event   | No coordinator, flow is hard to follow     |

Compensation is not a true rollback. A refund is a new transaction, and the customer may see both the charge and the refund on a statement. Place steps that are hard to undo as late as possible.

Sagas also lack isolation, so other requests can see intermediate state, such as stock that is reserved and then released. Mark such records as pending.

Messages can be redelivered, so every step and every compensation must be idempotent, meaning safe to repeat. Use the [outbox pattern](./Outbox%20pattern.md) to publish each step reliably, and persist the saga's state so it can resume after a crash.

_Usage:_

"Payment succeeded, but the courier booking failed and the order is stuck in pending."

"The saga isn't compensating for that failure. It should refund, release the stock and cancel automatically."
