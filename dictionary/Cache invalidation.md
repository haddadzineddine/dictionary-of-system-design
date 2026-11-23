---
description: Removing or updating cached copies when the source data changes, so that readers stop receiving stale values.
---

[Cache](./Cache.md) invalidation is the removal or updating of cached copies when the source data changes. A cache does not know that a database row has been modified, so something must tell it, or the copy must expire. Until that happens, readers get stale data. A customer changes their shipping address, sees the old one on the checkout page, and contacts support.

| Strategy               | How it works                                                  | Trade-off                          |
| ---------------------- | ------------------------------------------------------------- | ---------------------------------- |
| [TTL](./TTL.md) expiry | Copy expires after a set time                                 | Simple, but stale until it expires |
| Delete on write        | Writer removes the key after updating                         | Must find every affected key       |
| Versioned keys         | Key includes a version, such as `user:42:v7`                  | Old entries linger until evicted   |
| Change events          | Database changes are streamed to a consumer that deletes keys | Accurate, more infrastructure      |

Finding every affected key is often the difficult part. A product's price may appear in the product entry, a category listing, search results and a cached homepage. Deleting the first and forgetting the rest gives pages that disagree with each other.

Timing is the other difficulty. Between a database commit and the delete, readers can still get the old value, and a slow reader can write an old value back after the delete, as in the race described under [cache-aside](./Cache-aside.md). Several layers add to it, because clearing Redis does nothing for a copy in a [CDN](./CDN.md) or a browser.

The usual practice combines methods: delete the key on write for prompt updates, and set a TTL so that any missed invalidation corrects itself. Decide how much staleness each kind of data can tolerate. A follower count can lag by minutes, while a permission change should take effect quickly and may be better left uncached.

_Usage:_

"We removed her admin role, but she can still open the settings page."

"Permissions are cached per session and nothing handles invalidation on role change. Delete that key when the role is updated."
