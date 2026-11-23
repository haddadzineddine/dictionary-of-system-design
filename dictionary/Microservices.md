---
description: An architecture of small services that are deployed independently, each owning one business capability and its data.
---

Microservices is an architecture in which an application is split into small services that are deployed independently, each owning one business capability and its data. Services communicate over the network, through APIs or messages. Size matters less than independence: a team should be able to change and release its service without coordinating with others.

| Concern              | [Monolith](./Monolith.md) | [Modular monolith](./Modular%20monolith.md) | Microservices           |
| -------------------- | ------------------------- | ------------------------------------------- | ----------------------- |
| Deployment           | One unit                  | One unit                                    | Per service             |
| Internal calls       | Function call             | Function call via interface                 | Network call            |
| Transactions         | Single database           | Single database                             | Across services is hard |
| Scaling              | Whole app                 | Whole app                                   | Per service             |
| Failure isolation    | Low                       | Low                                         | Possible, with effort   |
| Operational overhead | Low                       | Low                                         | High                    |

The main reason to adopt them is organisational. When many teams share one codebase, coordination becomes the limit on delivery, and separate services let them ship on their own schedules. A small team usually gets the costs without this benefit.

Those costs are substantial. Every call can now fail or be slow, so you need [timeouts](./Timeout.md), retries and [circuit breakers](./Circuit%20breaker.md). A [cascading failure](./Cascading%20failure.md) can cross service boundaries. Operations that were one transaction become a [saga](./Saga.md). You also need [service discovery](./Service%20discovery.md), distributed tracing (following one request across services) and automated deployment.

A common failure is the distributed monolith. Services are separate on paper, but they share a database or call each other in long synchronous chains, so a feature requires five coordinated deploys and one service being down breaks all of them. That combines the coupling of a monolith with the failure modes of a network.

Draw boundaries around business capabilities that change independently. If two services always change together, they should probably be one service.

_Usage:_

"Why does adding one field to checkout need releases from five teams?"

"Because our microservices share a schema. We have a distributed monolith, and that's what we need to fix."
