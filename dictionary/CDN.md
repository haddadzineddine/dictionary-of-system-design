---
description: A content delivery network, a set of servers in many locations that cache content close to users.
---

A CDN, or content delivery network, is a set of servers in many geographic locations that [cache](./Cache.md) content close to users. Those servers are called the edge. Your own server is called the origin. A user in Tokyo can get an image from a nearby edge location instead of an origin in Virginia, which lowers [latency](./Latency.md) and origin load.

A request moves through these steps:

| Step | Who     | What happens                                                      |
| ---- | ------- | ----------------------------------------------------------------- |
| 1    | Network | DNS or network routing sends the user to a nearby edge location   |
| 2    | Edge    | Looks up the response by its cache key, usually the URL           |
| 3    | Edge    | On a hit, returns the stored copy                                 |
| 4    | Origin  | On a miss, receives the request and returns the response          |
| 5    | Edge    | Stores it according to its `Cache-Control` headers, then responds |

CDNs suit images, video, scripts and stylesheets, and can also cache API responses that are the same for many users.

The familiar failure is stale content. You deploy a fix and some users still get the old stylesheet, because edge locations hold it until its [TTL](./TTL.md) expires. Purging helps but takes time to reach every location and does not clear browser caches. A more dependable method is to put a content hash in the filename, such as `app.3f9c2.css`. Each release then has a new URL, and the file can be cached for a long time.

A more serious failure is caching personalised content. If an account page is cached under a URL shared by all users, one person can receive another's data. Mark such responses `Cache-Control: private` or `no-store`, and make sure the cache key includes anything the response varies on.

Cloudflare, Amazon CloudFront and Fastly are examples.

_Usage:_

"I shipped the CSS fix an hour ago, but customers still see the broken layout."

"The CDN is still serving the old file. We need hashed filenames so each release gets a new URL."
