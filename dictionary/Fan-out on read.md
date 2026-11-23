---
description: Assembling a user's feed at the moment they request it, by fetching and merging recent content from each source.
---

[Fan-out](./Fan-out.md) on read means assembling a user's feed at the moment they request it. Nothing is prepared in advance. When a user opens the app, the service looks up who they follow, fetches recent posts from each account, merges them by time or rank, and returns the result. It is also called the pull model, in contrast to [fan-out on write](./Fan-out%20on%20write.md), the push model.

| Concern                      | Fan-out on write           | Fan-out on read          |
| ---------------------------- | -------------------------- | ------------------------ |
| Cost of posting              | High, grows with followers | Low, one insert          |
| Cost of reading              | Low, one lookup            | High, grows with follows |
| Read [latency](./Latency.md) | Low and steady             | Higher and more variable |
| Storage                      | One copy per follower      | One copy of each post    |
| Freshness                    | Delayed by the queue       | Current at read time     |
| Inactive users               | Work wasted on them        | Cost nothing             |
| Very large accounts          | Expensive to post          | No extra cost            |

Fan-out on read suits data that is written often and read rarely, and products where most users are inactive on a given day. Deleting a post also takes effect immediately, since no copies exist.

The symptom of this design under load is a slow home screen for the most connected users. Someone who follows 5,000 accounts triggers 5,000 lookups, or a large query across many partitions, every time they refresh. Because the request waits on many sources, it is exposed to the tail-latency effect of fan-out, where one slow partition delays the whole response.

Common mitigations are to [cache](./Cache.md) the assembled feed for a short time, cap how many sources are read per request, and fetch only the first page. Many large products combine the two approaches: precompute timelines for ordinary accounts, and pull posts from very large accounts during the read.

_Usage:_

"Should we precompute notifications for every workspace member?"

"Most members never open that tab. Use fan-out on read and cache the result for a minute."
