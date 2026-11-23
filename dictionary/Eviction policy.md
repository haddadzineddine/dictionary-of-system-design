---
description: The rule a cache uses to choose which entries to remove when it runs out of memory.
---

An eviction policy is the rule a [cache](./Cache.md) uses to choose which entries to remove when it runs out of memory. Eviction differs from expiry. A [TTL](./TTL.md) removes data because it is old, while eviction removes data because space is needed, even if that data is still valid.

| Policy      | Removes                     | Works well when                   | Weakness                           |
| ----------- | --------------------------- | --------------------------------- | ---------------------------------- |
| LRU         | Least recently used entry   | Recent items tend to be reused    | One large scan flushes useful data |
| LFU         | Least frequently used entry | Popularity is stable              | Old favourites can linger          |
| FIFO        | Oldest inserted entry       | Items lose value with age         | Ignores how often items are used   |
| Random      | Any entry                   | Access is fairly uniform          | May remove popular items           |
| TTL-based   | Entry closest to expiry     | TTLs reflect value                | Depends on sensible TTLs           |
| No eviction | Nothing, new writes fail    | Data must not be silently dropped | Errors once memory is full         |

LRU's known weakness is scan pollution. A nightly export reads millions of rows once, each read counts as recent, and the entries people actually use are pushed out. The next morning the [cache hit ratio](./Cache%20hit%20ratio.md) is low and the database is busy until the cache refills. Fixes include having batch jobs bypass the cache, or using a scan-resistant policy such as an LFU variant.

Check the configured policy, because the failure mode depends on it. Redis's `noeviction` policy returns errors on writes once memory is full, which an application may experience as a sudden outage of the cache. If a store holds data with no other copy, such as a job queue, an evicting policy can delete it silently.

A rising eviction count alongside a falling hit ratio usually means the cache is too small for its working set.

_Usage:_

"Hit ratio tanks every morning around 3 a.m. and recovers by nine."

"That's the export job. Our LRU eviction policy treats every row it scans as recent, so make it skip the cache."
