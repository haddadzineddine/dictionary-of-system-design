---
description: Many concurrent requests missing the same cache entry at once and all hitting the database to rebuild it.
---

A [cache](./Cache.md) stampede happens when many concurrent requests miss the same entry and all go to the database to rebuild it. It is also called a thundering herd or dogpile. It usually starts when a popular key expires or the cache restarts empty.

The sequence feeds itself:

| Step | What happens                                                          |
| ---- | --------------------------------------------------------------------- |
| 1    | A [hot key](./Hot%20key.md) expires                                   |
| 2    | Hundreds of in-flight requests see a miss before anyone refills it    |
| 3    | Each one runs the same expensive query                                |
| 4    | The database slows under the duplicated work                          |
| 5    | Queries take longer, so the key stays empty and more requests pile in |
| 6    | Requests hit their [timeouts](./Timeout.md), and any retries add load |

The recognisable symptom is a database spike that recurs at regular intervals and matches a key's [TTL](./TTL.md), or an outage that begins moments after a cache restart. If the query takes 2 seconds and the key receives 1,000 requests per second, about 2,000 identical queries can start before the first one finishes.

Several defences exist, and they combine well:

| Technique              | How it works                                                      |
| ---------------------- | ----------------------------------------------------------------- |
| Request coalescing     | One request per key does the rebuild, and others await its result |
| Lock or lease          | Only the holder rebuilds, and the rest wait or get stale data     |
| Stale-while-revalidate | Serve the expired value while one worker refreshes it             |
| Early refresh          | Refresh popular keys shortly before they expire                   |
| TTL jitter             | Randomise expiry so that keys do not expire together              |
| Warming                | Load popular keys before sending traffic to a new cache           |

Coalescing inside each process, such as Go's `singleflight`, limits duplicates to one per server, and a distributed lock can limit them across the fleet. Give any lock its own expiry, so that a crashed holder cannot block the key forever.

_Usage:_

"The database falls over at the same minute every hour."

"That's a cache stampede on the homepage query. Add request coalescing and jitter the TTL."
