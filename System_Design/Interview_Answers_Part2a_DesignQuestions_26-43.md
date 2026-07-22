SYSTEM DESIGN INTERVIEW — CANDIDATE ANSWERS
PART 2A: SECTION B — CLASSIC "DESIGN A PRODUCT" QUESTIONS (26–43)

Each answer is written as a candidate would actually deliver it out loud — starting with a quick clarification of requirements, then the high-level design, then the one hardest sub-problem, told as a connected narrative rather than a bullet dump.


26. Design a URL shortener (e.g., TinyURL / bit.ly).

"Before I design anything, let me clarify scope — I'll assume we need to generate a short alias for a long URL, redirect users who visit it, and that reads will vastly outnumber writes, maybe 100 to 1. Roughly, if we're doing 100 million new URLs a month, that's about 40 writes a second and maybe 4,000 reads a second at peak — that tells me the read path deserves most of my attention.

For the architecture: a client hits a load balancer, which routes to a stateless app server. On write, I'll generate a unique short code using a counter-based approach — a global counter, base62-encoded, with each app server handed a pre-allocated range of IDs so they don't need to coordinate with each other on every single request. I'll store the short-code-to-long-URL mapping in a simple key-value store like DynamoDB, since the access pattern is purely 'get value by exact key,' nothing more complex than that.

Because this is so read-heavy, I'd put a cache like Redis in front of the database using cache-aside — check the cache first, and only fall back to the database on a miss. That absorbs almost all of my read traffic. For the redirect itself, I'd use a 302, not a 301, specifically because we probably want click analytics, and a 301 would get cached by the browser, meaning we'd never even see repeat visits.

The one thing I'd flag as the genuinely interesting design decision here is the ID generation strategy — a naive auto-increment column on a single database would become a write bottleneck and a single point of failure, so pre-allocating ranges to each app server avoids that coordination cost entirely."


27. Design a rate limiter (as a standalone service, not just the algorithm).

"I'll assume we need a service other internal APIs can call to check 'is this specific request currently allowed,' shared across many different services and many instances of each.

For the algorithm, I'd use token bucket — it allows short, reasonable bursts while still enforcing a steady average rate, which I think matches real traffic better than a fixed window, which has a known edge case where a client can burst right across a window boundary and briefly exceed the intended limit.

The part I actually think is more interesting here is the state-sharing problem. If this rate limiter is meant to be called by many different application instances, each instance can't just keep its own local counter — a client could trivially bypass the limit by having requests land on different servers. So I'd centralize the counters in Redis, keyed by client ID or API key, using Redis's atomic increment operations so concurrent requests from different app servers don't race each other. At very high scale, even one Redis instance can become a bottleneck, so I'd shard the counters across multiple Redis nodes using consistent hashing on the client key.

I'd also expose this as its own lightweight service or library, sitting either at the API Gateway layer — which is the natural, centralized place to enforce it before a request even reaches a backend service — or as a sidecar next to each service, depending on whether we want centralized policy control or per-service flexibility."


28. Design a distributed cache (e.g., "build your own Redis").

"This is really asking me to design the caching and data-distribution layer itself, not just use one. I'd distribute keys across multiple cache nodes using consistent hashing — placing both nodes and keys on a hash ring, so adding or removing a node only reshuffles the small slice of keys between it and its neighbor, not the whole dataset.

For eviction, once an individual node's memory is full, I'd default to LRU — evict whatever hasn't been accessed most recently — since it's a reasonable, low-overhead approximation of what's likely to be needed again soon. I'd mention LFU as an alternative if the interviewer wants something more resistant to a one-off traffic spike skewing the eviction decisions.

For fault tolerance, I'd replicate each node's data to at least one other node, so if a node dies, its replica can immediately take over instead of every key it held becoming a full cache miss all at once, which would otherwise dump a spike of traffic straight onto the database.

For how clients actually find the right node, I'd lean toward a proxy layer in front of the cache cluster rather than making every single client independently implement the hashing logic — simpler for callers, at the cost of one extra network hop. And for keeping data fresh when the underlying source changes, I'd use a pub-sub invalidation broadcast so every relevant node hears about a change and drops its stale copy."


29. Design a key-value store (e.g., "build your own DynamoDB").

"I'd design this as a leaderless, replicated system, since that's really what distinguishes a serious key-value store from a simple cache. Data is partitioned across nodes using consistent hashing. For each write, the client — or a coordinator on its behalf — writes to W out of N replica nodes; for each read, it reads from R nodes and reconciles if they disagree. If I set W plus R greater than N, I'm mathematically guaranteed that every read overlaps with at least one node that has the latest write, which gives me a tunable consistency guarantee without needing a single leader at all.

For conflict resolution when two nodes get concurrent, non-overlapping writes to the same key, I'd use vector clocks to determine whether one write causally happened before another, or whether they're genuinely concurrent and need to be merged rather than one arbitrarily overwriting the other.

For durability, every write is first appended to a write-ahead log before being applied, so a crash right after acknowledging a write doesn't lose it. I'd let W and R be tunable per operation, so callers who need strict consistency can set them higher, at the cost of latency, and callers who just want speed can set them to 1."


30. Design a notification system that supports push, email, and SMS.

"I'd design this as event-driven from the start — other services publish events like 'order shipped' to a pub-sub topic, and the notification service subscribes and reacts, fully decoupled from whatever triggered the notification in the first place.

Because one event might need to go out across multiple channels, I'd give each channel — push, email, SMS — its own separate queue and worker pool. That's a deliberate isolation choice: if the email provider is slow or down, it shouldn't delay or block push delivery, which has nothing to do with it.

For reliability, external providers fail transiently, so I'd retry with exponential backoff, and after some retry limit, move the notification to a dead letter queue rather than silently dropping it or retrying forever — that gives us visibility into failures instead of losing them quietly.

The part I'd specifically call out as the interesting product-and-systems overlap here is batching: if five related events fire for one user within a short window, I don't want to send five separate notifications — I'd batch them into one digest, both for user experience and to respect per-user rate limits, especially during something like a flash sale where a lot of events can fire back to back. And since delivery is at-least-once, I'd deduplicate by event ID on the client or server side so a redelivered event never results in the same notification twice."


31. Design a web crawler.

"I'd start from a set of seed URLs and maintain a crawl frontier — a priority queue of URLs still to visit — rather than simple FIFO, prioritizing pages by estimated importance, like how many other pages link to them, and by how frequently they've historically changed.

Politeness is a real, explicit requirement, not an afterthought — I'd respect each site's robots.txt, and rate-limit crawl requests strictly per domain, so I'm never hammering one specific website regardless of how much total crawling capacity I have available.

For duplicate detection, I'd separate two genuinely different problems: avoiding re-crawling a URL I've already visited, which I'd handle with a Bloom filter for fast, memory-efficient 'have I seen this' checks, accepting a small false-positive rate in exchange for massive memory savings; and detecting near-duplicate content served at two different URLs, which I'd handle with a content-fingerprinting technique like MinHash, so I'm not wastefully indexing the same effective page twice.

At scale, I'd distribute the crawling workload across many worker machines, partitioned by domain, specifically so that all the politeness state for one site naturally lives on one worker rather than being scattered across many workers that would each need to somehow coordinate rate limits with each other."


32. Design a search engine's autocomplete / typeahead feature.

"The key insight here is that autocomplete has a fundamentally different access pattern than regular search — I need to match on a PREFIX, on every single keystroke, within tens of milliseconds, not do a full-word inverted index lookup.

For the data structure, I'd use a trie — a tree where each path from the root spells out a prefix, so finding all completions for what the user's typed so far is just walking down the tree to that node and reading off the subtree below it. I'd annotate each complete word in the trie with a popularity score, computed offline from historical query logs, so I can return the top-N most likely completions ranked by popularity rather than an arbitrary order.

Since this needs to respond on every keystroke, I'd keep the trie fully in memory, sharded by prefix if it's too large for one machine, and I'd update it via a periodic offline batch job that reprocesses recent query logs — I don't need this to reflect the last five minutes of queries, daily or hourly freshness is totally fine for suggestion popularity.

I'd also add basic fuzzy matching for typo tolerance — if a prefix doesn't match anything exactly, fall back to suggestions within a small edit distance, so a one-character typo doesn't return zero suggestions."


33. Design Google Search (or a general-purpose search engine).

"I'd split this into three pieces: crawling, indexing, and serving. Crawling I've already described — a distributed crawler with a prioritized frontier and strict per-domain politeness.

For indexing, the core data structure is an inverted index — instead of storing which words are in a document, I store, for each word, which documents contain it, which turns 'find documents containing X' into a fast direct lookup instead of scanning every document. Before indexing, text goes through tokenization and stemming, so different forms of the same word — running, runs, ran — all match a search for any one of them.

For ranking, I'd start with a classic scoring approach like BM25, which combines how often a term appears in a specific document with how rare that term is across the whole collection, so common words like 'the' don't dominate scoring while rare, specific terms carry real signal.

At real scale, I'd shard the index across many machines, fan a query out to all relevant shards in parallel, and merge and re-rank the combined results centrally. And I'd explicitly say that ranking, in a real system, also blends in signals well beyond pure text relevance — click-through history, freshness, and personalization — but I'd frame those as extensions on top of the core text-relevance foundation I just described."


34. Design a chat application (e.g., WhatsApp / Messenger).

"I'd use WebSockets for real-time delivery — each client holds a persistent connection to one of many chat servers. The interesting problem is routing: if User A is connected to Server 1 and sends a message to User B, who's connected to Server 3, Server 1 needs to know that. I'd solve that with a presence service, backed by something fast like Redis, mapping user ID to their currently connected server, updated on every connect and disconnect.

For delivery guarantees, I'd accept at-least-once delivery rather than trying to guarantee exactly-once, which is expensive — each message gets a unique ID, and the client deduplicates by that ID so a rare redelivery never shows up twice on screen.

For ordering, I'd route all messages for a given conversation through the same partition, keyed by conversation ID, so messages within one conversation are always processed in order, even though there's no ordering guarantee across different, unrelated conversations.

For storage, I'd use a wide-column store like Cassandra rather than a relational database, since the access pattern — fetch messages for one conversation, ordered by time — is simple and the write volume is very high.

If end-to-end encryption is a requirement, I'd flag the real consequence of that upfront: the server can never read message content, which directly means server-side full-text search across message history becomes impossible — it would have to happen client-side instead, on the device that already holds the decrypted messages."


35. Design a social media news feed (e.g., Facebook / Instagram / Twitter).

"The core decision here is how the feed actually gets assembled — I'd call this the fan-out problem. Fan-out on write means the moment someone posts, I immediately push that post into every follower's precomputed feed, usually cached in Redis — reads are then instant, but a post from someone with millions of followers becomes a huge, expensive write. Fan-out on read means I do nothing special at post time and instead assemble a user's feed live, at load time, by pulling recent posts from everyone they follow — cheap to post, expensive to read.

I wouldn't pick one exclusively — I'd use a hybrid: fan-out on write for the vast majority of normal accounts, since their follower counts are small and pushing is cheap, but fan-out on read specifically for celebrity accounts, so one post from someone with 50 million followers doesn't trigger 50 million individual feed writes. A user's final feed is assembled by merging their precomputed feed with a live fetch of just the handful of celebrity accounts they follow.

I'd also rank the feed rather than showing pure chronological order — scoring each candidate post by recency, engagement so far, and relationship strength to the poster, and sorting by that score. And I'd flag that this whole design assumes eventual consistency for the feed — a post appearing a second or two late for some followers is an acceptable trade-off for the throughput this design buys us."


36. Design a video streaming platform (e.g., YouTube / Netflix).

"I'd split this into upload/processing and playback. On upload, the original file lands in object storage, and I'd immediately hand it off to a message queue for asynchronous transcoding — I don't want the uploader waiting minutes for the video to finish processing before their upload 'completes.' A pool of transcoding workers pulls from that queue and produces multiple resolutions and bitrates, storing each back in object storage.

For playback, I'd use adaptive bitrate streaming — the video is chunked into a few seconds each, and the player continuously requests whichever bitrate best matches the viewer's currently measured network speed, switching up or down as conditions change, rather than one fixed quality that either buffers on a bad connection or looks worse than necessary on a good one.

Because video is extremely bandwidth-heavy, a CDN is non-negotiable here — I'd cache popular content at edge locations close to viewers, since a small number of videos account for a hugely disproportionate share of total views, and serving those from the origin repeatedly would be both slow and very expensive.

Metadata — title, description, view count — is much smaller and goes in a regular database, with view counts specifically cached and updated somewhat loosely, since they don't need perfectly precise real-time accuracy on every single view. If this is more Netflix-flavored, I'd add that recommendation becomes a first-class part of the design, not just a nice-to-have on top of playback infrastructure."


37. Design a photo-sharing app (e.g., Instagram).

"This is really the social feed design I already described, with a heavier emphasis on media. Photos and videos go into object storage, served through a CDN, exactly like the video streaming case — large binary content doesn't belong in a structured database.

The feed itself uses the same hybrid fan-out approach — push for normal accounts, pull for high-follower accounts, merged at load time and ranked by relevance rather than pure chronology.

One thing I'd specifically add for a photo app is image processing on upload — generating multiple resized versions, thumbnails, and possibly applying filters, all done asynchronously via a queue-and-worker pipeline, the same pattern as video transcoding, just lighter weight since images process much faster than video.

I'd also mention that likes and comments need their own scalable counters — similar to the vote-counting problem in a system like Reddit — using atomic increments in a fast store rather than hitting the primary database directly on every single like, especially on a viral post getting thousands of likes a second."


38. Design a ride-sharing platform (e.g., Uber / Lyft).

"The foundational problem here is geospatial indexing — efficiently finding drivers near a specific rider without comparing that rider's location against every driver in the system. I'd use geohashing, which encodes latitude and longitude into a string where nearby locations share a common prefix, turning 'find nearby drivers' into a fast prefix lookup instead of a geometric distance calculation across everyone.

Drivers' apps send location updates every few seconds — at scale that's a huge continuous write stream, so I'd feed it through something like Kafka, both to update the live geospatial index and to feed downstream analytics without coupling the two.

For matching, given a rider request, I'd query the geo-index for nearby candidates and score them by distance, rating, and current status to assign one specific driver — this needs to be fast since the rider's actively watching a spinner waiting for a match.

Surge pricing is a real-time analytics problem layered on top — continuously computing the ratio of active requests to available drivers within small geographic regions and adjusting price where demand is currently outstripping supply.

For the trip lifecycle itself — match, drive, complete, pay — I'd use the Saga pattern, since it spans multiple services, with a compensating action, like releasing the driver assignment, if a later step like payment fails."


39. Design a food delivery platform (e.g., DoorDash / Swiggy).

"This is a genuine three-sided marketplace — customers, restaurants, and drivers — which pushes me toward splitting the system into independent services: a Restaurant Service for menus and availability, an Order Service owning the order lifecycle, and a Delivery Service handling driver assignment and tracking, communicating with each other through published events rather than tight, direct calls.

Restaurant discovery reuses the same geospatial indexing as ride-sharing, combined with a text search layer for cuisine and dish search.

The order lifecycle — placed, confirmed, paid, assigned, picked up, delivered — spans multiple services, so I'd use a Saga again, with a concrete compensating action like refunding the customer if a restaurant can't actually fulfill the order after accepting it.

I'd call out that driver assignment here is genuinely harder than in ride-sharing, because a delivery route has to account for both the restaurant's location and the customer's, and at real scale I'd batch nearby orders onto a single driver's route rather than assigning strictly one driver per order.

For live tracking, I'd push status updates — preparing, out for delivery, arriving soon — to the customer over a persistent connection like WebSockets or Server-Sent Events, rather than having the app poll repeatedly."


40. Design an e-commerce checkout and inventory system (e.g., Amazon).

"For the catalog, I'd use a dedicated search engine with an inverted index, supporting faceted filtering by category, price, and brand alongside free-text search.

The genuinely hard problem here is inventory correctness — never overselling the last unit of a popular item, even with many concurrent buyers trying to grab it at the same instant. I'd explicitly avoid the naive approach of reading the stock count, checking it's above zero, then separately decrementing it — that's a race condition, because two concurrent checkouts could both read '1 remaining' before either decrements. Instead I'd make the check and decrement one atomic database operation, so a second concurrent attempt safely and correctly fails rather than both succeeding.

The shopping cart needs to persist across sessions and devices, so I'd store it in a fast key-value store, similar to how I'd handle session state, just representing cart contents.

Checkout itself — reserve inventory, charge payment, dispatch to shipping — spans multiple independent concerns, so I'd use the Saga pattern again, with a compensating action to release reserved inventory if payment ultimately fails. If this were a flash-sale-specific follow-up, I'd emphasize that the atomic inventory operation is really the entire answer to 'how do you prevent overselling' — everything else in the design is secondary to getting that one operation right."


41. Design a file storage and sync service (e.g., Dropbox / Google Drive).

"Actual file content goes into object storage — large binary blobs don't belong in a relational database. Metadata — file names, folder structure, sharing permissions — goes into a regular database, since it benefits from strong consistency and structured queries like 'show me everything in this folder.'

The interesting technical problem is efficient sync: rather than re-uploading a whole file on every small change, I'd split files into fixed-size chunks, each identified by a hash of its own content. On a change, the client recomputes chunk hashes, compares them to what was previously uploaded, and only sends the chunks that actually changed. That same content hashing also gives me deduplication basically for free — if two different users happen to have an identical file or chunk, I only physically store it once.

For propagating changes to other devices, I'd use a publish-subscribe notification pushed to a user's other currently connected devices over a persistent connection, rather than each device repeatedly polling to check for updates.

For conflict resolution — the same file edited on two devices while both were offline — the simplest approach is keeping both versions as a 'conflicted copy' and letting the user sort it out. If this needs to support real-time collaborative editing, like Google Docs, I'd flag that as a meaningfully harder problem requiring either operational transformation or CRDTs to automatically merge concurrent character-level edits."


42. Design a payment processing system (e.g., Stripe / PayPal).

"Unlike most systems I design, strong consistency here isn't a trade-off I'm weighing — it's just required. Every operation touching a balance or transaction record needs full ACID guarantees.

Idempotency is critical, since clients will retry timed-out requests, and a payment retry must never risk a double charge. I'd have the client generate a unique idempotency key per attempt, and the server checks whether that key was already successfully processed before doing anything, returning the original result if so rather than reprocessing.

A single payment typically touches multiple systems — our own ledger, the card network, the receiving bank — so I'd use the Saga pattern across that flow, with careful compensating actions, like reversing an internal ledger entry if a downstream bank call fails partway through.

I'd also use the Outbox pattern for recording a transaction and notifying downstream systems like fraud detection or accounting — writing both the actual data change and the 'event to publish' into the same single atomic database transaction, so they can never end up out of sync, with a background process reliably publishing the outbox entries afterward.

On security, I'd keep raw card numbers out of our systems entirely, using a third-party processor for true card handling and tokenization, and I'd keep credentials in a dedicated secrets manager, never hardcoded."


43. Design a ticket-booking system (e.g., Ticketmaster) that prevents double-booking of the same seat.

"The defining problem is preventing two people from booking the exact same seat for the exact same event. I'd handle this with explicit concurrency control at the database level — either pessimistic locking, where the first booking attempt locks that specific seat row, forcing a second concurrent attempt to wait and then see it's taken, or optimistic concurrency control, where both attempts proceed but a version number check at commit time rejects whichever one loses the race, letting it retry against fresh data.

For a genuinely popular event — a huge on-sale moment with millions of concurrent users hitting the same small pool of good seats — I'd add a virtual waiting room in front of checkout, admitting users in controlled batches rather than letting everyone hit the booking endpoint simultaneously, which both protects the backend and gives users a fairer, less frustrating experience than a chaotic race.

The actual booking flow — hold the seat temporarily, process payment, confirm — is again a Saga, with a compensating action releasing the temporary hold if payment fails or if the hold simply times out because the user never completed checkout."
