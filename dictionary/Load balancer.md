---
description: A component that spreads incoming requests across a group of servers and stops sending traffic to unhealthy ones.
---

A load balancer is a component that receives incoming requests and spreads them across a group of backend servers. Clients see one address, while several instances do the work. This is what makes [horizontal scaling](./Horizontal%20scaling.md) possible for a web service, and it lets you replace instances during a deploy without clients noticing.

The balancer chooses a backend using an algorithm:

| Algorithm            | How it picks                               | Suits                                |
| -------------------- | ------------------------------------------ | ------------------------------------ |
| Round robin          | Each server in turn                        | Similar servers, similar requests    |
| Weighted round robin | In turn, in proportion to assigned weights | Servers of different sizes           |
| Least connections    | Server with the fewest active connections  | Requests of varying duration         |
| Hash                 | Same client or key goes to the same server | Affinity, local [caches](./Cache.md) |
| Power of two choices | Samples two at random, takes the less busy | Large fleets, several balancers      |

Balancers operate at different levels. [Layer 4 load balancing](./Layer%204%20load%20balancing.md) routes network connections without reading their content, while [layer 7 load balancing](./Layer%207%20load%20balancing.md) reads each HTTP request and can route on its path or headers.

Health checks decide which instances receive traffic, and they cause two familiar problems. A shallow check that only confirms the process is running can keep returning 200 while real requests fail, so users keep hitting a broken instance. A deep check that also tests the shared database has the opposite risk: one database blip marks every instance unhealthy at once, and the balancer may have nowhere to send traffic. A common compromise is to check what the instance itself needs to function and handle shared dependency failures separately.

The balancer sits on every request, so it can become a [single point of failure](./Single%20point%20of%20failure.md). Run it as a redundant pair or use a managed service.

_Usage:_

"Why are 25% of requests failing when only one of four pods is broken?"

"Its health endpoint still returns 200, so the load balancer keeps sending it traffic."
