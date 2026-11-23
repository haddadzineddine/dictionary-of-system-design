---
description: A helper process deployed alongside an application instance to provide supporting functions without changing its code.
---

A sidecar is a helper process deployed alongside each application instance to provide supporting functions without changes to the application's code. The two run together, start and stop together, and usually talk over localhost. In Kubernetes, a sidecar is typically a second container in the same pod, sharing its network and optionally its storage.

The pattern is useful when many services, possibly in different languages, need the same capability. Without it, every team installs and upgrades its own library, and versions drift.

| Sidecar job    | What it does                                            | Example                 |
| -------------- | ------------------------------------------------------- | ----------------------- |
| Traffic proxy  | Handles TLS, retries, [timeouts](./Timeout.md), routing | Envoy                   |
| Log shipping   | Reads local logs and forwards them                      | Fluent Bit              |
| Metrics        | Collects and exports telemetry                          | OpenTelemetry Collector |
| Secrets        | Fetches and refreshes credentials                       | Vault Agent             |
| Database proxy | Manages secure connections                              | Cloud SQL Auth Proxy    |

A service mesh, a dedicated layer for service-to-service traffic, commonly places a proxy sidecar beside every instance. Istio and Linkerd are examples.

Lifecycle ordering causes the most familiar bugs. An application starts before its proxy is ready, and its first network calls fail with connection refused. During shutdown, the proxy exits first, and in-flight requests are dropped. In a batch job, the main container finishes while the sidecar keeps running, so the job never completes. Kubernetes now has native sidecar containers that start before the main containers and stop after them, which addresses much of this.

The costs multiply with scale. A sidecar using 100 MB of memory across 1,000 pods consumes about 100 GB. A proxy also adds a local network hop to every request, adding some [latency](./Latency.md).

For a single service, a library is usually simpler.

_Usage:_

"Half the pods throw connection errors for the first few seconds after boot."

"The app starts before the Envoy sidecar is ready. Make it a native sidecar with a startup probe, so the app waits until the proxy is ready."
