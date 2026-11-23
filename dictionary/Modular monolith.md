---
description: A single deployable application divided into modules with enforced boundaries, each owning its own logic and data.
---

A modular [monolith](./Monolith.md) is a single deployable application divided into modules with enforced boundaries. Each module covers one business area, such as billing or inventory, owns its logic and data, and exposes a small public interface to the others. It deploys as one unit but is organised internally much like a set of services.

The pattern addresses a familiar symptom in large codebases. Someone renames a column in billing, and shipping breaks because it had been querying that table directly. The cause is that any code can reach any other code.

A modular monolith depends on a few rules:

| Rule                  | In practice                                                |
| --------------------- | ---------------------------------------------------------- |
| Public interface      | Modules call only what another module explicitly exposes   |
| Private internals     | Importing another module's internals fails the build       |
| Data ownership        | Each module owns its tables, with no cross-module joins    |
| Explicit dependencies | Allowed dependencies are declared, and cycles are rejected |
| Communication         | Direct calls, or in-process events for loose coupling      |

Enforcement is what separates this from a folder structure. Conventions erode under deadline pressure, so check boundaries automatically in CI. ArchUnit for Java and import-linter for Python are examples of tools for this.

The approach keeps most of the simplicity of a monolith: one deploy, in-process calls, and straightforward local development. A single transaction can still span modules when necessary, although relying on that makes a later split harder.

Its limits are those of any single deployable. Modules share a runtime, so a crash or memory leak in one affects all of them.

Clean boundaries keep future options open. If one module later needs independent scaling, its interface and data are already separated, so extracting it into one of a set of [microservices](./Microservices.md) is a contained piece of work.

_Usage:_

"We keep breaking each other's code, so should we move to services?"

"The problem is boundaries, not deployment. Try a modular monolith with import rules enforced in CI first."
