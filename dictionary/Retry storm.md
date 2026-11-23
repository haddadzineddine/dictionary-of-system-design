---
description: A surge of repeated requests from clients retrying failures, adding load to a service that is already struggling.
---

A retry storm is a surge of repeated requests from clients retrying failures, which adds load to a service that is already struggling. When a service is overloaded, most requests fail, every failure generates more requests, and the extra traffic keeps the service overloaded.

The recognisable symptom is a dependency that cannot recover. Requests to it are several times higher than real user traffic, it comes back for a moment after a restart, and it is knocked over again within seconds. The original cause may be long gone, while the retries alone sustain the outage.

Retries at several layers multiply. If each layer makes up to 3 attempts, the bottom of the call chain receives:

| Layers that retry | Maximum attempts per user request |
| ----------------- | --------------------------------- |
| 1                 | 3                                 |
| 2                 | 9                                 |
| 3                 | 27                                |
| 4                 | 81                                |

[Retry with backoff](./Retry%20with%20backoff.md) spreads attempts over time, but it does not reduce how many are made, so it is necessary and not sufficient.

Several controls limit the amplification:

- **Retry at one layer.** Usually the layer closest to the failing call. Others pass the error up.
- **Retry budgets.** Allow retries only up to a fixed share of normal traffic, such as 10%, per client.
- **[Circuit breakers](./Circuit%20breaker.md).** Stop calling a dependency that is failing.
- **Explicit overload signals.** Have the server mark an error as "do not retry", and have callers obey it.
- **[Load shedding](./Load%20shedding.md).** Reject excess work inexpensively, so that the server survives the surge.

During an incident, the quickest way out is often to cut incoming traffic at the edge and restore it gradually, since the service may be unable to recover while the backlog of retries continues. A retry storm is a common way for a small fault to become a [cascading failure](./Cascading%20failure.md).

_Usage:_

"Auth came back up, then fell over again thirty seconds later."

"That's a retry storm. Three layers each make three attempts, so it can see up to 27 times the real load."
