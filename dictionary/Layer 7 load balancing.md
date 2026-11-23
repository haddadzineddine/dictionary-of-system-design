---
description: Distributing individual requests by reading application data such as the URL path, headers and cookies.
---

Layer 7 load balancing distributes individual requests by reading application data such as the URL path, headers and cookies. The name refers to the application layer of the OSI networking model, where HTTP sits. The [load balancer](./Load%20balancer.md) ends the client's connection, reads the request, chooses a backend, and sends the request over its own connection. In this role it is a type of [reverse proxy](./Reverse%20proxy.md).

Reading the request allows routing rules that lower layers cannot express. `/api/*` can go to one service and `/images/*` to another, a header can send 5% of users to a canary release (a small trial deployment), and a cookie can support a [sticky session](./Sticky%20session.md).

| Concern                       | [Layer 4](./Layer%204%20load%20balancing.md) | Layer 7                          |
| ----------------------------- | -------------------------------------------- | -------------------------------- |
| Routes on                     | IP address and port                          | Path, host, headers, cookies     |
| Balances                      | Connections                                  | Individual requests              |
| Protocols                     | Any TCP or UDP                               | Those it can parse, such as HTTP |
| TLS                           | Can pass through                             | Usually terminated here          |
| Overhead                      | Lower                                        | Higher                           |
| Request retries, header edits | Not available                                | Supported                        |

The cost is more work per request. The balancer must decrypt traffic, parse it and hold state for two connections, and it needs access to your TLS private keys. It also adds its own limits, which produce confusing failures. A file upload that works locally returns 413 in production because of a body-size limit on the balancer. A long report returns 504 because the balancer's [timeout](./Timeout.md) is shorter than the application's. When an error never appears in your application logs, check the balancer's logs.

Automatic retries at this layer need care. Retrying a failed POST can apply it twice unless the operation is idempotent, meaning repeating it has the same effect as doing it once.

NGINX, Envoy and AWS Application Load Balancer are common examples.

_Usage:_

"Can we send only `/search` traffic to the new service?"

"Yes, the ingress does layer 7 load balancing, so it's one path rule."
