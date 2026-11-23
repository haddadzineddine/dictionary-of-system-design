---
description: A service that keeps no client data in its own memory between requests, so any instance can handle any request.
---

A stateless service keeps no client-specific data in its own memory or disk between requests. Everything needed to answer arrives with the request or is fetched from a shared store such as a database. Because of this, any running instance can handle any request.

The state still exists. It has moved out of the application process into something built to hold it:

| State             | Stateful version         | Stateless version                                             |
| ----------------- | ------------------------ | ------------------------------------------------------------- |
| Login session     | Object in server memory  | Shared session store or signed token                          |
| Shopping cart     | Variable on one instance | Database row keyed by user                                    |
| Uploaded file     | Local disk               | Object storage                                                |
| Progress of a job | In-process loop          | Record in a [message queue](./Message%20queue.md) or database |

The symptom of hidden state is behaviour that depends on which server you reach. Users get logged out at random, a file uploaded a moment ago returns a 404 on refresh, or everything breaks right after a deploy because the restart wiped memory. These faults often appear only once a second instance is added, since a single server hides the problem.

Statelessness is what makes [horizontal scaling](./Horizontal%20scaling.md) straightforward. A [load balancer](./Load%20balancer.md) can send traffic anywhere, instances can be replaced during a deploy, and a crashed instance loses only the requests it was processing at that moment. Without it, teams often resort to a [sticky session](./Sticky%20session.md), which keeps the problem and adds new ones.

An in-process [cache](./Cache.md) does not break the model, provided losing it affects only speed. The test is whether an instance can be killed at any moment with no lasting effect on users.

The cost is that each request may need a trip to the shared store, which adds [latency](./Latency.md) and makes that store a dependency of every request.

_Usage:_

"Why do people lose their carts every time we deploy?"

"The cart lives in process memory, so this isn't a stateless service. Move it to Redis and restarts stop mattering."
