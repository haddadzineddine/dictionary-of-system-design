---
description: The maximum data rate a network link can carry, usually in bits per second. It is a capacity, not the speed you achieve.
---

Bandwidth is the maximum rate at which a network link can carry data, usually stated in bits per second. A 10 Gbps network card can move at most 10 gigabits each second. It is a capacity. What you actually achieve is [throughput](./Throughput.md), which is lower because of protocol overhead, congestion and the speed of whatever sits at each end.

Network capacity is quoted in bits, while file sizes are quoted in bytes, and mixing them up produces estimates that are wrong by a factor of eight. A 1 Gbps link carries at most 125 megabytes per second.

| Link     | Maximum in bytes | Time to move 1 TB at full rate |
| -------- | ---------------- | ------------------------------ |
| 1 Gbps   | 125 MB/s         | About 2.2 hours                |
| 10 Gbps  | 1.25 GB/s        | About 13 minutes               |
| 100 Gbps | 12.5 GB/s        | About 80 seconds               |

Bandwidth and [latency](./Latency.md) solve different problems. Bandwidth is the width of the pipe, and latency is how long a byte takes to reach the other end. A larger pipe speeds up a big download but does little for a small API call, whose duration is dominated by round trips. Upgrading a link will not fix a chatty service that makes thirty sequential calls across regions.

Bandwidth becomes the [bottleneck](./Bottleneck.md) when payloads are large. A service sending 500 KB images 2,000 times per second needs 1 GB/s, or 8 Gbps, before overhead. On a 10 Gbps card that leaves little headroom, so the service can stall with low CPU and plenty of free memory. When a server looks idle and responses are still slow, check network utilisation.

Common remedies are compressing responses, resizing images, paginating large results, and serving static files from a [CDN](./CDN.md). These also reduce data transfer charges, which cloud providers typically bill on outbound traffic.

_Usage:_

"CPU is at 20% on the media servers, but downloads crawl at peak."

"Check the NICs. We're pushing 9 Gbps through 10 Gbps cards, so we've run out of bandwidth."
