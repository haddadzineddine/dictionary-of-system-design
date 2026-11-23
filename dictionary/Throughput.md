---
description: The amount of work a system completes per unit of time, such as requests per second or messages per minute.
---

Throughput is the amount of work a system completes per unit of time. It is measured in whatever unit fits the job: requests per second, orders per minute, messages per second, or gigabytes per hour. It counts completed work, so a server that accepts 10,000 requests per second and finishes 6,000 has a throughput of 6,000.

Throughput and [latency](./Latency.md) are related but separate. Latency is how long one request takes. Throughput is how many finish per second. A system can have high throughput and poor latency, as a batch job does, or low latency and low throughput, as a single fast worker does.

Little's law connects them. The average number of requests in flight equals throughput multiplied by the average time each one spends in the system:

| Throughput | Average latency | Requests in flight |
| ---------- | --------------- | ------------------ |
| 1,000/s    | 50 ms           | 50                 |
| 1,000/s    | 200 ms          | 200                |
| 1,000/s    | 2 s             | 2,000              |

The table explains a common incident. A database slows from 50 ms to 2 seconds, and at the same arrival rate the number of in-flight requests grows forty-fold. Connection pools and worker threads fill up, and the service starts failing even though traffic did not change.

Every system has a maximum throughput, set by its [bottleneck](./Bottleneck.md). As load approaches that limit, requests queue and latency rises. Beyond it, throughput can fall, because the system spends effort on requests that time out before finishing. This is the reason for [load shedding](./Load%20shedding.md).

To raise throughput, remove the limiting resource, add capacity with [horizontal scaling](./Horizontal%20scaling.md), or batch small operations so each unit of overhead covers more work. Load-test to find the real limit, because a capacity figure derived from one instance rarely holds once instances share a database.

_Usage:_

"We doubled the workers, but we still process about 800 jobs a second."

"Then workers aren't what limits throughput. Check the database, it's probably saturated."
