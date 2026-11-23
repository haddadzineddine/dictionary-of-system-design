---
description: One key that receives far more traffic than others, overloading the single node that stores it while the rest sit idle.
---

A hot key is a single key that receives far more traffic than the others, overloading the node that stores it. In a partitioned [cache](./Cache.md) or database, each key lives on one node or a small set of nodes, so a viral product page or a very popular account sends its traffic to one place regardless of how large the cluster is.

The recognisable symptom is one node at 100% CPU while the others sit at 15%. Cluster-wide averages look healthy, more nodes do not help, and requests touching unrelated keys on the same node also slow down. [Consistent hashing](./Consistent%20hashing.md) does not prevent this, because it balances how many keys each node owns and takes no account of how popular each key is.

Hot keys are found by sampling per-key traffic. Redis, for example, offers `redis-cli --hotkeys` when an LFU [eviction policy](./Eviction%20policy.md) is configured.

The fix depends on whether the traffic is reads or writes:

| Technique              | Works for | How it helps                                              |
| ---------------------- | --------- | --------------------------------------------------------- |
| Local in-process cache | Reads     | Each app server keeps the value for a second or two       |
| Replicate the key      | Reads     | Several nodes hold copies and share the requests          |
| Key splitting          | Reads     | Store copies as `item:1`, `item:2` and pick one at random |
| Counter splitting      | Writes    | Increment one of N sub-counters and sum them on read      |
| Write batching         | Writes    | Combine many updates in the app before writing            |
| Dedicated node         | Both      | Move the key to hardware of its own                       |

A one-second local cache limits each app server to about one fetch per second for that key.

A hot key that expires is a further risk. Every request misses simultaneously and goes to the database, producing a [cache stampede](./Cache%20stampede.md).

_Usage:_

"Redis shard 7 is maxed out and the other eleven are idle."

"That's a hot key, probably the homepage banner. Add a short local cache in the app."
