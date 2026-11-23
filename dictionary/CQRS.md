---
description: Command Query Responsibility Segregation, a pattern that uses separate models for changing data and for reading it.
---

CQRS, or Command Query Responsibility Segregation, is a pattern that uses separate models for changing data and for reading it. Commands such as `PlaceOrder` go through a write model that enforces business rules. Queries such as "show this customer's order history" go through a read model shaped for that screen.

The motivation is that the two sides often want different structures. A write model benefits from normalised tables and strict validation. A dashboard wants wide, denormalised rows it can fetch in one query. Forcing both through one model produces a domain object with dozens of fields and a reporting query that joins nine tables and slows checkout on the same database.

| Concern | Command side                 | Query side                                                                                            |
| ------- | ---------------------------- | ----------------------------------------------------------------------------------------------------- |
| Purpose | Change state                 | Return data                                                                                           |
| Model   | Business rules, invariants   | Shaped per screen or report                                                                           |
| Storage | Often normalised             | Often denormalised or a search index                                                                  |
| Scaling | Limited by write correctness | Copies [scale out](./Horizontal%20scaling.md) for a [read-heavy workload](./Read-heavy%20workload.md) |
| Returns | Success or failure           | Data, no side effects                                                                                 |

CQRS has levels. The simplest keeps one database and uses separate code paths for reads and writes, which adds little cost. A fuller version has separate read stores, updated from the write side through events, often using the [outbox pattern](./Outbox%20pattern.md).

Separate stores introduce replication lag, and this is the failure users notice. Someone saves a change, the page reloads from the read model, and the old value appears, so they save again. Common responses are to show the submitted value optimistically in the interface, or to have the client wait until the read model reaches a given version.

CQRS is often paired with [event sourcing](./Event%20sourcing.md), but neither requires the other. It adds moving parts, so for simple create-read-update-delete screens, a single model is usually the right choice.

_Usage:_

"The analytics page keeps locking the orders table during checkout."

"Use CQRS for that area. Project orders into a reporting table and let the page read from there."
