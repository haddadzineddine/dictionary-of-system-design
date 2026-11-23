---
description: Storing every change to an entity as an append-only sequence of events and deriving current state by replaying them.
---

Event sourcing stores every change to an entity as an append-only sequence of events, and derives the current state by replaying them. A conventional table keeps only the latest value and overwrites the previous one. An event-sourced system keeps the full history, in the way a bank ledger records transactions and calculates a balance from them.

| Sequence | Event           | Balance after replay |
| -------- | --------------- | -------------------- |
| 1        | `AccountOpened` | 0                    |
| 2        | `Deposited 100` | 100                  |
| 3        | `Withdrawn 30`  | 70                   |
| 4        | `Deposited 50`  | 120                  |

The event log is the source of truth. This gives a complete audit trail, lets you reconstruct the state at any past moment, and answers questions such as "how did this order end up cancelled?", which an overwritten row cannot. It also lets you build new read models from existing history, which is why it is often combined with [CQRS](./CQRS.md).

It is different from [event streaming](./Event%20streaming.md). A stream moves events between systems. Event sourcing uses events as the primary storage for an entity's state, and needs to load all events for one entity reliably and reject conflicting writes, usually by checking an expected version number on append.

The costs tend to appear later:

| Problem        | Why it happens                     | Usual response                            |
| -------------- | ---------------------------------- | ----------------------------------------- |
| Slow loads     | Long histories take time to replay | Periodic snapshots of state               |
| Schema change  | Old events are immutable           | Versioned events, upgrade them when read  |
| Personal data  | Logs are not meant to be edited    | Encrypt per user, delete the key to erase |
| Querying       | Log is poor for "find all X"       | Separate read models                      |
| Replay effects | Replaying could resend emails      | Keep side effects out of state rebuilding |

Use event sourcing where history is itself a business requirement, as in accounting or compliance. For ordinary data, a table plus an audit log is much simpler.

_Usage:_

"Support can't tell how this invoice reached a negative total."

"If billing used event sourcing, we could replay it step by step. Today we only have the final row."
