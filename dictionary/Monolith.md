---
description: An application built and deployed as a single unit, with all its features running together and usually sharing one database.
---

A monolith is an application built and deployed as a single unit. All its features run in the same process and usually share one database, and one part calls another with an ordinary function call. It can still run as many identical copies behind a [load balancer](./Load%20balancer.md), so a monolith is not limited to one server.

For most new products a monolith is a reasonable starting point. Function calls do not fail the way network calls do, a database transaction can cover a whole operation, a stack trace shows the full path of a request, and there is one thing to build, test and deploy.

The costs appear as the codebase and the team grow:

| Growing pain      | How it shows up                                              |
| ----------------- | ------------------------------------------------------------ |
| Slow pipeline     | A one-line fix waits for a long build and full test suite    |
| Deploy contention | Teams queue to release, and one bad change blocks everyone's |
| Shared fate       | A memory leak in reports takes down checkout too             |
| Uniform scaling   | Scaling one CPU-heavy feature means scaling the whole app    |
| Tangled code      | A change in billing breaks something in shipping             |

The last row is often blamed on the deployment model, but it comes from weak internal boundaries. A tangled monolith split into services usually becomes a tangled distributed system. Fixing the structure first, by moving to a [modular monolith](./Modular%20monolith.md), addresses the cause and keeps deployment simple.

Some pains have targeted fixes. Slow or risky background work can run as a separate worker process from the same codebase. A [bulkhead](./Bulkhead.md) can separate traffic types. Only when independent deployment or scaling is a demonstrated need does a move to [microservices](./Microservices.md) justify its cost, and the [strangler fig](./Strangler%20fig.md) approach allows that move in stages.

_Usage:_

"Should a team of four start with separate services for users, billing and orders?"

"No. Build a monolith with clean module boundaries, and split something out when there's a concrete reason."
