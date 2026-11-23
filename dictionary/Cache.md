---
description: A fast store holding copies of data so that repeated requests can avoid a slower source such as a database.
---

A cache is a fast store that holds copies of data so that repeated requests can avoid a slower source, such as a database, a disk or a remote service. When the requested item is present, that is a hit. When it is absent, that is a miss, and the data must be fetched from the source. Caching works because access is usually uneven, with a small share of the data receiving a large share of the requests.

Caches exist at several levels of a typical system:

| Level                                 | Where it lives            | Typical contents             |
| ------------------------------------- | ------------------------- | ---------------------------- |
| Browser                               | User's device             | Images, scripts, stylesheets |
| [CDN](./CDN.md)                       | Edge locations near users | Static files, public pages   |
| [Reverse proxy](./Reverse%20proxy.md) | In front of app servers   | Whole HTTP responses         |
| In-process                            | Application memory        | Config, small lookup tables  |
| Distributed                           | Redis or Memcached        | Sessions, query results      |
| Database                              | Database memory           | Recently used data pages     |

Every cache design answers three questions. How does data get in, for example [cache-aside](./Cache-aside.md) or a [write-through cache](./Write-through%20cache.md)? How long does it stay, as set by a [TTL](./TTL.md) and an [eviction policy](./Eviction%20policy.md)? How is it corrected when the source changes, which is [cache invalidation](./Cache%20invalidation.md)?

Two failures are common. The first is stale data: a user changes their address, and the old one keeps appearing because a cached copy outlived the change. The second is hidden dependence. A database is sized for the traffic left over after the cache, so when the cache restarts empty, the database receives the full load in a [cache stampede](./Cache%20stampede.md) and can go down. A cache that was added to improve speed has become necessary for availability.

Treat a cache as a copy that can vanish at any time, and track its [cache hit ratio](./Cache%20hit%20ratio.md). Test what happens when it is empty.

_Usage:_

"Can we restart the Redis cluster during the day?"

"Not safely. The cache absorbs about 95% of reads, and the database can't take that load cold."
