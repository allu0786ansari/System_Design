SYSTEM DESIGN STUDY NOTES
PHASE 1: SYSTEM DESIGN FUNDAMENTALS

These are the foundational concepts that every other phase builds on. Do not rush this phase — nearly every interview question and every later topic (caching, sharding, messaging, microservices) is explained using the vocabulary defined here.


1. WHAT IS SYSTEM DESIGN?

System design is the process of defining the architecture, components, modules, interfaces, and data flow of a system to satisfy a specific set of requirements. In interviews, it means: given a problem statement (e.g., "design Twitter"), you must decide how data is stored, how requests flow through the system, how the system stays fast and available as it grows, and what trade-offs you are making.

Two flavors of system design:
- High-Level Design (HLD): the big picture — services, databases, load balancers, how components talk to each other. This is what most interviews focus on.
- Low-Level Design (LLD): class diagrams, API contracts, database schema, design patterns inside a single service.

A system design interview is not about finding "the correct answer" — there isn't one. It is about:
- Clarifying requirements before jumping to a solution
- Making reasonable assumptions explicit
- Estimating scale (users, requests/sec, data size)
- Proposing a design and justifying trade-offs
- Identifying bottlenecks and iterating

Example: "Design a URL shortener" is not really about URL shortening — it's about testing whether you can reason about read/write ratios, database choice, ID generation, and caching.


2. FUNCTIONAL VS NON-FUNCTIONAL REQUIREMENTS

Every system design problem starts by splitting requirements into two buckets.

Functional Requirements (FR): What the system must DO. These are the features.
Example (URL Shortener):
- User can submit a long URL and get a short URL back
- User can visit the short URL and get redirected to the original URL
- User can set an optional expiry date

Non-Functional Requirements (NFR): How the system must BEHAVE while doing it — the quality attributes.
Example (URL Shortener):
- System must be highly available (redirects should almost never fail)
- Redirect latency should be under 100ms
- System should handle 100M redirects/day
- Short URLs must be unique

Why this split matters: FRs decide *what* you build (the API, the entities). NFRs decide *how* you build it (which database, whether you need caching, whether you need multiple regions). Most of the "hard" decisions in system design come from NFRs, not FRs.

Common NFR categories you will keep meeting in later phases: scalability, availability, reliability, durability, fault tolerance, latency, consistency, security, maintainability, cost.

Rule of thumb for interviews: always state your FRs and NFRs out loud in the first 5 minutes, and explicitly ask the interviewer which NFRs matter most (e.g., "should I optimize for consistency or availability here?").


3. SCALABILITY

Scalability is a system's ability to handle increasing load (more users, more data, more requests) by adding resources, without a proportional drop in performance.

Two dimensions of load growth:
- More traffic (requests per second)
- More data (storage growth)

A system is "scalable" if you can throw more resources at it and it keeps performing well — as opposed to a system that hits a wall no matter how much hardware you add (a badly designed single database with a full-table-scan query is a classic example: adding more RAM only delays the problem).

Related idea — "scale cube" (three axes of scaling, useful mental model):
- X-axis: run multiple identical copies of the app behind a load balancer (horizontal scaling)
- Y-axis: split the app by function/service (microservices — covered in Phase 10)
- Z-axis: split data by some key, e.g. user ID range (sharding — covered in Phase 4)

Example: A single web server handling 100 requests/sec starts timing out at 1000 requests/sec. Adding 10 identical web servers behind a load balancer to handle the 1000 requests/sec is horizontal scalability in action.


4. VERTICAL SCALING VS HORIZONTAL SCALING

Vertical Scaling (Scale Up): Add more power (CPU, RAM, disk) to an existing single machine.
- Pros: simple, no code changes, no distributed-system complexity (no network calls between nodes, no data consistency issues)
- Cons: hardware has a physical ceiling, single point of failure, expensive at the high end, requires downtime to upgrade in many cases

Horizontal Scaling (Scale Out): Add more machines and distribute the load across them.
- Pros: near-infinite scaling, better fault tolerance (one machine dying doesn't kill the system), can use commodity hardware
- Cons: added complexity — you now need a load balancer, data needs to be partitioned or replicated, you must handle network failures, distributed consistency becomes a real problem

Comparison table:

| Aspect | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| Method | Bigger machine | More machines |
| Ceiling | Limited by hardware | Practically unlimited |
| Complexity | Low | High |
| Fault tolerance | Poor (single point of failure) | Good |
| Cost curve | Expensive at high end | Cheaper, incremental |
| Example | Upgrading RDS instance size | Adding more app servers behind an ELB |

In practice, real systems use both: vertically scale a database up to a point, then horizontally scale the read layer (read replicas) and app layer.


5. AVAILABILITY

Availability is the percentage of time a system is operational and able to serve requests, over a given time period.

Formula: Availability = Uptime / (Uptime + Downtime)

Measured in "nines":

| Availability | Downtime per year |
|---|---|
| 99% (two nines) | ~3.65 days |
| 99.9% (three nines) | ~8.76 hours |
| 99.99% (four nines) | ~52.6 minutes |
| 99.999% (five nines) | ~5.26 minutes |

How to increase availability: redundancy (multiple servers/regions), failover mechanisms, load balancing, eliminating single points of failure, health checks + auto-restart.

Availability in distributed systems: if you have multiple independent components each with their own availability, and a request depends on ALL of them (serial dependency), overall availability is the product of each component's availability — it goes DOWN. If components are redundant (parallel, only need one to succeed), availability goes UP.

Example: If your service depends on 3 microservices each with 99.9% availability in series, your overall availability is roughly 99.9% × 99.9% × 99.9% ≈ 99.7% — worse than any individual piece.


6. RELIABILITY

Reliability is the probability that a system performs its intended function correctly for a specified period of time, without failure.

Availability vs Reliability — the classic confusion: a system can be available but not reliable (it responds to every request, but sometimes returns wrong/corrupted data), or reliable but not available (it always gives correct results, but is frequently down for maintenance).

Ways to improve reliability:
- Redundancy and replication
- Thorough testing (unit, integration, chaos testing)
- Graceful degradation (serve a stale cached response instead of failing completely)
- Monitoring and fast incident response
- Idempotent operations (retry-safe)

Example: An ATM that is online 99.99% of the time (highly available) but occasionally dispenses the wrong amount of cash due to a software bug is available but not reliable.


7. DURABILITY

Durability is the guarantee that once data is written/committed, it will not be lost, even in the face of crashes, power failures, or hardware faults.

This is a storage-layer concern (as opposed to availability, which is about the whole system responding).

How durability is achieved:
- Write-ahead logging (WAL) — write the intended change to a log before applying it, so it can be replayed after a crash
- Replication across multiple disks/machines/data centers
- Periodic backups and snapshots
- RAID configurations at the hardware level
- Fsync-ing data to disk instead of only keeping it in memory

Example: When you get a "payment successful" confirmation from a bank, you expect that transaction record to survive even if the database server crashes one second later. That guarantee is durability.

Durability vs Availability: data can be durable but temporarily unavailable (disk is fine, but the network to it is down), or available but not durable (an in-memory cache is highly available and fast, but a crash wipes everything).


8. FAULT TOLERANCE

Fault tolerance is a system's ability to continue operating correctly even when one or more of its components fail.

Key techniques:
- Redundancy: no single point of failure — every critical component has a backup
- Replication: keep copies of data/services on multiple nodes
- Failover: automatically switch to a backup when the primary fails
- Isolation (Bulkhead pattern — Phase 10): failures in one part of the system don't cascade to others
- Circuit Breakers (Phase 10): stop calling a failing downstream service to prevent pile-up
- Graceful degradation: serve a reduced but functional experience instead of total failure

Fault tolerance vs High Availability: fault tolerance is often a mechanism used to ACHIEVE high availability. A fault-tolerant system masks failures from the user; a merely "available" system might have brief blips during failover.

Example: Netflix's Chaos Monkey randomly kills production instances to verify the system tolerates faults without impacting users — this is fault tolerance being deliberately tested.


9. LATENCY

Latency is the time taken for a single request to travel from the client, get processed, and return a response — essentially, "how long does one operation take."

Usually measured in milliseconds (ms).

Percentiles matter more than averages in system design:
- p50 (median): half of requests are faster than this
- p95: 95% of requests are faster than this
- p99: 99% of requests are faster than this — this is what interviews usually care about, because it represents your worst "typical" user experience, not diluted by a huge number of fast requests

Sources of latency: network round trip, DNS lookup, TLS handshake, application processing time, database query time, disk I/O, serialization/deserialization.

Example: An API might have an average latency of 50ms but a p99 latency of 800ms — meaning 1 in 100 users has a noticeably slow, possibly frustrating experience, even though "on average" things look fine. This is why p99 (or p95) is the number engineers actually optimize.


10. THROUGHPUT

Throughput is the number of requests (or units of work) a system can process per unit of time — usually measured as requests per second (RPS) or queries per second (QPS), or transactions per second (TPS).

Latency vs Throughput — these are NOT the same thing and are often in tension:
- Latency: how fast is ONE request
- Throughput: how many requests total, over time

Example: A highway analogy — latency is how long it takes one car to travel from A to B; throughput is how many cars pass a point per hour. You can have low latency (empty highway, cars fly through) but low throughput (few cars total), or high throughput with higher latency (many cars, but traffic jams slow each individual car).

In system design, you typically calculate expected throughput (QPS) during capacity estimation, e.g.: "100M daily active users, each making 10 requests/day, gives ~1000 requests/sec average, with peak traffic often 3-5x the average."


11. BANDWIDTH

Bandwidth is the maximum rate of data transfer across a network path, usually measured in bits per second (Mbps, Gbps).

Bandwidth vs Throughput vs Latency:
- Bandwidth: the theoretical capacity of the pipe (how much data COULD flow per second)
- Throughput: the actual data/requests flowing per second (often less than bandwidth due to bottlenecks)
- Latency: the delay for a single unit of data to travel

Analogy: Bandwidth is the width of a water pipe, throughput is how much water actually flows through it, and latency is how long a single drop of water takes to travel from one end to the other.

Why it matters in system design: video streaming (Phase 15 — YouTube/Netflix) and file storage systems are bandwidth-heavy; they need to estimate total bandwidth (concurrent users × bitrate) to size CDNs and network infrastructure correctly.


12. PERFORMANCE METRICS

This is the summary category tying together 9, 10, and 11 plus a few more you should know cold for interviews:

- Latency (response time) — covered above
- Throughput (QPS/RPS/TPS) — covered above
- Bandwidth — covered above
- Error rate — percentage of requests that fail (5xx errors, timeouts)
- Availability — uptime percentage
- Resource utilization — CPU%, memory%, disk I/O%, network I/O% (used to detect bottlenecks and decide when to scale)
- Saturation — how "full" a resource is; a system near saturation shows rising latency well before it fully fails
- Concurrency — number of simultaneous requests/connections being handled at once

The "Four Golden Signals" (from Google's SRE book — good to mention in interviews): Latency, Traffic (throughput), Errors, Saturation. These four together give a near-complete picture of a system's health and are the basis of most production dashboards (covered more in Phase 13: Observability).


13. CAP THEOREM

CAP Theorem states that a distributed system can only guarantee at most TWO of the following three properties at the same time, in the presence of a network partition:

- Consistency (C): every read receives the most recent write or an error (all nodes see the same data at the same time)
- Availability (A): every request receives a (non-error) response, without guarantee it contains the latest write
- Partition Tolerance (P): the system continues to operate despite network partitions (messages between nodes being dropped/delayed)

The key insight: in any real distributed system, network partitions WILL happen (cables get cut, packets get lost) — so Partition Tolerance is not really optional, it's a given. That means the real-world choice is between C and A when a partition occurs.

- CP systems: choose consistency over availability during a partition — they may return an error or time out rather than serve stale data. Example: HBase, MongoDB (in certain configurations), ZooKeeper, etcd.
- AP systems: choose availability over consistency during a partition — they always respond, but might serve stale data temporarily. Example: Cassandra, DynamoDB, CouchDB.
- CA systems: only possible in a system with no network partitions at all, i.e. effectively a single-node system — not realistic for genuinely distributed systems, so this option is mostly theoretical.

Example scenario: Two data centers, A and B, get disconnected from each other (a partition). A user writes data to A.
- If the system is CP: data center B will refuse to serve reads (or error out) rather than risk showing stale data, until the partition heals.
- If the system is AP: data center B will keep serving reads using its last-known (possibly stale) copy of the data, prioritizing "always respond" over "always correct."

Interview tip: CAP is often oversimplified. In practice, most real systems are tunable — e.g., Cassandra lets you choose consistency level per query (ONE, QUORUM, ALL) — so the real answer is "it's a spectrum, not a strict binary," and PACELC (next topic) extends this thinking further.


14. PACELC THEOREM

PACELC is an extension of CAP that addresses a gap: CAP only describes trade-offs during a network Partition — but what about normal operation, when there's no partition at all?

PACELC states: If there is a Partition (P), a system must trade off Availability (A) vs Consistency (C) — this part is just CAP. Else (E), even when the system is running normally with no partition, it must trade off Latency (L) vs Consistency (C).

So the full trade-off is: PA/EL, PC/EC, etc.

- PA/EL systems: during a partition, favor availability; during normal operation, favor low latency over strict consistency. Example: Cassandra, DynamoDB.
- PC/EC systems: during a partition, favor consistency; during normal operation, also favor consistency over latency, even if that means waiting longer. Example: traditional relational databases run in a distributed/replicated mode, Google Spanner (leans PC/EC but engineered to minimize the latency cost).

Why this matters more than plain CAP: partitions are actually rare events; the everyday trade-off engineers make constantly is latency vs consistency (e.g., "should this read wait for the leader to confirm, or can it read from a nearby replica that might be a few milliseconds stale but is much faster?"). PACELC captures that this trade-off exists even when everything is healthy.


15. CONSISTENCY MODELS

A consistency model defines the rules about what value a read operation is allowed to return, relative to previous writes, in a system with multiple copies of data (replicas).

Main consistency models to know (in decreasing strictness):

- Strict/Linearizable Consistency: every read returns the most recent write, as if there were only a single copy of the data and operations happened instantaneously in real-time order. Strongest and most expensive.
- Sequential Consistency: all operations appear to happen in some single global order that respects each individual process's own program order (but that order need not match real/wall-clock time).
- Causal Consistency: operations that are causally related (e.g., a reply to a comment) are seen by everyone in the same order; unrelated (concurrent) operations may be seen in different orders by different nodes.
- Eventual Consistency: given enough time with no new writes, all replicas will converge to the same value — but at any given moment, different replicas might return different (stale) values. (Expanded in topic 18 below.)

Where this matters: choosing a consistency model is one of the most important decisions in distributed database design (Phase 3 and 4), and it directly ties back to CAP/PACELC — stronger consistency generally costs you latency and/or availability.

Example: In a chat app, message ordering within one conversation typically needs causal consistency (replies should appear after the message they're replying to) — but the "online/offline" status indicator can safely be eventually consistent (a few seconds of staleness is harmless).


16. ACID PROPERTIES

ACID is a set of properties that guarantee reliable processing of database transactions, primarily associated with relational (SQL) databases.

- Atomicity: a transaction is all-or-nothing. If any part of it fails, the entire transaction is rolled back — the database is never left in a partially-updated state. Example: transferring money between two bank accounts must debit one account AND credit the other, or do neither.
- Consistency (the "C" in ACID, different from the "C" in CAP): a transaction takes the database from one valid state to another, respecting all defined rules, constraints, and triggers (e.g., foreign key constraints, unique constraints).
- Isolation: concurrent transactions execute as if they ran one after another (serially), even though they may physically run in parallel. Prevents issues like dirty reads, non-repeatable reads, and phantom reads. Controlled via isolation levels: Read Uncommitted, Read Committed, Repeatable Read, Serializable (each trading off strictness vs performance).
- Durability: once a transaction is committed, it stays committed, even if the system crashes immediately afterward (same concept as topic 7 above, applied specifically to transactions).

Example: A bank transfer transaction — BEGIN TRANSACTION → debit $100 from Account A → credit $100 to Account B → COMMIT. If the system crashes after the debit but before the credit, Atomicity guarantees the whole transaction rolls back, so the $100 isn't lost in limbo.


17. BASE PROPERTIES

BASE is the philosophical opposite of ACID, associated with many NoSQL/distributed databases that prioritize availability and scalability over strict transactional guarantees.

BASE stands for:
- Basically Available: the system guarantees availability, in the CAP sense — it will respond to requests, even during failures, even if the response is based on stale data.
- Soft state: the state of the system may change over time, even without new input, as data propagates between replicas to reach consistency (i.e., the "current" state isn't guaranteed fixed until it settles).
- Eventual consistency: given enough time without new writes, all replicas will eventually converge to the same value (same concept as topic 18).

ACID vs BASE comparison:

| Aspect | ACID | BASE |
|---|---|---|
| Priority | Consistency & correctness | Availability & scalability |
| Typical databases | MySQL, PostgreSQL, Oracle | Cassandra, DynamoDB, MongoDB (default) |
| Consistency | Strong, immediate | Eventual |
| Use case | Banking, financial transactions | Social media feeds, product catalogs, analytics |
| CAP alignment | Leans CP | Leans AP |

Example: A banking ledger needs ACID — you cannot show a "soft," temporarily-wrong account balance. A "like count" on a social media post can be BASE — it's fine if it takes a few seconds for a like to be reflected everywhere, in exchange for the system never going down.


18. EVENTUAL CONSISTENCY

Eventual consistency is a specific, weaker consistency model where, if no new updates are made to a piece of data, all replicas will eventually converge to the same value — but there is no guarantee about HOW LONG "eventually" takes, and reads during that window may return stale/different data from different replicas.

Why systems choose it: it allows for much higher availability and lower latency, because a node can respond immediately using its local copy of data without needing to coordinate with (and wait for) other replicas first.

Mechanisms used to achieve convergence: 
- Anti-entropy protocols (background comparison and repair of replicas)
- Read repair (fixing stale replicas when detected during a read)
- Gossip protocols (Phase 7) to propagate updates
- Conflict resolution strategies for when two replicas received different concurrent writes: "last write wins" (using timestamps), vector clocks (Phase 7), or application-specific merge logic (e.g., CRDTs)

Example: DNS is a classic real-world example of eventual consistency — when you update a DNS record, it can take minutes to hours to propagate to all DNS servers globally; during that window, different users querying different DNS servers may get different (old vs new) answers. Amazon's shopping cart (historically, via DynamoDB) is another classic example — items you add might briefly not appear if you hit a different replica, but they always show up eventually.


19. STRONG CONSISTENCY

Strong consistency guarantees that once a write is acknowledged/completed, any subsequent read (from any client, on any replica) will see that write immediately — there is no window of staleness.

This typically requires coordination between nodes before acknowledging a write (e.g., waiting for a quorum of replicas to confirm, or routing all reads through a single leader/primary).

Trade-offs: strong consistency generally costs you latency (you have to wait for coordination) and can cost you availability during a network partition (per CAP — a CP system will refuse requests rather than break the consistency guarantee).

How it's implemented in practice:
- Single-leader replication where all reads AND writes go to the leader
- Quorum-based reads/writes (e.g., in Cassandra, you can force QUORUM consistency level to get strong-consistency-like guarantees at the cost of latency)
- Consensus protocols like Raft or Paxos (Phase 7) to agree on the order of operations across nodes

Example: When you check your bank balance right after a transaction, you expect strong consistency — you should never see your old balance on one screen and a debit notification on another. Compare this to a "views count" on a video, which can be eventually consistent without anyone noticing or caring.

Strong vs Eventual — the interview framing: this is fundamentally a business/product decision disguised as a technical one. Ask "what happens if a user briefly sees stale data here — is that a minor annoyance or a serious problem (money, safety, legal)?" The answer tells you which consistency model to pick.


20. DISTRIBUTED SYSTEMS BASICS

A distributed system is a collection of independent computers (nodes) that appear to their users as a single coherent system, working together over a network to achieve a common goal.

Why we build distributed systems at all (instead of one giant machine): scalability beyond a single machine's limits, fault tolerance (no single point of failure), geographic distribution (serve users close to them, lower latency), and cost-effectiveness (many cheap commodity machines vs one extremely expensive supercomputer).

Core challenges that make distributed systems hard — these are the recurring themes of Phases 2 through 17:
- Network is unreliable: messages can be delayed, dropped, duplicated, or arrive out of order. You cannot assume a message you sent was received.
- No global clock: different machines' clocks drift; you cannot simply compare timestamps across nodes to determine "what happened first" (this motivates logical/vector clocks in Phase 7).
- Partial failures: in a single machine, either the whole thing works or it crashes. In a distributed system, SOME nodes can fail while others keep working — and it's often hard to tell the difference between "a node is slow" and "a node is dead."
- Concurrency: multiple nodes may try to read/modify shared data at the same time, requiring coordination (locks, consensus).
- Consistency vs Availability trade-offs: as covered in CAP/PACELC above.

The "Fallacies of Distributed Computing" (a famous list, good to know for interviews): assumptions engineers wrongly make, such as "the network is reliable," "latency is zero," "bandwidth is infinite," "the network is secure," "topology doesn't change," "there is one administrator," "transport cost is zero," and "the network is homogeneous." Every distributed system design should be built assuming these are all FALSE.

Example: Designing a distributed system to store user sessions across 5 servers means confronting: what happens if the network between two servers is temporarily cut (partition)? What happens if a server appears slow — do you wait or assume it's dead and fail over? What happens if two servers both try to update the same session at once? These exact questions are what Phases 2–17 teach you how to answer.


END OF PHASE 1

Quick self-check before moving to Phase 2 (Networking): you should be able to explain, without notes, the difference between availability and reliability, why CAP forces a choice between C and A (not B), when you'd pick ACID over BASE, and why "the network is reliable" is a dangerous assumption. If any of these feel shaky, re-read that section before continuing — Phase 2 builds directly on top of this vocabulary.


PHASE 2: NETWORKING IN SYSTEM DESIGN

Phase 1 gave you the vocabulary (latency, availability, consistency). Phase 2 is about the actual plumbing that connects clients to servers and servers to each other. Every system you design later — caches, databases, microservices — sits on top of this networking layer, so understanding it conceptually (not the OS-level packet details, just how and why each piece is used) is essential.


1. CLIENT-SERVER ARCHITECTURE

Client-server is the foundational architecture pattern where a "client" (browser, mobile app, another service) sends a request, and a "server" processes it and sends back a response. The client initiates; the server responds.

Key characteristics:
- Clients and servers are decoupled — they can be built with different technologies as long as they agree on a communication protocol (e.g., HTTP + JSON)
- One server can serve many clients concurrently
- Servers are usually stateless or externalize their state (to a database/cache) so that any server instance can handle any client's request — this is what makes horizontal scaling (Phase 1) possible

Variants worth knowing:
- Two-tier: client talks directly to a database
- Three-tier: client → application server → database (the most common web architecture)
- N-tier / microservices: client → API gateway → many backend services → multiple databases

Example: When you open a website, your browser (client) sends an HTTP request to a web server, which processes the request (maybe querying a database) and sends back an HTML response — classic client-server interaction.


2. HTTP (HyperText Transfer Protocol)

HTTP is the application-layer protocol that defines how clients and servers exchange messages on the web. It is a request-response protocol: the client sends a request (method + URL + headers + optional body), and the server replies with a response (status code + headers + optional body).

Key HTTP methods:
- GET: retrieve a resource (safe, no side effects, cacheable)
- POST: create a resource / submit data (not idempotent)
- PUT: replace a resource entirely (idempotent)
- PATCH: partially update a resource
- DELETE: remove a resource (idempotent)

Key status code ranges:
- 2xx: success (200 OK, 201 Created, 204 No Content)
- 3xx: redirection (301 Moved Permanently, 304 Not Modified)
- 4xx: client error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests)
- 5xx: server error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout)

HTTP is stateless: each request is independent; the server does not automatically remember previous requests from the same client. This is why cookies/sessions/tokens exist — to layer statefulness on top of a stateless protocol when needed (covered more in Phase 11: Security).

Example: A REST API returning 404 when you request a user ID that doesn't exist, or 429 when you've hit a rate limit (Phase 11), are both everyday uses of HTTP status codes to communicate outcome without needing a special response body format.


3. HTTPS

HTTPS is HTTP layered on top of TLS/SSL (topic 14 below) — it encrypts the data exchanged between client and server so that a third party intercepting the traffic cannot read or tamper with it.

Why it matters: without HTTPS, anyone on the network path (public WiFi, ISP, an attacker doing a man-in-the-middle attack) can read passwords, session tokens, and personal data in plain text, or inject malicious content into the response.

What HTTPS adds over HTTP:
- Encryption: data is unreadable to eavesdroppers
- Integrity: data cannot be silently modified in transit without detection
- Authentication: the client can verify it's actually talking to the real server (via certificates), not an impersonator

Practical note for system design: HTTPS termination (decrypting incoming HTTPS traffic) is often done at the load balancer or reverse proxy (topics 12/13 below), so internal traffic between your own services can run as plain HTTP inside a trusted private network — a common design decision to reduce CPU overhead on every backend service.


4. HTTP/2

HTTP/2 is a major revision of HTTP designed to fix performance problems in HTTP/1.1, while keeping the same semantics (same methods, status codes, headers) — it's a transport-level upgrade, not a new API model.

Key improvements over HTTP/1.1:
- Multiplexing: multiple requests/responses can be sent over a single TCP connection simultaneously, instead of HTTP/1.1's one-request-at-a-time-per-connection (or limited parallel connections) — this eliminates "head-of-line blocking" at the HTTP layer
- Header compression (HPACK): reduces overhead from repetitive headers sent with every request
- Server push: server can proactively send resources the client will likely need, without waiting for a request (though this is used sparingly in practice)
- Binary framing: HTTP/2 uses a binary protocol instead of HTTP/1.1's plain text, which is more efficient to parse

Example: A web page with 50 images used to require up to 50 separate (or queued) connections/requests under HTTP/1.1's limits; with HTTP/2 multiplexing, all 50 can be fetched in parallel over one connection, significantly speeding up page load.


5. HTTP/3

HTTP/3 is the next revision, which changes the underlying transport protocol from TCP to QUIC (built on UDP).

Why the change: HTTP/2's multiplexing happens at the HTTP layer, but it still runs on a single TCP connection — so if a single TCP packet is lost, TCP blocks ALL streams until that packet is retransmitted (TCP-level head-of-line blocking). QUIC solves this by handling multiple streams independently at the transport layer, so one lost packet only affects its own stream.

Other QUIC/HTTP3 benefits:
- Faster connection setup (combines the transport and TLS handshake into fewer round trips)
- Built-in encryption (QUIC always uses TLS 1.3)
- Better performance on unstable networks (e.g., mobile switching from WiFi to cellular) because QUIC connections are tied to a connection ID, not an IP/port tuple, so they survive network changes

Example: Google, YouTube, and Facebook use HTTP/3 heavily because their traffic is dominated by mobile users on unreliable networks, where QUIC's resilience to packet loss and network switching gives a real, measurable performance improvement.


6. REST (Representational State Transfer)

REST is an architectural style (not a strict protocol) for designing networked APIs, built around the idea of "resources" (nouns, like /users/123) manipulated via standard HTTP methods (verbs, like GET/POST/PUT/DELETE).

Core REST principles:
- Resources are identified by URLs (e.g., /users/123/orders)
- Standard HTTP methods express the action (GET to read, POST to create, etc.)
- Stateless: each request contains all information needed to process it; server holds no client session state between requests
- Representations: a resource can be returned in different formats (usually JSON, sometimes XML) — the "representation" of the underlying resource "state" (hence the name)
- Uses standard HTTP status codes to communicate results

Example REST API design:
- GET /users/123 → fetch user 123
- POST /users → create a new user
- PUT /users/123 → replace user 123's data
- DELETE /users/123 → delete user 123

Why REST is so widely used: it's simple, leverages existing HTTP infrastructure (caching, status codes, load balancers all understand it natively), and is easy for different teams/clients to consume without shared tooling.

Limitations that led to alternatives (GraphQL, gRPC below): REST often causes over-fetching (getting more data than you need) or under-fetching (needing multiple round trips to assemble what you actually need, e.g., fetching a user then separately fetching their orders).


7. GRAPHQL

GraphQL is a query language and runtime for APIs, where the CLIENT specifies exactly what data it needs in a single request, and the server returns exactly that — nothing more, nothing less.

How it differs from REST:
- REST: server defines fixed endpoints and fixed response shapes; client takes what it's given
- GraphQL: single endpoint (typically /graphql); client sends a query describing the exact fields it wants, potentially spanning multiple "resources" in one request

Example: Instead of REST's two round trips (GET /users/123, then GET /users/123/orders), a GraphQL client can send one query asking for the user's name, email, and their last 5 orders' totals — all in a single request, with exactly those fields returned, nothing extra.

Benefits: eliminates over-fetching and under-fetching, strongly typed schema (self-documenting API, tooling can validate queries), great for mobile clients where minimizing round trips and payload size matters.

Trade-offs: more complex server-side implementation (resolvers), harder to cache at the HTTP layer (since it's typically all POST requests to one endpoint, unlike REST's cacheable GET URLs), possible for a client to send an expensive/abusive query that touches too much data (requires query complexity limits).


8. gRPC

gRPC is a high-performance Remote Procedure Call (RPC) framework built by Google, using HTTP/2 as transport and Protocol Buffers (protobuf) as the data serialization format.

How it differs from REST/GraphQL: instead of thinking in terms of "resources" or "queries," gRPC lets you define and call remote FUNCTIONS directly, as if they were local — e.g., a client calls GetUser(id) and it feels like a normal function call, even though it's going over the network to a different service.

Key features:
- Uses Protocol Buffers: a compact, binary serialization format (much smaller and faster to parse than JSON)
- Built on HTTP/2: gets multiplexing, header compression, and streaming for free
- Supports 4 communication patterns: unary (single request/single response, like a normal function call), server streaming, client streaming, and bidirectional streaming
- Strongly typed contracts defined in a .proto file, from which client/server code is auto-generated in many languages

Why it's popular for internal (service-to-service) communication: it's significantly faster and more bandwidth-efficient than JSON-over-REST, which matters a lot in high-throughput microservices architectures (Phase 10) where services call each other constantly.

Trade-off: not human-readable (binary format, harder to debug with plain curl/browser compared to REST+JSON), less ideal for public-facing/browser-facing APIs (browsers don't support gRPC natively without a proxy like gRPC-Web).

Example: Two internal microservices — an Order Service and an Inventory Service — communicating via gRPC to check stock before confirming an order, prioritizing speed and low overhead since this happens on every single order.


9. WEBSOCKETS

WebSockets provide a persistent, full-duplex (two-way) communication channel between client and server over a single long-lived TCP connection — unlike regular HTTP where the client always initiates and the server can only respond.

Why it's needed: standard HTTP is request-response only — the server cannot push data to the client whenever it wants, only in reply to a request. Many real-time applications need the server to proactively send updates the instant something happens.

How it works: the connection starts as a normal HTTP request with an "Upgrade" header, then the connection is upgraded to the WebSocket protocol, after which both client and server can send messages to each other at any time, without the overhead of repeated HTTP headers on every message.

Example use cases: chat applications (Phase 15 — WhatsApp, Discord, Slack), live sports score updates, multiplayer online games (Phase 15), collaborative editing tools (like Google Docs), live stock tickers.

Trade-offs: maintaining many persistent open connections is resource-intensive on the server (each connection consumes memory), requires sticky sessions or a shared state layer (like Redis Pub/Sub) if you have multiple server instances behind a load balancer so messages reach the right connected client.


10. SERVER-SENT EVENTS (SSE)

SSE is a simpler alternative to WebSockets for cases where you ONLY need one-directional communication: server pushes updates to the client, but the client doesn't need to send data back over the same channel.

How it works: the client opens a regular HTTP connection and keeps it open; the server sends a stream of text-based events over time (using the text/event-stream content type), and the browser's built-in EventSource API automatically handles reconnection if the connection drops.

SSE vs WebSockets:

| Aspect | SSE | WebSockets |
|---|---|---|
| Direction | One-way (server to client) | Two-way (full duplex) |
| Protocol | Plain HTTP | Upgraded/custom protocol |
| Browser reconnection | Automatic (built-in) | Manual (you implement it) |
| Data format | Text only | Text or binary |
| Use case | Live feeds, notifications, stock tickers | Chat, gaming, collaborative editing |

Example: A notification bell icon that updates in real-time when you get a new notification is a perfect SSE use case — the server needs to push events to you, but your browser doesn't need to send anything back over that same channel.


11. LONG POLLING

Long polling is a technique to simulate real-time updates using plain HTTP, useful when WebSockets/SSE aren't available or supported.

How it works: the client sends a request to the server; instead of responding immediately, the server HOLDS the request open until either new data becomes available or a timeout is reached. Once the server responds (with data or an empty timeout response), the client immediately sends another request, repeating the cycle.

Regular (short) polling vs long polling:
- Short polling: client asks "any updates?" every few seconds, regardless of whether there's anything new — wastes requests and has an inherent delay
- Long polling: client asks once, and the server holds the connection until it actually has something to say — feels much closer to real-time, with fewer wasted requests

Long polling vs WebSockets: long polling is simpler to implement (works over plain HTTP, no protocol upgrade needed, works through more restrictive proxies/firewalls) but has more overhead (repeated HTTP headers, connection setup) compared to a single persistent WebSocket connection. It's often used as a fallback when WebSockets aren't available.

Example: Older chat applications and some notification systems used long polling before WebSockets became widely supported across browsers and infrastructure.


12. DNS (Domain Name System)

DNS is the system that translates human-readable domain names (e.g., www.example.com) into machine-readable IP addresses (e.g., 93.184.216.34), which is what's actually needed to route network traffic.

How DNS resolution works (simplified chain):
1. Browser checks its local cache for the domain
2. If not cached, it asks a DNS Resolver (usually provided by your ISP or a public resolver like Google's 8.8.8.8)
3. The resolver queries a Root DNS Server, which points to the correct Top-Level Domain (TLD) Server (e.g., for .com)
4. The TLD server points to the Authoritative Name Server for that specific domain
5. The authoritative server returns the actual IP address
6. The result is cached at multiple levels (browser, OS, resolver) for a duration defined by TTL (Time To Live), so future lookups are fast

Relevant DNS record types: A (maps domain to IPv4 address), AAAA (IPv6), CNAME (alias — maps one domain to another domain name), MX (mail server), TXT (arbitrary text, often used for verification).

Why DNS matters in system design: DNS is also used for load balancing and failover at a coarse level — e.g., DNS-based load balancing can return different server IPs to different users (geographic routing), or a health-check-aware DNS provider can stop returning the IP of a data center that's down (disaster recovery, Phase 12).

Example: When multiple data centers exist in different regions (US, EU, Asia), DNS can be configured to route a user's request to whichever regional data center is geographically closest to them, reducing latency.


13. REVERSE PROXY

A reverse proxy sits in front of one or more backend servers and intercepts client requests, forwarding them to the appropriate backend server, then returning the backend's response back to the client — the client only ever talks to the reverse proxy, never directly to the real servers.

What reverse proxies are used for:
- Load balancing (distributing requests across multiple backend servers — see topic 15)
- SSL/TLS termination (decrypting HTTPS traffic once at the proxy, so backend servers can run plain HTTP internally)
- Caching frequently requested responses
- Compression of responses before sending to the client
- Hiding internal server architecture/IP addresses from the outside world (security benefit)
- Request routing (sending /api/* to one service, /static/* to another)

Common tools: Nginx, HAProxy, Envoy, AWS Application Load Balancer.

Example: A company runs 10 backend servers behind Nginx configured as a reverse proxy; users only ever see and connect to a single public domain/IP, while Nginx quietly distributes and manages traffic to whichever backend servers are healthy.


14. FORWARD PROXY

A forward proxy sits in front of CLIENTS (not servers) and forwards their requests out to the internet on their behalf — the destination server sees the proxy's identity, not the actual client's.

Forward Proxy vs Reverse Proxy — the key distinction:
- Forward proxy: acts on behalf of the CLIENT, hiding the client from the server. ("I am one of many clients, going out through a shared gateway.")
- Reverse proxy: acts on behalf of the SERVER, hiding the server from the client. ("I am one of many servers, being accessed through a shared gateway.")

What forward proxies are used for: bypassing geographic content restrictions, corporate network content filtering/monitoring (companies routing all employee traffic through a proxy for security/compliance), anonymizing client identity (VPNs are conceptually related), caching for a group of clients to reduce redundant external requests.

Example: A company might route all employee internet traffic through a forward proxy to block access to certain websites and log outbound traffic for security auditing — the external websites see requests coming from the company's proxy, not each individual employee's machine.


15. TLS/SSL

TLS (Transport Layer Security) — the modern name; SSL (Secure Sockets Layer) is its older, now-deprecated predecessor, but the term "SSL" is still used colloquially to refer to TLS.

TLS provides three guarantees for data in transit: encryption (data can't be read by eavesdroppers), integrity (data can't be tampered with undetected), authentication (you can verify you're talking to the legitimate server, via digital certificates).

Simplified TLS handshake process:
1. Client says hello, listing supported encryption methods (cipher suites)
2. Server responds with its chosen cipher suite and its digital certificate (which contains its public key, issued/signed by a trusted Certificate Authority)
3. Client verifies the certificate is valid and trusted
4. Client and server use asymmetric encryption (public/private key pair) briefly to securely agree on a shared symmetric key
5. From that point on, all communication uses fast symmetric encryption with that shared key

Why the switch from asymmetric to symmetric mid-handshake: asymmetric encryption (public/private key) is secure but computationally expensive; symmetric encryption (same key both ways) is much faster — so TLS uses asymmetric encryption only briefly, to safely exchange a symmetric key, then uses the fast symmetric method for the actual data.

Where this connects to system design: TLS termination (decrypting the traffic) is a real architectural decision — do you terminate TLS at the load balancer/reverse proxy (simpler, less CPU load on every backend server) or pass it all the way through to each backend service (higher security, "TLS everywhere," but more overhead)? Large systems often do both: TLS terminates at the edge, then a separate internal TLS layer re-encrypts traffic between internal services (mutual TLS / mTLS) for zero-trust security.


16. LOAD BALANCER

A load balancer distributes incoming network traffic across multiple backend servers, so no single server becomes a bottleneck or a single point of failure, and to enable horizontal scaling (Phase 1).

Where load balancers operate (OSI layers):
- Layer 4 (Transport layer) load balancing: routes traffic based on IP address and port only, without inspecting the actual content of the request — faster, less flexible
- Layer 7 (Application layer) load balancing: inspects the actual HTTP request (URL path, headers, cookies) and can route based on that — e.g., /api/* to one service, /images/* to another — slower per-request but much more flexible

Common load balancing algorithms:
- Round Robin: requests distributed sequentially across servers, one after another
- Least Connections: send the next request to whichever server currently has the fewest active connections
- Weighted Round Robin / Weighted Least Connections: same as above, but some servers get proportionally more traffic (e.g., a more powerful server gets a higher weight)
- IP Hash: route based on a hash of the client's IP, so the same client generally reaches the same backend server (a simple way to get "sticky sessions")
- Least Response Time: route to the server currently responding fastest

Health checks: load balancers continuously ping backend servers; if a server fails a health check, it's automatically removed from rotation until it recovers — a key mechanism for fault tolerance (Phase 1).

Where load balancers sit in an architecture: at the edge (in front of web/app servers, as covered here), but also often in front of databases (routing reads to replicas) and between internal microservices (client-side or sidecar load balancing, covered more in Phase 10: Service Mesh).

Example: A load balancer in front of 5 web servers, using Round Robin, sends request 1 to server A, request 2 to server B, and so on — and if server C fails a health check, it's automatically skipped until it comes back online.


17. API GATEWAY

An API Gateway is a single entry point that sits in front of a collection of backend services (especially in a microservices architecture — Phase 10) and handles cross-cutting concerns before routing requests to the appropriate internal service.

What an API Gateway typically handles:
- Request routing (directing /users/* to the User Service, /orders/* to the Order Service, etc.)
- Authentication and authorization (verifying tokens once, at the edge, instead of every service reimplementing it)
- Rate limiting (Phase 11) — throttling how many requests a client can make
- Request/response transformation (e.g., aggregating responses from multiple services into one response for the client)
- Load balancing across instances of each backend service
- Logging, monitoring, and analytics for all incoming traffic in one place
- Caching common responses

API Gateway vs plain Reverse Proxy: a reverse proxy is a general-purpose traffic router; an API Gateway is a specialized reverse proxy built specifically for managing APIs, with built-in support for the concerns listed above (auth, rate limiting, transformation) rather than just routing/load balancing.

Example: In a microservices e-commerce app, a mobile client makes one call to the API Gateway for "get order details," and the gateway internally calls the Order Service, the Inventory Service, and the Shipping Service, aggregates their responses into a single JSON payload, and returns it — the mobile client never needs to know these are 3 separate backend services.

(Note: API Gateway also appears again in Phase 10 — Microservices — because it's central to that architecture; this Phase 2 entry covers it purely as a networking/routing concept.)


18. CDN (Content Delivery Network)

A CDN is a globally distributed network of caching servers (called "edge servers" or "points of presence," POPs) that store copies of static (and sometimes dynamic) content close to end users geographically, so content is served from a nearby location instead of a far-away origin server.

Why it matters: physical distance directly impacts latency (data can only travel so fast, bounded by the speed of light through fiber). A user in India requesting content from a server in the US will always have higher latency than requesting the same content from a CDN edge server located in India.

What CDNs typically cache/serve: static assets (images, videos, CSS, JS files), and increasingly, dynamic content via edge computing capabilities.

How CDN caching works: when a user requests content, the request is routed (often via DNS) to the nearest edge server. If that edge server has the content cached, it serves it directly (a "cache hit"). If not, it fetches it from the origin server, caches it, and then serves it (a "cache miss," slower the first time, fast for subsequent requests from other nearby users).

Benefits: dramatically reduced latency for users, reduced load on the origin server (most requests never even reach it), improved availability/resilience (if the origin goes down temporarily, cached content can often still be served), protection against traffic spikes and some DDoS attacks (Phase 11).

Example: Netflix and YouTube (Phase 15) rely heavily on CDNs to serve video content — without a CDN, every video stream would have to travel all the way from a centralized data center to every viewer worldwide, which would be both slow and prohibitively expensive in bandwidth.


19. SERVICE DISCOVERY

Service discovery is the mechanism by which services in a distributed system (especially microservices, Phase 10) automatically find the network location (IP address and port) of other services they need to talk to, without hardcoding those addresses.

Why it's needed: in modern cloud/container environments (Phase 12), service instances are created and destroyed dynamically (auto-scaling, deployments, crashes and restarts), so their IP addresses constantly change — hardcoding IPs would break constantly.

Two main patterns:
- Client-side discovery: the client (or a library within it) queries a service registry directly to find available instances of a service, then picks one itself (possibly with client-side load balancing). Example: Netflix Eureka.
- Server-side discovery: the client sends a request to a load balancer/router, which itself queries the service registry and forwards the request to an appropriate instance — the client doesn't need to know about the registry at all. Example: AWS ELB combined with ECS/Kubernetes service discovery.

Service Registry: the central database that keeps track of which service instances are currently alive and where they're located. Services register themselves on startup (and deregister on shutdown), often with periodic heartbeats to prove they're still alive. Tools: Consul, etcd, ZooKeeper, or built-in mechanisms in Kubernetes.

Example: In a Kubernetes cluster, when the Order Service needs to call the Inventory Service, it doesn't need to know any specific pod's IP address — it simply calls a stable service name (like inventory-service), and Kubernetes' internal DNS-based service discovery resolves that name to a currently healthy pod, even as pods are constantly being created and destroyed by auto-scaling.


END OF PHASE 2

Quick self-check before moving to Phase 3 (Storage Systems): you should be able to explain why HTTP is stateless and why that matters, when you'd choose gRPC over REST, the real difference between a forward proxy and a reverse proxy, why a CDN reduces latency specifically (not just "it caches stuff"), and how a load balancer decides which server gets the next request. If any of these feel shaky, re-read that section — Phase 3 (databases) assumes you understand how requests physically arrive at a server in the first place.


PHASE 3: STORAGE SYSTEMS

Phase 1 gave you the vocabulary (consistency, durability, ACID/BASE). Phase 2 covered how requests physically travel to a server. Phase 3 is about where data actually lives once it arrives — the different types of databases, why each one exists, and the indexing/query mechanics that make them fast (or slow). Almost every system design interview eventually asks "which database would you use here, and why" — this phase gives you the vocabulary to answer that well.


1. SQL DATABASE

A SQL (relational) database stores data in structured tables made up of rows and columns, where each table has a predefined schema (column names and data types fixed in advance). Tables can be related to each other via foreign keys, and data is queried using SQL (Structured Query Language).

Key characteristics:
- Fixed schema: you must define the structure of your data upfront (e.g., a "users" table with columns id, name, email)
- Relationships: tables reference each other (e.g., an "orders" table has a user_id foreign key pointing to the "users" table), avoiding data duplication
- ACID transactions (Phase 1): strong consistency and reliability guarantees, ideal for data where correctness is critical
- Powerful querying: SQL supports complex joins, aggregations, and filters across multiple tables in a single query

Popular SQL databases: PostgreSQL, MySQL, Oracle Database, Microsoft SQL Server, SQLite.

When to use SQL: when your data is structured and relationships between entities matter (e.g., users, orders, products, payments), when you need strong consistency (financial transactions, inventory counts), and when your query patterns are varied and not known fully in advance (SQL's flexibility with joins/filters shines here).

Trade-offs: schema changes (migrations) can be costly on large tables, and traditional SQL databases are harder to horizontally scale (sharding a relational database while preserving joins and ACID guarantees is a genuinely hard problem — covered in Phase 4).

Example: An e-commerce order system — orders, order_items, users, products, and payments tables are naturally relational (an order belongs to a user, contains multiple order_items, each referencing a product) — this is a textbook SQL use case.


2. NOSQL DATABASE

NoSQL ("Not Only SQL") is an umbrella term for databases that don't follow the traditional relational model — they typically favor flexible schemas, horizontal scalability, and availability over strict relational structure and immediate consistency (leaning toward BASE properties from Phase 1, though this varies by database).

Why NoSQL emerged: as internet-scale companies (Google, Amazon, Facebook) hit the limits of scaling traditional relational databases horizontally, they built purpose-specific data stores optimized for their exact access patterns, sacrificing general-purpose flexibility (joins, strict schemas) for massive scale and availability.

NoSQL is not one thing — it's a category containing several very different data models, each covered individually below: Key-Value Stores, Document Databases, Column (wide-column) Databases, Graph Databases, and Time Series Databases.

SQL vs NoSQL comparison:

| Aspect | SQL | NoSQL |
|---|---|---|
| Schema | Fixed, defined upfront | Flexible / dynamic |
| Scaling | Primarily vertical (harder to shard) | Designed for horizontal scaling |
| Relationships | Strong (joins, foreign keys) | Weak or denormalized |
| Consistency | Strong (ACID) | Often eventual (BASE), tunable in some |
| Query language | SQL (standardized) | Varies per database |
| Best for | Structured, relational data | High-scale, flexible, high-velocity data |

Important nuance for interviews: this is not "NoSQL is better/newer/faster than SQL" — it is a trade-off decision based on your data's shape and your system's access patterns and scale requirements. Many real production systems use both (polyglot persistence): SQL for the transactional core, NoSQL for specific high-scale needs like session storage or activity feeds.


3. KEY-VALUE STORE

A key-value store is the simplest NoSQL data model: data is stored as a collection of key-value pairs, where the key is a unique identifier and the value is an opaque blob (string, JSON, binary) that the database doesn't need to understand the internal structure of.

How it works: you can only look up data by its exact key (GET key), and write/update it (SET key value) — there is generally no querying by value content, no joins, no complex filtering. This simplicity is exactly what makes it extremely fast.

Popular key-value stores: Redis, Amazon DynamoDB, Riak, Memcached (technically a cache, but conceptually key-value).

When to use it: when access is almost always "give me the value for this exact key" — session storage, caching (Phase 5), shopping carts, user preferences, leaderboards (with some databases adding specialized data structures on top, like Redis's sorted sets).

Trade-off: extremely fast and simple, but very limited querying — if you later need to search/filter by something other than the key, a key-value store is the wrong tool.

Example: Storing a user's session data as session:abc123 → {"user_id": 42, "logged_in_at": "..."} in Redis — you'll only ever fetch this by the exact session ID, never by searching "find all sessions where user_id > 40."


4. DOCUMENT DATABASE

A document database stores data as self-contained "documents" (usually JSON or BSON), where each document can have a different structure/fields even within the same collection — there's no rigid schema forcing every record to look identical.

How it works: related data is often embedded within a single document instead of being split across multiple linked tables (denormalization) — e.g., a "user" document might directly contain an array of their recent orders, instead of requiring a join to a separate orders table.

Popular document databases: MongoDB, Couchbase, Amazon DocumentDB, Firestore.

When to use it: when your data is naturally hierarchical/nested and usually accessed as a whole unit (e.g., a blog post with embedded comments, a product catalog entry with varying attributes per product category), and when your schema is likely to evolve frequently (adding new fields without a formal migration).

Trade-offs: joins across documents/collections are either unsupported or much less efficient than in SQL, so you must think carefully about what to embed vs what to reference; denormalization can lead to data duplication and consistency challenges when the same data appears in multiple documents.

Example: A content management system storing blog posts — each post document might contain the title, body, author info, tags, and an embedded array of comments, all fetched in one single read, no joins needed.


5. COLUMN DATABASE

A column (wide-column) database stores data by column rather than by row, and organizes it into column families — this is different from both relational tables and document stores; it's optimized specifically for very high write throughput and queries that scan large amounts of data across specific columns.

Row-oriented vs column-oriented storage: a traditional SQL database stores each row's data contiguously on disk (fast for "get everything about one record"); a column database stores each column's data contiguously (fast for "scan this one column across millions of records," and highly compressible since similar data is stored together).

Popular column databases: Apache Cassandra, HBase, Google Bigtable, ScyllaDB.

When to use it: extremely high write volumes with wide, sparse data (many possible columns, but each row only populates a few), time-series-like or event data at massive scale, systems that need to stay available and fast even across multiple data centers (many column databases, like Cassandra, are designed as AP systems per CAP theorem, from Phase 1).

Trade-offs: query flexibility is limited — you generally need to know your access patterns in advance and design your "table" (column family) around them, rather than querying ad hoc like SQL; no real joins.

Example: Cassandra is famously used for systems like Netflix's viewing history or messaging systems' message stores, where writes happen at enormous scale continuously and the query pattern (e.g., "get all messages for this conversation, ordered by time") is known in advance and can be optimized for directly.


6. GRAPH DATABASE

A graph database stores data as nodes (entities) and edges (relationships between entities), explicitly optimized for traversing and querying relationships — something relational databases can do via joins, but which becomes very slow as the number of "hops" (levels of relationship) increases.

How it works: instead of storing a relationship as a foreign key requiring a join at query time, a graph database stores the relationship itself as a first-class object (an edge), often with its own properties (e.g., a "FOLLOWS" edge might have a "since" date) — traversing from node to node along edges is extremely fast because the connections are stored directly, not computed via search.

Popular graph databases: Neo4j, Amazon Neptune, ArangoDB.

When to use it: social networks (who follows whom, mutual friends, degrees of separation), recommendation engines (people who bought X also bought Y, based on shared connections), fraud detection (finding rings of connected suspicious accounts), knowledge graphs.

Trade-offs: not designed for simple aggregate queries or high-volume transactional writes the way SQL/column stores are; a specialized tool for a specific access pattern (relationship traversal), not a general-purpose database.

Example: "Find all friends-of-friends who are not already my friend, and suggest them as connections" (a classic friend-suggestion feature) requires traversing 2 hops of relationships — trivial and fast in a graph database, but requires expensive multi-way joins in a relational database that get slower with each additional hop.


7. TIME SERIES DATABASE

A time series database (TSDB) is optimized specifically for data points indexed by time — metrics, sensor readings, logs, events — where data is written continuously (often at very high volume) and queried primarily by time range (e.g., "show me CPU usage for the last hour").

Why a specialized database exists for this: time-series data has unique characteristics — writes are mostly append-only (new data points, rarely updates to old ones), queries almost always involve time-range filters and aggregations (avg/min/max over a window), and old data is often "rolled up" (downsampled) or deleted after a retention period — general-purpose databases don't optimize for these patterns as well.

Popular time series databases: InfluxDB, TimescaleDB (a PostgreSQL extension), Prometheus (used heavily for metrics, see Phase 13: Observability).

Key features TSDBs provide: efficient compression (consecutive time-stamped values are often similar, compressing very well), automatic data retention/downsampling policies, and built-in time-based aggregation functions.

When to use it: application performance monitoring metrics (Phase 13), IoT sensor data (Phase 15/17), financial market tick data, any data where "what happened over time" is the primary access pattern.

Example: A monitoring system collecting CPU/memory/network metrics from thousands of servers every few seconds needs a time series database to efficiently store this ever-growing stream of timestamped data and answer queries like "what was the average CPU usage across all servers between 2pm and 3pm yesterday" without scanning the entire dataset.


8. DATABASE INDEXING

A database index is a separate data structure that stores a sorted (or otherwise organized) reference to the data in a table, allowing the database to find specific rows much faster than scanning the entire table (a "full table scan") for every query.

Analogy: an index in a database works exactly like an index at the back of a textbook — instead of reading every page to find mentions of "photosynthesis," you look it up in the index, which tells you exactly which page(s) to jump to.

How it works internally: most indexes are implemented as a B-Tree (or B+ Tree) data structure — a balanced tree structure that keeps data sorted and allows searches, insertions, and deletions in logarithmic time (O(log n)) instead of linear time (O(n)) for a full scan. Some databases also support hash indexes (very fast for exact-match lookups, but can't support range queries) and specialized indexes for full-text search (Phase 9) or geospatial data.

The core trade-off of indexing: indexes dramatically speed up READS (specifically, queries that filter/sort on the indexed column), but they slow down WRITES (every INSERT/UPDATE/DELETE must also update every index on that table) and consume additional storage space. This is why you shouldn't blindly index every column — you index columns that are actually queried/filtered/sorted on frequently.

Example: A "users" table with 10 million rows and a query like SELECT * FROM users WHERE email = 'x@example.com' would require scanning all 10 million rows without an index on email; with an index on email, the database can find the matching row almost instantly using the B-Tree structure.


9. COMPOSITE INDEX

A composite index (also called a compound index) is an index built on TWO OR MORE columns together, rather than a single column — useful when queries frequently filter or sort by a combination of columns.

How column order matters: a composite index on (last_name, first_name) is efficient for queries filtering by last_name alone, or by last_name AND first_name together — but it is generally NOT useful for queries filtering by first_name alone, because the index is sorted primarily by last_name first. This is often called the "leftmost prefix rule."

Why not just create separate single-column indexes on each column: the database can sometimes combine two single-column indexes, but a well-designed composite index is usually far more efficient for queries that consistently filter on the same combination of columns, because it directly narrows down the search in one structure rather than intersecting two separate result sets.

Example: An orders table frequently queried with WHERE user_id = ? AND status = ? (e.g., "get all pending orders for this user") benefits from a composite index on (user_id, status) — far more efficient than two separate indexes on user_id and status individually for this specific, common query pattern.


10. CLUSTERED INDEX

A clustered index determines the PHYSICAL order in which data rows are actually stored on disk — the table data itself is organized according to the clustered index's key. Because of this, a table can have at most ONE clustered index (data can only be physically sorted one way).

How it works: when you query using the clustered index key, the database can go directly to the physical location of the data, since the index IS essentially the table's storage order — there's no separate lookup step needed.

Common default: in many databases (like SQL Server, MySQL's InnoDB engine), the primary key automatically becomes the clustered index unless specified otherwise, meaning rows are physically stored in primary-key order on disk.

Example: In MySQL's InnoDB engine, a table with an auto-incrementing integer primary key id will have its rows physically stored on disk in ascending id order — querying by a range of IDs (WHERE id BETWEEN 100 AND 200) is very efficient because those rows are physically adjacent on disk.


11. NON-CLUSTERED INDEX

A non-clustered index is a SEPARATE data structure from the actual table data — it stores the indexed column's values along with a pointer (reference) back to the actual row's location, rather than physically reordering the table itself. A table can have MANY non-clustered indexes (unlike the single clustered index).

How it works: querying via a non-clustered index requires an extra step compared to a clustered index — first, the database looks up the value in the index structure to find the pointer/reference, then it follows that pointer to fetch the actual row data (sometimes called a "bookmark lookup" or "key lookup") — slightly slower than a clustered index lookup, but still vastly faster than a full table scan.

Clustered vs Non-Clustered comparison:

| Aspect | Clustered Index | Non-Clustered Index |
|---|---|---|
| Physical data order | Determines it | Independent of it |
| How many per table | Only 1 | Many |
| Lookup speed | Faster (direct) | Slightly slower (extra pointer lookup) |
| Typical use | Primary key | Frequently filtered/sorted columns other than the primary key |

Example: On the same "users" table with a clustered index on id, you might additionally create a non-clustered index on email (since users frequently log in via email) — looking up by email uses the non-clustered index to find a pointer, then follows it to the actual row.


12. QUERY OPTIMIZATION

Query optimization is the process (largely automatic, performed by the database's query optimizer, but also something engineers actively do) of finding the most efficient way to execute a given query — deciding which indexes to use, what order to join tables in, and which operations to perform first, to minimize the total work done.

How the database's built-in query optimizer works: when you submit a SQL query, the database doesn't execute it literally step by step as written — it first analyzes multiple possible "execution plans" (different strategies to get the same correct result) and estimates the cost of each (based on table sizes, existing indexes, and statistics about the data distribution), then picks the plan it estimates will be cheapest.

Practical query optimization techniques engineers use:
- Add appropriate indexes on frequently filtered/joined/sorted columns (topic 8/9/10/11 above)
- Avoid SELECT * — only fetch the columns you actually need, reducing data transfer
- Avoid functions on indexed columns in WHERE clauses (e.g., WHERE YEAR(created_at) = 2024 often can't use an index on created_at efficiently, while WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01' can)
- Be mindful of join order and join types, especially on large tables
- Use LIMIT/pagination for large result sets instead of fetching everything
- Keep database statistics up to date so the optimizer's cost estimates are accurate

Example: A slow query filtering millions of rows by an unindexed column might take several seconds; adding the right index (and confirming via the execution plan, topic 13 below, that it's actually being used) can often bring that down to milliseconds.


13. EXECUTION PLANS

An execution plan (also called a query plan) is the actual step-by-step strategy the database's query optimizer decided to use to execute a specific query — and critically, it's something engineers can and should inspect to understand (and fix) why a query is slow.

How to view one: most databases provide a command to show the execution plan without actually running the full query — e.g., EXPLAIN in MySQL/PostgreSQL, or EXPLAIN ANALYZE (which actually runs the query and shows real timing alongside the plan).

What to look for in an execution plan:
- Whether the query is using an index ("Index Scan"/"Index Seek") or doing a full table scan ("Seq Scan"/"Table Scan") — a full scan on a large table is usually a red flag
- The estimated (or actual, with ANALYZE) number of rows processed at each step
- The join strategy used (nested loop, hash join, merge join) and whether it's appropriate for the data sizes involved
- Where the most "cost" or time is being spent, so you know exactly which part of the query to optimize

Why this matters in system design interviews: being able to say "I'd check the execution plan to confirm the index is actually being used, rather than assuming" signals real hands-on database experience, not just theoretical knowledge.

Example: Running EXPLAIN ANALYZE on a slow query might reveal it's doing a "Seq Scan" on a 50-million-row table despite an index existing on the filtered column — this could be because the query used a function on that column (breaking index usage, as in the query optimization example above), telling the engineer exactly what to fix.


END OF PHASE 3

Quick self-check before moving to Phase 4 (Database Scaling): you should be able to explain when you'd reach for a document database over a relational one, why a graph database beats SQL joins for multi-hop relationship queries, the practical difference between a clustered and non-clustered index, and why indexing isn't a free win (the read/write trade-off). If any of these feel shaky, revisit that section — Phase 4 builds directly on these storage concepts by asking "now how do we scale this database once one machine isn't enough?"


PHASE 4: DATABASE SCALING

Phase 3 covered what kind of database to use. Phase 4 covers what happens once a single database instance can no longer handle your load — either because there's too much traffic (needs replication) or too much data (needs sharding), or because your operations now span multiple databases (needs distributed transactions). This is one of the most heavily tested areas in system design interviews, because almost every "design X at scale" question eventually runs into "one database server isn't enough."


1. REPLICATION

Replication is the practice of keeping multiple copies (replicas) of the same data on different servers/nodes, so that data survives a single server failure and read traffic can be spread across multiple machines.

Why replicate:
- Fault tolerance / durability (Phase 1): if one server dies, another copy of the data still exists
- Availability: if one node is down, requests can be served by another replica
- Read scalability: read traffic can be distributed across multiple replicas instead of hammering one server
- Geographic distribution: replicas placed closer to users in different regions reduce latency

The core challenge replication introduces: once you have multiple copies of the same data, you must decide how writes propagate to all copies, and what happens if two copies temporarily disagree — this directly connects back to the CAP theorem and consistency models from Phase 1. The three main replication strategies (below) are different answers to "who is allowed to accept writes, and how do other copies find out about them."


2. LEADER-FOLLOWER REPLICATION

Also called Master-Slave or Primary-Replica replication. One node (the leader/primary) accepts ALL writes. The leader then propagates those changes to one or more follower/replica nodes, which typically only serve reads.

How it works: a client sends a write to the leader; the leader applies it and records it in a replication log; followers continuously pull (or receive pushed) changes from this log and apply them locally to stay in sync.

Synchronous vs Asynchronous replication:
- Synchronous: the leader waits for at least one follower to confirm it received the write before acknowledging success to the client — stronger durability guarantee, but higher write latency
- Asynchronous: the leader acknowledges the write immediately without waiting for followers — lower latency, but risk of data loss if the leader crashes before followers catch up (the write existed only on the leader)
- Semi-synchronous: a middle ground — wait for at least one follower (not all) to confirm, balancing safety and speed

What happens if the leader fails: a follower must be promoted to become the new leader (failover) — this can be done manually or automatically (via a consensus mechanism, Phase 7), but there's a risk of losing very recent writes that hadn't yet replicated to the promoted follower.

Example: A typical PostgreSQL or MySQL setup with one primary handling all writes and 2-3 read replicas serving read traffic — the application routes writes to the primary and can route reads to any replica, reducing load on the primary substantially.


3. MULTI-LEADER REPLICATION

Also called Master-Master replication. MULTIPLE nodes can each accept writes independently (not just one leader), and these nodes replicate their changes to each other.

Why you'd want this: useful in multi-datacenter setups, where each region has its own leader accepting local writes (so local users get fast writes without crossing continents) — writes are then asynchronously replicated between the regional leaders.

The core new problem this introduces: write conflicts. If two different leaders accept different writes to the SAME piece of data at roughly the same time (before they've replicated to each other), which one wins? Multi-leader replication requires a conflict resolution strategy (e.g., last-write-wins based on timestamp, custom merge logic, or techniques like CRDTs — Conflict-free Replicated Data Types).

Trade-off: multi-leader gives you better write availability and lower write latency in each region (no need to cross the ocean for every write), at the cost of significantly increased complexity around conflict detection and resolution — this is a direct CAP/PACELC trade-off (Phase 1) favoring availability/latency over strict consistency.

Example: A global collaborative document editor with users in the US and Europe, where each region has its own leader database to keep local writes fast, and conflicting simultaneous edits to the same document section are reconciled using a merge strategy.


4. LEADERLESS REPLICATION

In leaderless replication, there is no designated leader at all — ANY replica can accept both reads and writes directly from clients, and consistency is achieved through client-side coordination rather than through a single ordering authority.

How it typically works (quorum-based approach): the client (or a coordinator on its behalf) writes to multiple replicas simultaneously (say, W replicas out of N total), and reads from multiple replicas as well (R replicas), then reconciles differences if replicas disagree (e.g., using version numbers or vector clocks — Phase 7 — to determine which value is more recent).

The quorum formula (important for interviews): if W + R > N, you are mathematically guaranteed that every read overlaps with at least one replica that has the most recent write — giving you strong-ish consistency guarantees even without a single leader. This is tunable: you can favor faster writes (lower W) or faster/more consistent reads (higher R), or vice versa.

Popular databases using this model: Amazon DynamoDB (originally), Apache Cassandra, Riak.

Trade-off: no single point of failure for writes (unlike leader-follower, where the leader is briefly a bottleneck/SPOF until failover completes) and very high availability, but read-repair and conflict resolution logic add complexity, and achieving strict consistency requires careful tuning of W and R.

Example: Cassandra allows you to specify a consistency level per query — e.g., QUORUM for both reads and writes gives strong-ish guarantees, while ONE for both favors maximum speed and availability at the cost of potentially reading stale data.


5. SHARDING

Sharding is the practice of splitting a large dataset horizontally across multiple database servers (shards), where each shard holds a SUBSET of the total data (as opposed to replication, where every replica holds a full copy of the SAME data).

Why sharding is necessary: replication solves the "too much traffic" problem (many copies handling reads), but it doesn't solve the "too much data" problem — if your dataset is simply too large to fit on (or be efficiently queried from) a single machine, you must split the data itself across multiple machines. This is the Z-axis of the "scale cube" mentioned in Phase 1.

Core decision in sharding: choosing a shard key (also called a partition key) — the field used to decide which shard a given piece of data belongs to (e.g., sharding a users table by user_id). A well-chosen shard key distributes data (and load) evenly across shards; a poorly chosen one creates "hot shards" that receive disproportionate traffic.

Sharding strategies:
- Range-based sharding: data is split by ranges of the shard key (e.g., user IDs 1-1M on shard A, 1M-2M on shard B). Simple, but can create hot spots if writes cluster at one end of the range (e.g., IDs are sequential and all new users hit the newest shard).
- Hash-based sharding: apply a hash function to the shard key to determine which shard the data goes to, distributing data much more evenly, but making range queries (e.g., "get all users created this month") harder, since sequential data ends up scattered across shards.
- Directory-based sharding: maintain a separate lookup service that explicitly maps each key to its shard — flexible (can rebalance/move data easily) but adds an extra lookup hop and a new potential single point of failure/bottleneck (the lookup service itself).

Consequences of sharding: cross-shard joins and cross-shard transactions become difficult or impossible without special handling (see Distributed Transactions, topic 12 below) — this is the single biggest trade-off engineers accept when sharding a relational database.

Example: A large social media platform might shard its "posts" table by user_id (hash-based), so all of a given user's posts always live on the same shard — making "get this user's posts" fast (single-shard query), while "get the 10 most recent posts across all users globally" becomes much harder (requires querying every shard and merging results).


6. CONSISTENT HASHING

Consistent hashing is a specific hashing technique used to distribute data (or requests) across a set of nodes (shards, cache servers, etc.) in a way that minimizes redistribution when nodes are added or removed — solving a major pain point of naive hash-based sharding.

The problem with naive hashing: if you shard using a simple formula like hash(key) % N (where N is the number of servers), then adding or removing even ONE server changes the value of N, which changes the result of the modulo operation for almost EVERY key — meaning almost all data would need to be moved/rebalanced, which is extremely expensive and disruptive at scale.

How consistent hashing solves this: imagine a circular hash space (a "ring") from 0 to some maximum value. Both the servers AND the data keys are hashed onto this same ring (based on their ID/name and their key, respectively). Each key is assigned to the first server found by moving clockwise around the ring from the key's position. When a server is added or removed, only the keys that fall between that server and its neighbor on the ring need to move — everything else stays exactly where it was.

Virtual nodes: in practice, each physical server is mapped to MULTIPLE points on the ring (virtual nodes), not just one — this ensures a more even distribution of data/load across servers, and means that when one server is removed, the resulting extra load is spread across several other servers rather than dumped entirely onto just one neighbor.

Where it's used: distributed caches (Phase 5 — e.g., Memcached client-side consistent hashing), NoSQL databases like Cassandra and DynamoDB for partitioning data across nodes, CDN request routing (Phase 2).

Example: A distributed cache with 4 servers using consistent hashing needs to add a 5th server; instead of nearly all cached keys needing to move (as with naive modulo hashing), only roughly 1/5th of the keys (the ones that now fall into the new server's range on the ring) need to be relocated — dramatically reducing cache invalidation and data movement during scaling events.


7. PARTITIONING

Partitioning is the general term for dividing data into distinct segments — sharding (topic 5) is one specific form of partitioning (splitting data across multiple SEPARATE servers/machines), but partitioning can also happen WITHIN a single database instance.

Partitioning within a single database (not distributed sharding):
- Horizontal partitioning: splitting a table's ROWS into multiple physical partitions based on some criteria (e.g., splitting an "orders" table into partitions by year: orders_2023, orders_2024) — often done within one database server to improve query performance and manageability (e.g., dropping an entire old partition is much faster than deleting millions of individual rows)
- Vertical partitioning: splitting a table's COLUMNS into separate tables — e.g., separating rarely-accessed large text/blob columns from frequently-accessed small columns, so common queries don't have to read past unnecessary large data

Partitioning vs Sharding — the distinction worth knowing for interviews: sharding is specifically horizontal partitioning that is distributed ACROSS multiple separate servers/nodes; partitioning is the broader concept that can happen even on a single machine. In casual conversation the terms are often used interchangeably, but knowing the precise distinction signals depth.

Example: A time-series-heavy orders table might be horizontally partitioned by month within a single PostgreSQL instance, so queries that only care about "this month's orders" only need to scan one partition, not the entire multi-year table — and old partitions can be archived or dropped entirely, very cheaply, once they're no longer needed.


8. REBALANCING

Rebalancing is the process of redistributing data across shards/nodes after the number of nodes changes (a node is added for more capacity, or removed due to failure/decommissioning) — the goal is to restore an even distribution of data and load.

Why it's needed: as discussed in sharding and consistent hashing above, when you add or remove nodes, some data needs to move to maintain balance — rebalancing is the operational process of actually doing that data movement safely.

Key requirements for good rebalancing:
- Minimize data movement: move only what's necessary (this is exactly what consistent hashing, topic 6, is designed to optimize for)
- Keep the system available during rebalancing: reads/writes should continue to work correctly even while data is actively being moved between nodes (often achieved by temporarily routing to both old and new locations, or using a lookup/directory service to track the current location during the transition)
- Automate it: manually moving data at scale is error-prone; most modern distributed databases (Cassandra, MongoDB, DynamoDB) handle rebalancing automatically in the background

Example: When a new shard is added to a MongoDB sharded cluster, MongoDB's balancer process automatically and gradually migrates chunks of data from existing shards to the new one in the background, without taking the database offline, until data is evenly distributed again.


9. READ REPLICAS

A read replica is a copy of a database used SPECIFICALLY to serve read queries, offloading read traffic away from the primary/leader database, which continues to handle all writes. (This is essentially the practical, applied use of leader-follower replication, topic 2, specifically for scaling reads.)

Why this matters as its own concept: in most real-world applications, reads vastly outnumber writes (often 80-95%+ of traffic is reads) — so scaling read capacity independently, by simply adding more read replicas, is one of the highest-leverage, simplest scaling techniques available before you need to reach for the much more complex option of sharding.

Application-level considerations: the application needs read/write splitting logic — routing INSERT/UPDATE/DELETE statements to the primary, and SELECT statements to a replica (sometimes handled by a proxy/middleware layer instead of the application itself). You must also handle replication lag: since replication is often asynchronous, a read replica might briefly serve slightly stale data compared to the primary — this connects directly back to the consistency model discussion in Phase 1 (a system using read replicas is, in that moment, offering eventual rather than strong consistency for reads).

Example: A read-heavy blog platform might have 1 primary database (handling post creation/edits) and 5 read replicas (handling the far larger volume of people simply reading posts) — dramatically increasing total read throughput without touching the write path at all.


10. WRITE SCALING

Write scaling refers to the set of techniques used to handle a growing volume of WRITE operations — a fundamentally harder problem than read scaling, because unlike reads (which can simply be duplicated across replicas), writes must ultimately be applied consistently, and naively adding more nodes doesn't automatically help unless writes can somehow be distributed.

Key techniques for write scaling:
- Sharding (topic 5): distribute writes across multiple shards based on a shard key, so no single node bears the full write load
- Batching: group multiple writes together into a single operation/transaction to reduce per-write overhead
- Asynchronous writes / write-behind: acknowledge the write quickly (e.g., after writing to a fast queue or cache) and persist it to the durable store slightly later in the background (trading some durability guarantee for speed — connects to Write Back caching in Phase 5)
- Using a database/architecture built for high write throughput (e.g., column stores like Cassandra, topic 5 in Phase 3, are specifically optimized for very high write volumes)
- Write-optimized storage engines: some databases use Log-Structured Merge (LSM) trees instead of B-Trees internally specifically because they handle very high write volumes more efficiently (writes are appended sequentially rather than requiring in-place updates)

Why write scaling is harder than read scaling, in one sentence: reads only need "a" correct copy of the data (any replica will do), but writes need to be durably and (usually) consistently applied to the authoritative copy/copies, which is a fundamentally more constrained problem.

Example: A high-volume IoT sensor platform ingesting millions of data points per second uses sharding (by device ID) combined with a write-optimized time-series database (Phase 3) to distribute and efficiently absorb this write load, rather than trying to funnel everything through a single write-heavy relational database.


11. DATA MIGRATION

Data migration is the process of moving data from one system, schema, or format to another — a routine but high-stakes operational activity that comes up constantly as systems evolve (changing databases, changing schemas, splitting a monolith's database in a microservices migration, etc.).

Common migration scenarios: changing a column's data type, adding a new required field to millions of existing rows, moving from one database technology to another entirely (e.g., MySQL to PostgreSQL, or a monolith's single database to per-service databases in a microservices migration, Phase 10), or re-sharding data onto a new sharding scheme.

Key strategies for safe, low-downtime migrations:
- Dual writes: temporarily write every new piece of data to BOTH the old and new systems simultaneously, while backfilling historical data separately, until you're confident the new system is correct and complete, then cut over reads
- Backfilling: a background process gradually copies/transforms existing historical data from the old system to the new one, often in batches, to avoid overwhelming production traffic
- Expand-contract pattern (a common safe schema-change pattern): first EXPAND the schema (add the new column/table alongside the old one, without removing anything), migrate all reads/writes to use the new structure gradually, then CONTRACT (remove the old, now-unused structure) — this avoids ever having a moment where the schema doesn't support both old and new code simultaneously, which is critical for zero-downtime deployments
- Feature flags: gradually roll out reliance on the new data source to a percentage of traffic, so any issues affect a limited blast radius and can be rolled back quickly

Example: Migrating a "user address" field from a single flat string column to a structured set of columns (street, city, state, zip) without downtime would use the expand-contract pattern — add the new columns, backfill them from the old string field, update the application to write to both then read from the new columns, verify correctness, and only then drop the old string column.


12. DISTRIBUTED TRANSACTIONS

A distributed transaction is a transaction that spans MULTIPLE databases or services, where you need an all-or-nothing (atomic) guarantee across all of them — a direct consequence of sharding (topic 5) and microservices (Phase 10), both of which split data that used to live in one place across multiple independent systems.

Why this is hard: a normal ACID transaction (Phase 1) relies on a single database engine's ability to coordinate locks and rollbacks internally. Once your data lives across multiple independent systems (that may not even use the same database technology), there is no single engine that can atomically guarantee "commit everywhere, or commit nowhere" — you need explicit coordination protocols to achieve something similar, and even then, the guarantees are usually weaker or come at a real performance cost.

This challenge is exactly why the next two topics (Two-Phase Commit and the Saga Pattern) exist — they are the two dominant approaches to handling transactions that span multiple systems, representing a classic strong-consistency-and-lower-availability vs. eventual-consistency-and-higher-availability trade-off (echoing CAP/PACELC from Phase 1).

Example: An e-commerce checkout that needs to simultaneously deduct inventory (in an Inventory Service/database), charge a payment (in a Payment Service), and create an order record (in an Order Service) is a distributed transaction — if the payment succeeds but inventory deduction fails, you have an inconsistent, incorrect state unless you use one of the following coordination strategies.


13. TWO-PHASE COMMIT (2PC)

Two-Phase Commit is a protocol for achieving atomic commits across multiple distributed systems (nodes/databases), coordinated by a single "coordinator" node, using exactly two phases.

How it works:
- Phase 1 (Prepare/Voting phase): the coordinator asks every participant node "can you commit this transaction?" Each participant does whatever preparation is needed (locks resources, writes to a temporary log) and responds "yes, I'm ready" or "no."
- Phase 2 (Commit/Abort phase): if ALL participants voted "yes," the coordinator tells everyone to actually COMMIT. If even ONE participant voted "no" (or failed to respond), the coordinator tells everyone to ABORT/ROLLBACK instead.

Guarantee provided: strong consistency — either every participant commits, or every participant rolls back; there's no partial/inconsistent outcome (in the successful case).

The big weakness: 2PC is a blocking protocol. If the coordinator crashes after participants have "voted yes" (prepared) but before sending the final commit/abort instruction, those participants are stuck holding their locks/resources indefinitely, unable to proceed, until the coordinator recovers — this significantly hurts availability, directly illustrating the CAP theorem trade-off (favoring consistency at a real cost to availability).

When it's still used: scenarios genuinely requiring strong consistency across systems, often within a single, well-controlled infrastructure (e.g., some traditional distributed relational database setups) — but it's largely avoided in large-scale, loosely-coupled microservices architectures because of the availability/blocking risk, which is why the Saga Pattern (topic 14) is generally preferred there.

Example: A traditional distributed database spanning two data centers might use 2PC to ensure a transaction either commits fully in both locations or not at all — but if the network between the two data centers drops right after the "prepare" phase, both sides may be stuck waiting, unable to serve related requests, until connectivity is restored.


14. SAGA PATTERN

The Saga Pattern is an alternative to Two-Phase Commit specifically designed for microservices architectures (Phase 10), where you accept that you CANNOT get true atomicity across services, and instead break a distributed transaction into a sequence of LOCAL transactions, each with a corresponding COMPENSATING action to undo it if a later step fails.

How it works: instead of one big atomic transaction, a saga is a series of individual steps, each committed locally and immediately (not held/locked waiting for other steps) — Step 1 commits in Service A, then Step 2 commits in Service B, and so on. If a later step fails, the saga executes compensating transactions to undo the effects of the earlier steps that already succeeded, working backward.

Two common coordination styles:
- Choreography: there's no central coordinator — each service publishes an event when it completes its local step, and other services listen for relevant events and react accordingly (connects to Event-Driven Architecture, Phase 6). Simple for small sagas, but can become hard to trace/debug as the number of steps grows (no single place to see the whole flow).
- Orchestration: a central "saga orchestrator" service explicitly tells each participant what to do next and handles the compensating logic if something fails — easier to understand, monitor, and debug as a single source of truth for the workflow, at the cost of introducing a new central component.

Saga vs Two-Phase Commit — the key trade-off: Sagas favor availability and are non-blocking (each local step commits immediately, no held locks waiting on a coordinator), but only offer eventual consistency (there's a window of time where the overall operation is partially complete) and are notably harder to design correctly (you must think through and implement a compensating action for every single step, which isn't always trivial or even fully reversible in the real world, e.g., "un-sending" an email confirmation).

Example: Returning to the e-commerce checkout example — using a saga, Step 1 reserves inventory (committed immediately), Step 2 charges payment (committed immediately); if Step 2's payment fails, the saga triggers a compensating transaction for Step 1 (release the reserved inventory) — the system was briefly in an inconsistent state (inventory reserved but no successful payment) but self-corrects, rather than holding locks and blocking other operations the way 2PC would.


END OF PHASE 4

Quick self-check before moving to Phase 5 (Caching): you should be able to explain the practical difference between replication and sharding (and why they solve different problems), why consistent hashing matters more than "just use hash % N," what W + R > N actually guarantees you in leaderless replication, and why most modern microservices architectures prefer the Saga Pattern over Two-Phase Commit despite Sagas only offering eventual consistency. If any of these feel shaky, revisit that section — Phase 5 (Caching) assumes you're comfortable with the idea that data can legitimately be stale for a while, which is exactly what several of these Phase 4 topics already introduced.
