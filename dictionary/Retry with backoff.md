---
description: Repeating a failed call after a wait that grows with each attempt, usually with added randomness to spread clients out.
---

Retry with backoff means repeating a failed call after a wait that grows with each attempt. Many failures are transient, such as a dropped connection or an instance restarting, and a second attempt moments later succeeds. Waiting longer each time gives a struggling service room to recover.

The usual form is exponential backoff with jitter. Jitter is added randomness, and it matters because clients that failed together would otherwise retry together in synchronised waves. With "full jitter", each wait is a random value between zero and a growing ceiling. With a base of 100 ms and a cap of 2 seconds:

| Retry | Ceiling (doubles, capped) | Actual wait      |
| ----- | ------------------------- | ---------------- |
| 1     | 100 ms                    | Random, 0–100 ms |
| 2     | 200 ms                    | Random, 0–200 ms |
| 3     | 400 ms                    | Random, 0–400 ms |
| 4     | 800 ms                    | Random, 0–800 ms |
| 5     | 1,600 ms                  | Random, 0–1.6 s  |
| 6     | 2,000 ms (cap)            | Random, 0–2 s    |

Retry only what can succeed on a second attempt. Connection errors, [timeouts](./Timeout.md), 429 and 503 are reasonable candidates. A 400 or 403 will usually fail the same way, although some APIs also use 403 for [rate limits](./Rate%20limiting.md). If the server sends a `Retry-After` header, honour it.

Safety is the other condition. If a payment request times out, the charge may have gone through, and a blind retry charges the customer twice. Retry only idempotent operations, those where repeating has no additional effect. For others, send an idempotency key, a unique ID the server uses to recognise a repeated request.

Always limit the attempts and the total time, and keep that total inside the caller's own deadline. Retrying at several layers multiplies load and can cause a [retry storm](./Retry%20storm.md). A [circuit breaker](./Circuit%20breaker.md) stops retries when a dependency is down.

_Usage:_

"The sync job fails whenever the vendor API has a brief 503."

"Wrap it in retry with backoff and jitter, five attempts at most, and honour their `Retry-After`."
