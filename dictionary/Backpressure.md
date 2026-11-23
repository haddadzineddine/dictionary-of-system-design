---
description: A way for a slower part of a system to make faster upstream parts slow down, so that work does not pile up without limit.
---

Backpressure is a way for a slower part of a system to make faster upstream parts reduce their sending rate. Without it, work piles up wherever the producer outruns the consumer, usually in an in-memory buffer. That buffer grows until the process runs out of memory and crashes, losing everything it held.

The recognisable symptom is memory that climbs steadily during a traffic peak, followed by a crash, with [latency](./Latency.md) rising the whole time because each new item waits behind a longer backlog. A queue with no size limit hides overload until it fails.

When a bounded buffer fills up, a component has a limited set of choices:

| Response        | What happens                       | Suitable when                           |
| --------------- | ---------------------------------- | --------------------------------------- |
| Block           | Producer waits until there is room | Producer can safely pause               |
| Reject          | Return an error such as 429 or 503 | Caller can retry or give up             |
| Drop            | Discard new or old items           | Losing some data is fine, as in metrics |
| Sample          | Keep a fraction of items           | Approximate data is acceptable          |
| Scale consumers | Add capacity                       | Slowdown is sustained, not a spike      |

Strictly, only blocking and rejecting push back on the sender. Dropping and sampling protect the system without slowing the producer. All of them depend on the buffer having a limit.

TCP has backpressure built in. A receiver advertises how much data it can accept, and the sender slows when that window shrinks.

The signal has to travel all the way to the source. If a worker slows but its API keeps accepting requests into an unbounded [message queue](./Message%20queue.md), the problem has moved without being solved. At the system edge, the final form of backpressure is telling clients to wait, through [rate limiting](./Rate%20limiting.md) or [load shedding](./Load%20shedding.md).

_Usage:_

"The ingest service gets OOM-killed every time a big customer uploads."

"It buffers without a limit. Bound the queue and add backpressure so uploads wait when the writer falls behind."
