---
description: A wrapper that stops calling a failing dependency for a while and returns an immediate error so the caller stays healthy.
---

A circuit breaker is a wrapper around calls to a dependency that stops sending requests once the failure rate gets too high. Callers then receive an immediate error instead of waiting. The name comes from the electrical device that cuts a circuit to prevent damage.

The problem it addresses is slow failure. When a dependency hangs, each call holds a thread until its [timeout](./Timeout.md) expires, so a service with healthy code runs out of workers waiting on something that will not answer. Failing in a millisecond keeps those workers free for other requests and gives the dependency room to recover.

A breaker moves between three states:

| State     | Behaviour                                    | Moves to                                  |
| --------- | -------------------------------------------- | ----------------------------------------- |
| Closed    | Calls pass through, and failures are counted | Open, when failures cross a threshold     |
| Open      | Calls fail immediately                       | Half-open, after a cool-down period       |
| Half-open | A few trial calls are allowed                | Closed if they succeed, open if they fail |

Thresholds are normally a failure percentage over a sliding period with a minimum number of calls, so that two failures out of three at a quiet moment do not trip it. Count slow calls as well as errors, since slowness is often how a failure begins. Client errors such as 400 should not count, because they say nothing about the dependency's health.

What the caller does while the breaker is open is a product decision. Options include a cached value, a default, or a reduced page, which is [graceful degradation](./Graceful%20degradation.md). If there is no fallback, return a clear error.

Use one breaker per dependency, and sometimes per endpoint, so that one bad route does not block a healthy one. Breakers also limit a [retry storm](./Retry%20storm.md), because retries against an open breaker fail locally. Resilience4j is an example library.

_Usage:_

"Recommendations is down and it's dragging the product page with it."

"Put a circuit breaker on that call and render the page without the widget while it's open."
