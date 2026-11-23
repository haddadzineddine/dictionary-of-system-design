---
description: A server that sits in front of backend servers, receives client requests on their behalf, and forwards them.
---

A reverse proxy is a server that sits in front of one or more backend servers, receives client requests on their behalf, and forwards them. The client talks only to the proxy and usually cannot tell what is behind it. A forward proxy is the mirror image: it acts for clients, for example a company proxy that controls outbound web access.

Putting one component at the entrance lets it handle work that every backend would otherwise repeat:

| Job                   | What the proxy does                                                 |
| --------------------- | ------------------------------------------------------------------- |
| TLS termination       | Handles HTTPS so backends can receive decrypted traffic             |
| Routing               | Sends paths or hostnames to different services                      |
| Balancing             | Spreads requests, acting as a [load balancer](./Load%20balancer.md) |
| [Caching](./Cache.md) | Stores responses and serves repeats                                 |
| Compression           | Shrinks responses before sending them                               |
| Buffering             | Absorbs slow clients so backend workers are freed quickly           |
| Static files          | Serves assets without involving the application                     |

The proxy also introduces a class of errors that can confuse debugging. A 502 Bad Gateway means the proxy got an invalid response or no usable connection from the backend, often because the application crashed or is not listening. A 504 Gateway [Timeout](./Timeout.md) means the backend did not answer within the proxy's time limit. In both cases the error page comes from the proxy, so the application's logs may show nothing, or show a request that completed successfully after the proxy gave up.

Another common surprise is that every request appears to come from the proxy's IP address. Proxies pass the original address in a header such as `X-Forwarded-For`. Clients can forge that header, so trust only the values added by your own proxies, particularly for [rate limiting](./Rate%20limiting.md) and audit logs.

NGINX, HAProxy, Caddy and Envoy are examples. An [API gateway](./API%20gateway.md) is a reverse proxy with API-specific features added.

_Usage:_

"Users get a 502, but there's nothing in the app logs."

"The reverse proxy generates that. The app probably crashed and restarted, so check for container restarts."
