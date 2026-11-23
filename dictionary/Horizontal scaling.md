---
description: Handling more load by adding more machines and spreading work across them, also called scaling out.
---

Horizontal scaling means handling more load by adding more machines and spreading the work across them. It is also called scaling out. Instead of one larger server, you run ten ordinary ones behind a [load balancer](./Load%20balancer.md), and add an eleventh when traffic grows.

The comparison with [vertical scaling](./Vertical%20scaling.md) is mostly a trade of simplicity for headroom:

| Concern         | Vertical                  | Horizontal                              |
| --------------- | ------------------------- | --------------------------------------- |
| Capacity limit  | Largest available machine | Much higher, depends on the design      |
| Code changes    | Usually none              | Must tolerate several running copies    |
| Machine failure | Full outage               | Lose a fraction of capacity             |
| Deploys         | Restart causes downtime   | Replace instances one at a time         |
| Operations      | One machine to manage     | Networking, discovery, partial failures |

Horizontal scaling works well for a [stateless service](./Stateless%20service.md), because any instance can answer any request. State is what makes it hard. If each server keeps sessions in its own memory, a user's next request can land on a server that has never seen them. Databases are harder still, since spreading writes means sharding (splitting data across machines by a key), which complicates queries and transactions.

A common disappointment is adding servers and getting no improvement. If every instance waits on one database, that database is the [bottleneck](./Bottleneck.md), and extra instances add connections to it. Scaling only helps the part that is actually constrained.

Costs also appear between machines. A process that kept a counter in memory now has several disagreeing counters, and a scheduled job runs once per instance unless something coordinates it. Check for assumptions like these before scaling a service that has only ever run as one copy.

_Usage:_

"Traffic doubles for the sale, can we add API pods?"

"Yes, the API scales horizontally. The checkout worker doesn't, because it keeps a lock in memory, so fix that before relying on horizontal scaling there."
