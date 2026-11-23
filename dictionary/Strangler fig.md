---
description: Replacing a legacy system gradually by routing one feature at a time to a new system until the old one can be removed.
---

The strangler fig is a migration pattern that replaces a legacy system gradually, by routing one feature at a time to a new system until the old one can be removed. The name comes from a plant that grows around a host tree and can eventually replace it.

The alternative is a full rewrite, and its failure is a familiar one. A team spends a long time rebuilding the product while the old system keeps gaining features, so the target keeps moving, nothing reaches users until the end, and the cutover is a single high-risk event. This pattern ships in small, reversible steps.

| Step | What happens                                                                                                                          |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Put a routing layer, such as a [reverse proxy](./Reverse%20proxy.md) or [API gateway](./API%20gateway.md), in front of the old system |
| 2    | Pick one well-bounded feature and build it in the new system                                                                          |
| 3    | Send a small share of its traffic to the new version                                                                                  |
| 4    | Compare results and error rates, and increase the share                                                                               |
| 5    | Delete the old code for that feature                                                                                                  |
| 6    | Repeat, then remove the routing layer when nothing old remains                                                                        |

Rollback at step 3 or 4 is a routing change. Shadowing helps before that: the new version receives a copy of live requests, and its responses are compared but not returned.

Data is usually the hardest part. Options include having the new service call the old database temporarily, syncing changes between stores, or writing to both during a transition. Decide which system is the source of truth for each entity at each stage.

The common failure is stopping halfway. Once the easy features have moved, the hard ones remain, priorities shift, and the team now runs two systems indefinitely. Track the traffic still served by the old system.

_Usage:_

"Can we rewrite the billing [monolith](./Monolith.md) in one go over the next year?"

"Too risky. Use a strangler fig: put a proxy in front and move invoices first."
