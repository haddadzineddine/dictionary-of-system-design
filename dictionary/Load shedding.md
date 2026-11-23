---
description: Deliberately rejecting some requests when a system is overloaded, so that it can keep serving the rest.
---

Load shedding means deliberately rejecting some requests when a system is overloaded, so that it can keep serving the rest. The name comes from electricity grids, which cut power to some areas to protect the whole network.

It is needed because overloaded systems often do less useful work as load rises. Past capacity, requests queue and wait so long that clients give up, while the server keeps working on answers nobody will receive. CPU stays at 100% and [throughput](./Throughput.md) of successful responses falls toward zero. Rejecting 20% of requests early can let the other 80% succeed, where accepting all of them might mean that almost none do.

The decision has to be inexpensive, so it happens as early as possible, before authentication, database calls or parsing a large body. Useful overload signals include the number of requests in flight and the time requests spend queued. These react faster and more directly than average CPU. Rejected requests typically get a 503, optionally with `Retry-After`.

Shedding works better when requests have priorities:

| Priority | Example                    | Under overload |
| -------- | -------------------------- | -------------- |
| Critical | Checkout, login            | Shed last      |
| Normal   | Product pages, search      | Shed if needed |
| Low      | Recommendations, analytics | Shed early     |
| Batch    | Exports, background sync   | Shed first     |

Also drop requests whose deadline has already passed.

Load shedding differs from [rate limiting](./Rate%20limiting.md), which enforces per-client quotas regardless of server health. Shedding responds to the server's current state and can affect any client. It pairs with [graceful degradation](./Graceful%20degradation.md), where the system does less for each request instead of refusing it.

Clients must cooperate. If every rejected request is retried immediately, shedding feeds a [retry storm](./Retry%20storm.md). Test the behaviour with a load test beyond capacity, and confirm that successful throughput holds steady.

_Usage:_

"At peak the API serves nothing, even though it's only 30% over capacity."

"It's accepting everything and timing out. Add load shedding on queue time and drop the batch traffic first."
