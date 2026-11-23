---
description: A single entry point for API clients that handles routing, authentication, rate limits and other shared request concerns.
---

An API gateway is a single entry point for API clients that routes each request to the right backend service and handles concerns shared by all of them. It is a specialised [reverse proxy](./Reverse%20proxy.md). It is common in [microservices](./Microservices.md), where clients would otherwise need to know the address and rules of many separate services.

Without one, each service reimplements login checks and limits, usually with small differences. One team's endpoint ends up with weaker validation, or a security fix has to ship in twelve places. A gateway puts those rules in one layer.

| Concern                               | What the gateway does                          |
| ------------------------------------- | ---------------------------------------------- |
| Routing                               | Maps public paths to internal services         |
| Authentication                        | Validates tokens or API keys before forwarding |
| [Rate limiting](./Rate%20limiting.md) | Enforces quotas per client or plan             |
| Transformation                        | Rewrites headers, paths or payload formats     |
| Versioning                            | Sends `/v1` and `/v2` to different backends    |
| Observability                         | Records logs and metrics for every call        |

The neighbouring components overlap. A [load balancer](./Load%20balancer.md) spreads traffic across copies of one service. A reverse proxy forwards traffic and handles generic HTTP work. An API gateway adds knowledge of API consumers, such as who is calling and what plan they are on.

The main risk is logic creep. Teams add business rules, data joins and special cases to the gateway because it is convenient, and it becomes a shared component that every release depends on and one team must approve. Keep rules about the product inside services, and keep the gateway to cross-cutting concerns. If different clients need different response shapes, a [backend for frontend](./Backend%20for%20frontend.md) is a better location.

The gateway also adds a network hop and sits on every request, so a bad configuration change can take down all APIs at once. Kong and Amazon API Gateway are examples.

_Usage:_

"Should the discount calculation go in a gateway plugin? It's the one place every request passes through."

"No, that's business logic. Keep the API gateway to auth, routing and quotas."
