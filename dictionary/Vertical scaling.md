---
description: Handling more load by giving one machine more CPU, memory or faster storage, also called scaling up.
---

Vertical scaling means handling more load by giving one machine more resources: more CPU cores, more memory, faster disks or a faster network card. It is also called scaling up. On a cloud provider it can be as small a change as selecting a larger instance type.

The main advantage is that the software usually does not change. There is no data to split, no coordination between machines, and no new failure modes from network calls. For databases in particular, moving to a larger machine is often the sensible first step, because the alternative of splitting data across servers adds lasting complexity.

Vertical scaling has limits that eventually force a different approach:

| Limit            | What happens                                                                               |
| ---------------- | ------------------------------------------------------------------------------------------ |
| Hardware ceiling | There is a largest machine you can buy or rent                                             |
| Cost curve       | The largest machines can cost more per unit of capacity                                    |
| Resize downtime  | Moving to a larger machine often requires a restart                                        |
| Availability     | One large machine is still a [single point of failure](./Single%20point%20of%20failure.md) |
| Software limits  | Locks or single-threaded code may not use the added cores                                  |

The last limit is a common surprise. A team moves a service to a 64-core machine and sees no improvement, because one thread is pinned at 100% while the rest sit idle. Before resizing, confirm which resource is the [bottleneck](./Bottleneck.md). More cores do nothing for a service limited by a lock, and more memory does nothing for one limited by disk writes.

Most production systems combine both directions: [stateless](./Stateless%20service.md) application servers grow through [horizontal scaling](./Horizontal%20scaling.md), while the database grows vertically for as long as that remains practical.

_Usage:_

"Do we need to shard the orders database this quarter?"

"Not yet. Vertical scaling buys us a year or so. Move it to the next instance size and revisit when we're near the top of the range."
