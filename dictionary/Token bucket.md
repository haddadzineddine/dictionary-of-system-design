---
description: A rate limiting algorithm where requests spend tokens that refill at a steady rate, allowing short bursts up to a set size.
---

A token bucket is a [rate limiting](./Rate%20limiting.md) algorithm in which each request spends a token from a bucket that refills at a steady rate. It has two settings: capacity, the most tokens the bucket can hold, and refill rate, how many tokens are added per second. Capacity controls the size of a burst, and refill rate controls the sustained average.

Take a bucket with a capacity of 20 and a refill rate of 5 tokens per second:

| Time | Event                | Tokens after | Result                |
| ---- | -------------------- | ------------ | --------------------- |
| 0 s  | Client has been idle | 20           | Bucket is full        |
| 0 s  | 20 requests arrive   | 0            | All allowed           |
| 0 s  | 1 more request       | 0            | Rejected              |
| 1 s  | 5 tokens added       | 5            | Refilled              |
| 1 s  | 8 requests arrive    | 0            | 5 allowed, 3 rejected |

This burst behaviour suits real clients. A page load may fire ten API calls at once and then go quiet. A strict per-second limit would reject part of that page even though the user's average rate is low. A token bucket lets it through and still holds the long-run rate.

Implementations do not need a timer adding tokens. Store only the token count and the last update time. On each request, compute `tokens = min(capacity, tokens + elapsed × rate)`, then subtract one if at least one token is available. Across several servers, this read-and-update has to be atomic, for example through a Lua script in Redis. Otherwise two concurrent requests can spend the same token.

Choosing the capacity is the main tuning decision. If it is too small, normal bursts receive 429 errors even though the client is under its average limit. If it is too large, many clients can burst together and still overload the backend.

_Usage:_

"Customers get 429s on dashboard load, but they're well under 300 requests a minute."

"The page sends 40 calls at once. Use a token bucket with a burst capacity of 50."
