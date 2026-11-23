---
description: A measurable statement of how well a system must work, covering qualities like speed, availability, durability and cost.
---

Where a [functional requirement](./Functional%20requirement.md) says what a system must do, a non-functional requirement says how well it must do it. For "users can search products", the non-functional version is "95% of searches return within 200 ms at 5,000 searches per second".

These requirements shape the architecture more than the feature list does. A photo-sharing app for 50 people and one for 50 million have similar features and very different designs. A system that works in a demo and times out on launch day usually met its functional requirements and had no stated target for load.

| Quality                       | Example target                      | Design consequence                                                           |
| ----------------------------- | ----------------------------------- | ---------------------------------------------------------------------------- |
| [Latency](./Latency.md)       | 99% of page loads under 500 ms      | [Caches](./Cache.md), fewer network hops                                     |
| [Throughput](./Throughput.md) | 5,000 orders per second at peak     | [Horizontal scaling](./Horizontal%20scaling.md), queues                      |
| Availability                  | 99.9% of requests succeed monthly   | Redundancy, no [single point of failure](./Single%20point%20of%20failure.md) |
| Durability                    | A confirmed payment is never lost   | Replicated storage, backups                                                  |
| Consistency                   | Balance is correct after a transfer | Limits on caching and asynchronous updates                                   |
| Cost                          | Under a set monthly budget          | Smaller fleet, simpler components                                            |

Targets need numbers to be useful. "The system must be fast" cannot be tested and cannot settle an argument. A number also shows what a goal costs. An availability target of 99.9% allows about 43 minutes of downtime in a 30-day month, while 99.99% allows about 4 minutes, which is too short for a person to respond to most failures and generally requires automated failover.

The qualities also compete. Stronger consistency tends to raise latency, and higher availability raises cost. Ask which requirement wins when two conflict, because that answer decides many later design choices. A payment system and a social feed will answer differently.

_Usage:_

"Do we really need multi-region for the internal reporting tool?"

"Check the non-functional requirements. It needs 99.5% during office hours, so one region with a tested restore is enough."
