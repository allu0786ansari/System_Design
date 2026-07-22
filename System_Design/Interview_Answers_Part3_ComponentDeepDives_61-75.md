SYSTEM DESIGN INTERVIEW — CANDIDATE ANSWERS
PART 3: SECTION C — COMPONENT-LEVEL AND INFRASTRUCTURE DEEP DIVES (61–75)


61. How would you design the database schema for [whichever product was just discussed], and how would you shard it as it grows?

"I'd start from the core entities and their relationships — for something like the e-commerce system we just discussed, that's users, products, orders, and order_items, with foreign keys tying orders to users and order_items to both orders and products. I'd normalize this initially, since strong relational integrity is genuinely valuable early on, and only denormalize specific hot paths later once I have real evidence a particular join is actually a bottleneck.

For sharding, the critical decision is the shard key, since it determines both data distribution and which queries stay fast versus which become expensive cross-shard operations. I'd shard orders by user_id, since almost every order-related query is naturally scoped to one user — 'show me my orders' becomes a single-shard query. I'd explicitly flag the trade-off: a query like 'show me the 10 most recent orders across all users globally' becomes much harder under this scheme, since it now has to query every shard and merge results — but that's a query pattern I'd expect to be rare and could serve from a separate, purpose-built analytics store instead of forcing my primary transactional schema to support it efficiently."


62. Walk through what happens end-to-end when a client sends a single HTTP request to your system — every hop.

"The client first needs to resolve the domain name to an IP address via DNS, which is often cached at the browser or OS level from a previous lookup. The request then, if it's over HTTPS, goes through a TLS handshake to establish an encrypted connection — that's where the client verifies the server's certificate and the two sides agree on a symmetric key for the actual data exchange.

The request hits a load balancer, which picks a healthy backend instance using something like round robin or least connections, and forwards it — for a Layer 7 load balancer, it might route based on the URL path, sending API calls to one service and static content to a CDN-backed path.

At the application server, middleware typically runs first — authentication, checking a rate limiter, logging the request. The application logic then executes, which might check a cache first before querying the database, and might call out to other internal services if needed.

The response flows back through the same load balancer to the client, and the whole round trip gets logged and its latency recorded as a metric, feeding into the monitoring dashboards I'd have watching this system in production. I'd walk through this end-to-end specifically because it's a good way to show I understand every layer isn't just the application code — DNS, TLS, load balancing, and caching all sit in the critical path too."


63. How would you design a distributed lock, and what happens if the node holding the lock crashes?

"I'd implement this using an external coordination system like Redis or ZooKeeper. A node acquires the lock by writing a unique key, only if that key doesn't already exist — an atomic 'set if not exists' operation — and it holds the lock as long as that key exists. To handle the case where the holding node crashes without releasing it, I'd attach a TTL to the lock key, so it automatically expires after some timeout even if nobody explicitly releases it.

The subtle danger I'd specifically flag is what happens if a node holds the lock, then experiences a long, unexpected pause — a garbage collection pause or a slow network — long enough that the TTL expires while the node still believes it holds the lock and keeps working. Meanwhile, a different node acquires the now-available lock and starts its own work, and now two nodes both believe they have exclusive access at the same time. To guard against that, I'd use fencing tokens — a monotonically increasing number issued with each lock acquisition — and have any downstream system that's protected by the lock reject requests carrying a stale, lower token number than one it's already seen, so even if two nodes briefly both think they hold the lock, only the genuinely current one can actually cause any effect."


64. How would you design a service registry / service discovery mechanism for a microservices system?

"I'd maintain a centralized registry tracking every currently-alive service instance and its network location. Each instance registers itself on startup and sends periodic heartbeats to prove it's still alive; if heartbeats stop, the registry automatically removes that instance after some timeout, without requiring any manual intervention.

For how other services actually find and use this registry, I'd distinguish two patterns: client-side discovery, where the calling service directly queries the registry and picks an instance itself, possibly load balancing client-side; or server-side discovery, where the caller just talks to a load balancer or router, which itself queries the registry on the caller's behalf, meaning the caller doesn't need any registry-aware logic at all.

I'd lean toward server-side discovery for most services, since it keeps application code simpler and centralizes the discovery logic in one well-tested place, and I'd back the registry itself with something like etcd or Consul, which use a consensus protocol internally to stay reliably consistent even if some of their own nodes fail — since if the registry itself is wrong or unavailable, requests start getting routed to dead instances, which defeats the entire purpose."


65. How would you design an event bus or message broker (e.g., "build your own Kafka")?

"I'd split each topic into multiple partitions, distributed across broker machines, with messages sharing the same key — like the same conversation ID or user ID — always routed to the same partition, which guarantees ordering within that partition even though there's no ordering guarantee across different partitions.

Each partition is durably replicated across multiple brokers, so a single broker failing never loses committed data, and an event bus is explicitly expected to retain events for some retention window, letting consumers replay history, not just pass messages through transiently.

For consumers, I'd support consumer groups — multiple consumer processes that split up a topic's partitions among themselves, each partition read by exactly one consumer within a given group at a time, letting me process a topic in parallel while ensuring no two consumers in the same group redundantly process the same partition. Coordinating exactly which consumer owns which partition, especially as consumers join or leave, requires a form of distributed coordination, which I'd build on the same kind of consensus mechanism I'd use for the registry.

For delivery guarantees, I'd default to at-least-once, paired with idempotent consumer logic, since genuinely guaranteeing exactly-once delivery is both harder and more expensive than most systems actually need."


66. How would you design a health-check and auto-recovery system for a fleet of servers?

"I'd have each service expose a lightweight health endpoint, and I'd distinguish liveness from readiness explicitly, since they mean genuinely different things. Liveness answers 'is this process fundamentally still running and not deadlocked' — a failed liveness check should trigger a full restart of that instance. Readiness answers 'is this instance currently able to accept new traffic right now' — a service might be alive but temporarily not ready, like still starting up or briefly unable to reach its database, and a failed readiness check should just pull it out of load balancer rotation temporarily, not kill it.

A central monitoring component, or the load balancer itself, polls these health endpoints on a regular interval; a few consecutive failures, not just one, trigger the actual action, since I want to tolerate a single transient blip without unnecessary restart churn.

For auto-recovery, a failed liveness check triggers an automatic restart, and if a node keeps failing repeatedly even after restarts, I'd escalate — pull it fully out of the fleet and page an engineer, rather than restart-looping it forever, which would just mask a real underlying problem."


67. How would you design multi-region failover for a system that currently runs in one region?

"First I'd clarify the actual RTO and RPO requirements, since that determines which of several genuinely different strategies makes sense, ranging from a fairly cheap 'backup and restore' approach all the way up to a fully active-active multi-region deployment, and the right answer here depends entirely on how much downtime and data loss the business can actually tolerate.

Assuming this needs to be reasonably fast to fail over, I'd set up a warm standby — a scaled-down but fully functional copy of the production environment running continuously in a second region, with data continuously replicated to it. On a regional failure, DNS or a global load balancer redirects traffic to the standby region, and its capacity is scaled up to handle full production load.

The genuinely hard part is data replication across regions — I'd use asynchronous replication to avoid the latency cost of synchronous cross-region writes, which directly means accepting some replication lag, and therefore some risk of losing the most recent few seconds of writes in an actual failover. I'd make that trade-off explicit rather than pretending failover can be instant and lossless, and I'd periodically actually test the failover process itself, since an untested failover plan is one I genuinely wouldn't trust to work when it's actually needed."


68. How would you design a system to safely deploy a new version of a service with zero downtime?

"I'd use a rolling deployment as the default — replacing instances of the old version with the new version a few at a time, behind a load balancer, so there's always a healthy mix of instances serving traffic throughout the rollout, rather than taking the whole service down and back up at once.

For something higher-risk, I'd add a canary step first — deploy the new version to a small percentage of instances, closely monitor error rate and latency for a defined period, and only proceed to a full rolling deployment if the canary looks healthy; if it doesn't, I'd roll back that small percentage immediately, having only affected a small fraction of traffic.

I'd also make sure the new version stays backward compatible with the old version during the transition, particularly for anything involving a shared database schema — using an expand-contract approach for schema changes, adding new columns or tables without removing the old ones until every instance is fully migrated to the new version, so old and new code can safely run side by side during the rollout without either one breaking."


69. How would you design a geospatial index to find "nearby" entities (drivers, restaurants, stores) efficiently?

"I'd use geohashing — encoding latitude and longitude into a string specifically designed so that nearby physical locations share a common prefix. That turns 'find things near this point' into a fast prefix-based lookup rather than a geometric distance calculation against every single entity in the system.

For a concrete implementation, I'd store entities in a sorted structure keyed by their geohash, and to find nearby entities, I'd query for the geohash prefix covering the searcher's location, expanding the prefix search to a coarser, shorter prefix if too few results come back in that immediate cell, since a shorter prefix covers a larger geographic area.

I'd mention quadtrees as an alternative — recursively subdividing the map into smaller regions wherever entity density is high — which adapts more naturally to very uneven density, like drivers heavily clustered in a city center versus sparse in rural areas, at the cost of being a bit more complex to implement and to keep balanced as entities move around.

For a real-time use case like ride matching, I'd keep this index in memory, backed by something like Redis's native geospatial commands, specifically because entity locations update every few seconds and I need sub-second query latency on top of that constantly-changing dataset."


70. How would you design an inverted index for a full-text search system?

"For each unique term across the document collection, I'd maintain a postings list — the set of document IDs containing that term, along with enough extra information, like term frequency and position within the document, to support both relevance scoring and phrase matching later.

On ingestion, each document goes through tokenization, splitting text into individual terms, followed by stemming, so different forms of the same word map to a shared root and match each other in search. I'd also strip common stop words like 'the' and 'a' before indexing, since they add index size and lookup overhead while carrying essentially no distinguishing relevance signal.

At query time, a multi-word query looks up the postings list for each individual term and combines them — intersecting for an implicit AND, unioning for an OR — to get the candidate matching document set, then scores and ranks that candidate set using something like BM25.

At real scale, I'd shard the index itself across multiple machines, with each shard holding a subset of the total document collection, fanning a query out to every shard in parallel and merging the ranked results centrally, rather than trying to hold one single, enormous index on one machine."


71. How would you design a system to compute trending topics or "what's happening now" in real time?

"I'd treat this as a real-time streaming aggregation problem. Every relevant event — a post, a hashtag mention — flows through a durable event stream like Kafka, and a stream processor continuously maintains a rolling count of mentions per topic within a sliding time window, say the last hour, updating that count continuously rather than recomputing it from scratch on some fixed schedule.

To actually surface what's 'trending' rather than just 'popular,' I'd compare the current rate of mentions against each topic's own recent historical baseline, so a topic that's suddenly spiking relative to its normal volume ranks higher than a topic that's simply always popular — a celebrity's name might always get moderate volume, but a sudden, sharp spike relative to its own baseline is what actually signals something newsworthy just happened.

I'd maintain a continuously-updated top-N leaderboard, similar to the game leaderboard design, using a sorted-set-style structure for fast ranked reads, and refresh the publicly displayed 'trending now' list every few seconds to a minute, rather than on every single individual event, since that level of freshness is more than sufficient for this use case and meaningfully cheaper to serve."


72. How would you design a distributed transaction across two microservices that each own their own database?

"I'd first say that I'd try hard to avoid needing this at all — if two pieces of data need strict transactional consistency together, that's often a sign they actually belong in the same service and same database, and splitting them was premature. But assuming the split is genuinely necessary, I'd reach for the Saga pattern rather than Two-Phase Commit for most cases, specifically because Two-Phase Commit is a blocking protocol — if the coordinator fails between the prepare and commit phases, every participant is stuck holding locks, which is a real availability cost I'd rather avoid in a microservices setting.

With a Saga, each service commits its own local step immediately and independently, and I define a compensating action for each step to undo it if a later step in the sequence fails — for example, if Service A reserves inventory and Service B's payment charge subsequently fails, Service A executes a compensating action to release that reservation.

I'd choose between orchestration, where a central coordinator explicitly directs each step, or choreography, where each service reacts to events published by the previous step, based mainly on how many steps are involved and how much visibility I want into the overall flow — orchestration is easier to monitor and debug as the number of steps grows, so I'd lean toward it for anything beyond two or three steps."


73. How would you design a schema migration process for a large, always-on production database with zero downtime?

"I'd use the expand-contract pattern rather than making the change directly in one step. First, expand — add the new column, table, or structure alongside the existing one, without touching or removing anything the current application code relies on, so old code keeps working completely unaffected.

Then I'd update the application to write to both the old and new structures simultaneously, and run a background backfill job to populate the new structure with existing historical data, all while the old structure remains the actual source of truth that reads still come from.

Once I've verified the new structure is fully populated and correct — comparing it against the old structure for consistency — I'd switch reads over to the new structure, and only once that's been running safely and correctly for a while would I move to the contract phase: removing the old, now-unused structure.

The reason I'd insist on this multi-step approach rather than a single direct schema change is that it means the schema always supports both the old and new application code simultaneously during the transition — which is exactly what's required for a genuinely zero-downtime rolling deployment, since old and new code inevitably run side by side for at least a brief window during any rollout."


74. How would you design an audit-logging system that guarantees no event is ever lost?

"I'd have every service write audit events to a durable, replicated event stream like Kafka rather than directly to a database from the application's own request path, since I don't want audit logging to ever add latency or a new failure mode to the actual user-facing request.

I'd specifically use the Outbox pattern for this: the service's own primary state change and the corresponding audit event both get written in the exact same local database transaction, guaranteeing they either both succeed or both fail together, with a separate background process reliably publishing the outbox entries onward to the durable event stream, retrying as needed until it succeeds. Without this, a database write and a separate audit-log write are two independent operations that could each succeed or fail on their own, risking a real gap in the audit trail exactly when something worth auditing actually happened.

From there, a consumer reads from that durable stream and writes into a purpose-built audit store, one that's append-only and never allows updates or deletes to existing entries, specifically because an audit log that can be edited after the fact isn't a trustworthy audit log at all. I'd also replicate that audit store across regions, since it's exactly the kind of data where losing history is unacceptable even in a full regional failure."


75. How would you design a feature flag / experimentation (A/B testing) rollout system?

"I'd store flag and experiment definitions centrally — which flags exist, their possible values, and the targeting rules deciding which users see which variant, like 'this experiment applies to 10% of users in this region.'

For actually evaluating a flag at request time, I want this to be extremely fast and never a dependency the whole request can fail on, so I'd have each application instance pull the current flag configuration into local memory, refreshed periodically or pushed via a pub-sub update, rather than making a network call to a central flag service on every single request.

For consistent bucketing — the same user should reliably see the same variant across repeated visits, not randomly flip between variants — I'd deterministically hash the user's ID combined with the experiment's ID into a bucket, rather than using genuine randomness, so the assignment is stable and reproducible without needing to store an explicit assignment per user.

For measuring results, every relevant user action gets logged along with which experiment variant that user was in, and a separate analytics pipeline aggregates outcomes per variant and runs statistical significance testing before anyone concludes one variant is actually, genuinely better than another, rather than reacting to a difference that could just be noise."
