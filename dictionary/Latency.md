---
description: The time one operation takes from start to result, usually reported as percentiles because averages hide slow requests.
---

Latency is the time one operation takes, from the moment it starts to the moment its result arrives. For a web request, that includes network travel, time spent waiting in queues, processing, and the trip back. Some texts reserve "latency" for the waiting part and use "response time" for the total, but in everyday use engineers mean the total.

Latency is reported in percentiles because averages hide the slow requests. The p50 is the median. The p99 is the value that 99% of requests beat. A dashboard can show an 80 ms average while users complain about a slow checkout, because the p99 is 3 seconds. A person who loads many pages per session will run into that slow 1% regularly.

Rough magnitudes are useful for reasoning about where time goes:

| Operation                             | Rough time                       |
| ------------------------------------- | -------------------------------- |
| Read from main memory                 | About 100 nanoseconds            |
| Random read from an SSD               | Tens to hundreds of microseconds |
| Network round trip in a data centre   | Under 1 millisecond              |
| Network round trip between continents | About 50 to 300 milliseconds     |

These gaps explain many design choices. A [cache](./Cache.md) in memory avoids a slower lookup, and a [CDN](./CDN.md) moves content closer to the user because no amount of server tuning shortens the distance light travels through fibre.

Latency also depends on load. As a component approaches its capacity, requests spend more time queueing, so latency climbs sharply before [throughput](./Throughput.md) reaches its limit. A rising p99 is often the first visible sign of a [bottleneck](./Bottleneck.md).

To reduce latency, measure each stage of a request first. Then remove network hops, run independent calls in parallel, or do the work ahead of time.

_Usage:_

"The average is fine, so why is support getting complaints about search?"

"Look at the p99 latency. One request in a hundred takes four seconds, and the average hides it."
