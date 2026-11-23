---
description: A load balancer setting that keeps sending one client to the same server, usually because that server holds its session.
---

A sticky session is a [load balancer](./Load%20balancer.md) setting that keeps sending the same client to the same backend server. It is also called session affinity. It is typically used when a server holds something about that client in local memory, such as a login session, and other servers could not answer without it.

There are two common ways to implement it:

| Method    | How the client is identified                    | Weakness                                                             |
| --------- | ----------------------------------------------- | -------------------------------------------------------------------- |
| Cookie    | Balancer sets or reads a cookie naming a server | Requires [layer 7 load balancing](./Layer%207%20load%20balancing.md) |
| Source IP | Balancer hashes the client's IP address         | Many users behind one office or mobile network share an IP           |

Stickiness makes a stateful application run on several servers with little code change, which is why it is often adopted. The costs show up later. Load becomes uneven, because a few heavy users stay pinned to one instance while others idle. Adding servers does not relieve the busy ones right away, since existing clients stay where they are.

Failure is the most visible problem. When a server crashes or is replaced during a deploy, every user pinned to it loses their session, and people find themselves logged out or with an empty form. Rolling deploys become slower too, because you must either wait for sessions to drain from an instance or accept that those users lose them.

The usual fix is to make the application a [stateless service](./Stateless%20service.md) by moving session data to a shared store, or into a signed token held by the client. Stickiness can then be removed.

It still has legitimate uses as an optimisation. Routing a user back to a server with a warm local [cache](./Cache.md) improves speed, and the system still works correctly if the routing changes.

_Usage:_

"Why are a third of users logged out after each deploy?"

"We use sticky sessions and keep the session in memory, so restarting a pod logs out everyone pinned to it."
