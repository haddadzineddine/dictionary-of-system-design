---
description: The one resource or component that limits a system's overall throughput, so that improving anything else changes nothing.
---

A bottleneck is the resource or component that limits how much work a whole system can do. A request path can only move as much work as its most constrained step allows. If the web tier handles 10,000 requests per second and the database handles 2,000, the system's [throughput](./Throughput.md) is 2,000.

The usual symptom is that adding capacity has no effect. A team doubles the application servers, and the graphs do not move, because every server is waiting on the same saturated database. In some cases things get worse, since more servers open more connections to the component that is already struggling.

To find a bottleneck, look for the place where work piles up:

| Step | What to check                                                                       |
| ---- | ----------------------------------------------------------------------------------- |
| 1    | Trace a slow request and see which stage takes the time                             |
| 2    | Check utilisation of CPU, memory, disk and [bandwidth](./Bandwidth.md) on each tier |
| 3    | Look for growing queues, full connection pools and lock waits                       |
| 4    | Fix the limiting resource                                                           |
| 5    | Load-test again, because the limit will now sit somewhere else                      |

Step 5 matters because removing one bottleneck exposes the next. Add a [cache](./Cache.md) in front of the database and the limit may move to the network or to a single busy partition.

A bottleneck is not always a whole machine. It can be one lock, one [hot key](./Hot%20key.md), a single-threaded process using one CPU while the others sit idle, or an external API with a quota. Average utilisation can hide all of these, so inspect individual instances and resources.

Fixes depend on the cause. Options include doing less work per request, adding capacity, splitting the load, or moving slow work to a [message queue](./Message%20queue.md) so that it no longer holds up the request.

_Usage:_

"We scaled the API to twenty pods and [latency](./Latency.md) didn't change."

"Then the API isn't the bottleneck. The database has been pinned at 100% CPU since this morning."
