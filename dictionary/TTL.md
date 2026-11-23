---
description: Time to live, a duration after which a stored item such as a cache entry or DNS record is treated as expired.
---

TTL stands for time to live, a duration after which a stored item is treated as expired. A [cache](./Cache.md) entry with a TTL of 300 seconds is served for five minutes, and after that the next request must fetch a fresh copy. DNS records carry a TTL too, and so do many session stores and message systems. In IP networking the same name refers to a hop limit on a packet, which is a separate concept.

Setting a TTL is a trade between freshness and load:

| TTL     | Staleness   | Load on the source | Typical use                 |
| ------- | ----------- | ------------------ | --------------------------- |
| Seconds | Very little | High               | Stock levels, live scores   |
| Minutes | Small       | Moderate           | Product pages, feeds        |
| Hours   | Noticeable  | Low                | Reference data, config      |
| Days+   | High        | Very low           | Versioned, unchanging files |

A TTL also acts as a safety net for [cache invalidation](./Cache%20invalidation.md). If a delete is missed, the wrong value still disappears when it expires. For that reason, entries with no TTL are risky, because a single stale value can stay indefinitely.

Identical TTLs cause a recognisable problem. If a job loads 100,000 keys at once with a one-hour TTL, they all expire together, and the database gets a load spike every hour on the hour. Adding jitter, a small random variation such as 3,600 seconds plus or minus 10%, spreads the expiries out. For a single popular key, expiry can trigger a [cache stampede](./Cache%20stampede.md).

DNS TTLs catch people during migrations. If a record has a 24-hour TTL and you change the IP, some clients keep using the old address for up to a day. Lower the TTL at least one full old-TTL period before the change, then raise it again afterwards.

_Usage:_

"We cut over the DNS an hour ago and half the traffic still hits the old [load balancer](./Load%20balancer.md)."

"The record had a TTL of a day. We should have lowered it yesterday."
