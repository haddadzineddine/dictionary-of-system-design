---
description: The mechanism services use to find the current network addresses of the other services they call.
---

Service discovery is the mechanism services use to find the current network addresses of the services they call. Hard-coded addresses stop working once instances are created and destroyed by autoscaling and deploys, because each new instance can have a different IP. A discovery system maintains a live list of healthy instances under a stable service name, such as `payments`.

The list lives in a registry. Instances either register themselves on startup or are registered by the platform, and health checks remove those that stop responding. Callers then reach an instance in one of three ways:

| Approach    | How a caller finds an instance                                            | Trade-off                                                         |
| ----------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| DNS-based   | Resolves the service name to one or more IPs                              | Simple, but resolvers and clients [cache](./Cache.md) old records |
| Client-side | Fetches the list from the registry and picks an instance itself           | No extra hop, but logic in every client                           |
| Server-side | Calls a stable address, and a [load balancer](./Load%20balancer.md) picks | Simple clients, one more hop                                      |

Kubernetes Services and Consul are examples.

The usual symptom of a discovery problem is a burst of connection errors during every deploy. An old pod has shut down, but callers still hold its address for a few seconds, or longer if a client caches DNS results for too long and ignores the record's [TTL](./TTL.md). To reduce this, have an instance leave the registry first, keep serving during a short draining period, and only then exit. Callers should also retry a failed connection on a different instance.

The registry becomes a dependency of nearly everything, so an outage there can spread widely. Run it with redundancy, and have clients keep using their last known list of addresses if the registry cannot be reached.

_Usage:_

"Every rollout gives us thirty seconds of connection refused from the orders service."

"Pods exit before service discovery removes them. Add a pre-stop delay so they drain first."
