---
description: Copying new content to every recipient's feed when it is created, so that later reads are fast and already prepared.
---

[Fan-out](./Fan-out.md) on write means copying new content to every recipient's feed at the moment it is created. Each user has a precomputed timeline, usually a list of post IDs in a fast store, and reading it is one lookup. It is one of two ways to build a feed, the other being [fan-out on read](./Fan-out%20on%20read.md). Both are forms of data fan-out.

The write path usually runs in the background:

| Step | Who     | What happens                                                                              |
| ---- | ------- | ----------------------------------------------------------------------------------------- |
| 1    | API     | Saves the post                                                                            |
| 2    | API     | Puts a job on a [message queue](./Message%20queue.md), then returns success to the author |
| 3    | Worker  | Looks up the author's followers                                                           |
| 4    | Workers | Insert the post ID into each follower's stored timeline                                   |
| 5    | Reader  | Fetches their timeline with a single read                                                 |

This suits a [read-heavy workload](./Read-heavy%20workload.md). The expensive work happens once per post, and the many reads that follow are inexpensive and predictable.

The weakness is accounts with very large audiences. One post from an account with 20 million followers creates 20 million timeline writes. The queue backs up, and followers see that post minutes late, while posts from smaller accounts queued behind it are delayed too.

Storage grows as well, since the same post ID is stored once per follower. Systems limit this by keeping only the most recent few hundred entries per timeline and by skipping inactive users. Edits and deletes are also harder, because removing a post means finding every copy, or filtering deleted posts at read time.

Large systems commonly use a hybrid. Ordinary accounts fan out on write, while posts from very large accounts are fetched and merged when a follower reads.

_Usage:_

"Why did the feed queue spike to two million jobs at noon?"

"A celebrity account posted. With fan-out on write, that's one job per follower, so we should exclude big accounts and merge them at read time."
