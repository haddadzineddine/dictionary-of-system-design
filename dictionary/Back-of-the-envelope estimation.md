---
description: A quick calculation with rounded numbers that sizes traffic, storage and bandwidth before any design decision is made.
---

Back-of-the-envelope estimation is a quick calculation with rounded numbers that tells you the rough size of a system before you design it. The aim is the right order of magnitude. Knowing whether a service handles 10, 1,000 or 100,000 requests per second decides whether one server is enough or a distributed design is needed.

A useful shortcut is that a day has 86,400 seconds, which rounds to 100,000. Here is a worked estimate for a social app with 10 million daily users:

| Step | Quantity                         | Calculation                       | Result             |
| ---- | -------------------------------- | --------------------------------- | ------------------ |
| 1    | Reads per day                    | 10M users × 20 reads              | 200M               |
| 2    | Average reads                    | 200M ÷ 86,400 s                   | About 2,300/s      |
| 3    | Peak reads                       | 2,300/s × 3 (assumed peak factor) | About 7,000/s      |
| 4    | Writes per day                   | 10M users × 0.1 posts             | 1M, about 12/s     |
| 5    | Storage per year                 | 1M posts × 2 KB × 365             | About 730 GB       |
| 6    | Peak [bandwidth](./Bandwidth.md) | 7,000/s × 50 KB                   | 350 MB/s, 2.8 Gbps |

The result already suggests a design. Reads outnumber writes 200 to 1, so this is a [read-heavy workload](./Read-heavy%20workload.md) and a [cache](./Cache.md) will help. Storage fits on one machine for years, so splitting the database across machines is not yet justified.

Estimation prevents two opposite mistakes. One is building a sharded, multi-region platform for traffic a single server could handle. The other is shipping a design that hits its [bottleneck](./Bottleneck.md) in the first week because nobody multiplied the expected users by the payload size.

State every assumption next to its number, because the assumptions are what reviewers can challenge. Plan for peak rather than average traffic, and remember that bits and bytes differ by a factor of eight.

_Usage:_

"We'll need Kafka and a dozen nodes for the audit log, right?"

"Do the back-of-the-envelope estimation first. It's 40 writes per second at peak, and one Postgres instance can handle that."
