---
description: One request or message that triggers many downstream calls or copies. The number of targets is the fan-out factor.
---

Fan-out is when one incoming request or message triggers many downstream calls or copies. A search query sent to 50 index partitions has a fan-out of 50. A post delivered to 10,000 followers has a fan-out of 10,000. The term appears in two main settings.

| Setting         | What multiplies                     | Typical example                      |
| --------------- | ----------------------------------- | ------------------------------------ |
| Request fan-out | Synchronous calls to other services | A page calling 12 services to render |
| Data fan-out    | Copies of a message or record       | One event delivered to many inboxes  |

Request fan-out has a counter-intuitive effect on [latency](./Latency.md). If a request must wait for all its downstream calls, it is as slow as the slowest one, so rare slowness becomes common. Suppose each backend exceeds one second on 1% of calls. A request that calls 100 of them in parallel has a 1 − 0.99¹⁰⁰ chance, about 63%, of hitting at least one slow response. This is why a page can feel slow when every service's dashboard looks healthy.

Several techniques reduce the effect. Call backends in parallel, give each call a [timeout](./Timeout.md), and return partial results when an optional backend is late, which is a form of [graceful degradation](./Graceful%20degradation.md). A hedged request sends a duplicate to another replica if the first has not answered after a short delay, and uses whichever reply comes first.

Fan-out also multiplies load. One user action becoming 100 internal calls means a modest traffic increase can overload deep dependencies, and any [retries](./Retry%20with%20backoff.md) multiply it again.

Data fan-out raises a timing question: do the copying when content is written, or assemble it when someone reads? Those two designs are [fan-out on write](./Fan-out%20on%20write.md) and [fan-out on read](./Fan-out%20on%20read.md).

_Usage:_

"Every service is under 100 ms at p99, but the dashboard page takes three seconds."

"It has a fan-out of about 40 calls, partly in sequence. The slow ones add up."
