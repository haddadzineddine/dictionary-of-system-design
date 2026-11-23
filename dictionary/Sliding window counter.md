---
description: A rate limiting algorithm that estimates recent traffic by weighting the counts of the current and previous time windows.
---

A sliding window counter is a [rate limiting](./Rate%20limiting.md) algorithm that estimates recent traffic by combining the counts of the current and previous fixed time windows. It uses about as little memory as a plain counter and reduces that counter's main flaw.

The flaw is the boundary burst. A fixed window limit of 100 requests per minute resets at the top of each minute. A client can send 100 requests at 12:00:59 and another 100 at 12:01:00. Each window is within its limit, yet the backend receives 200 requests in about one second, double the intended rate.

The sliding window counter keeps two numbers per client and weights the older one by how much of it still overlaps the last 60 seconds:

| Step | Calculation                             | Example             |
| ---- | --------------------------------------- | ------------------- |
| 1    | Count in the previous window            | 80                  |
| 2    | Count so far in the current window      | 30                  |
| 3    | Position in the current window          | 15 s of 60 s        |
| 4    | Share of previous window still in range | 1 − 15/60 = 75%     |
| 5    | Estimate = previous × share + current   | 80 × 0.75 + 30 = 90 |
| 6    | Compare with the limit of 100           | Allowed             |

The estimate assumes requests in the previous window were spread evenly. When they were bunched at one end, the estimate can be far off: a client can fit nearly twice the limit into a rolling window, or is rejected after a minute of sending nothing. On evenly spread traffic the error is small. For quotas and abuse protection that error is normally acceptable. A sliding log is exact, at a higher memory cost.

Compared with a [token bucket](./Token%20bucket.md), this algorithm expresses a limit directly as "N per window" and does not grant an explicit burst allowance.

_Usage:_

"Clients are doubling their quota by bursting right around the minute mark."

"That's the fixed window edge. Switch to a sliding window counter, which only needs one extra integer per client."
