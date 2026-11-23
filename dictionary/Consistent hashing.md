---
description: A way of assigning keys to servers so that adding or removing a server moves only a small share of the keys.
---

Consistent hashing is a way of assigning keys to servers so that adding or removing a server moves only a small share of the keys. It is used in distributed [caches](./Cache.md) and partitioned data stores, where each key must live on a predictable node.

The simpler approach is `hash(key) % N`, where N is the number of servers. It spreads keys evenly, but changing N changes the result for nearly every key. Going from 4 servers to 5 sends about 80% of keys to a different server. In a cache, that shows up as a sudden collapse in [cache hit ratio](./Cache%20hit%20ratio.md) right after someone adds a node, with the database taking the resulting load.

Consistent hashing avoids this by placing both servers and keys on the same circular range of hash values:

| Step | What happens                                         |
| ---- | ---------------------------------------------------- |
| 1    | Hash each server's name to a position on the ring    |
| 2    | Hash the key to a position on the same ring          |
| 3    | Walk clockwise from the key until you reach a server |
| 4    | That server owns the key                             |

When a server joins, it takes over only the keys between itself and the previous server on the ring. When one leaves, its keys pass to the next server. Going from 4 to 5 servers moves about 20% of keys.

With a few servers placed at random positions, some arcs of the ring are much longer than others, so load is uneven. The standard fix is virtual nodes: each physical server appears at many positions on the ring. A larger machine can be given more positions.

Consistent hashing balances the number of keys and does nothing about uneven popularity. A single [hot key](./Hot%20key.md) still lands on one server and can overload it.

_Usage:_

"Every time we add a Redis node, the database gets hammered for ten minutes."

"The client is doing modulo hashing, so nearly every key remaps. Switch it to consistent hashing."
