SYSTEM DESIGN INTERVIEW — CANDIDATE ANSWERS
PART 2B: SECTION B — CLASSIC "DESIGN A PRODUCT" QUESTIONS (44–60)


44. Design a hotel or flight reservation system.

"This has the same double-booking problem as ticket booking, but harder, because availability is date-range-based rather than a single seat for a single event — a hotel room is unavailable only for a specific span of dates, so I need a range-overlap check rather than a simple flag.

I'd use the same concurrency control options — pessimistic locking on the specific room-and-date-range during a booking attempt, or optimistic version checking at commit time — to make sure two overlapping booking attempts for the same room can never both succeed.

Search — 'available hotels in this city, these dates, this price range' — is a faceted search problem, similar to product search, while the underlying availability data itself I'd partition by hotel or region to keep any single database's data volume manageable.

The booking flow is again a Saga: temporarily hold the room, charge payment, confirm — with a compensating action releasing the hold if payment fails or the hold simply expires."


45. Design a parking garage system.

"I'd clarify first whether this needs real-time spot availability display, multiple entry/exit points, and different vehicle/spot sizes — I'll assume yes to all three since that's the more interesting version of the problem.

I'd model the garage as floors, each with rows of spots, each spot having a type — compact, regular, oversized — and a current status. On entry, a sensor or ticket kiosk assigns the nearest available spot matching the vehicle's size, and I'd maintain a fast in-memory count of available spots per type per floor so a display board can show live availability without scanning every individual spot on every request.

For payment, I'd track entry time, compute a fee based on duration at exit, and support multiple payment methods. This is a much lower-scale system than most of what I've discussed today, so I'd deliberately keep it simpler — a single well-indexed relational database is genuinely sufficient here; I wouldn't reach for sharding or a distributed cache unless we were talking about a garage network at genuinely large multi-city scale."


46. Design an ATM system, including how to handle a machine running out of cash mid-transaction.

"For the core flow — authenticate card and PIN, verify the account has sufficient balance, dispense cash, update the balance — the critical requirement is that the balance update and the physical cash dispensing need to be treated as one logical transaction, even though one happens in a database and the other happens in physical hardware.

For the specific failure case — the ATM runs out of cash after the balance has already been deducted but before cash is actually dispensed — I'd design the flow so the balance deduction only commits AFTER the machine confirms successful physical dispensing, not before. If the dispensing step fails partway, I'd have the transaction roll back automatically, and if there's any ambiguity about whether cash was actually dispensed, I'd flag the transaction as 'pending reconciliation' rather than either blindly deducting or blindly not deducting, and route it to a manual review queue.

I'd also mention idempotency here again — if the network connection between the ATM and the bank's server drops right after dispensing but before confirming success back to the bank, a retry must not cause a double deduction, so I'd use the same idempotency-key approach from the payment system design."


47. Design a distributed task scheduler / cron job service.

"I'd need this to reliably trigger jobs at specified times or intervals, even as individual worker machines come and go, and to guarantee a given job runs on exactly one worker, not zero and not multiple.

I'd store job definitions — schedule, target, payload — in a database, and use a leader-elected coordinator, or a small consensus-backed cluster of coordinators, to periodically scan for jobs that are due and dispatch them to available workers. To avoid double-execution if two coordinator instances somehow both think they're responsible, I'd use a distributed lock on each specific job right before dispatch, so only one coordinator instance can actually claim and trigger it.

Workers pick up dispatched jobs from a queue, execute them, and report success or failure back; on failure, I'd retry with backoff up to a limit, then move to a dead letter queue for manual investigation rather than silently dropping a failed job.

For very high job volume, I'd shard job definitions across multiple coordinator instances by job ID, so no single coordinator has to scan the entire job table on every tick."


48. Design a distributed logging and monitoring system.

"I'd split this into log/metric collection, storage, and querying/alerting. Each application instance emits structured logs and metrics; a lightweight local agent collects and forwards them, rather than each application making a network call per log line, which would be slow and fragile.

For metrics specifically, I'd use a time-series-optimized store, since the access pattern is almost always 'aggregate this metric over a time range,' and a pull-based collection model like Prometheus's, where the central system periodically scrapes each service's exposed metrics endpoint — that also gives me an easy way to detect a fully unresponsive service, since a failed scrape is itself informative.

For logs, I'd centralize them into a search-optimized store like Elasticsearch, so an engineer can search across every service's logs by request ID, user ID, or free text, rather than SSHing into individual machines.

For alerting, I'd favor symptom-based alerts — like elevated error rate or latency — over purely cause-based alerts like disk usage, since symptom-based alerts reflect genuine user impact regardless of the specific underlying cause, and I'd require a sustained threshold over a few minutes, not a single data point, to avoid paging someone for a brief, harmless blip."


49. Design a real-time collaborative document editor (e.g., Google Docs).

"The core problem is letting multiple users edit the same document simultaneously and have everyone's changes converge to the same final result, even though edits can arrive out of order or overlap.

I'd use either Operational Transformation or CRDTs to handle this. OT works by transforming each incoming edit against any edits that happened concurrently, so it can be applied correctly regardless of the order it arrives in relative to other users' edits — this is what Google Docs itself historically used. CRDTs instead design the underlying data structure itself so that concurrent edits are always automatically, mathematically mergeable without needing a central transformation step — this tends to be simpler to reason about correctness for, though the underlying data structures are less intuitive to implement from scratch.

Each client maintains a local copy of the document and applies edits optimistically for instant local responsiveness, while a central server (or fully peer-to-peer, in some CRDT designs) is responsible for correctly ordering and broadcasting edits to all other currently connected clients over a persistent connection like WebSockets.

I'd also mention presence — showing other users' live cursors — as a lighter-weight real-time broadcast layered on top of the same connection, and periodic snapshotting of document state so a client reconnecting after being offline doesn't need to replay the entire edit history from the beginning."


50. Design a video conferencing system (e.g., Zoom).

"For real-time audio/video transport, I'd use WebRTC over UDP, specifically because a slightly late or dropped frame should be skipped, not waited for, or the whole call feels laggy.

For handling multiple participants, I'd explicitly reject a mesh architecture, where everyone sends directly to everyone else, because bandwidth and CPU cost scale quadratically with participant count. Instead I'd use an SFU — a Selective Forwarding Unit — where each participant sends one stream to a central server, which relays it to everyone else who needs it; that keeps the cost on each individual device linear regardless of call size, with the heavier relay work concentrated on infrastructure that's much easier to scale than someone's home internet connection.

Screen sharing is just another stream through the same SFU pipeline. Recording is a separate, asynchronous process that captures and writes the combined media to object storage, deliberately decoupled from the live call so it never adds delay.

I'd also mention NAT traversal — since most devices sit behind routers hiding their real network address, I'd need STUN servers to help establish direct connections where possible, and TURN servers as a relay of last resort for devices that genuinely can't connect directly."


51. Design an online multiplayer game backend.

"The core challenge is synchronizing fast-moving game state across all players with minimal perceived lag. I'd use UDP-based transport rather than TCP or WebSockets, for the same reason as video conferencing — a stale position update should be skipped, not queued up behind a guaranteed-in-order delivery mechanism that would otherwise delay everything behind a single lost packet.

I'd use an authoritative server model to prevent cheating — clients send their inputs, not raw state changes, and the server computes and validates the actual resulting game state, then broadcasts it back out. To keep gameplay feeling responsive despite real network latency, I'd add client-side prediction — the client optimistically renders what it expects to happen locally, and reconciles or corrects itself if the server's authoritative response ends up different.

For matchmaking, I'd match players by skill rating and region, similar in spirit to the geospatial matching in ride-sharing, just matching on skill and latency rather than physical location.

For persistent data like player profiles and inventory, that's a much more traditional database-plus-cache design, completely separate from the real-time gameplay path. And I'd deploy game servers across multiple regions so players are matched against whichever server is physically closest, minimizing the latency that this whole design is fundamentally built around minimizing."


52. Design a leaderboard system (e.g., for a game or fitness app).

"The core requirement is efficiently maintaining and querying rankings — 'what's my rank,' 'show me the top 100,' 'show me players near my rank' — for potentially millions of players, updated very frequently as scores change.

I'd use a sorted set data structure, like Redis's ZSET, which keeps entries automatically ordered by score and supports getting a rank or a range of ranks in logarithmic time — that's a much better fit than repeatedly running an ORDER BY query against a relational database on every single request. Score updates are simple atomic operations against that same sorted set.

For a genuinely massive player base, I'd shard leaderboards logically — for example, by region or by game mode — rather than maintaining one single giant global structure, and if I need a combined global view, I'd periodically merge the top entries from each shard rather than trying to query across all of them live on every request.

I'd also cache the very top of the leaderboard, like the top 100, aggressively, since that's disproportionately requested compared to a random player's individual rank lookup, which I'd compute on demand instead."


53. Design a distributed unique ID generator (e.g., Twitter's Snowflake).

"The requirement is generating unique, roughly time-sortable IDs across many machines without any central coordination on every single ID generated, since that would be a bottleneck and single point of failure.

I'd use the Snowflake approach — each ID is a single 64-bit number composed of a timestamp, a machine or worker ID, and a per-machine sequence number that increments for IDs generated within the same millisecond. Because the timestamp is the highest-order bits, IDs are naturally, roughly sortable by creation time even across many different machines. Because each machine has its own unique worker ID baked into the ID, and its own local sequence counter, no two machines can ever generate the same ID without needing to talk to each other at all.

The one operational risk I'd flag is clock skew — if a machine's clock jumps backward, it could generate a duplicate or out-of-order ID, so I'd have each machine explicitly refuse to generate IDs if it detects its own clock has moved backward, and page an engineer rather than silently risking a collision."


54. Design a system to detect duplicate content or plagiarism at scale.

"I'd separate this into exact-duplicate detection and near-duplicate detection, since they need genuinely different techniques.

For exact duplicates, I'd hash the full content and compare hashes — trivially fast, but obviously useless the moment even one character differs.

For near-duplicates — genuinely the harder and more interesting part — I'd use a technique like MinHash combined with locality-sensitive hashing. The idea is to break content into overlapping shingles, hash a representative sample of them in a way specifically designed so that similar documents produce similar hash signatures, and then bucket documents by those signatures so I only need to compare documents that landed in the same bucket, rather than comparing every document against every other document, which wouldn't scale at all.

At real scale, I'd run this as an offline or near-real-time pipeline rather than checking every single piece of new content synchronously against the entire existing corpus — new content gets hashed and bucketed on ingestion, and a comparison against just its bucket-mates happens shortly after, asynchronously, flagging likely matches for either automatic action or human review depending on confidence."


55. Design an ad-serving / ad-auction system with real-time bidding.

"The defining constraint here is latency — a real-time auction among multiple bidders typically needs to complete within around 100 milliseconds end to end, which is tighter than almost anything else I'd design.

Given that, I'd push as much work as possible into precomputation and caching ahead of time — candidate ad sets per targeting segment, user profile summaries — rather than computing anything expensive fresh on each individual request. At actual request time, I'd narrow the full ad inventory down by targeting criteria to a smaller candidate set, then run the real-time auction just among those candidates, scoring by a combination of bid value and predicted relevance.

For pacing an advertiser's budget smoothly across a whole day, and for frequency capping so a single user doesn't see the same ad excessively often, I'd use fast atomic counters in Redis, since both need to be checked and updated within that same extremely tight latency budget.

I'd explicitly say that nearly every design decision here — the aggressive caching, the precomputation, keeping the live computation path as narrow as possible — exists specifically because of that one hard latency constraint, more so than in most other systems I'd design."


56. Design a stock trading platform with an order-matching engine.

"This is one of the most latency-sensitive systems I'd ever design — the matching engine often needs to process orders within microseconds, since even small latency differences between traders can be genuinely, financially exploited.

For the matching engine itself, incoming buy and sell orders are organized into an order book, sorted by price, and the engine looks for compatible matches. Given the extreme latency requirement, I'd deliberately implement this as a single-threaded, entirely in-memory process per security — which sounds like it goes against normal 'scale by distributing' instincts, but that's the point: any cross-machine coordination, like locks or network calls, would introduce exactly the latency this design is trying to eliminate in its single hottest path.

Every order event is still durably logged using an event-sourcing approach, both for a complete audit trail and so the exact order book state can be reconstructed if the process needs to restart after a crash.

For scaling, I wouldn't try to distribute the matching logic for any one security — I'd instead run many independent single-threaded matching engines in parallel, one per security, achieving horizontal scale across securities rather than within any single security's own order book."


57. Design a distributed file system (e.g., HDFS / Google File System).

"I'd split large files into fixed-size chunks — much bigger than a typical local filesystem's blocks, since this is optimized for large sequential reads rather than small random access — and replicate each chunk, typically three times, across physically separate machines and racks for fault tolerance.

I'd use a centralized coordinator, or NameNode-style component, that tracks metadata — which files exist, which chunks make up each file, and which specific machines hold each chunk — while the actual chunk data lives entirely on separate worker machines. A client reading a file first asks the coordinator where the relevant chunks are, then reads the bulk data directly from those workers — critically, the actual data volume never has to flow through the coordinator itself, which would otherwise become an overwhelming bottleneck.

Because the coordinator is a genuinely critical single point of failure, I'd run it as a small cluster using a consensus protocol like Raft to stay reliably in sync, rather than a single unreplicated machine.

For fault detection, workers send periodic heartbeats to the coordinator, and if one goes silent, the coordinator automatically triggers re-replication of the chunks that worker was responsible for, restoring the target replica count elsewhere."


58. Design a URL-based web analytics / click-tracking system (e.g., Google Analytics).

"I'd treat this as a high-volume event ingestion and aggregation pipeline. Every pageview or click gets sent as an event to a durable, high-throughput streaming platform like Kafka, which acts as the reliable backbone the rest of the pipeline builds on.

I'd run two processing paths off that same event stream: a stream processor maintaining real-time rolling aggregates, like 'pageviews in the last hour,' feeding a fast-access store for live dashboards; and a separate batch job, running periodically, computing more thorough historical aggregates across the full dataset, since some metrics genuinely need the complete picture, not just a rolling window.

Raw events also land untouched in a data lake in their original form, so we're never locked out of computing some new metric later that we didn't think to precompute at ingestion time. Processed, structured aggregates get loaded into a data warehouse specifically for fast, reliable business-intelligence-style querying.

Given the sheer event volume here, I'd also mention sampling as a real, pragmatic option for extremely high-traffic sites — recording, say, 10% of events with a multiplier applied at query time, trading a small amount of precision for a large reduction in ingestion and storage cost."


59. Design a system to track and prevent fraudulent reviews (e.g., Amazon reviews abuse).

"I'd frame this as a mix of rule-based signals and an ML classification model, feeding into a moderation queue rather than fully automated silent removal, given how costly a false positive is here — wrongly removing a genuine review is a real trust problem.

For features, I'd look at account-level signals, like account age and past review velocity, and behavioral signals, like unusually large clusters of reviews posted for one product within a short window, or accounts that only ever review products from one specific seller — these patterns are strong fraud signals on their own, even before considering the review text itself.

I'd run a fast, cheap rule-based check first, catching the most obvious cases immediately, and route genuinely ambiguous cases to a full ML model for a more careful score. Given that fraudulent review tactics evolve as bad actors adapt to whatever detection is currently in place, I'd treat this as a case of real concept drift, requiring a tighter, more frequent retraining loop than a typical, more stable ML use case.

For the ones the model flags as likely fraudulent but not with full confidence, I'd route to human review rather than automatic removal, and I'd specifically monitor the model's false-positive rate as closely as its detection rate, since over-flagging genuine reviews has a real cost to the business too."


60. Design an API rate-limited third-party integration platform (e.g., Zapier-style workflow automation).

"The core requirement is letting users connect a trigger from one third-party service to an action in another, reliably, even though each third-party API has its own rate limits, authentication requirements, and occasional downtime.

I'd model each user's automation as a defined workflow — a trigger, and one or more subsequent actions — stored as configuration. Triggers are handled either via polling, for services without webhook support, on a schedule appropriate to that specific integration, or via inbound webhooks for services that support pushing events to us.

Each triggered execution gets published as an event and processed asynchronously by workers, rather than synchronously in the request path, since third-party APIs can be slow or temporarily unavailable, and I don't want that directly blocking anything user-facing.

Critically, I'd rate-limit our own outbound calls to each specific third-party service, separately per service and per user's connected account, respecting whatever limits that service imposes — using the same token-bucket approach I described for the standalone rate limiter, just applied outbound instead of inbound. For reliability, I'd retry failed third-party calls with backoff, and move persistently failing executions to a dead letter queue with clear surfacing back to the user about what failed and why, rather than silently dropping a workflow step."
