---
description: A statement of what a system must do, such as letting a user upload a photo or cancel an order.
---

A functional requirement is a statement of what a system must do: a user can upload a photo, a customer can cancel an order, an admin can export last month's invoices. It describes behaviour you can check with a yes or a no.

Functional requirements come first in a design because they determine the data and the operations. "Users post short messages and see posts from people they follow" already implies a store of users, a follow relationship, a store of posts and a timeline read. How that timeline gets built, for example with [fan-out on write](./Fan-out%20on%20write.md), depends on what the feature has to show.

They are paired with [non-functional requirements](./Non-functional%20requirement.md), which say how well the system must do those things. "Users can search products" is functional. "Search returns within 200 ms" is non-functional.

A short set of questions usually produces the list:

| Question                   | Example answer                             |
| -------------------------- | ------------------------------------------ |
| Who uses it?               | Shoppers, warehouse staff, support agents  |
| What can they do?          | Place an order, track it, request a refund |
| What data goes in and out? | Order lines in, order status out           |
| What is out of scope?      | Gift cards, third-party sellers            |

The out-of-scope row prevents a common failure. A team builds for weeks, then learns in review that "notifications" meant SMS as well as email, or that refunds had to support part of an order. Writing the list down and getting it confirmed exposes that disagreement while it is still inexpensive to fix.

Keep the list short during a design discussion. Pick the few operations that drive the architecture, and leave settings pages and profile edits aside unless they change the shape of the system.

_Usage:_

"Should the feed include reposts, or only original posts?"

"Nobody wrote that down. Let's settle the functional requirements before we argue about storage, because reposts change what each timeline has to hold."
