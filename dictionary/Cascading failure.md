---
description: A failure in one component that overloads or stalls others, spreading until a large part of the system is down.
---

A cascading failure is a failure in one component that causes others to fail, spreading until a large part of the system is down. The initial fault is often small, such as one slow database query or one lost instance.

There are two common ways it spreads. One is load transfer: when a server fails, its traffic moves to the survivors, which then exceed their capacity and fail too. The other is resource exhaustion: a slow dependency holds its callers' threads and connections until those callers stop responding, and their own callers then stall in turn.

| Step | What happens                                                   |
| ---- | -------------------------------------------------------------- |
| 1    | A database slows down                                          |
| 2    | Service A's threads fill up waiting on it                      |
| 3    | Health checks on A time out, and instances are removed         |
| 4    | Remaining A instances receive more traffic and fail            |
| 5    | Service B, which calls A, runs out of threads                  |
| 6    | Clients and services retry, multiplying the load               |
| 7    | The site is down, including features unrelated to the database |

Fixing the original fault often does not end the incident. A system can stay down after its trigger is gone, because retries, cold [caches](./Cache.md) and queued work keep it overloaded. This is sometimes called a metastable failure. Restarting everything at once tends to fail again, since instances come up with empty caches into full traffic.

Each defence interrupts a different step:

| Defence                                   | What it stops                       |
| ----------------------------------------- | ----------------------------------- |
| [Timeout](./Timeout.md)                   | Threads waiting indefinitely        |
| [Circuit breaker](./Circuit%20breaker.md) | Calls to a failing dependency       |
| [Bulkhead](./Bulkhead.md)                 | One dependency using all capacity   |
| [Load shedding](./Load%20shedding.md)     | Overload on survivors               |
| Retry budget                              | A [retry storm](./Retry%20storm.md) |

To recover, reduce traffic at the edge, let the system drain, and bring load back in stages. Keep enough spare capacity that losing one instance or zone does not overload what remains.

_Usage:_

"How did a slow reporting query take down login?"

"It was a cascading failure. The query held the connection pool, the API stalled, and retries finished it off."
