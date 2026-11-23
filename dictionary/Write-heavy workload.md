---
description: A workload dominated by incoming writes, such as logs or sensor data, where caches and read copies give little relief.
---

A write-heavy workload is one where a large share of the traffic creates or changes data. Metrics collection, application logs, sensor readings, click tracking and chat messages are typical examples.

Writes are harder to scale than reads because each one must reach durable storage and stay correct. A [cache](./Cache.md) does little to absorb them. Read replicas (database copies that serve queries) do not help either, because every replica has to apply every write, and heavy write load can make those copies fall behind.

The symptoms differ from a read problem. Disk I/O saturates, lock waits climb, replicas lag by minutes, and insert [latency](./Latency.md) grows as tables and their indexes get larger. Each index adds work to every insert, so a table with many indexes can be slow to write even when its queries are fast.

| Technique                             | How it helps                                | Trade-off                       |
| ------------------------------------- | ------------------------------------------- | ------------------------------- |
| Batching                              | Groups many rows into one write             | Small delay, larger loss window |
| Fewer indexes                         | Reduces work per insert                     | Some queries get slower         |
| [Message queue](./Message%20queue.md) | Absorbs bursts ahead of the database        | Data is visible later           |
| Write-optimised storage               | Appends to disk and merges later (LSM tree) | Reads can cost more             |
| Partitioning                          | Splits writes across several machines       | Harder queries, rebalancing     |

Partitioning needs a well-chosen key. If all writes for the current hour land on one partition, as happens when data is range-partitioned by timestamp, one machine takes the entire load. This hot partition has the same effect as a [hot key](./Hot%20key.md). Mixing in a device or user ID spreads the writes.

Check whether every write needs the same guarantee. Losing one click event may be acceptable, while losing a payment is not. Data you can afford to lose can use batching or a [write-behind cache](./Write-behind%20cache.md).

_Usage:_

"Replica lag hits ten minutes whenever the sensors report."

"It's a write-heavy workload. More replicas won't help, so batch the inserts and drop the indexes nobody uses."
