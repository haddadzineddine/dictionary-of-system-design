# System Design Dictionary

A plain-English dictionary of how to shape a system: estimation, scaling, traffic, caching, asynchronous work, resilience patterns and architecture styles.

Why did adding a cache make the outage worse? Why did retries take the service down? When is a monolith the right answer?

Each of these has a short explanation once you know the right words. Entries define the term, describe the failure it explains where there is one, and end with a short example of engineers using it.

---

## Table of contents

- **[Section 1 — Thinking in Systems](#section-1--thinking-in-systems)**
  - [Functional requirement](#functional-requirement)
  - [Non-functional requirement](#non-functional-requirement)
  - [Back-of-the-envelope estimation](#back-of-the-envelope-estimation)
  - [Latency](#latency)
  - [Throughput](#throughput)
  - [Bandwidth](#bandwidth)
  - [Bottleneck](#bottleneck)
  - [Single point of failure](#single-point-of-failure)
  - [Read-heavy workload](#read-heavy-workload)
  - [Write-heavy workload](#write-heavy-workload)
- **[Section 2 — Scaling](#section-2--scaling)**
  - [Vertical scaling](#vertical-scaling)
  - [Horizontal scaling](#horizontal-scaling)
  - [Stateless service](#stateless-service)
  - [Sticky session](#sticky-session)
  - [Consistent hashing](#consistent-hashing)
  - [Fan-out](#fan-out)
  - [Fan-out on write](#fan-out-on-write)
  - [Fan-out on read](#fan-out-on-read)
  - [Hot key](#hot-key)
- **[Section 3 — Traffic](#section-3--traffic)**
  - [Load balancer](#load-balancer)
  - [Layer 4 load balancing](#layer-4-load-balancing)
  - [Layer 7 load balancing](#layer-7-load-balancing)
  - [Reverse proxy](#reverse-proxy)
  - [API gateway](#api-gateway)
  - [CDN](#cdn)
  - [Service discovery](#service-discovery)
  - [Rate limiting](#rate-limiting)
  - [Token bucket](#token-bucket)
  - [Sliding window counter](#sliding-window-counter)
- **[Section 4 — Caching](#section-4--caching)**
  - [Cache](#cache)
  - [Cache hit ratio](#cache-hit-ratio)
  - [Cache-aside](#cache-aside)
  - [Write-through cache](#write-through-cache)
  - [Write-behind cache](#write-behind-cache)
  - [Cache invalidation](#cache-invalidation)
  - [TTL](#ttl)
  - [Eviction policy](#eviction-policy)
  - [Cache stampede](#cache-stampede)
- **[Section 5 — Asynchronous Work](#section-5--asynchronous-work)**
  - [Message queue](#message-queue)
  - [Publish-subscribe](#publish-subscribe)
  - [Event streaming](#event-streaming)
  - [Consumer group](#consumer-group)
  - [Backpressure](#backpressure)
  - [Dead letter queue](#dead-letter-queue)
  - [Outbox pattern](#outbox-pattern)
- **[Section 6 — Resilience](#section-6--resilience)**
  - [Timeout](#timeout)
  - [Retry with backoff](#retry-with-backoff)
  - [Retry storm](#retry-storm)
  - [Circuit breaker](#circuit-breaker)
  - [Bulkhead](#bulkhead)
  - [Graceful degradation](#graceful-degradation)
  - [Load shedding](#load-shedding)
  - [Cascading failure](#cascading-failure)
- **[Section 7 — Architecture Styles](#section-7--architecture-styles)**
  - [Monolith](#monolith)
  - [Modular monolith](#modular-monolith)
  - [Microservices](#microservices)
  - [Event-driven architecture](#event-driven-architecture)
  - [CQRS](#cqrs)
  - [Event sourcing](#event-sourcing)
  - [Saga](#saga)
  - [Strangler fig](#strangler-fig)
  - [Backend for frontend](#backend-for-frontend)
  - [Sidecar](#sidecar)

## Section 1 — Thinking in Systems

### Functional requirement

A functional requirement is a statement of what a system must do: a user can upload a photo, a customer can cancel an order, an admin can export last month's invoices. It describes behaviour you can check with a yes or a no.

Functional requirements come first in a design because they determine the data and the operations. "Users post short messages and see posts from people they follow" already implies a store of users, a follow relationship, a store of posts and a timeline read. How that timeline gets built, for example with [fan-out on write](#fan-out-on-write), depends on what the feature has to show.

They are paired with [non-functional requirements](#non-functional-requirement), which say how well the system must do those things. "Users can search products" is functional. "Search returns within 200 ms" is non-functional.

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

### Non-functional requirement

Where a [functional requirement](#functional-requirement) says what a system must do, a non-functional requirement says how well it must do it. For "users can search products", the non-functional version is "95% of searches return within 200 ms at 5,000 searches per second".

These requirements shape the architecture more than the feature list does. A photo-sharing app for 50 people and one for 50 million have similar features and very different designs. A system that works in a demo and times out on launch day usually met its functional requirements and had no stated target for load.

| Quality                       | Example target                      | Design consequence                                                           |
| ----------------------------- | ----------------------------------- | ---------------------------------------------------------------------------- |
| [Latency](#latency)       | 99% of page loads under 500 ms      | [Caches](#cache), fewer network hops                                     |
| [Throughput](#throughput) | 5,000 orders per second at peak     | [Horizontal scaling](#horizontal-scaling), queues                      |
| Availability                  | 99.9% of requests succeed monthly   | Redundancy, no [single point of failure](#single-point-of-failure) |
| Durability                    | A confirmed payment is never lost   | Replicated storage, backups                                                  |
| Consistency                   | Balance is correct after a transfer | Limits on caching and asynchronous updates                                   |
| Cost                          | Under a set monthly budget          | Smaller fleet, simpler components                                            |

Targets need numbers to be useful. "The system must be fast" cannot be tested and cannot settle an argument. A number also shows what a goal costs. An availability target of 99.9% allows about 43 minutes of downtime in a 30-day month, while 99.99% allows about 4 minutes, which is too short for a person to respond to most failures and generally requires automated failover.

The qualities also compete. Stronger consistency tends to raise latency, and higher availability raises cost. Ask which requirement wins when two conflict, because that answer decides many later design choices. A payment system and a social feed will answer differently.

_Usage:_

"Do we really need multi-region for the internal reporting tool?"

"Check the non-functional requirements. It needs 99.5% during office hours, so one region with a tested restore is enough."

### Back-of-the-envelope estimation

Back-of-the-envelope estimation is a quick calculation with rounded numbers that tells you the rough size of a system before you design it. The aim is the right order of magnitude. Knowing whether a service handles 10, 1,000 or 100,000 requests per second decides whether one server is enough or a distributed design is needed.

A useful shortcut is that a day has 86,400 seconds, which rounds to 100,000. Here is a worked estimate for a social app with 10 million daily users:

| Step | Quantity                         | Calculation                       | Result             |
| ---- | -------------------------------- | --------------------------------- | ------------------ |
| 1    | Reads per day                    | 10M users × 20 reads              | 200M               |
| 2    | Average reads                    | 200M ÷ 86,400 s                   | About 2,300/s      |
| 3    | Peak reads                       | 2,300/s × 3 (assumed peak factor) | About 7,000/s      |
| 4    | Writes per day                   | 10M users × 0.1 posts             | 1M, about 12/s     |
| 5    | Storage per year                 | 1M posts × 2 KB × 365             | About 730 GB       |
| 6    | Peak [bandwidth](#bandwidth) | 7,000/s × 50 KB                   | 350 MB/s, 2.8 Gbps |

The result already suggests a design. Reads outnumber writes 200 to 1, so this is a [read-heavy workload](#read-heavy-workload) and a [cache](#cache) will help. Storage fits on one machine for years, so splitting the database across machines is not yet justified.

Estimation prevents two opposite mistakes. One is building a sharded, multi-region platform for traffic a single server could handle. The other is shipping a design that hits its [bottleneck](#bottleneck) in the first week because nobody multiplied the expected users by the payload size.

State every assumption next to its number, because the assumptions are what reviewers can challenge. Plan for peak rather than average traffic, and remember that bits and bytes differ by a factor of eight.

_Usage:_

"We'll need Kafka and a dozen nodes for the audit log, right?"

"Do the back-of-the-envelope estimation first. It's 40 writes per second at peak, and one Postgres instance can handle that."

### Latency

Latency is the time one operation takes, from the moment it starts to the moment its result arrives. For a web request, that includes network travel, time spent waiting in queues, processing, and the trip back. Some texts reserve "latency" for the waiting part and use "response time" for the total, but in everyday use engineers mean the total.

Latency is reported in percentiles because averages hide the slow requests. The p50 is the median. The p99 is the value that 99% of requests beat. A dashboard can show an 80 ms average while users complain about a slow checkout, because the p99 is 3 seconds. A person who loads many pages per session will run into that slow 1% regularly.

Rough magnitudes are useful for reasoning about where time goes:

| Operation                             | Rough time                       |
| ------------------------------------- | -------------------------------- |
| Read from main memory                 | About 100 nanoseconds            |
| Random read from an SSD               | Tens to hundreds of microseconds |
| Network round trip in a data centre   | Under 1 millisecond              |
| Network round trip between continents | About 50 to 300 milliseconds     |

These gaps explain many design choices. A [cache](#cache) in memory avoids a slower lookup, and a [CDN](#cdn) moves content closer to the user because no amount of server tuning shortens the distance light travels through fibre.

Latency also depends on load. As a component approaches its capacity, requests spend more time queueing, so latency climbs sharply before [throughput](#throughput) reaches its limit. A rising p99 is often the first visible sign of a [bottleneck](#bottleneck).

To reduce latency, measure each stage of a request first. Then remove network hops, run independent calls in parallel, or do the work ahead of time.

_Usage:_

"The average is fine, so why is support getting complaints about search?"

"Look at the p99 latency. One request in a hundred takes four seconds, and the average hides it."

### Throughput

Throughput is the amount of work a system completes per unit of time. It is measured in whatever unit fits the job: requests per second, orders per minute, messages per second, or gigabytes per hour. It counts completed work, so a server that accepts 10,000 requests per second and finishes 6,000 has a throughput of 6,000.

Throughput and [latency](#latency) are related but separate. Latency is how long one request takes. Throughput is how many finish per second. A system can have high throughput and poor latency, as a batch job does, or low latency and low throughput, as a single fast worker does.

Little's law connects them. The average number of requests in flight equals throughput multiplied by the average time each one spends in the system:

| Throughput | Average latency | Requests in flight |
| ---------- | --------------- | ------------------ |
| 1,000/s    | 50 ms           | 50                 |
| 1,000/s    | 200 ms          | 200                |
| 1,000/s    | 2 s             | 2,000              |

The table explains a common incident. A database slows from 50 ms to 2 seconds, and at the same arrival rate the number of in-flight requests grows forty-fold. Connection pools and worker threads fill up, and the service starts failing even though traffic did not change.

Every system has a maximum throughput, set by its [bottleneck](#bottleneck). As load approaches that limit, requests queue and latency rises. Beyond it, throughput can fall, because the system spends effort on requests that time out before finishing. This is the reason for [load shedding](#load-shedding).

To raise throughput, remove the limiting resource, add capacity with [horizontal scaling](#horizontal-scaling), or batch small operations so each unit of overhead covers more work. Load-test to find the real limit, because a capacity figure derived from one instance rarely holds once instances share a database.

_Usage:_

"We doubled the workers, but we still process about 800 jobs a second."

"Then workers aren't what limits throughput. Check the database, it's probably saturated."

### Bandwidth

Bandwidth is the maximum rate at which a network link can carry data, usually stated in bits per second. A 10 Gbps network card can move at most 10 gigabits each second. It is a capacity. What you actually achieve is [throughput](#throughput), which is lower because of protocol overhead, congestion and the speed of whatever sits at each end.

Network capacity is quoted in bits, while file sizes are quoted in bytes, and mixing them up produces estimates that are wrong by a factor of eight. A 1 Gbps link carries at most 125 megabytes per second.

| Link     | Maximum in bytes | Time to move 1 TB at full rate |
| -------- | ---------------- | ------------------------------ |
| 1 Gbps   | 125 MB/s         | About 2.2 hours                |
| 10 Gbps  | 1.25 GB/s        | About 13 minutes               |
| 100 Gbps | 12.5 GB/s        | About 80 seconds               |

Bandwidth and [latency](#latency) solve different problems. Bandwidth is the width of the pipe, and latency is how long a byte takes to reach the other end. A larger pipe speeds up a big download but does little for a small API call, whose duration is dominated by round trips. Upgrading a link will not fix a chatty service that makes thirty sequential calls across regions.

Bandwidth becomes the [bottleneck](#bottleneck) when payloads are large. A service sending 500 KB images 2,000 times per second needs 1 GB/s, or 8 Gbps, before overhead. On a 10 Gbps card that leaves little headroom, so the service can stall with low CPU and plenty of free memory. When a server looks idle and responses are still slow, check network utilisation.

Common remedies are compressing responses, resizing images, paginating large results, and serving static files from a [CDN](#cdn). These also reduce data transfer charges, which cloud providers typically bill on outbound traffic.

_Usage:_

"CPU is at 20% on the media servers, but downloads crawl at peak."

"Check the NICs. We're pushing 9 Gbps through 10 Gbps cards, so we've run out of bandwidth."

### Bottleneck

A bottleneck is the resource or component that limits how much work a whole system can do. A request path can only move as much work as its most constrained step allows. If the web tier handles 10,000 requests per second and the database handles 2,000, the system's [throughput](#throughput) is 2,000.

The usual symptom is that adding capacity has no effect. A team doubles the application servers, and the graphs do not move, because every server is waiting on the same saturated database. In some cases things get worse, since more servers open more connections to the component that is already struggling.

To find a bottleneck, look for the place where work piles up:

| Step | What to check                                                                       |
| ---- | ----------------------------------------------------------------------------------- |
| 1    | Trace a slow request and see which stage takes the time                             |
| 2    | Check utilisation of CPU, memory, disk and [bandwidth](#bandwidth) on each tier |
| 3    | Look for growing queues, full connection pools and lock waits                       |
| 4    | Fix the limiting resource                                                           |
| 5    | Load-test again, because the limit will now sit somewhere else                      |

Step 5 matters because removing one bottleneck exposes the next. Add a [cache](#cache) in front of the database and the limit may move to the network or to a single busy partition.

A bottleneck is not always a whole machine. It can be one lock, one [hot key](#hot-key), a single-threaded process using one CPU while the others sit idle, or an external API with a quota. Average utilisation can hide all of these, so inspect individual instances and resources.

Fixes depend on the cause. Options include doing less work per request, adding capacity, splitting the load, or moving slow work to a [message queue](#message-queue) so that it no longer holds up the request.

_Usage:_

"We scaled the API to twenty pods and [latency](#latency) didn't change."

"Then the API isn't the bottleneck. The database has been pinned at 100% CPU since this morning."

### Single point of failure

A single point of failure is a component whose failure stops the whole system because nothing else can take over its work. It may be an obvious part, such as the only database server, or a less visible one, such as a single DNS provider, one expiring TLS certificate, or a configuration service every application reads at startup.

The way to remove one is redundancy: run more than one copy, and have a tested way to move work to a surviving copy.

| Component           | Failure without redundancy | Common remedy                                                    |
| ------------------- | -------------------------- | ---------------------------------------------------------------- |
| App server          | Site goes down             | Several instances behind a [load balancer](#load-balancer) |
| Load balancer       | All traffic stops          | Active-passive pair or a managed service                         |
| Primary database    | Writes stop                | Standby copy with automatic failover                             |
| [Cache](#cache) | Database takes full load   | Replicated cache, capacity to survive a cold start               |
| Data centre         | Everything in it stops     | Deploy across availability zones (isolated sites)                |

Redundancy only helps if the copies fail independently. Two servers in one rack share a power feed and a network switch. Two services in different zones may both depend on one authentication service. A design diagram can show everything doubled while a shared dependency remains underneath, and teams often discover it during the outage.

Failover also has to work when it is needed. A standby that has silently fallen behind, or a procedure nobody has run in a year, gives false confidence. Test failover on a schedule by actually removing the component.

Removing every single point of failure is rarely worthwhile. Each layer of redundancy adds cost and operational complexity, and the failover machinery can itself fail. Compare the cost of an outage with the cost of the extra copy, using the availability target from your [non-functional requirements](#non-functional-requirement).

_Usage:_

"We run three API instances across zones, so we're safe if one goes."

"All three read their config from that one Redis box at boot. That's still a single point of failure."

### Read-heavy workload

A read-heavy workload is one where reads far outnumber writes. Product catalogues, news sites, documentation and social feeds are typical examples, with ratios of 100 reads to one write being common. The ratio matters because reads and writes scale in different ways.

Reads are easier to scale because they do not change anything. The same answer can be copied to many places and served from any of them. The options form a ladder, and each rung adds some complexity:

| Option              | What it does                             | Trade-off                  |
| ------------------- | ---------------------------------------- | -------------------------- |
| Add indexes         | Lets the database find rows faster       | Slightly slower writes     |
| [Cache](#cache) | Serves repeated reads from memory        | Data can be stale          |
| [CDN](#cdn)     | Serves content from locations near users | Harder to purge            |
| Read replicas       | Database copies that answer read queries | Copies can lag the primary |
| Precomputed views   | Stores the finished answer ahead of time | Extra work on each write   |

The usual symptom of an unaddressed read-heavy workload is a database at high CPU while running the same few queries thousands of times per second. A [cache-aside](#cache-aside) layer often removes most of that load.

The price of these techniques is staleness. A read replica applies changes shortly after the primary, so a user can save a profile, reload and see the old value because the second request hit a copy that has not caught up. For that case, send a user's reads to the primary for a short period after they write, or read that user's own data from the primary.

Measure the ratio before relying on it. Some workloads that look read-heavy have write bursts, and a [write-heavy workload](#write-heavy-workload) calls for different techniques.

_Usage:_

"The catalogue database is at 90% CPU, should we shard it?"

"It's a read-heavy workload with about a thousand reads per write. Put a cache in front before splitting anything."

### Write-heavy workload

A write-heavy workload is one where a large share of the traffic creates or changes data. Metrics collection, application logs, sensor readings, click tracking and chat messages are typical examples.

Writes are harder to scale than reads because each one must reach durable storage and stay correct. A [cache](#cache) does little to absorb them. Read replicas (database copies that serve queries) do not help either, because every replica has to apply every write, and heavy write load can make those copies fall behind.

The symptoms differ from a read problem. Disk I/O saturates, lock waits climb, replicas lag by minutes, and insert [latency](#latency) grows as tables and their indexes get larger. Each index adds work to every insert, so a table with many indexes can be slow to write even when its queries are fast.

| Technique                             | How it helps                                | Trade-off                       |
| ------------------------------------- | ------------------------------------------- | ------------------------------- |
| Batching                              | Groups many rows into one write             | Small delay, larger loss window |
| Fewer indexes                         | Reduces work per insert                     | Some queries get slower         |
| [Message queue](#message-queue) | Absorbs bursts ahead of the database        | Data is visible later           |
| Write-optimised storage               | Appends to disk and merges later (LSM tree) | Reads can cost more             |
| Partitioning                          | Splits writes across several machines       | Harder queries, rebalancing     |

Partitioning needs a well-chosen key. If all writes for the current hour land on one partition, as happens when data is range-partitioned by timestamp, one machine takes the entire load. This hot partition has the same effect as a [hot key](#hot-key). Mixing in a device or user ID spreads the writes.

Check whether every write needs the same guarantee. Losing one click event may be acceptable, while losing a payment is not. Data you can afford to lose can use batching or a [write-behind cache](#write-behind-cache).

_Usage:_

"Replica lag hits ten minutes whenever the sensors report."

"It's a write-heavy workload. More replicas won't help, so batch the inserts and drop the indexes nobody uses."

## Section 2 — Scaling

### Vertical scaling

Vertical scaling means handling more load by giving one machine more resources: more CPU cores, more memory, faster disks or a faster network card. It is also called scaling up. On a cloud provider it can be as small a change as selecting a larger instance type.

The main advantage is that the software usually does not change. There is no data to split, no coordination between machines, and no new failure modes from network calls. For databases in particular, moving to a larger machine is often the sensible first step, because the alternative of splitting data across servers adds lasting complexity.

Vertical scaling has limits that eventually force a different approach:

| Limit            | What happens                                                                               |
| ---------------- | ------------------------------------------------------------------------------------------ |
| Hardware ceiling | There is a largest machine you can buy or rent                                             |
| Cost curve       | The largest machines can cost more per unit of capacity                                    |
| Resize downtime  | Moving to a larger machine often requires a restart                                        |
| Availability     | One large machine is still a [single point of failure](#single-point-of-failure) |
| Software limits  | Locks or single-threaded code may not use the added cores                                  |

The last limit is a common surprise. A team moves a service to a 64-core machine and sees no improvement, because one thread is pinned at 100% while the rest sit idle. Before resizing, confirm which resource is the [bottleneck](#bottleneck). More cores do nothing for a service limited by a lock, and more memory does nothing for one limited by disk writes.

Most production systems combine both directions: [stateless](#stateless-service) application servers grow through [horizontal scaling](#horizontal-scaling), while the database grows vertically for as long as that remains practical.

_Usage:_

"Do we need to shard the orders database this quarter?"

"Not yet. Vertical scaling buys us a year or so. Move it to the next instance size and revisit when we're near the top of the range."

### Horizontal scaling

Horizontal scaling means handling more load by adding more machines and spreading the work across them. It is also called scaling out. Instead of one larger server, you run ten ordinary ones behind a [load balancer](#load-balancer), and add an eleventh when traffic grows.

The comparison with [vertical scaling](#vertical-scaling) is mostly a trade of simplicity for headroom:

| Concern         | Vertical                  | Horizontal                              |
| --------------- | ------------------------- | --------------------------------------- |
| Capacity limit  | Largest available machine | Much higher, depends on the design      |
| Code changes    | Usually none              | Must tolerate several running copies    |
| Machine failure | Full outage               | Lose a fraction of capacity             |
| Deploys         | Restart causes downtime   | Replace instances one at a time         |
| Operations      | One machine to manage     | Networking, discovery, partial failures |

Horizontal scaling works well for a [stateless service](#stateless-service), because any instance can answer any request. State is what makes it hard. If each server keeps sessions in its own memory, a user's next request can land on a server that has never seen them. Databases are harder still, since spreading writes means sharding (splitting data across machines by a key), which complicates queries and transactions.

A common disappointment is adding servers and getting no improvement. If every instance waits on one database, that database is the [bottleneck](#bottleneck), and extra instances add connections to it. Scaling only helps the part that is actually constrained.

Costs also appear between machines. A process that kept a counter in memory now has several disagreeing counters, and a scheduled job runs once per instance unless something coordinates it. Check for assumptions like these before scaling a service that has only ever run as one copy.

_Usage:_

"Traffic doubles for the sale, can we add API pods?"

"Yes, the API scales horizontally. The checkout worker doesn't, because it keeps a lock in memory, so fix that before relying on horizontal scaling there."

### Stateless service

A stateless service keeps no client-specific data in its own memory or disk between requests. Everything needed to answer arrives with the request or is fetched from a shared store such as a database. Because of this, any running instance can handle any request.

The state still exists. It has moved out of the application process into something built to hold it:

| State             | Stateful version         | Stateless version                                             |
| ----------------- | ------------------------ | ------------------------------------------------------------- |
| Login session     | Object in server memory  | Shared session store or signed token                          |
| Shopping cart     | Variable on one instance | Database row keyed by user                                    |
| Uploaded file     | Local disk               | Object storage                                                |
| Progress of a job | In-process loop          | Record in a [message queue](#message-queue) or database |

The symptom of hidden state is behaviour that depends on which server you reach. Users get logged out at random, a file uploaded a moment ago returns a 404 on refresh, or everything breaks right after a deploy because the restart wiped memory. These faults often appear only once a second instance is added, since a single server hides the problem.

Statelessness is what makes [horizontal scaling](#horizontal-scaling) straightforward. A [load balancer](#load-balancer) can send traffic anywhere, instances can be replaced during a deploy, and a crashed instance loses only the requests it was processing at that moment. Without it, teams often resort to a [sticky session](#sticky-session), which keeps the problem and adds new ones.

An in-process [cache](#cache) does not break the model, provided losing it affects only speed. The test is whether an instance can be killed at any moment with no lasting effect on users.

The cost is that each request may need a trip to the shared store, which adds [latency](#latency) and makes that store a dependency of every request.

_Usage:_

"Why do people lose their carts every time we deploy?"

"The cart lives in process memory, so this isn't a stateless service. Move it to Redis and restarts stop mattering."

### Sticky session

A sticky session is a [load balancer](#load-balancer) setting that keeps sending the same client to the same backend server. It is also called session affinity. It is typically used when a server holds something about that client in local memory, such as a login session, and other servers could not answer without it.

There are two common ways to implement it:

| Method    | How the client is identified                    | Weakness                                                             |
| --------- | ----------------------------------------------- | -------------------------------------------------------------------- |
| Cookie    | Balancer sets or reads a cookie naming a server | Requires [layer 7 load balancing](#layer-7-load-balancing) |
| Source IP | Balancer hashes the client's IP address         | Many users behind one office or mobile network share an IP           |

Stickiness makes a stateful application run on several servers with little code change, which is why it is often adopted. The costs show up later. Load becomes uneven, because a few heavy users stay pinned to one instance while others idle. Adding servers does not relieve the busy ones right away, since existing clients stay where they are.

Failure is the most visible problem. When a server crashes or is replaced during a deploy, every user pinned to it loses their session, and people find themselves logged out or with an empty form. Rolling deploys become slower too, because you must either wait for sessions to drain from an instance or accept that those users lose them.

The usual fix is to make the application a [stateless service](#stateless-service) by moving session data to a shared store, or into a signed token held by the client. Stickiness can then be removed.

It still has legitimate uses as an optimisation. Routing a user back to a server with a warm local [cache](#cache) improves speed, and the system still works correctly if the routing changes.

_Usage:_

"Why are a third of users logged out after each deploy?"

"We use sticky sessions and keep the session in memory, so restarting a pod logs out everyone pinned to it."

### Consistent hashing

Consistent hashing is a way of assigning keys to servers so that adding or removing a server moves only a small share of the keys. It is used in distributed [caches](#cache) and partitioned data stores, where each key must live on a predictable node.

The simpler approach is `hash(key) % N`, where N is the number of servers. It spreads keys evenly, but changing N changes the result for nearly every key. Going from 4 servers to 5 sends about 80% of keys to a different server. In a cache, that shows up as a sudden collapse in [cache hit ratio](#cache-hit-ratio) right after someone adds a node, with the database taking the resulting load.

Consistent hashing avoids this by placing both servers and keys on the same circular range of hash values:

| Step | What happens                                         |
| ---- | ---------------------------------------------------- |
| 1    | Hash each server's name to a position on the ring    |
| 2    | Hash the key to a position on the same ring          |
| 3    | Walk clockwise from the key until you reach a server |
| 4    | That server owns the key                             |

When a server joins, it takes over only the keys between itself and the previous server on the ring. When one leaves, its keys pass to the next server. Going from 4 to 5 servers moves about 20% of keys.

With a few servers placed at random positions, some arcs of the ring are much longer than others, so load is uneven. The standard fix is virtual nodes: each physical server appears at many positions on the ring. A larger machine can be given more positions.

Consistent hashing balances the number of keys and does nothing about uneven popularity. A single [hot key](#hot-key) still lands on one server and can overload it.

_Usage:_

"Every time we add a Redis node, the database gets hammered for ten minutes."

"The client is doing modulo hashing, so nearly every key remaps. Switch it to consistent hashing."

### Fan-out

Fan-out is when one incoming request or message triggers many downstream calls or copies. A search query sent to 50 index partitions has a fan-out of 50. A post delivered to 10,000 followers has a fan-out of 10,000. The term appears in two main settings.

| Setting         | What multiplies                     | Typical example                      |
| --------------- | ----------------------------------- | ------------------------------------ |
| Request fan-out | Synchronous calls to other services | A page calling 12 services to render |
| Data fan-out    | Copies of a message or record       | One event delivered to many inboxes  |

Request fan-out has a counter-intuitive effect on [latency](#latency). If a request must wait for all its downstream calls, it is as slow as the slowest one, so rare slowness becomes common. Suppose each backend exceeds one second on 1% of calls. A request that calls 100 of them in parallel has a 1 − 0.99¹⁰⁰ chance, about 63%, of hitting at least one slow response. This is why a page can feel slow when every service's dashboard looks healthy.

Several techniques reduce the effect. Call backends in parallel, give each call a [timeout](#timeout), and return partial results when an optional backend is late, which is a form of [graceful degradation](#graceful-degradation). A hedged request sends a duplicate to another replica if the first has not answered after a short delay, and uses whichever reply comes first.

Fan-out also multiplies load. One user action becoming 100 internal calls means a modest traffic increase can overload deep dependencies, and any [retries](#retry-with-backoff) multiply it again.

Data fan-out raises a timing question: do the copying when content is written, or assemble it when someone reads? Those two designs are [fan-out on write](#fan-out-on-write) and [fan-out on read](#fan-out-on-read).

_Usage:_

"Every service is under 100 ms at p99, but the dashboard page takes three seconds."

"It has a fan-out of about 40 calls, partly in sequence. The slow ones add up."

### Fan-out on write

[Fan-out](#fan-out) on write means copying new content to every recipient's feed at the moment it is created. Each user has a precomputed timeline, usually a list of post IDs in a fast store, and reading it is one lookup. It is one of two ways to build a feed, the other being [fan-out on read](#fan-out-on-read). Both are forms of data fan-out.

The write path usually runs in the background:

| Step | Who     | What happens                                                                              |
| ---- | ------- | ----------------------------------------------------------------------------------------- |
| 1    | API     | Saves the post                                                                            |
| 2    | API     | Puts a job on a [message queue](#message-queue), then returns success to the author |
| 3    | Worker  | Looks up the author's followers                                                           |
| 4    | Workers | Insert the post ID into each follower's stored timeline                                   |
| 5    | Reader  | Fetches their timeline with a single read                                                 |

This suits a [read-heavy workload](#read-heavy-workload). The expensive work happens once per post, and the many reads that follow are inexpensive and predictable.

The weakness is accounts with very large audiences. One post from an account with 20 million followers creates 20 million timeline writes. The queue backs up, and followers see that post minutes late, while posts from smaller accounts queued behind it are delayed too.

Storage grows as well, since the same post ID is stored once per follower. Systems limit this by keeping only the most recent few hundred entries per timeline and by skipping inactive users. Edits and deletes are also harder, because removing a post means finding every copy, or filtering deleted posts at read time.

Large systems commonly use a hybrid. Ordinary accounts fan out on write, while posts from very large accounts are fetched and merged when a follower reads.

_Usage:_

"Why did the feed queue spike to two million jobs at noon?"

"A celebrity account posted. With fan-out on write, that's one job per follower, so we should exclude big accounts and merge them at read time."

### Fan-out on read

[Fan-out](#fan-out) on read means assembling a user's feed at the moment they request it. Nothing is prepared in advance. When a user opens the app, the service looks up who they follow, fetches recent posts from each account, merges them by time or rank, and returns the result. It is also called the pull model, in contrast to [fan-out on write](#fan-out-on-write), the push model.

| Concern                      | Fan-out on write           | Fan-out on read          |
| ---------------------------- | -------------------------- | ------------------------ |
| Cost of posting              | High, grows with followers | Low, one insert          |
| Cost of reading              | Low, one lookup            | High, grows with follows |
| Read [latency](#latency) | Low and steady             | Higher and more variable |
| Storage                      | One copy per follower      | One copy of each post    |
| Freshness                    | Delayed by the queue       | Current at read time     |
| Inactive users               | Work wasted on them        | Cost nothing             |
| Very large accounts          | Expensive to post          | No extra cost            |

Fan-out on read suits data that is written often and read rarely, and products where most users are inactive on a given day. Deleting a post also takes effect immediately, since no copies exist.

The symptom of this design under load is a slow home screen for the most connected users. Someone who follows 5,000 accounts triggers 5,000 lookups, or a large query across many partitions, every time they refresh. Because the request waits on many sources, it is exposed to the tail-latency effect of fan-out, where one slow partition delays the whole response.

Common mitigations are to [cache](#cache) the assembled feed for a short time, cap how many sources are read per request, and fetch only the first page. Many large products combine the two approaches: precompute timelines for ordinary accounts, and pull posts from very large accounts during the read.

_Usage:_

"Should we precompute notifications for every workspace member?"

"Most members never open that tab. Use fan-out on read and cache the result for a minute."

### Hot key

A hot key is a single key that receives far more traffic than the others, overloading the node that stores it. In a partitioned [cache](#cache) or database, each key lives on one node or a small set of nodes, so a viral product page or a very popular account sends its traffic to one place regardless of how large the cluster is.

The recognisable symptom is one node at 100% CPU while the others sit at 15%. Cluster-wide averages look healthy, more nodes do not help, and requests touching unrelated keys on the same node also slow down. [Consistent hashing](#consistent-hashing) does not prevent this, because it balances how many keys each node owns and takes no account of how popular each key is.

Hot keys are found by sampling per-key traffic. Redis, for example, offers `redis-cli --hotkeys` when an LFU [eviction policy](#eviction-policy) is configured.

The fix depends on whether the traffic is reads or writes:

| Technique              | Works for | How it helps                                              |
| ---------------------- | --------- | --------------------------------------------------------- |
| Local in-process cache | Reads     | Each app server keeps the value for a second or two       |
| Replicate the key      | Reads     | Several nodes hold copies and share the requests          |
| Key splitting          | Reads     | Store copies as `item:1`, `item:2` and pick one at random |
| Counter splitting      | Writes    | Increment one of N sub-counters and sum them on read      |
| Write batching         | Writes    | Combine many updates in the app before writing            |
| Dedicated node         | Both      | Move the key to hardware of its own                       |

A one-second local cache limits each app server to about one fetch per second for that key.

A hot key that expires is a further risk. Every request misses simultaneously and goes to the database, producing a [cache stampede](#cache-stampede).

_Usage:_

"Redis shard 7 is maxed out and the other eleven are idle."

"That's a hot key, probably the homepage banner. Add a short local cache in the app."

## Section 3 — Traffic

### Load balancer

A load balancer is a component that receives incoming requests and spreads them across a group of backend servers. Clients see one address, while several instances do the work. This is what makes [horizontal scaling](#horizontal-scaling) possible for a web service, and it lets you replace instances during a deploy without clients noticing.

The balancer chooses a backend using an algorithm:

| Algorithm            | How it picks                               | Suits                                |
| -------------------- | ------------------------------------------ | ------------------------------------ |
| Round robin          | Each server in turn                        | Similar servers, similar requests    |
| Weighted round robin | In turn, in proportion to assigned weights | Servers of different sizes           |
| Least connections    | Server with the fewest active connections  | Requests of varying duration         |
| Hash                 | Same client or key goes to the same server | Affinity, local [caches](#cache) |
| Power of two choices | Samples two at random, takes the less busy | Large fleets, several balancers      |

Balancers operate at different levels. [Layer 4 load balancing](#layer-4-load-balancing) routes network connections without reading their content, while [layer 7 load balancing](#layer-7-load-balancing) reads each HTTP request and can route on its path or headers.

Health checks decide which instances receive traffic, and they cause two familiar problems. A shallow check that only confirms the process is running can keep returning 200 while real requests fail, so users keep hitting a broken instance. A deep check that also tests the shared database has the opposite risk: one database blip marks every instance unhealthy at once, and the balancer may have nowhere to send traffic. A common compromise is to check what the instance itself needs to function and handle shared dependency failures separately.

The balancer sits on every request, so it can become a [single point of failure](#single-point-of-failure). Run it as a redundant pair or use a managed service.

_Usage:_

"Why are 25% of requests failing when only one of four pods is broken?"

"Its health endpoint still returns 200, so the load balancer keeps sending it traffic."

### Layer 4 load balancing

Layer 4 load balancing distributes network connections using only IP addresses, ports and the protocol, without reading the data inside them. The name comes from the transport layer, layer 4 of the OSI networking model, which is where TCP and UDP sit. The [load balancer](#load-balancer) picks a backend when a connection opens, and every packet in that connection then goes to the same backend.

Not parsing the content has these consequences:

| Property                           | Consequence                                    |
| ---------------------------------- | ---------------------------------------------- |
| No parsing of application data     | Low overhead per connection                    |
| Works with any TCP or UDP protocol | Suits databases, message brokers, game traffic |
| Encrypted traffic passes through   | TLS can stay end-to-end, up to the backend     |
| Sees no URL, header or cookie      | Cannot route by path or user                   |
| Decides once per connection        | Balances connections, not requests             |

The last property causes a well-known problem with long-lived connections. HTTP/2 and gRPC (a remote procedure call framework) send many requests over one connection. A layer 4 balancer sees one connection and pins it to one backend, so a busy client can load one server heavily while others sit idle.

The same effect shows up during scaling. You add three new pods, and they receive almost no traffic, because existing clients keep their open connections to the old pods.

There are three usual fixes. Set a maximum connection age so clients reconnect and spread out over time. Use client-side balancing, where the client holds connections to several backends. Or move to [layer 7 load balancing](#layer-7-load-balancing), which balances each request individually.

Layer 4 remains a good fit for non-HTTP protocols and very high connection counts. AWS Network Load Balancer and HAProxy in TCP mode are examples.

_Usage:_

"We scaled gRPC to ten pods, but three of them are doing all the work."

"The service is behind layer 4 load balancing, so each connection is stuck to one pod. Set a max connection age."

### Layer 7 load balancing

Layer 7 load balancing distributes individual requests by reading application data such as the URL path, headers and cookies. The name refers to the application layer of the OSI networking model, where HTTP sits. The [load balancer](#load-balancer) ends the client's connection, reads the request, chooses a backend, and sends the request over its own connection. In this role it is a type of [reverse proxy](#reverse-proxy).

Reading the request allows routing rules that lower layers cannot express. `/api/*` can go to one service and `/images/*` to another, a header can send 5% of users to a canary release (a small trial deployment), and a cookie can support a [sticky session](#sticky-session).

| Concern                       | [Layer 4](#layer-4-load-balancing) | Layer 7                          |
| ----------------------------- | -------------------------------------------- | -------------------------------- |
| Routes on                     | IP address and port                          | Path, host, headers, cookies     |
| Balances                      | Connections                                  | Individual requests              |
| Protocols                     | Any TCP or UDP                               | Those it can parse, such as HTTP |
| TLS                           | Can pass through                             | Usually terminated here          |
| Overhead                      | Lower                                        | Higher                           |
| Request retries, header edits | Not available                                | Supported                        |

The cost is more work per request. The balancer must decrypt traffic, parse it and hold state for two connections, and it needs access to your TLS private keys. It also adds its own limits, which produce confusing failures. A file upload that works locally returns 413 in production because of a body-size limit on the balancer. A long report returns 504 because the balancer's [timeout](#timeout) is shorter than the application's. When an error never appears in your application logs, check the balancer's logs.

Automatic retries at this layer need care. Retrying a failed POST can apply it twice unless the operation is idempotent, meaning repeating it has the same effect as doing it once.

NGINX, Envoy and AWS Application Load Balancer are common examples.

_Usage:_

"Can we send only `/search` traffic to the new service?"

"Yes, the ingress does layer 7 load balancing, so it's one path rule."

### Reverse proxy

A reverse proxy is a server that sits in front of one or more backend servers, receives client requests on their behalf, and forwards them. The client talks only to the proxy and usually cannot tell what is behind it. A forward proxy is the mirror image: it acts for clients, for example a company proxy that controls outbound web access.

Putting one component at the entrance lets it handle work that every backend would otherwise repeat:

| Job                   | What the proxy does                                                 |
| --------------------- | ------------------------------------------------------------------- |
| TLS termination       | Handles HTTPS so backends can receive decrypted traffic             |
| Routing               | Sends paths or hostnames to different services                      |
| Balancing             | Spreads requests, acting as a [load balancer](#load-balancer) |
| [Caching](#cache) | Stores responses and serves repeats                                 |
| Compression           | Shrinks responses before sending them                               |
| Buffering             | Absorbs slow clients so backend workers are freed quickly           |
| Static files          | Serves assets without involving the application                     |

The proxy also introduces a class of errors that can confuse debugging. A 502 Bad Gateway means the proxy got an invalid response or no usable connection from the backend, often because the application crashed or is not listening. A 504 Gateway [Timeout](#timeout) means the backend did not answer within the proxy's time limit. In both cases the error page comes from the proxy, so the application's logs may show nothing, or show a request that completed successfully after the proxy gave up.

Another common surprise is that every request appears to come from the proxy's IP address. Proxies pass the original address in a header such as `X-Forwarded-For`. Clients can forge that header, so trust only the values added by your own proxies, particularly for [rate limiting](#rate-limiting) and audit logs.

NGINX, HAProxy, Caddy and Envoy are examples. An [API gateway](#api-gateway) is a reverse proxy with API-specific features added.

_Usage:_

"Users get a 502, but there's nothing in the app logs."

"The reverse proxy generates that. The app probably crashed and restarted, so check for container restarts."

### API gateway

An API gateway is a single entry point for API clients that routes each request to the right backend service and handles concerns shared by all of them. It is a specialised [reverse proxy](#reverse-proxy). It is common in [microservices](#microservices), where clients would otherwise need to know the address and rules of many separate services.

Without one, each service reimplements login checks and limits, usually with small differences. One team's endpoint ends up with weaker validation, or a security fix has to ship in twelve places. A gateway puts those rules in one layer.

| Concern                               | What the gateway does                          |
| ------------------------------------- | ---------------------------------------------- |
| Routing                               | Maps public paths to internal services         |
| Authentication                        | Validates tokens or API keys before forwarding |
| [Rate limiting](#rate-limiting) | Enforces quotas per client or plan             |
| Transformation                        | Rewrites headers, paths or payload formats     |
| Versioning                            | Sends `/v1` and `/v2` to different backends    |
| Observability                         | Records logs and metrics for every call        |

The neighbouring components overlap. A [load balancer](#load-balancer) spreads traffic across copies of one service. A reverse proxy forwards traffic and handles generic HTTP work. An API gateway adds knowledge of API consumers, such as who is calling and what plan they are on.

The main risk is logic creep. Teams add business rules, data joins and special cases to the gateway because it is convenient, and it becomes a shared component that every release depends on and one team must approve. Keep rules about the product inside services, and keep the gateway to cross-cutting concerns. If different clients need different response shapes, a [backend for frontend](#backend-for-frontend) is a better location.

The gateway also adds a network hop and sits on every request, so a bad configuration change can take down all APIs at once. Kong and Amazon API Gateway are examples.

_Usage:_

"Should the discount calculation go in a gateway plugin? It's the one place every request passes through."

"No, that's business logic. Keep the API gateway to auth, routing and quotas."

### CDN

A CDN, or content delivery network, is a set of servers in many geographic locations that [cache](#cache) content close to users. Those servers are called the edge. Your own server is called the origin. A user in Tokyo can get an image from a nearby edge location instead of an origin in Virginia, which lowers [latency](#latency) and origin load.

A request moves through these steps:

| Step | Who     | What happens                                                      |
| ---- | ------- | ----------------------------------------------------------------- |
| 1    | Network | DNS or network routing sends the user to a nearby edge location   |
| 2    | Edge    | Looks up the response by its cache key, usually the URL           |
| 3    | Edge    | On a hit, returns the stored copy                                 |
| 4    | Origin  | On a miss, receives the request and returns the response          |
| 5    | Edge    | Stores it according to its `Cache-Control` headers, then responds |

CDNs suit images, video, scripts and stylesheets, and can also cache API responses that are the same for many users.

The familiar failure is stale content. You deploy a fix and some users still get the old stylesheet, because edge locations hold it until its [TTL](#ttl) expires. Purging helps but takes time to reach every location and does not clear browser caches. A more dependable method is to put a content hash in the filename, such as `app.3f9c2.css`. Each release then has a new URL, and the file can be cached for a long time.

A more serious failure is caching personalised content. If an account page is cached under a URL shared by all users, one person can receive another's data. Mark such responses `Cache-Control: private` or `no-store`, and make sure the cache key includes anything the response varies on.

Cloudflare, Amazon CloudFront and Fastly are examples.

_Usage:_

"I shipped the CSS fix an hour ago, but customers still see the broken layout."

"The CDN is still serving the old file. We need hashed filenames so each release gets a new URL."

### Service discovery

Service discovery is the mechanism services use to find the current network addresses of the services they call. Hard-coded addresses stop working once instances are created and destroyed by autoscaling and deploys, because each new instance can have a different IP. A discovery system maintains a live list of healthy instances under a stable service name, such as `payments`.

The list lives in a registry. Instances either register themselves on startup or are registered by the platform, and health checks remove those that stop responding. Callers then reach an instance in one of three ways:

| Approach    | How a caller finds an instance                                            | Trade-off                                                         |
| ----------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| DNS-based   | Resolves the service name to one or more IPs                              | Simple, but resolvers and clients [cache](#cache) old records |
| Client-side | Fetches the list from the registry and picks an instance itself           | No extra hop, but logic in every client                           |
| Server-side | Calls a stable address, and a [load balancer](#load-balancer) picks | Simple clients, one more hop                                      |

Kubernetes Services and Consul are examples.

The usual symptom of a discovery problem is a burst of connection errors during every deploy. An old pod has shut down, but callers still hold its address for a few seconds, or longer if a client caches DNS results for too long and ignores the record's [TTL](#ttl). To reduce this, have an instance leave the registry first, keep serving during a short draining period, and only then exit. Callers should also retry a failed connection on a different instance.

The registry becomes a dependency of nearly everything, so an outage there can spread widely. Run it with redundancy, and have clients keep using their last known list of addresses if the registry cannot be reached.

_Usage:_

"Every rollout gives us thirty seconds of connection refused from the orders service."

"Pods exit before service discovery removes them. Add a pre-stop delay so they drain first."

### Rate limiting

Rate limiting caps how many requests a client can make in a period of time and rejects the excess. A rule might allow 100 requests per minute per API key. It protects a service from abusive clients, buggy scripts stuck in a loop, and one tenant using capacity that others need. Rejected HTTP requests normally receive a 429 Too Many Requests status, often with a `Retry-After` header saying when to try again.

Several algorithms can enforce the limit:

| Algorithm                                                 | How it counts                      | Trade-off                         |
| --------------------------------------------------------- | ---------------------------------- | --------------------------------- |
| Fixed window                                              | Counter per calendar minute        | Allows a double burst at the edge |
| Sliding log                                               | Stores a timestamp per request     | Exact, but uses more memory       |
| [Sliding window counter](#sliding-window-counter) | Weights two adjacent counters      | Approximate, low memory           |
| [Token bucket](#token-bucket)                       | Spends tokens that refill steadily | Permits controlled bursts         |
| Leaky bucket                                              | Drains a queue at a fixed rate     | Smooth output, adds waiting       |

Identifying the client needs thought. Limiting by IP address punishes everyone behind one office network or mobile carrier, so a whole company can receive 429s because of one colleague's script. Prefer an API key or user ID, and keep IP limits for unauthenticated endpoints such as login.

Running several instances introduces a common bug. If each instance counts on its own, a limit of 100 per minute across three instances really allows 300. A shared store such as Redis fixes this, provided the check and increment happen atomically. You also need to decide what happens when that store is down: allow traffic and lose protection, or reject it and cause an outage.

Rate limiting differs from [load shedding](#load-shedding). Rate limiting enforces a per-client quota even when the system is healthy, while load shedding drops work because the system as a whole is overloaded.

_Usage:_

"A partner's sync job is eating half of our API capacity."

"Add rate limiting per API key and return `Retry-After`, so their client backs off."

### Token bucket

A token bucket is a [rate limiting](#rate-limiting) algorithm in which each request spends a token from a bucket that refills at a steady rate. It has two settings: capacity, the most tokens the bucket can hold, and refill rate, how many tokens are added per second. Capacity controls the size of a burst, and refill rate controls the sustained average.

Take a bucket with a capacity of 20 and a refill rate of 5 tokens per second:

| Time | Event                | Tokens after | Result                |
| ---- | -------------------- | ------------ | --------------------- |
| 0 s  | Client has been idle | 20           | Bucket is full        |
| 0 s  | 20 requests arrive   | 0            | All allowed           |
| 0 s  | 1 more request       | 0            | Rejected              |
| 1 s  | 5 tokens added       | 5            | Refilled              |
| 1 s  | 8 requests arrive    | 0            | 5 allowed, 3 rejected |

This burst behaviour suits real clients. A page load may fire ten API calls at once and then go quiet. A strict per-second limit would reject part of that page even though the user's average rate is low. A token bucket lets it through and still holds the long-run rate.

Implementations do not need a timer adding tokens. Store only the token count and the last update time. On each request, compute `tokens = min(capacity, tokens + elapsed × rate)`, then subtract one if at least one token is available. Across several servers, this read-and-update has to be atomic, for example through a Lua script in Redis. Otherwise two concurrent requests can spend the same token.

Choosing the capacity is the main tuning decision. If it is too small, normal bursts receive 429 errors even though the client is under its average limit. If it is too large, many clients can burst together and still overload the backend.

_Usage:_

"Customers get 429s on dashboard load, but they're well under 300 requests a minute."

"The page sends 40 calls at once. Use a token bucket with a burst capacity of 50."

### Sliding window counter

A sliding window counter is a [rate limiting](#rate-limiting) algorithm that estimates recent traffic by combining the counts of the current and previous fixed time windows. It uses about as little memory as a plain counter and reduces that counter's main flaw.

The flaw is the boundary burst. A fixed window limit of 100 requests per minute resets at the top of each minute. A client can send 100 requests at 12:00:59 and another 100 at 12:01:00. Each window is within its limit, yet the backend receives 200 requests in about one second, double the intended rate.

The sliding window counter keeps two numbers per client and weights the older one by how much of it still overlaps the last 60 seconds:

| Step | Calculation                             | Example             |
| ---- | --------------------------------------- | ------------------- |
| 1    | Count in the previous window            | 80                  |
| 2    | Count so far in the current window      | 30                  |
| 3    | Position in the current window          | 15 s of 60 s        |
| 4    | Share of previous window still in range | 1 − 15/60 = 75%     |
| 5    | Estimate = previous × share + current   | 80 × 0.75 + 30 = 90 |
| 6    | Compare with the limit of 100           | Allowed             |

The estimate assumes requests in the previous window were spread evenly. When they were bunched at one end, the estimate can be far off: a client can fit nearly twice the limit into a rolling window, or is rejected after a minute of sending nothing. On evenly spread traffic the error is small. For quotas and abuse protection that error is normally acceptable. A sliding log is exact, at a higher memory cost.

Compared with a [token bucket](#token-bucket), this algorithm expresses a limit directly as "N per window" and does not grant an explicit burst allowance.

_Usage:_

"Clients are doubling their quota by bursting right around the minute mark."

"That's the fixed window edge. Switch to a sliding window counter, which only needs one extra integer per client."

## Section 4 — Caching

### Cache

A cache is a fast store that holds copies of data so that repeated requests can avoid a slower source, such as a database, a disk or a remote service. When the requested item is present, that is a hit. When it is absent, that is a miss, and the data must be fetched from the source. Caching works because access is usually uneven, with a small share of the data receiving a large share of the requests.

Caches exist at several levels of a typical system:

| Level                                 | Where it lives            | Typical contents             |
| ------------------------------------- | ------------------------- | ---------------------------- |
| Browser                               | User's device             | Images, scripts, stylesheets |
| [CDN](#cdn)                       | Edge locations near users | Static files, public pages   |
| [Reverse proxy](#reverse-proxy) | In front of app servers   | Whole HTTP responses         |
| In-process                            | Application memory        | Config, small lookup tables  |
| Distributed                           | Redis or Memcached        | Sessions, query results      |
| Database                              | Database memory           | Recently used data pages     |

Every cache design answers three questions. How does data get in, for example [cache-aside](#cache-aside) or a [write-through cache](#write-through-cache)? How long does it stay, as set by a [TTL](#ttl) and an [eviction policy](#eviction-policy)? How is it corrected when the source changes, which is [cache invalidation](#cache-invalidation)?

Two failures are common. The first is stale data: a user changes their address, and the old one keeps appearing because a cached copy outlived the change. The second is hidden dependence. A database is sized for the traffic left over after the cache, so when the cache restarts empty, the database receives the full load in a [cache stampede](#cache-stampede) and can go down. A cache that was added to improve speed has become necessary for availability.

Treat a cache as a copy that can vanish at any time, and track its [cache hit ratio](#cache-hit-ratio). Test what happens when it is empty.

_Usage:_

"Can we restart the Redis cluster during the day?"

"Not safely. The cache absorbs about 95% of reads, and the database can't take that load cold."

### Cache hit ratio

[Cache](#cache) hit ratio is the share of lookups that a cache answers without going to the underlying source. It is calculated as hits divided by hits plus misses. A cache that answers 950 of 1,000 lookups has a hit ratio of 95%.

The number that matters for capacity is the miss ratio, because misses are what reach the database. Small changes near the top of the scale have large effects:

| Hit ratio | Requests reaching the database, per 10,000 |
| --------- | ------------------------------------------ |
| 90%       | 1,000                                      |
| 95%       | 500                                        |
| 99%       | 100                                        |
| 99.9%     | 10                                         |

A drop from 99% to 98% looks minor on a dashboard, yet it doubles database load. This is why a small change, such as a new field in the cache key or a shorter [TTL](#ttl), can produce a database incident with no increase in traffic.

Hit ratio also drives average [latency](#latency). If a hit takes 1 ms and a miss costs 1 ms for the check plus 50 ms for the database, a 90% hit ratio averages 6 ms and a 99% ratio averages 1.5 ms.

Several things commonly lower the ratio: keys that include a timestamp or request ID, so that nothing repeats; a cache too small for its working set, the data in regular use; and restarts or deploys that leave the cache empty. Adding nodes with naive hashing has the same effect, which [consistent hashing](#consistent-hashing) avoids.

Read the number with care. A single overall ratio can hide one important endpoint at 40% behind many trivial lookups at 99.9%, so measure it per key type. A high ratio also says nothing about whether the data served is correct.

_Usage:_

"Database CPU doubled after the release, but traffic is flat."

"Cache hit ratio fell from 99 to 98. The new locale field in the key split every entry."

### Cache-aside

[Cache](#cache)-aside is a caching pattern in which the application manages the cache itself, loading data into it only when that data is requested. It is also called lazy loading. The cache sits beside the database and has no connection to it.

| Step | Path  | What the application does                     |
| ---- | ----- | --------------------------------------------- |
| 1    | Read  | Looks up the key in the cache                 |
| 2    | Read  | On a hit, returns the value                   |
| 3    | Read  | On a miss, queries the database               |
| 4    | Read  | Stores the result in the cache and returns it |
| 5    | Write | Updates the database                          |
| 6    | Write | Deletes the key from the cache                |

The pattern has useful properties. Only requested data takes up cache memory, and if the cache goes down, reads can still reach the database, at the price of more load on it.

Writes delete the key instead of updating it because concurrent updates can land in the wrong order and leave an old value in place. Deleting forces the next reader to load the current value.

A race remains. Reader A misses and fetches the old row. Writer B updates the database and deletes the key. Reader A then stores the old value it fetched earlier. The cache now holds stale data with nothing scheduled to remove it, which is how a record can stay wrong for hours and then correct itself. Always set a [TTL](#ttl) as a backstop, so that any such error has a limited lifetime.

Each miss costs three trips: cache, database, cache. After a deploy or restart everything misses at once, and a popular key that expires can cause a [cache stampede](#cache-stampede). For data that must be current right after a write, compare a [write-through cache](#write-through-cache).

_Usage:_

"One customer's plan shows the old tier, but the database row is right."

"That's the cache-aside race. The TTL is a week, so shorten it and delete that key by hand."

### Write-through cache

A write-through [cache](#cache) is one where each write goes to both the cache and the backing database before the caller receives confirmation. Because the two are updated as part of one operation, a read that follows a successful write finds the new value in the cache.

| Step | Who      | What happens                               |
| ---- | -------- | ------------------------------------------ |
| 1    | App      | Sends the write to the cache layer         |
| 2    | Cache    | Stores the new value                       |
| 3    | Cache    | Writes the value to the database and waits |
| 4    | Database | Confirms the write                         |
| 5    | Cache    | Confirms success to the app                |

Some products provide this behaviour directly. With a plain store such as Redis, the application usually performs both writes itself, which gives weaker guarantees, because a crash between the two leaves them different.

This pattern addresses a common complaint with [cache-aside](#cache-aside): a user saves a change and the next read is a miss or returns something old. Recently written data is already cached, so reads of recent data are fast and current.

The costs fall on the write path. Each write now waits for two systems, so write [latency](#latency) is at least that of the database, and it does nothing to reduce database write load. A [write-heavy workload](#write-heavy-workload) gets little benefit. The cache also fills with data that may never be read, such as records created once and left alone, so pair it with a [TTL](#ttl) or an [eviction policy](#eviction-policy).

Failure handling needs a decision. If the database write fails, the cached value must be removed or never exposed, or readers will see data that was never saved. Other systems that write directly to the database also bypass the cache and leave it stale, so route all writes through the same path.

_Usage:_

"Users edit their profile and sometimes see the old name for a few seconds."

"Profile reads vastly outnumber edits, so a write-through cache fits. The extra few milliseconds on save won't matter."

### Write-behind cache

A write-behind [cache](#cache) confirms a write as soon as it holds the value, then saves it to the database later in the background. It is also called a write-back cache. The caller gets a fast response, and the database receives the change some time afterwards.

The three common write strategies differ in when the database is updated:

| Pattern                                     | Database updated     | Write speed               | Risk of data loss |
| ------------------------------------------- | -------------------- | ------------------------- | ----------------- |
| [Cache-aside](#cache-aside)             | By the app, on write | Database speed            | Low               |
| [Write-through](#write-through-cache) | Before confirming    | Database speed plus cache | Low               |
| Write-behind                                | After confirming     | Cache speed               | Higher            |

Delaying the write allows batching and combining. If a view counter is incremented 1,000 times in ten seconds, the cache can send one update with the final value. This reduces database load, which makes the pattern attractive for a [write-heavy workload](#write-heavy-workload) such as counters, analytics or game state.

The cost is durability. A write that was confirmed to the user exists only in the cache until it is flushed. If that node dies first, the write is gone, even though the client was told it succeeded. Replicating the cache or keeping a durable log of pending writes reduces the risk but does not make the pattern equal to a synchronous database commit.

Other problems follow from the delay. Anything reading the database directly, such as a report, sees old data. If the database is down or slow, unflushed writes accumulate in memory. Ordering between related records can also break when they are flushed separately.

Use this pattern for data you could afford to lose or recompute. Do not use it for payments, orders or anything else that a user has been promised is saved.

_Usage:_

"Can we buffer the like counts in Redis and flush every few seconds?"

"Yes, write-behind is fine there. Losing a few likes in a crash is acceptable, unlike losing an order."

### Cache invalidation

[Cache](#cache) invalidation is the removal or updating of cached copies when the source data changes. A cache does not know that a database row has been modified, so something must tell it, or the copy must expire. Until that happens, readers get stale data. A customer changes their shipping address, sees the old one on the checkout page, and contacts support.

| Strategy               | How it works                                                  | Trade-off                          |
| ---------------------- | ------------------------------------------------------------- | ---------------------------------- |
| [TTL](#ttl) expiry | Copy expires after a set time                                 | Simple, but stale until it expires |
| Delete on write        | Writer removes the key after updating                         | Must find every affected key       |
| Versioned keys         | Key includes a version, such as `user:42:v7`                  | Old entries linger until evicted   |
| Change events          | Database changes are streamed to a consumer that deletes keys | Accurate, more infrastructure      |

Finding every affected key is often the difficult part. A product's price may appear in the product entry, a category listing, search results and a cached homepage. Deleting the first and forgetting the rest gives pages that disagree with each other.

Timing is the other difficulty. Between a database commit and the delete, readers can still get the old value, and a slow reader can write an old value back after the delete, as in the race described under [cache-aside](#cache-aside). Several layers add to it, because clearing Redis does nothing for a copy in a [CDN](#cdn) or a browser.

The usual practice combines methods: delete the key on write for prompt updates, and set a TTL so that any missed invalidation corrects itself. Decide how much staleness each kind of data can tolerate. A follower count can lag by minutes, while a permission change should take effect quickly and may be better left uncached.

_Usage:_

"We removed her admin role, but she can still open the settings page."

"Permissions are cached per session and nothing handles invalidation on role change. Delete that key when the role is updated."

### TTL

TTL stands for time to live, a duration after which a stored item is treated as expired. A [cache](#cache) entry with a TTL of 300 seconds is served for five minutes, and after that the next request must fetch a fresh copy. DNS records carry a TTL too, and so do many session stores and message systems. In IP networking the same name refers to a hop limit on a packet, which is a separate concept.

Setting a TTL is a trade between freshness and load:

| TTL     | Staleness   | Load on the source | Typical use                 |
| ------- | ----------- | ------------------ | --------------------------- |
| Seconds | Very little | High               | Stock levels, live scores   |
| Minutes | Small       | Moderate           | Product pages, feeds        |
| Hours   | Noticeable  | Low                | Reference data, config      |
| Days+   | High        | Very low           | Versioned, unchanging files |

A TTL also acts as a safety net for [cache invalidation](#cache-invalidation). If a delete is missed, the wrong value still disappears when it expires. For that reason, entries with no TTL are risky, because a single stale value can stay indefinitely.

Identical TTLs cause a recognisable problem. If a job loads 100,000 keys at once with a one-hour TTL, they all expire together, and the database gets a load spike every hour on the hour. Adding jitter, a small random variation such as 3,600 seconds plus or minus 10%, spreads the expiries out. For a single popular key, expiry can trigger a [cache stampede](#cache-stampede).

DNS TTLs catch people during migrations. If a record has a 24-hour TTL and you change the IP, some clients keep using the old address for up to a day. Lower the TTL at least one full old-TTL period before the change, then raise it again afterwards.

_Usage:_

"We cut over the DNS an hour ago and half the traffic still hits the old [load balancer](#load-balancer)."

"The record had a TTL of a day. We should have lowered it yesterday."

### Eviction policy

An eviction policy is the rule a [cache](#cache) uses to choose which entries to remove when it runs out of memory. Eviction differs from expiry. A [TTL](#ttl) removes data because it is old, while eviction removes data because space is needed, even if that data is still valid.

| Policy      | Removes                     | Works well when                   | Weakness                           |
| ----------- | --------------------------- | --------------------------------- | ---------------------------------- |
| LRU         | Least recently used entry   | Recent items tend to be reused    | One large scan flushes useful data |
| LFU         | Least frequently used entry | Popularity is stable              | Old favourites can linger          |
| FIFO        | Oldest inserted entry       | Items lose value with age         | Ignores how often items are used   |
| Random      | Any entry                   | Access is fairly uniform          | May remove popular items           |
| TTL-based   | Entry closest to expiry     | TTLs reflect value                | Depends on sensible TTLs           |
| No eviction | Nothing, new writes fail    | Data must not be silently dropped | Errors once memory is full         |

LRU's known weakness is scan pollution. A nightly export reads millions of rows once, each read counts as recent, and the entries people actually use are pushed out. The next morning the [cache hit ratio](#cache-hit-ratio) is low and the database is busy until the cache refills. Fixes include having batch jobs bypass the cache, or using a scan-resistant policy such as an LFU variant.

Check the configured policy, because the failure mode depends on it. Redis's `noeviction` policy returns errors on writes once memory is full, which an application may experience as a sudden outage of the cache. If a store holds data with no other copy, such as a job queue, an evicting policy can delete it silently.

A rising eviction count alongside a falling hit ratio usually means the cache is too small for its working set.

_Usage:_

"Hit ratio tanks every morning around 3 a.m. and recovers by nine."

"That's the export job. Our LRU eviction policy treats every row it scans as recent, so make it skip the cache."

### Cache stampede

A [cache](#cache) stampede happens when many concurrent requests miss the same entry and all go to the database to rebuild it. It is also called a thundering herd or dogpile. It usually starts when a popular key expires or the cache restarts empty.

The sequence feeds itself:

| Step | What happens                                                          |
| ---- | --------------------------------------------------------------------- |
| 1    | A [hot key](#hot-key) expires                                   |
| 2    | Hundreds of in-flight requests see a miss before anyone refills it    |
| 3    | Each one runs the same expensive query                                |
| 4    | The database slows under the duplicated work                          |
| 5    | Queries take longer, so the key stays empty and more requests pile in |
| 6    | Requests hit their [timeouts](#timeout), and any retries add load |

The recognisable symptom is a database spike that recurs at regular intervals and matches a key's [TTL](#ttl), or an outage that begins moments after a cache restart. If the query takes 2 seconds and the key receives 1,000 requests per second, about 2,000 identical queries can start before the first one finishes.

Several defences exist, and they combine well:

| Technique              | How it works                                                      |
| ---------------------- | ----------------------------------------------------------------- |
| Request coalescing     | One request per key does the rebuild, and others await its result |
| Lock or lease          | Only the holder rebuilds, and the rest wait or get stale data     |
| Stale-while-revalidate | Serve the expired value while one worker refreshes it             |
| Early refresh          | Refresh popular keys shortly before they expire                   |
| TTL jitter             | Randomise expiry so that keys do not expire together              |
| Warming                | Load popular keys before sending traffic to a new cache           |

Coalescing inside each process, such as Go's `singleflight`, limits duplicates to one per server, and a distributed lock can limit them across the fleet. Give any lock its own expiry, so that a crashed holder cannot block the key forever.

_Usage:_

"The database falls over at the same minute every hour."

"That's a cache stampede on the homepage query. Add request coalescing and jitter the TTL."

## Section 5 — Asynchronous Work

### Message queue

A message queue is a buffer that holds messages from producers until a consumer processes them. The producer adds a task and moves on, without waiting for the work to finish. A checkout service can queue "send receipt" and respond to the customer at once, even if the email provider is slow or briefly down.

Each message is normally handled by one consumer, and several consumers can share a queue to work through it faster:

| Step | Who      | What happens                                                  |
| ---- | -------- | ------------------------------------------------------------- |
| 1    | Producer | Sends a message to the broker, the server that runs the queue |
| 2    | Broker   | Stores it                                                     |
| 3    | Consumer | Receives it, and the broker hides it from other consumers     |
| 4    | Consumer | Does the work                                                 |
| 5    | Consumer | Acknowledges it, and the broker removes it                    |
| 6    | Broker   | Redelivers it if no acknowledgement arrives in time           |

Step 6 means most queues provide at-least-once delivery, so a message can arrive twice. If a consumer finishes sending an email and crashes before acknowledging, another consumer sends it again, and the customer receives two receipts. Consumers should be idempotent, meaning that processing a message twice has the same effect as processing it once. A common method is to record each message ID and skip those already handled.

A job that runs longer than the redelivery window causes the same duplication, because the broker assumes the consumer died. Extend the window for long jobs.

Queue depth is the number to watch. A depth that keeps growing means producers are outrunning consumers, and users see work delayed by minutes or hours while every service reports healthy. Add consumers, or apply [backpressure](#backpressure). Messages that fail repeatedly belong in a [dead letter queue](#dead-letter-queue).

RabbitMQ and Amazon SQS are examples. To deliver one message to several independent consumers, use [publish-subscribe](#publish-subscribe).

_Usage:_

"Customers are getting their invoice PDF two or three times."

"The job takes longer than the message queue's visibility [timeout](#timeout), so it gets redelivered while still running."

### Publish-subscribe

Publish-subscribe, or pub/sub, is a messaging pattern in which publishers send messages to a named topic and every subscriber to that topic receives its own copy. The publisher does not know who is listening. When an order is placed, the order service publishes one `order.placed` message, and the email, analytics and warehouse services each receive it.

The difference from a [message queue](#message-queue) is who gets each message:

| Concern           | Message queue            | Publish-subscribe                 |
| ----------------- | ------------------------ | --------------------------------- |
| Delivery          | One consumer per message | Every subscriber gets a copy      |
| Purpose           | Share out work           | Broadcast that something happened |
| Adding a consumer | Splits the existing work | Adds another independent copy     |
| Typical message   | "Resize this image"      | "An image was uploaded"           |

The main benefit is decoupling. A new service can react to orders by subscribing, with no change to the order service. This makes pub/sub a common building block of [event-driven architecture](#event-driven-architecture). The two patterns are often combined, with each subscriber having its own queue so that it can process its copy at its own pace and retry on failure.

Durability varies by product, and this catches people out. With Redis Pub/Sub, a subscriber that is disconnected when a message is published misses it permanently. A service restarts during a deploy, and the orders published in those seconds never reach it, with no error anywhere. Systems such as Google Cloud Pub/Sub retain messages for each subscription until they are acknowledged or a retention period ends. If missing a message matters, choose a durable option.

Decoupling also hides dependencies. A publisher can change a field in its message and break a consumer its team did not know existed. Keep a record of who subscribes to what, and treat message formats as a public contract.

_Usage:_

"Finance wants a copy of every refund. Do we need to change the payments service?"

"No. It already publishes refund events, so with publish-subscribe they add a subscription."

### Event streaming

Event streaming records events in a durable, append-only log that many consumers read independently. Unlike a [message queue](#message-queue), reading an event does not remove it. Each consumer tracks its own position in the log, called an offset, and events stay until a retention limit is reached. Apache Kafka and Amazon Kinesis are examples.

| Concern        | Message queue              | Event stream                          |
| -------------- | -------------------------- | ------------------------------------- |
| After reading  | Message is removed         | Event stays in the log                |
| Consumer state | Broker tracks each message | Consumer tracks an offset             |
| Replay         | Generally not possible     | Rewind the offset and read again      |
| Ordering       | Often loose                | Guaranteed within a partition         |
| Typical use    | Task distribution          | Data pipelines, activity feeds, audit |

Replay is the main practical benefit. If a bug corrupts a derived table, you can fix the code, reset the offset and rebuild the table from history.

A stream is split into partitions so it can scale, and a partition key decides where each event goes. Order is only guaranteed inside a partition. If events for one order are keyed randomly, "order shipped" can be processed before "order paid", which produces failures that are hard to reproduce. Key by the entity whose order matters, such as the order ID. Avoid low-variety keys, which concentrate traffic in the same way as a [hot key](#hot-key).

Consumers share partitions through a [consumer group](#consumer-group). The health metric is consumer lag, the distance between the newest event and a consumer's offset. Growing lag means the consumer is falling behind and downstream data is getting older.

Retention sets a hard limit. If a consumer is down for longer than the retention period, the events it missed are deleted before it can read them. Delivery is typically at-least-once, so consumers must tolerate duplicates.

_Usage:_

"We shipped a bug that miscalculated revenue for the last three days."

"Event streaming saves us here. Fix the consumer, reset its offset to Monday and rebuild the table."

### Consumer group

A consumer group is a set of consumers that share the work of reading a stream, with each partition assigned to one member at a time. The term comes from [event streaming](#event-streaming) systems such as Apache Kafka. Separate groups each read the full stream independently, so a billing group and an analytics group can consume the same topic at different speeds.

Because a partition has only one active reader per group, the partition count sets the upper limit on parallelism. With 6 partitions:

| Consumers in group | Assignment               | Effect                     |
| ------------------ | ------------------------ | -------------------------- |
| 1                  | One consumer reads all 6 | No parallelism             |
| 2                  | 3 each                   | Twice the capacity         |
| 6                  | 1 each                   | Maximum parallelism        |
| 8                  | 6 active, 2 idle         | Extra consumers do nothing |

The last row is a common surprise. A team sees lag growing, scales from 6 pods to 12, and nothing improves because half of them hold no partitions. The fix is more partitions, which can change how keys map to partitions and therefore affects ordering, or faster processing per message.

When members join or leave, the group performs a rebalance, moving partitions between consumers. During a rebalance, some or all consumption may pause. A consumer that takes too long between polls is presumed dead and removed from the group. Its partitions move to another member, which reprocesses the same slow batch and may also time out, producing a loop of rebalances with little progress. In Kafka this is governed by `max.poll.interval.ms`. Reduce batch size or move slow work elsewhere.

Each group commits offsets to record its progress. A consumer that crashes after processing but before committing causes those events to be processed again, so handlers need to be idempotent, meaning safe to run twice.

_Usage:_

"Lag keeps climbing, so I doubled the consumers, and half of them are idle."

"The topic has 12 partitions, and the consumer group already had 12 members before you doubled it. We need more partitions."

### Backpressure

Backpressure is a way for a slower part of a system to make faster upstream parts reduce their sending rate. Without it, work piles up wherever the producer outruns the consumer, usually in an in-memory buffer. That buffer grows until the process runs out of memory and crashes, losing everything it held.

The recognisable symptom is memory that climbs steadily during a traffic peak, followed by a crash, with [latency](#latency) rising the whole time because each new item waits behind a longer backlog. A queue with no size limit hides overload until it fails.

When a bounded buffer fills up, a component has a limited set of choices:

| Response        | What happens                       | Suitable when                           |
| --------------- | ---------------------------------- | --------------------------------------- |
| Block           | Producer waits until there is room | Producer can safely pause               |
| Reject          | Return an error such as 429 or 503 | Caller can retry or give up             |
| Drop            | Discard new or old items           | Losing some data is fine, as in metrics |
| Sample          | Keep a fraction of items           | Approximate data is acceptable          |
| Scale consumers | Add capacity                       | Slowdown is sustained, not a spike      |

Strictly, only blocking and rejecting push back on the sender. Dropping and sampling protect the system without slowing the producer. All of them depend on the buffer having a limit.

TCP has backpressure built in. A receiver advertises how much data it can accept, and the sender slows when that window shrinks.

The signal has to travel all the way to the source. If a worker slows but its API keeps accepting requests into an unbounded [message queue](#message-queue), the problem has moved without being solved. At the system edge, the final form of backpressure is telling clients to wait, through [rate limiting](#rate-limiting) or [load shedding](#load-shedding).

_Usage:_

"The ingest service gets OOM-killed every time a big customer uploads."

"It buffers without a limit. Bound the queue and add backpressure so uploads wait when the writer falls behind."

### Dead letter queue

A dead letter queue is a holding queue for messages that could not be processed after a set number of attempts. It removes the failing message from the main flow while keeping it for inspection.

The usual reason for needing one is a poison message: a message that always fails, such as one with a malformed field or a reference to a deleted record. In a [message queue](#message-queue) with automatic redelivery, the consumer picks it up, crashes, restarts and picks it up again. Logs fill with one repeated stack trace, and in an ordered system the messages behind it stop moving.

| Step | Who      | What happens                                                |
| ---- | -------- | ----------------------------------------------------------- |
| 1    | Consumer | Fails to process a message                                  |
| 2    | Broker   | Redelivers it and counts the attempt                        |
| 3    | Broker   | Moves it to the dead letter queue once the limit is reached |
| 4    | Alerting | Notifies the team that the queue is no longer empty         |
| 5    | Engineer | Finds the cause and fixes the code or the data              |
| 6    | Engineer | Redrives the message to the main queue, or discards it      |

An attempt limit that is too low sends thousands of good messages to the dead letter queue during a brief dependency outage. A consumer that recognises a permanent error, such as a validation failure, can send the message there at once, without further [retries](#retry-with-backoff).

The common mistake is creating one and never watching it. Messages accumulate unnoticed until retention deletes them. Alert on its depth, and store the error reason and original metadata with each message.

Check ordering before redriving, because a replayed old update can overwrite newer data. Amazon SQS and RabbitMQ support this pattern directly. With Apache Kafka it is usually implemented in the consumer or its framework.

_Usage:_

"The worker has been crash-looping on the same order for an hour."

"That's a poison message. Set a max receive count so it lands in the dead letter queue and the rest can flow."

### Outbox pattern

The outbox pattern saves an outgoing event in the same database transaction as the data change, then publishes it to a message broker afterwards. It solves the dual-write problem, which arises when a service must update its database and also send a message, since those are two systems with no shared transaction.

Doing the two writes in sequence leaves a gap whichever order you choose. If the service commits the order and crashes before publishing, the order exists, but the warehouse never hears about it. If it publishes first and the commit then fails, the warehouse ships an order that does not exist. These bugs surface as a few records that get stuck with no error logged.

With an outbox, one database transaction covers both writes:

| Step | Who     | What happens                                        |
| ---- | ------- | --------------------------------------------------- |
| 1    | Service | Begins a database transaction                       |
| 2    | Service | Writes the business change, such as a new order row |
| 3    | Service | Inserts the event into an `outbox` table            |
| 4    | Service | Commits, so both are saved or neither is            |
| 5    | Relay   | Reads unpublished rows and sends them to the broker |
| 6    | Relay   | Marks them as published once the broker confirms    |

The relay either polls the table or uses change data capture, which reads the database's transaction log. Debezium is a common tool for the second approach.

The relay can crash between steps 5 and 6 and send the event again, so delivery is at-least-once. Consumers must therefore be idempotent, meaning a duplicate has no additional effect.

Delete or archive published rows, or the table grows without limit. Keep events for one entity in order if several relay workers run. It is often used to drive a [saga](#saga) or feed [event-driven architecture](#event-driven-architecture).

_Usage:_

"A few orders each week never reach the warehouse, and there's no error."

"We commit and then publish, so a crash in between loses the event. Use the outbox pattern."

## Section 6 — Resilience

### Timeout

A timeout is a limit on how long a caller waits for an operation before giving up and treating it as failed. Without one, a call to a stalled dependency can wait indefinitely, holding a thread and a connection. Some libraries, such as Python's `requests`, wait indefinitely unless you pass a timeout.

The missing timeout has a recognisable symptom. A service hangs with low CPU and no errors in its logs, because every worker is parked waiting on a slow downstream call. One stalled dependency can then produce a [cascading failure](#cascading-failure) through everything that calls this service.

| Kind            | What it limits                                  | Typical size                    |
| --------------- | ----------------------------------------------- | ------------------------------- |
| Connect         | Establishing the connection                     | Short, often around a second    |
| Read            | Waiting between bytes of the response           | Based on the dependency's p99   |
| Overall request | The whole call from start to finish             | Slightly above the read timeout |
| Deadline        | A budget shared by every hop in a request chain | Set once at the entry point     |

A read timeout alone misses a server that sends one byte every few seconds, so set an overall limit too.

To choose a value, measure the dependency's p99 [latency](#latency) and add a margin. Too short, and healthy requests fail during small slowdowns. Too long, and the protection arrives after your threads are exhausted.

Timeouts must fit together across layers. If a gateway gives up after 5 seconds while its backend keeps working for 30, the backend spends capacity on an answer nobody will read. Pass a deadline with the request, so that each service knows how much time remains.

A timeout does not mean the operation failed. The server may have finished the payment after the client stopped waiting. Before using [retry with backoff](#retry-with-backoff), make sure repeating the operation is safe.

_Usage:_

"The checkout pods are all stuck, but CPU is at 3% and nothing's logged."

"The fraud API is hanging and that client has no timeout, so every worker is waiting on it."

### Retry with backoff

Retry with backoff means repeating a failed call after a wait that grows with each attempt. Many failures are transient, such as a dropped connection or an instance restarting, and a second attempt moments later succeeds. Waiting longer each time gives a struggling service room to recover.

The usual form is exponential backoff with jitter. Jitter is added randomness, and it matters because clients that failed together would otherwise retry together in synchronised waves. With "full jitter", each wait is a random value between zero and a growing ceiling. With a base of 100 ms and a cap of 2 seconds:

| Retry | Ceiling (doubles, capped) | Actual wait      |
| ----- | ------------------------- | ---------------- |
| 1     | 100 ms                    | Random, 0–100 ms |
| 2     | 200 ms                    | Random, 0–200 ms |
| 3     | 400 ms                    | Random, 0–400 ms |
| 4     | 800 ms                    | Random, 0–800 ms |
| 5     | 1,600 ms                  | Random, 0–1.6 s  |
| 6     | 2,000 ms (cap)            | Random, 0–2 s    |

Retry only what can succeed on a second attempt. Connection errors, [timeouts](#timeout), 429 and 503 are reasonable candidates. A 400 or 403 will usually fail the same way, although some APIs also use 403 for [rate limits](#rate-limiting). If the server sends a `Retry-After` header, honour it.

Safety is the other condition. If a payment request times out, the charge may have gone through, and a blind retry charges the customer twice. Retry only idempotent operations, those where repeating has no additional effect. For others, send an idempotency key, a unique ID the server uses to recognise a repeated request.

Always limit the attempts and the total time, and keep that total inside the caller's own deadline. Retrying at several layers multiplies load and can cause a [retry storm](#retry-storm). A [circuit breaker](#circuit-breaker) stops retries when a dependency is down.

_Usage:_

"The sync job fails whenever the vendor API has a brief 503."

"Wrap it in retry with backoff and jitter, five attempts at most, and honour their `Retry-After`."

### Retry storm

A retry storm is a surge of repeated requests from clients retrying failures, which adds load to a service that is already struggling. When a service is overloaded, most requests fail, every failure generates more requests, and the extra traffic keeps the service overloaded.

The recognisable symptom is a dependency that cannot recover. Requests to it are several times higher than real user traffic, it comes back for a moment after a restart, and it is knocked over again within seconds. The original cause may be long gone, while the retries alone sustain the outage.

Retries at several layers multiply. If each layer makes up to 3 attempts, the bottom of the call chain receives:

| Layers that retry | Maximum attempts per user request |
| ----------------- | --------------------------------- |
| 1                 | 3                                 |
| 2                 | 9                                 |
| 3                 | 27                                |
| 4                 | 81                                |

[Retry with backoff](#retry-with-backoff) spreads attempts over time, but it does not reduce how many are made, so it is necessary and not sufficient.

Several controls limit the amplification:

- **Retry at one layer.** Usually the layer closest to the failing call. Others pass the error up.
- **Retry budgets.** Allow retries only up to a fixed share of normal traffic, such as 10%, per client.
- **[Circuit breakers](#circuit-breaker).** Stop calling a dependency that is failing.
- **Explicit overload signals.** Have the server mark an error as "do not retry", and have callers obey it.
- **[Load shedding](#load-shedding).** Reject excess work inexpensively, so that the server survives the surge.

During an incident, the quickest way out is often to cut incoming traffic at the edge and restore it gradually, since the service may be unable to recover while the backlog of retries continues. A retry storm is a common way for a small fault to become a [cascading failure](#cascading-failure).

_Usage:_

"Auth came back up, then fell over again thirty seconds later."

"That's a retry storm. Three layers each make three attempts, so it can see up to 27 times the real load."

### Circuit breaker

A circuit breaker is a wrapper around calls to a dependency that stops sending requests once the failure rate gets too high. Callers then receive an immediate error instead of waiting. The name comes from the electrical device that cuts a circuit to prevent damage.

The problem it addresses is slow failure. When a dependency hangs, each call holds a thread until its [timeout](#timeout) expires, so a service with healthy code runs out of workers waiting on something that will not answer. Failing in a millisecond keeps those workers free for other requests and gives the dependency room to recover.

A breaker moves between three states:

| State     | Behaviour                                    | Moves to                                  |
| --------- | -------------------------------------------- | ----------------------------------------- |
| Closed    | Calls pass through, and failures are counted | Open, when failures cross a threshold     |
| Open      | Calls fail immediately                       | Half-open, after a cool-down period       |
| Half-open | A few trial calls are allowed                | Closed if they succeed, open if they fail |

Thresholds are normally a failure percentage over a sliding period with a minimum number of calls, so that two failures out of three at a quiet moment do not trip it. Count slow calls as well as errors, since slowness is often how a failure begins. Client errors such as 400 should not count, because they say nothing about the dependency's health.

What the caller does while the breaker is open is a product decision. Options include a cached value, a default, or a reduced page, which is [graceful degradation](#graceful-degradation). If there is no fallback, return a clear error.

Use one breaker per dependency, and sometimes per endpoint, so that one bad route does not block a healthy one. Breakers also limit a [retry storm](#retry-storm), because retries against an open breaker fail locally. Resilience4j is an example library.

_Usage:_

"Recommendations is down and it's dragging the product page with it."

"Put a circuit breaker on that call and render the page without the widget while it's open."

### Bulkhead

A bulkhead is an isolation technique that divides resources into separate pools, so that a failure or overload in one part cannot use up capacity needed by the rest. The name comes from the watertight compartments in a ship's hull, which keep a breach from flooding the whole vessel.

The problem it solves is a shared pool. Suppose a service has 200 worker threads and calls three dependencies. One dependency becomes slow, its calls start holding threads, and soon all 200 are waiting on it. Requests that have nothing to do with the slow dependency now fail too. Users find login broken because the PDF report service is slow, and the two features appear unrelated.

With a bulkhead, the slow dependency can only exhaust its own allocation:

| Pool    | Threads | When reports hang                     |
| ------- | ------- | ------------------------------------- |
| Login   | 80      | Unaffected                            |
| Search  | 80      | Unaffected                            |
| Reports | 40      | Full, and extra requests are rejected |

Isolation can be applied at several levels:

| Level           | Example                                                |
| --------------- | ------------------------------------------------------ |
| Concurrency     | A semaphore capping in-flight calls to one dependency  |
| Thread pool     | A separate pool for each downstream service            |
| Connection pool | Separate database pools for web traffic and batch jobs |
| Instance        | Dedicated servers for admin or reporting endpoints     |
| Tenant          | A separate cluster for a very large customer           |
| Infrastructure  | Separate availability zones (isolated sites)           |

The trade-off is reduced efficiency. Capacity reserved for one pool sits idle while another is full, so sizes must be chosen from measured load and reviewed as traffic changes. Too small a pool rejects healthy requests during a normal peak.

Bulkheads work alongside other controls. A [timeout](#timeout) frees a held resource, a [circuit breaker](#circuit-breaker) stops new calls to a failing dependency, and the bulkhead limits how much damage occurs before either acts. Together they help contain a [cascading failure](#cascading-failure).

_Usage:_

"A slow export took down the whole admin API again."

"They share one connection pool. Add a bulkhead so exports get their own small pool."

### Graceful degradation

Graceful degradation means keeping the main functions of a system working, with reduced features, when part of it fails or is overloaded. The alternative is all-or-nothing behaviour, where one optional component takes the whole page down. A product page that returns a 500 because the recommendations widget timed out is a typical example. The customer could have bought the item without seeing recommendations.

Designing for it starts with sorting features by how much they matter, then deciding what each one does when its dependency is unavailable:

| Feature           | Normal       | Degraded                            |
| ----------------- | ------------ | ----------------------------------- |
| Recommendations   | Personalised | Generic bestseller list, or hidden  |
| Search            | Full ranking | Simpler ranking, fewer results      |
| Reviews           | Live data    | [Cache](#cache) copy, or hidden |
| Video             | High quality | Lower bitrate                       |
| Shipping estimate | Exact date   | "Usually ships in 3–5 days"         |
| Checkout          | Full flow    | Stays on, other features give way   |

The mechanics are usually simple. Give each optional call a short [timeout](#timeout), wrap it in a [circuit breaker](#circuit-breaker), and write a fallback. Feature flags let operators turn off expensive features by hand during an incident, which pairs well with [load shedding](#load-shedding).

Not everything should degrade. A fallback that quietly shows an old account balance or a stale stock count can cause more harm than an error. For payments, permissions and other data that must be correct, return an error. Tell the user when something is reduced if it affects their decisions.

Fallback paths need testing. Code that runs only during outages is often broken when it is finally needed, and a fallback that calls the same failing database gives no protection at all. Inject failures in a test environment to verify each one. Record when a fallback is active as well, because a degraded mode that nobody can see can run unnoticed for weeks.

_Usage:_

"The reviews service is down. Do we need to pull the product page?"

"No, it has graceful degradation. The page renders from cache and hides the review section."

### Load shedding

Load shedding means deliberately rejecting some requests when a system is overloaded, so that it can keep serving the rest. The name comes from electricity grids, which cut power to some areas to protect the whole network.

It is needed because overloaded systems often do less useful work as load rises. Past capacity, requests queue and wait so long that clients give up, while the server keeps working on answers nobody will receive. CPU stays at 100% and [throughput](#throughput) of successful responses falls toward zero. Rejecting 20% of requests early can let the other 80% succeed, where accepting all of them might mean that almost none do.

The decision has to be inexpensive, so it happens as early as possible, before authentication, database calls or parsing a large body. Useful overload signals include the number of requests in flight and the time requests spend queued. These react faster and more directly than average CPU. Rejected requests typically get a 503, optionally with `Retry-After`.

Shedding works better when requests have priorities:

| Priority | Example                    | Under overload |
| -------- | -------------------------- | -------------- |
| Critical | Checkout, login            | Shed last      |
| Normal   | Product pages, search      | Shed if needed |
| Low      | Recommendations, analytics | Shed early     |
| Batch    | Exports, background sync   | Shed first     |

Also drop requests whose deadline has already passed.

Load shedding differs from [rate limiting](#rate-limiting), which enforces per-client quotas regardless of server health. Shedding responds to the server's current state and can affect any client. It pairs with [graceful degradation](#graceful-degradation), where the system does less for each request instead of refusing it.

Clients must cooperate. If every rejected request is retried immediately, shedding feeds a [retry storm](#retry-storm). Test the behaviour with a load test beyond capacity, and confirm that successful throughput holds steady.

_Usage:_

"At peak the API serves nothing, even though it's only 30% over capacity."

"It's accepting everything and timing out. Add load shedding on queue time and drop the batch traffic first."

### Cascading failure

A cascading failure is a failure in one component that causes others to fail, spreading until a large part of the system is down. The initial fault is often small, such as one slow database query or one lost instance.

There are two common ways it spreads. One is load transfer: when a server fails, its traffic moves to the survivors, which then exceed their capacity and fail too. The other is resource exhaustion: a slow dependency holds its callers' threads and connections until those callers stop responding, and their own callers then stall in turn.

| Step | What happens                                                   |
| ---- | -------------------------------------------------------------- |
| 1    | A database slows down                                          |
| 2    | Service A's threads fill up waiting on it                      |
| 3    | Health checks on A time out, and instances are removed         |
| 4    | Remaining A instances receive more traffic and fail            |
| 5    | Service B, which calls A, runs out of threads                  |
| 6    | Clients and services retry, multiplying the load               |
| 7    | The site is down, including features unrelated to the database |

Fixing the original fault often does not end the incident. A system can stay down after its trigger is gone, because retries, cold [caches](#cache) and queued work keep it overloaded. This is sometimes called a metastable failure. Restarting everything at once tends to fail again, since instances come up with empty caches into full traffic.

Each defence interrupts a different step:

| Defence                                   | What it stops                       |
| ----------------------------------------- | ----------------------------------- |
| [Timeout](#timeout)                   | Threads waiting indefinitely        |
| [Circuit breaker](#circuit-breaker) | Calls to a failing dependency       |
| [Bulkhead](#bulkhead)                 | One dependency using all capacity   |
| [Load shedding](#load-shedding)     | Overload on survivors               |
| Retry budget                              | A [retry storm](#retry-storm) |

To recover, reduce traffic at the edge, let the system drain, and bring load back in stages. Keep enough spare capacity that losing one instance or zone does not overload what remains.

_Usage:_

"How did a slow reporting query take down login?"

"It was a cascading failure. The query held the connection pool, the API stalled, and retries finished it off."

## Section 7 — Architecture Styles

### Monolith

A monolith is an application built and deployed as a single unit. All its features run in the same process and usually share one database, and one part calls another with an ordinary function call. It can still run as many identical copies behind a [load balancer](#load-balancer), so a monolith is not limited to one server.

For most new products a monolith is a reasonable starting point. Function calls do not fail the way network calls do, a database transaction can cover a whole operation, a stack trace shows the full path of a request, and there is one thing to build, test and deploy.

The costs appear as the codebase and the team grow:

| Growing pain      | How it shows up                                              |
| ----------------- | ------------------------------------------------------------ |
| Slow pipeline     | A one-line fix waits for a long build and full test suite    |
| Deploy contention | Teams queue to release, and one bad change blocks everyone's |
| Shared fate       | A memory leak in reports takes down checkout too             |
| Uniform scaling   | Scaling one CPU-heavy feature means scaling the whole app    |
| Tangled code      | A change in billing breaks something in shipping             |

The last row is often blamed on the deployment model, but it comes from weak internal boundaries. A tangled monolith split into services usually becomes a tangled distributed system. Fixing the structure first, by moving to a [modular monolith](#modular-monolith), addresses the cause and keeps deployment simple.

Some pains have targeted fixes. Slow or risky background work can run as a separate worker process from the same codebase. A [bulkhead](#bulkhead) can separate traffic types. Only when independent deployment or scaling is a demonstrated need does a move to [microservices](#microservices) justify its cost, and the [strangler fig](#strangler-fig) approach allows that move in stages.

_Usage:_

"Should a team of four start with separate services for users, billing and orders?"

"No. Build a monolith with clean module boundaries, and split something out when there's a concrete reason."

### Modular monolith

A modular [monolith](#monolith) is a single deployable application divided into modules with enforced boundaries. Each module covers one business area, such as billing or inventory, owns its logic and data, and exposes a small public interface to the others. It deploys as one unit but is organised internally much like a set of services.

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

Clean boundaries keep future options open. If one module later needs independent scaling, its interface and data are already separated, so extracting it into one of a set of [microservices](#microservices) is a contained piece of work.

_Usage:_

"We keep breaking each other's code, so should we move to services?"

"The problem is boundaries, not deployment. Try a modular monolith with import rules enforced in CI first."

### Microservices

Microservices is an architecture in which an application is split into small services that are deployed independently, each owning one business capability and its data. Services communicate over the network, through APIs or messages. Size matters less than independence: a team should be able to change and release its service without coordinating with others.

| Concern              | [Monolith](#monolith) | [Modular monolith](#modular-monolith) | Microservices           |
| -------------------- | ------------------------- | ------------------------------------------- | ----------------------- |
| Deployment           | One unit                  | One unit                                    | Per service             |
| Internal calls       | Function call             | Function call via interface                 | Network call            |
| Transactions         | Single database           | Single database                             | Across services is hard |
| Scaling              | Whole app                 | Whole app                                   | Per service             |
| Failure isolation    | Low                       | Low                                         | Possible, with effort   |
| Operational overhead | Low                       | Low                                         | High                    |

The main reason to adopt them is organisational. When many teams share one codebase, coordination becomes the limit on delivery, and separate services let them ship on their own schedules. A small team usually gets the costs without this benefit.

Those costs are substantial. Every call can now fail or be slow, so you need [timeouts](#timeout), retries and [circuit breakers](#circuit-breaker). A [cascading failure](#cascading-failure) can cross service boundaries. Operations that were one transaction become a [saga](#saga). You also need [service discovery](#service-discovery), distributed tracing (following one request across services) and automated deployment.

A common failure is the distributed monolith. Services are separate on paper, but they share a database or call each other in long synchronous chains, so a feature requires five coordinated deploys and one service being down breaks all of them. That combines the coupling of a monolith with the failure modes of a network.

Draw boundaries around business capabilities that change independently. If two services always change together, they should probably be one service.

_Usage:_

"Why does adding one field to checkout need releases from five teams?"

"Because our microservices share a schema. We have a distributed monolith, and that's what we need to fix."

### Event-driven architecture

Event-driven architecture is a design in which components announce what has happened by emitting events, and other components react to those events without being called directly. An event is a record of a past fact, such as `OrderPlaced` or `PaymentFailed`. The sender does not know or care who consumes it.

Events differ from commands, and confusing them leads to tightly coupled designs:

| Aspect       | Command                  | Event                   |
| ------------ | ------------------------ | ----------------------- |
| Meaning      | "Do this"                | "This happened"         |
| Tense        | Imperative: `ChargeCard` | Past: `CardCharged`     |
| Recipients   | One specific handler     | Zero, one or many       |
| Sender knows | Who will do the work     | Nothing about consumers |
| Can refuse?  | Yes                      | No, it already happened |

Events usually travel through [publish-subscribe](#publish-subscribe) or [event streaming](#event-streaming). The benefit is loose coupling. To send a welcome gift when a user signs up, add a consumer of `UserSignedUp` and leave the signup service alone. If a consumer is down, the rest of the system keeps running and it can catch up later.

The costs show up in understanding and debugging. There is no single place to read the flow of a business process, so "what happens after an order is placed?" requires tracing several consumers. When a step silently does not happen, there may be no error anywhere. Attach a correlation ID (one identifier carried through every related event) and use tracing to reconstruct what occurred.

Results are also eventually consistent, meaning different parts of the system catch up at different times. A customer places an order and their order history does not show it for a few seconds. The interface has to allow for that.

Consumers must handle duplicate and out-of-order events. Publish reliably with the [outbox pattern](#outbox-pattern), and treat event schemas as contracts, because you may not know everyone who depends on them.

_Usage:_

"Do we have to touch checkout to add loyalty points?"

"No, that flow uses event-driven architecture. Loyalty can consume `OrderPaid` without checkout knowing it exists."

### CQRS

CQRS, or Command Query Responsibility Segregation, is a pattern that uses separate models for changing data and for reading it. Commands such as `PlaceOrder` go through a write model that enforces business rules. Queries such as "show this customer's order history" go through a read model shaped for that screen.

The motivation is that the two sides often want different structures. A write model benefits from normalised tables and strict validation. A dashboard wants wide, denormalised rows it can fetch in one query. Forcing both through one model produces a domain object with dozens of fields and a reporting query that joins nine tables and slows checkout on the same database.

| Concern | Command side                 | Query side                                                                                            |
| ------- | ---------------------------- | ----------------------------------------------------------------------------------------------------- |
| Purpose | Change state                 | Return data                                                                                           |
| Model   | Business rules, invariants   | Shaped per screen or report                                                                           |
| Storage | Often normalised             | Often denormalised or a search index                                                                  |
| Scaling | Limited by write correctness | Copies [scale out](#horizontal-scaling) for a [read-heavy workload](#read-heavy-workload) |
| Returns | Success or failure           | Data, no side effects                                                                                 |

CQRS has levels. The simplest keeps one database and uses separate code paths for reads and writes, which adds little cost. A fuller version has separate read stores, updated from the write side through events, often using the [outbox pattern](#outbox-pattern).

Separate stores introduce replication lag, and this is the failure users notice. Someone saves a change, the page reloads from the read model, and the old value appears, so they save again. Common responses are to show the submitted value optimistically in the interface, or to have the client wait until the read model reaches a given version.

CQRS is often paired with [event sourcing](#event-sourcing), but neither requires the other. It adds moving parts, so for simple create-read-update-delete screens, a single model is usually the right choice.

_Usage:_

"The analytics page keeps locking the orders table during checkout."

"Use CQRS for that area. Project orders into a reporting table and let the page read from there."

### Event sourcing

Event sourcing stores every change to an entity as an append-only sequence of events, and derives the current state by replaying them. A conventional table keeps only the latest value and overwrites the previous one. An event-sourced system keeps the full history, in the way a bank ledger records transactions and calculates a balance from them.

| Sequence | Event           | Balance after replay |
| -------- | --------------- | -------------------- |
| 1        | `AccountOpened` | 0                    |
| 2        | `Deposited 100` | 100                  |
| 3        | `Withdrawn 30`  | 70                   |
| 4        | `Deposited 50`  | 120                  |

The event log is the source of truth. This gives a complete audit trail, lets you reconstruct the state at any past moment, and answers questions such as "how did this order end up cancelled?", which an overwritten row cannot. It also lets you build new read models from existing history, which is why it is often combined with [CQRS](#cqrs).

It is different from [event streaming](#event-streaming). A stream moves events between systems. Event sourcing uses events as the primary storage for an entity's state, and needs to load all events for one entity reliably and reject conflicting writes, usually by checking an expected version number on append.

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

### Saga

A saga is a multi-service operation run as a series of local transactions, with a compensating action to undo each completed step if a later one fails. It is used when one business action spans several databases, as is common in [microservices](#microservices), and a single atomic transaction is unavailable or impractical.

Placing an order might look like this:

| Step | Service   | Action            | Compensation   |
| ---- | --------- | ----------------- | -------------- |
| 1    | Orders    | Create as pending | Cancel order   |
| 2    | Inventory | Reserve stock     | Release stock  |
| 3    | Payments  | Charge card       | Refund         |
| 4    | Shipping  | Book courier      | Cancel booking |

If payment fails at step 3, the saga runs the compensations for steps 2 and 1 in reverse: release the stock, then cancel the order. If the courier booking fails at step 4, it refunds the card first. Without this logic, the familiar result is a customer who was charged for an order that stays stuck.

There are two ways to coordinate the steps:

| Style         | How it works                                | Trade-off                                  |
| ------------- | ------------------------------------------- | ------------------------------------------ |
| Orchestration | A coordinator tells each service what to do | Flow is visible, coordinator is extra code |
| Choreography  | Each service reacts to the previous event   | No coordinator, flow is hard to follow     |

Compensation is not a true rollback. A refund is a new transaction, and the customer may see both the charge and the refund on a statement. Place steps that are hard to undo as late as possible.

Sagas also lack isolation, so other requests can see intermediate state, such as stock that is reserved and then released. Mark such records as pending.

Messages can be redelivered, so every step and every compensation must be idempotent, meaning safe to repeat. Use the [outbox pattern](#outbox-pattern) to publish each step reliably, and persist the saga's state so it can resume after a crash.

_Usage:_

"Payment succeeded, but the courier booking failed and the order is stuck in pending."

"The saga isn't compensating for that failure. It should refund, release the stock and cancel automatically."

### Strangler fig

The strangler fig is a migration pattern that replaces a legacy system gradually, by routing one feature at a time to a new system until the old one can be removed. The name comes from a plant that grows around a host tree and can eventually replace it.

The alternative is a full rewrite, and its failure is a familiar one. A team spends a long time rebuilding the product while the old system keeps gaining features, so the target keeps moving, nothing reaches users until the end, and the cutover is a single high-risk event. This pattern ships in small, reversible steps.

| Step | What happens                                                                                                                          |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Put a routing layer, such as a [reverse proxy](#reverse-proxy) or [API gateway](#api-gateway), in front of the old system |
| 2    | Pick one well-bounded feature and build it in the new system                                                                          |
| 3    | Send a small share of its traffic to the new version                                                                                  |
| 4    | Compare results and error rates, and increase the share                                                                               |
| 5    | Delete the old code for that feature                                                                                                  |
| 6    | Repeat, then remove the routing layer when nothing old remains                                                                        |

Rollback at step 3 or 4 is a routing change. Shadowing helps before that: the new version receives a copy of live requests, and its responses are compared but not returned.

Data is usually the hardest part. Options include having the new service call the old database temporarily, syncing changes between stores, or writing to both during a transition. Decide which system is the source of truth for each entity at each stage.

The common failure is stopping halfway. Once the easy features have moved, the hard ones remain, priorities shift, and the team now runs two systems indefinitely. Track the traffic still served by the old system.

_Usage:_

"Can we rewrite the billing [monolith](#monolith) in one go over the next year?"

"Too risky. Use a strangler fig: put a proxy in front and move invoices first."

### Backend for frontend

A backend for frontend, or BFF, is a dedicated server-side layer built for one type of client, such as the web app or the mobile app. It calls the underlying services and returns data shaped for that client's screens.

The pattern addresses a recognisable problem with a shared API. Mobile users download large responses and discard most of the fields. A single phone screen needs six round trips, which is slow on a poor connection. Meanwhile, the API accumulates parameters and `if (client == "ios")` branches, and a change requested by one client team risks breaking another.

| Client  | What its BFF does                                          |
| ------- | ---------------------------------------------------------- |
| Mobile  | Combines calls into one, trims fields, resizes image URLs  |
| Web     | Returns richer data, manages sessions with secure cookies  |
| TV      | Returns simple layouts suited to remote-control navigation |
| Partner | Exposes a stable, versioned contract                       |

A BFF is usually owned by the same team as the client it serves, so that team can change its API without waiting for another group. For browser apps, it can also keep access tokens on the server and give the browser only a session cookie, which limits what a malicious script could steal.

It differs from an [API gateway](#api-gateway). A gateway is one shared entry point handling generic concerns such as authentication and [rate limiting](#rate-limiting). A BFF contains logic specific to one client, and often sits behind a gateway.

The main risk is duplication. Business rules that creep into each BFF will drift apart, so the web app and the mobile app calculate a discount differently. Keep BFFs to aggregation and formatting, and leave rules in the underlying services. GraphQL, a query language that lets clients request specific fields, is a common alternative when the main need is flexible data fetching.

_Usage:_

"The app makes eleven calls to render the home screen."

"Add a backend for frontend for mobile, so it makes one call and gets exactly what the screen needs."

### Sidecar

A sidecar is a helper process deployed alongside each application instance to provide supporting functions without changes to the application's code. The two run together, start and stop together, and usually talk over localhost. In Kubernetes, a sidecar is typically a second container in the same pod, sharing its network and optionally its storage.

The pattern is useful when many services, possibly in different languages, need the same capability. Without it, every team installs and upgrades its own library, and versions drift.

| Sidecar job    | What it does                                            | Example                 |
| -------------- | ------------------------------------------------------- | ----------------------- |
| Traffic proxy  | Handles TLS, retries, [timeouts](#timeout), routing | Envoy                   |
| Log shipping   | Reads local logs and forwards them                      | Fluent Bit              |
| Metrics        | Collects and exports telemetry                          | OpenTelemetry Collector |
| Secrets        | Fetches and refreshes credentials                       | Vault Agent             |
| Database proxy | Manages secure connections                              | Cloud SQL Auth Proxy    |

A service mesh, a dedicated layer for service-to-service traffic, commonly places a proxy sidecar beside every instance. Istio and Linkerd are examples.

Lifecycle ordering causes the most familiar bugs. An application starts before its proxy is ready, and its first network calls fail with connection refused. During shutdown, the proxy exits first, and in-flight requests are dropped. In a batch job, the main container finishes while the sidecar keeps running, so the job never completes. Kubernetes now has native sidecar containers that start before the main containers and stop after them, which addresses much of this.

The costs multiply with scale. A sidecar using 100 MB of memory across 1,000 pods consumes about 100 GB. A proxy also adds a local network hop to every request, adding some [latency](#latency).

For a single service, a library is usually simpler.

_Usage:_

"Half the pods throw connection errors for the first few seconds after boot."

"The app starts before the Envoy sidecar is ready. Make it a native sidecar with a startup probe, so the app waits until the proxy is ready."
