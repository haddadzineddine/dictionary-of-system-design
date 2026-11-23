---
description: A dedicated backend layer for one type of client, shaping data and calls to suit that client's needs.
---

A backend for frontend, or BFF, is a dedicated server-side layer built for one type of client, such as the web app or the mobile app. It calls the underlying services and returns data shaped for that client's screens.

The pattern addresses a recognisable problem with a shared API. Mobile users download large responses and discard most of the fields. A single phone screen needs six round trips, which is slow on a poor connection. Meanwhile, the API accumulates parameters and `if (client == "ios")` branches, and a change requested by one client team risks breaking another.

| Client  | What its BFF does                                          |
| ------- | ---------------------------------------------------------- |
| Mobile  | Combines calls into one, trims fields, resizes image URLs  |
| Web     | Returns richer data, manages sessions with secure cookies  |
| TV      | Returns simple layouts suited to remote-control navigation |
| Partner | Exposes a stable, versioned contract                       |

A BFF is usually owned by the same team as the client it serves, so that team can change its API without waiting for another group. For browser apps, it can also keep access tokens on the server and give the browser only a session cookie, which limits what a malicious script could steal.

It differs from an [API gateway](./API%20gateway.md). A gateway is one shared entry point handling generic concerns such as authentication and [rate limiting](./Rate%20limiting.md). A BFF contains logic specific to one client, and often sits behind a gateway.

The main risk is duplication. Business rules that creep into each BFF will drift apart, so the web app and the mobile app calculate a discount differently. Keep BFFs to aggregation and formatting, and leave rules in the underlying services. GraphQL, a query language that lets clients request specific fields, is a common alternative when the main need is flexible data fetching.

_Usage:_

"The app makes eleven calls to render the home screen."

"Add a backend for frontend for mobile, so it makes one call and gets exactly what the screen needs."
