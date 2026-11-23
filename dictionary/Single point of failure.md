---
description: A component whose failure stops the whole system because nothing else can take over its work.
---

A single point of failure is a component whose failure stops the whole system because nothing else can take over its work. It may be an obvious part, such as the only database server, or a less visible one, such as a single DNS provider, one expiring TLS certificate, or a configuration service every application reads at startup.

The way to remove one is redundancy: run more than one copy, and have a tested way to move work to a surviving copy.

| Component           | Failure without redundancy | Common remedy                                                    |
| ------------------- | -------------------------- | ---------------------------------------------------------------- |
| App server          | Site goes down             | Several instances behind a [load balancer](./Load%20balancer.md) |
| Load balancer       | All traffic stops          | Active-passive pair or a managed service                         |
| Primary database    | Writes stop                | Standby copy with automatic failover                             |
| [Cache](./Cache.md) | Database takes full load   | Replicated cache, capacity to survive a cold start               |
| Data centre         | Everything in it stops     | Deploy across availability zones (isolated sites)                |

Redundancy only helps if the copies fail independently. Two servers in one rack share a power feed and a network switch. Two services in different zones may both depend on one authentication service. A design diagram can show everything doubled while a shared dependency remains underneath, and teams often discover it during the outage.

Failover also has to work when it is needed. A standby that has silently fallen behind, or a procedure nobody has run in a year, gives false confidence. Test failover on a schedule by actually removing the component.

Removing every single point of failure is rarely worthwhile. Each layer of redundancy adds cost and operational complexity, and the failover machinery can itself fail. Compare the cost of an outage with the cost of the extra copy, using the availability target from your [non-functional requirements](./Non-functional%20requirement.md).

_Usage:_

"We run three API instances across zones, so we're safe if one goes."

"All three read their config from that one Redis box at boot. That's still a single point of failure."
