---
description: A workload where reads far outnumber writes, which makes caching and read copies effective ways to scale.
---

A read-heavy workload is one where reads far outnumber writes. Product catalogues, news sites, documentation and social feeds are typical examples, with ratios of 100 reads to one write being common. The ratio matters because reads and writes scale in different ways.

Reads are easier to scale because they do not change anything. The same answer can be copied to many places and served from any of them. The options form a ladder, and each rung adds some complexity:

| Option              | What it does                             | Trade-off                  |
| ------------------- | ---------------------------------------- | -------------------------- |
| Add indexes         | Lets the database find rows faster       | Slightly slower writes     |
| [Cache](./Cache.md) | Serves repeated reads from memory        | Data can be stale          |
| [CDN](./CDN.md)     | Serves content from locations near users | Harder to purge            |
| Read replicas       | Database copies that answer read queries | Copies can lag the primary |
| Precomputed views   | Stores the finished answer ahead of time | Extra work on each write   |

The usual symptom of an unaddressed read-heavy workload is a database at high CPU while running the same few queries thousands of times per second. A [cache-aside](./Cache-aside.md) layer often removes most of that load.

The price of these techniques is staleness. A read replica applies changes shortly after the primary, so a user can save a profile, reload and see the old value because the second request hit a copy that has not caught up. For that case, send a user's reads to the primary for a short period after they write, or read that user's own data from the primary.

Measure the ratio before relying on it. Some workloads that look read-heavy have write bursts, and a [write-heavy workload](./Write-heavy%20workload.md) calls for different techniques.

_Usage:_

"The catalogue database is at 90% CPU, should we shard it?"

"It's a read-heavy workload with about a thousand reads per write. Put a cache in front before splitting anything."
