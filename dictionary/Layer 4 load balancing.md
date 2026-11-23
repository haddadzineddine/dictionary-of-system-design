---
description: Distributing network connections using only IP addresses and ports, without reading the data inside them.
---

Layer 4 load balancing distributes network connections using only IP addresses, ports and the protocol, without reading the data inside them. The name comes from the transport layer, layer 4 of the OSI networking model, which is where TCP and UDP sit. The [load balancer](./Load%20balancer.md) picks a backend when a connection opens, and every packet in that connection then goes to the same backend.

Not parsing the content has these consequences:

| Property                           | Consequence                                    |
| ---------------------------------- | ---------------------------------------------- |
| No parsing of application data     | Low overhead per connection                    |
| Works with any TCP or UDP protocol | Suits databases, message brokers, game traffic |
| Encrypted traffic passes through   | TLS can stay end-to-end, up to the backend     |
| Sees no URL, header or cookie      | Cannot route by path or user                   |
| Decides once per connection        | Balances connections, not requests             |

The last property causes a well-known problem with long-lived connections. HTTP/2 and gRPC (a remote procedure call framework) send many requests over one connection. A layer 4 balancer sees one connection and pins it to one backend, so a busy client can load one server heavily while others sit idle.

The same effect shows up during scaling. You add three new pods, and they receive almost no traffic, because existing clients keep their open connections to the old pods.

There are three usual fixes. Set a maximum connection age so clients reconnect and spread out over time. Use client-side balancing, where the client holds connections to several backends. Or move to [layer 7 load balancing](./Layer%207%20load%20balancing.md), which balances each request individually.

Layer 4 remains a good fit for non-HTTP protocols and very high connection counts. AWS Network Load Balancer and HAProxy in TCP mode are examples.

_Usage:_

"We scaled gRPC to ten pods, but three of them are doing all the work."

"The service is behind layer 4 load balancing, so each connection is stuck to one pod. Set a max connection age."
