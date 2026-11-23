---
description: A set of consumers that share the work of reading a stream, with each partition assigned to one member at a time.
---

A consumer group is a set of consumers that share the work of reading a stream, with each partition assigned to one member at a time. The term comes from [event streaming](./Event%20streaming.md) systems such as Apache Kafka. Separate groups each read the full stream independently, so a billing group and an analytics group can consume the same topic at different speeds.

Because a partition has only one active reader per group, the partition count sets the upper limit on parallelism. With 6 partitions:

| Consumers in group | Assignment               | Effect                     |
| ------------------ | ------------------------ | -------------------------- |
| 1                  | One consumer reads all 6 | No parallelism             |
| 2                  | 3 each                   | Twice the capacity         |
| 6                  | 1 each                   | Maximum parallelism        |
| 8                  | 6 active, 2 idle         | Extra consumers do nothing |

The last row is a common surprise. A team sees lag growing, scales from 6 pods to 12, and nothing improves because half of them hold no partitions. The fix is more partitions, which can change how keys map to partitions and therefore affects ordering, or faster processing per message.

When members join or leave, the group performs a rebalance, moving partitions between consumers. During a rebalance, some or all consumption may pause. A consumer that takes too long between polls is presumed dead and removed from the group. Its partitions move to another member, which reprocesses the same slow batch and may also time out, producing a loop of rebalances with little progress. In Kafka this is governed by `max.poll.interval.ms`. Reduce batch size or move slow work elsewhere.

Each group commits offsets to record its progress. A consumer that crashes after processing but before committing causes those events to be processed again, so handlers need to be idempotent, meaning safe to run twice.

_Usage:_

"Lag keeps climbing, so I doubled the consumers, and half of them are idle."

"The topic has 12 partitions, and the consumer group already had 12 members before you doubled it. We need more partitions."
