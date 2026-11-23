---
description: A caching pattern where the application checks the cache, loads from the database on a miss, and stores the result.
---

[Cache](./Cache.md)-aside is a caching pattern in which the application manages the cache itself, loading data into it only when that data is requested. It is also called lazy loading. The cache sits beside the database and has no connection to it.

| Step | Path  | What the application does                     |
| ---- | ----- | --------------------------------------------- |
| 1    | Read  | Looks up the key in the cache                 |
| 2    | Read  | On a hit, returns the value                   |
| 3    | Read  | On a miss, queries the database               |
| 4    | Read  | Stores the result in the cache and returns it |
| 5    | Write | Updates the database                          |
| 6    | Write | Deletes the key from the cache                |

The pattern has useful properties. Only requested data takes up cache memory, and if the cache goes down, reads can still reach the database, at the price of more load on it.

Writes delete the key instead of updating it because concurrent updates can land in the wrong order and leave an old value in place. Deleting forces the next reader to load the current value.

A race remains. Reader A misses and fetches the old row. Writer B updates the database and deletes the key. Reader A then stores the old value it fetched earlier. The cache now holds stale data with nothing scheduled to remove it, which is how a record can stay wrong for hours and then correct itself. Always set a [TTL](./TTL.md) as a backstop, so that any such error has a limited lifetime.

Each miss costs three trips: cache, database, cache. After a deploy or restart everything misses at once, and a popular key that expires can cause a [cache stampede](./Cache%20stampede.md). For data that must be current right after a write, compare a [write-through cache](./Write-through%20cache.md).

_Usage:_

"One customer's plan shows the old tier, but the database row is right."

"That's the cache-aside race. The TTL is a week, so shorten it and delete that key by hand."
