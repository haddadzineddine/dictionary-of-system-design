---
description: Isolating resources into separate pools so that a failure or overload in one part cannot consume capacity for the rest.
---

A bulkhead is an isolation technique that divides resources into separate pools, so that a failure or overload in one part cannot use up capacity needed by the rest. The name comes from the watertight compartments in a ship's hull, which keep a breach from flooding the whole vessel.

The problem it solves is a shared pool. Suppose a service has 200 worker threads and calls three dependencies. One dependency becomes slow, its calls start holding threads, and soon all 200 are waiting on it. Requests that have nothing to do with the slow dependency now fail too. Users find login broken because the PDF report service is slow, and the two features appear unrelated.

With a bulkhead, the slow dependency can only exhaust its own allocation:

| Pool    | Threads | When reports hang                     |
| ------- | ------- | ------------------------------------- |
| Login   | 80      | Unaffected                            |
| Search  | 80      | Unaffected                            |
| Reports | 40      | Full, and extra requests are rejected |

Isolation can be applied at several levels:

| Level           | Example                                                |
| --------------- | ------------------------------------------------------ |
| Concurrency     | A semaphore capping in-flight calls to one dependency  |
| Thread pool     | A separate pool for each downstream service            |
| Connection pool | Separate database pools for web traffic and batch jobs |
| Instance        | Dedicated servers for admin or reporting endpoints     |
| Tenant          | A separate cluster for a very large customer           |
| Infrastructure  | Separate availability zones (isolated sites)           |

The trade-off is reduced efficiency. Capacity reserved for one pool sits idle while another is full, so sizes must be chosen from measured load and reviewed as traffic changes. Too small a pool rejects healthy requests during a normal peak.

Bulkheads work alongside other controls. A [timeout](./Timeout.md) frees a held resource, a [circuit breaker](./Circuit%20breaker.md) stops new calls to a failing dependency, and the bulkhead limits how much damage occurs before either acts. Together they help contain a [cascading failure](./Cascading%20failure.md).

_Usage:_

"A slow export took down the whole admin API again."

"They share one connection pool. Add a bulkhead so exports get their own small pool."
