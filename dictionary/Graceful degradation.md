---
description: Keeping the main functions of a system working with reduced features when a part of it fails or is overloaded.
---

Graceful degradation means keeping the main functions of a system working, with reduced features, when part of it fails or is overloaded. The alternative is all-or-nothing behaviour, where one optional component takes the whole page down. A product page that returns a 500 because the recommendations widget timed out is a typical example. The customer could have bought the item without seeing recommendations.

Designing for it starts with sorting features by how much they matter, then deciding what each one does when its dependency is unavailable:

| Feature           | Normal       | Degraded                            |
| ----------------- | ------------ | ----------------------------------- |
| Recommendations   | Personalised | Generic bestseller list, or hidden  |
| Search            | Full ranking | Simpler ranking, fewer results      |
| Reviews           | Live data    | [Cache](./Cache.md) copy, or hidden |
| Video             | High quality | Lower bitrate                       |
| Shipping estimate | Exact date   | "Usually ships in 3–5 days"         |
| Checkout          | Full flow    | Stays on, other features give way   |

The mechanics are usually simple. Give each optional call a short [timeout](./Timeout.md), wrap it in a [circuit breaker](./Circuit%20breaker.md), and write a fallback. Feature flags let operators turn off expensive features by hand during an incident, which pairs well with [load shedding](./Load%20shedding.md).

Not everything should degrade. A fallback that quietly shows an old account balance or a stale stock count can cause more harm than an error. For payments, permissions and other data that must be correct, return an error. Tell the user when something is reduced if it affects their decisions.

Fallback paths need testing. Code that runs only during outages is often broken when it is finally needed, and a fallback that calls the same failing database gives no protection at all. Inject failures in a test environment to verify each one. Record when a fallback is active as well, because a degraded mode that nobody can see can run unnoticed for weeks.

_Usage:_

"The reviews service is down. Do we need to pull the product page?"

"No, it has graceful degradation. The page renders from cache and hides the review section."
