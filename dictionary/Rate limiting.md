---
description: Capping how many requests a client can make in a period of time and rejecting the excess, usually with HTTP 429.
---

Rate limiting caps how many requests a client can make in a period of time and rejects the excess. A rule might allow 100 requests per minute per API key. It protects a service from abusive clients, buggy scripts stuck in a loop, and one tenant using capacity that others need. Rejected HTTP requests normally receive a 429 Too Many Requests status, often with a `Retry-After` header saying when to try again.

Several algorithms can enforce the limit:

| Algorithm                                                 | How it counts                      | Trade-off                         |
| --------------------------------------------------------- | ---------------------------------- | --------------------------------- |
| Fixed window                                              | Counter per calendar minute        | Allows a double burst at the edge |
| Sliding log                                               | Stores a timestamp per request     | Exact, but uses more memory       |
| [Sliding window counter](./Sliding%20window%20counter.md) | Weights two adjacent counters      | Approximate, low memory           |
| [Token bucket](./Token%20bucket.md)                       | Spends tokens that refill steadily | Permits controlled bursts         |
| Leaky bucket                                              | Drains a queue at a fixed rate     | Smooth output, adds waiting       |

Identifying the client needs thought. Limiting by IP address punishes everyone behind one office network or mobile carrier, so a whole company can receive 429s because of one colleague's script. Prefer an API key or user ID, and keep IP limits for unauthenticated endpoints such as login.

Running several instances introduces a common bug. If each instance counts on its own, a limit of 100 per minute across three instances really allows 300. A shared store such as Redis fixes this, provided the check and increment happen atomically. You also need to decide what happens when that store is down: allow traffic and lose protection, or reject it and cause an outage.

Rate limiting differs from [load shedding](./Load%20shedding.md). Rate limiting enforces a per-client quota even when the system is healthy, while load shedding drops work because the system as a whole is overloaded.

_Usage:_

"A partner's sync job is eating half of our API capacity."

"Add rate limiting per API key and return `Retry-After`, so their client backs off."
