---
description: A limit on how long a caller waits for an operation before giving up and treating it as failed.
---

A timeout is a limit on how long a caller waits for an operation before giving up and treating it as failed. Without one, a call to a stalled dependency can wait indefinitely, holding a thread and a connection. Some libraries, such as Python's `requests`, wait indefinitely unless you pass a timeout.

The missing timeout has a recognisable symptom. A service hangs with low CPU and no errors in its logs, because every worker is parked waiting on a slow downstream call. One stalled dependency can then produce a [cascading failure](./Cascading%20failure.md) through everything that calls this service.

| Kind            | What it limits                                  | Typical size                    |
| --------------- | ----------------------------------------------- | ------------------------------- |
| Connect         | Establishing the connection                     | Short, often around a second    |
| Read            | Waiting between bytes of the response           | Based on the dependency's p99   |
| Overall request | The whole call from start to finish             | Slightly above the read timeout |
| Deadline        | A budget shared by every hop in a request chain | Set once at the entry point     |

A read timeout alone misses a server that sends one byte every few seconds, so set an overall limit too.

To choose a value, measure the dependency's p99 [latency](./Latency.md) and add a margin. Too short, and healthy requests fail during small slowdowns. Too long, and the protection arrives after your threads are exhausted.

Timeouts must fit together across layers. If a gateway gives up after 5 seconds while its backend keeps working for 30, the backend spends capacity on an answer nobody will read. Pass a deadline with the request, so that each service knows how much time remains.

A timeout does not mean the operation failed. The server may have finished the payment after the client stopped waiting. Before using [retry with backoff](./Retry%20with%20backoff.md), make sure repeating the operation is safe.

_Usage:_

"The checkout pods are all stuck, but CPU is at 3% and nothing's logged."

"The fraud API is hanging and that client has no timeout, so every worker is waiting on it."
