---
description: The share of lookups a cache answers without going to the source, calculated as hits divided by total lookups.
---

[Cache](./Cache.md) hit ratio is the share of lookups that a cache answers without going to the underlying source. It is calculated as hits divided by hits plus misses. A cache that answers 950 of 1,000 lookups has a hit ratio of 95%.

The number that matters for capacity is the miss ratio, because misses are what reach the database. Small changes near the top of the scale have large effects:

| Hit ratio | Requests reaching the database, per 10,000 |
| --------- | ------------------------------------------ |
| 90%       | 1,000                                      |
| 95%       | 500                                        |
| 99%       | 100                                        |
| 99.9%     | 10                                         |

A drop from 99% to 98% looks minor on a dashboard, yet it doubles database load. This is why a small change, such as a new field in the cache key or a shorter [TTL](./TTL.md), can produce a database incident with no increase in traffic.

Hit ratio also drives average [latency](./Latency.md). If a hit takes 1 ms and a miss costs 1 ms for the check plus 50 ms for the database, a 90% hit ratio averages 6 ms and a 99% ratio averages 1.5 ms.

Several things commonly lower the ratio: keys that include a timestamp or request ID, so that nothing repeats; a cache too small for its working set, the data in regular use; and restarts or deploys that leave the cache empty. Adding nodes with naive hashing has the same effect, which [consistent hashing](./Consistent%20hashing.md) avoids.

Read the number with care. A single overall ratio can hide one important endpoint at 40% behind many trivial lookups at 99.9%, so measure it per key type. A high ratio also says nothing about whether the data served is correct.

_Usage:_

"Database CPU doubled after the release, but traffic is flat."

"Cache hit ratio fell from 99 to 98. The new locale field in the key split every entry."
