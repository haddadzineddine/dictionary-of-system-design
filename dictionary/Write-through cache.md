---
description: A cache where each write goes to both the cache and the database before the write is confirmed to the caller.
---

A write-through [cache](./Cache.md) is one where each write goes to both the cache and the backing database before the caller receives confirmation. Because the two are updated as part of one operation, a read that follows a successful write finds the new value in the cache.

| Step | Who      | What happens                               |
| ---- | -------- | ------------------------------------------ |
| 1    | App      | Sends the write to the cache layer         |
| 2    | Cache    | Stores the new value                       |
| 3    | Cache    | Writes the value to the database and waits |
| 4    | Database | Confirms the write                         |
| 5    | Cache    | Confirms success to the app                |

Some products provide this behaviour directly. With a plain store such as Redis, the application usually performs both writes itself, which gives weaker guarantees, because a crash between the two leaves them different.

This pattern addresses a common complaint with [cache-aside](./Cache-aside.md): a user saves a change and the next read is a miss or returns something old. Recently written data is already cached, so reads of recent data are fast and current.

The costs fall on the write path. Each write now waits for two systems, so write [latency](./Latency.md) is at least that of the database, and it does nothing to reduce database write load. A [write-heavy workload](./Write-heavy%20workload.md) gets little benefit. The cache also fills with data that may never be read, such as records created once and left alone, so pair it with a [TTL](./TTL.md) or an [eviction policy](./Eviction%20policy.md).

Failure handling needs a decision. If the database write fails, the cached value must be removed or never exposed, or readers will see data that was never saved. Other systems that write directly to the database also bypass the cache and leave it stale, so route all writes through the same path.

_Usage:_

"Users edit their profile and sometimes see the old name for a few seconds."

"Profile reads vastly outnumber edits, so a write-through cache fits. The extra few milliseconds on save won't matter."
