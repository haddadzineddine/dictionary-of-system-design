---
description: A cache that confirms a write immediately and saves it to the database later, trading durability for write speed.
---

A write-behind [cache](./Cache.md) confirms a write as soon as it holds the value, then saves it to the database later in the background. It is also called a write-back cache. The caller gets a fast response, and the database receives the change some time afterwards.

The three common write strategies differ in when the database is updated:

| Pattern                                     | Database updated     | Write speed               | Risk of data loss |
| ------------------------------------------- | -------------------- | ------------------------- | ----------------- |
| [Cache-aside](./Cache-aside.md)             | By the app, on write | Database speed            | Low               |
| [Write-through](./Write-through%20cache.md) | Before confirming    | Database speed plus cache | Low               |
| Write-behind                                | After confirming     | Cache speed               | Higher            |

Delaying the write allows batching and combining. If a view counter is incremented 1,000 times in ten seconds, the cache can send one update with the final value. This reduces database load, which makes the pattern attractive for a [write-heavy workload](./Write-heavy%20workload.md) such as counters, analytics or game state.

The cost is durability. A write that was confirmed to the user exists only in the cache until it is flushed. If that node dies first, the write is gone, even though the client was told it succeeded. Replicating the cache or keeping a durable log of pending writes reduces the risk but does not make the pattern equal to a synchronous database commit.

Other problems follow from the delay. Anything reading the database directly, such as a report, sees old data. If the database is down or slow, unflushed writes accumulate in memory. Ordering between related records can also break when they are flushed separately.

Use this pattern for data you could afford to lose or recompute. Do not use it for payments, orders or anything else that a user has been promised is saved.

_Usage:_

"Can we buffer the like counts in Redis and flush every few seconds?"

"Yes, write-behind is fine there. Losing a few likes in a crash is acceptable, unlike losing an order."
