SYSTEM DESIGN INTERVIEW — CANDIDATE ANSWERS
PART 1: SECTION A — CORE FUNDAMENTALS AND CONCEPTUAL QUESTIONS

Each answer below is written the way a strong candidate would actually say it out loud to an interviewer — direct, structured, and willing to state a clear opinion rather than hedging.


1. What is the CAP theorem, and how does it influence database choice in a distributed system?

"CAP says a distributed system can only guarantee two of three things — consistency, availability, and partition tolerance — at the same time. In practice, though, partition tolerance isn't really optional: networks fail, so any real distributed system has to tolerate partitions. That means the actual decision I'm making, day to day, is consistency versus availability whenever a partition happens. If I'm designing something like a banking ledger, I'll lean CP — I'd rather return an error or a timeout than show someone the wrong balance. If I'm designing something like a social media feed or a shopping cart, I'll lean AP — showing a slightly stale feed is a much better user experience than the whole app going down. So when I pick a database, I'm really asking: what does this specific system need to be true about, versus what does it need to always respond to. That's what pushes me toward something like Postgres with synchronous replication for the ledger case, versus Cassandra or DynamoDB for the feed case."


2. What is the difference between horizontal and vertical scaling?

"Vertical scaling means making one machine bigger — more CPU, more RAM. It's simple, there's no distributed systems complexity, but it has a hard ceiling, and that one machine becomes a single point of failure. Horizontal scaling means adding more machines and spreading the load across them. It scales much further and gives you fault tolerance — if one node dies, the others keep serving — but it introduces real complexity: you need a load balancer, you need to think about whether your application is stateless, and if you're scaling a database, you're now dealing with sharding and replication. In practice I'll vertically scale a database as far as it reasonably goes because it's simpler, and horizontally scale the stateless application/web tier from day one, since that's cheap to do and gives me redundancy for free."


3. What is the difference between a SQL and a NoSQL database, and when would you choose each?

"SQL databases give me a fixed schema, strong ACID transactions, and the ability to do complex joins across tables — that's the right choice when my data is genuinely relational, like orders referencing users and products, and I need strong consistency, like financial data. NoSQL is really an umbrella — key-value, document, wide-column, graph — and what they share is a more flexible schema and a design that's built to scale out horizontally more easily than a traditional relational database. I'd reach for NoSQL when my access pattern is simple and known in advance, like 'give me this user's session by ID,' or when I need to write at very high volume and can tolerate eventual consistency, like a messaging system's message store. In a real interview I wouldn't say 'NoSQL is more scalable' as a blanket statement — I'd tie the choice directly to the specific access pattern I just described for that system."


4. What is database sharding, and how is it different from partitioning?

"Sharding is splitting your data across multiple separate database servers, so each server, or shard, holds only a subset of the total data — that's how you get past the point where a single machine can't hold or serve your dataset anymore. Partitioning is really the broader term — it just means dividing data into segments — and it can happen even within a single database instance, for example splitting one big orders table into monthly partitions so old data can be archived cheaply. Sharding is specifically partitioning that's distributed across separate machines. The real cost of sharding is that joins and transactions that used to be trivial within one database now have to span multiple machines, which is a lot harder — so I only reach for it once I actually need it, not preemptively."


5. What is consistent hashing, and why is it preferred over simple modulo-based hashing?

"If I distribute keys using something like hash(key) mod N, where N is the number of servers, then the moment I add or remove a server, N changes, and almost every key's assigned server changes with it — that means a massive, unnecessary reshuffle of data every time I scale. Consistent hashing fixes this by placing both the servers and the keys on a conceptual ring, using the same hash function, and assigning each key to the next server it finds going clockwise. When I add or remove a server, only the keys between that server and its neighbor need to move — everything else on the ring stays exactly where it was. I'd also mention virtual nodes — instead of putting each physical server at just one point on the ring, you put it at many points, which spreads the load much more evenly and means that when one server drops out, its load is absorbed by many other servers instead of dumping onto just one neighbor."


6. Explain the difference between strong consistency and eventual consistency, with a real example of each.

"Strong consistency means that once a write completes, every subsequent read, from any node, sees that write immediately — there's no window where you could see stale data. That's what I want for something like an account balance: if I just transferred money, I need to see the updated balance right away, not five seconds later. Eventual consistency means that if no new writes happen, all replicas will eventually converge to the same value, but there's no guarantee about how long that takes, and a read in that window might return stale data. A good example is a 'like count' on a social post — if it takes a couple seconds to update everywhere, nobody notices or cares, and in exchange I get much better availability and lower latency. The way I frame this in an interview is: I ask what happens if the user sees stale data for a moment — is that a minor annoyance, or a real problem? That answer tells me which consistency model to reach for."


7. What is a load balancer, and what algorithms can it use to distribute traffic?

"A load balancer sits in front of multiple backend servers and distributes incoming requests across them, so no single server gets overwhelmed, and so if one server goes down, traffic automatically routes around it. For algorithms — round robin just cycles through servers one at a time; least connections sends the next request to whichever server currently has the fewest active connections, which handles uneven request durations better than round robin; and weighted versions of both let you send more traffic to more powerful servers. I'd also distinguish Layer 4 versus Layer 7 — Layer 4 just looks at IP and port and is fast but dumb, while Layer 7 actually inspects the HTTP request and can route based on the URL path, which lets me do things like send /api/* to one service and /static/* to a CDN-backed service. And critically, the load balancer needs to do health checks — if a server stops responding, it gets pulled out of rotation automatically."


8. What is the difference between a forward proxy and a reverse proxy?

"They're mirror images of each other in terms of who they're hiding. A forward proxy sits in front of clients and makes requests on their behalf — the destination server sees the proxy, not the actual client. That's used for things like corporate network filtering, or bypassing geographic restrictions. A reverse proxy sits in front of servers and handles requests coming in from clients — the client only ever talks to the reverse proxy, never directly to the real backend servers. That's used for load balancing, SSL termination, and hiding your actual server architecture from the outside world. The simplest way I remember it: forward proxy protects and represents the client; reverse proxy protects and represents the server."


9. What are the different caching strategies, and when would you use each?

"Cache-aside is the most common — the application checks the cache first, and on a miss, it queries the database itself, populates the cache, and returns the result. It's simple and resilient, since if the cache is down, the app still works, just slower. Write-through writes to the cache and the database at the same time, synchronously, so the cache is always fresh, but every write pays the cost of touching both systems. Write-back writes to the cache immediately and asynchronously flushes to the database later — that's much faster for writes, but if the cache crashes before that flush happens, you can lose data, so I'd only use it where losing a few seconds of the most recent writes is genuinely acceptable, like a view counter. I'd pick cache-aside as my default for most read-heavy systems, and only reach for write-through specifically when I can't tolerate the cache and database ever disagreeing, even briefly."


10. How do you handle cache invalidation, and why is it considered a genuinely hard problem?

"It's hard because you have two copies of the truth — the cache and the source database — and the moment the underlying data changes, you have to make sure the cache doesn't keep serving the old value. The simplest approach is a TTL — every cache entry just expires after some time, so staleness is bounded even if you forget to explicitly invalidate something. On top of that, I'd explicitly delete or update the cache entry whenever the application writes to the database, so users don't have to wait out the TTL to see a fresh value. At real scale I'd also worry about the thundering herd problem — if a very popular key expires, a huge number of requests can all miss the cache at the same instant and hammer the database simultaneously. I'd handle that with either a lock so only one request actually recomputes the value while others wait, or with a refresh-ahead approach that proactively refreshes hot keys before they actually expire."


11. What is a content delivery network, and how does it reduce latency?

"A CDN is a globally distributed network of caching servers, positioned physically close to end users in lots of different regions. Instead of every single user's request traveling all the way back to one central origin server, it gets served from a nearby edge location instead — and because distance directly drives network latency, that's a real, physical speedup, not just a caching trick. It also takes a huge amount of load off my origin servers, since popular content — images, videos, static JS/CSS — gets served almost entirely from the edge after the first request. I'd use a CDN for anything read-heavy and not hyper-personalized: static assets always, and for something like video streaming, it's basically mandatory, since video is so bandwidth-heavy that serving it all from one region would be both slow and extremely expensive."


12. What is database replication, and what's the difference between leader-follower and leaderless replication?

"Replication means keeping multiple copies of the same data on different nodes, mainly for fault tolerance and to scale out read traffic. In leader-follower replication, one node accepts all writes and then propagates them to followers, which typically only serve reads — it's simple to reason about, but if the leader goes down, you need a failover process to promote a new one, and there's a brief window where that's a single point of failure. In leaderless replication, any node can accept both reads and writes, and consistency comes from the client writing to and reading from a quorum of nodes — so there's no single leader to fail over at all, which gives you better write availability, at the cost of needing conflict resolution when two nodes get concurrent, conflicting writes. I'd pick leader-follower for something like a typical relational workload where I want a clear source of truth, and leaderless for something like Cassandra-style high-write-volume systems where I specifically want to avoid any single point of failure for writes."


13. What is the difference between a message queue and a publish-subscribe system?

"A message queue is point-to-point — a message goes onto the queue, and exactly one consumer picks it up and processes it, even if multiple consumers are listening; they're competing for messages, not each getting a copy. Publish-subscribe is fan-out — a publisher sends a message to a topic, and every single subscriber currently listening to that topic gets its own copy. I'd use a queue for distributing work, like processing image uploads, where I want the work done exactly once by whichever worker picks it up. I'd use pub-sub for broadcasting an event that multiple, independent systems care about — like 'order placed,' which the inventory service, the email service, and the analytics service might all need to react to independently, without the order service needing to know or care who's listening."


14. How would you design a system to be highly available versus highly consistent — walk through a concrete trade-off?

"Let's say I'm designing a shopping cart. If I prioritize availability, I'd let a user add to their cart on any server, using leaderless-style replication, and worry about merging conflicting concurrent writes later — if the network partitions, the cart still works, worst case I merge two versions afterward. If I prioritize consistency, I'd route all writes for a given cart through one specific node or use a strongly consistent database, and if that node's unreachable during a partition, the user simply can't modify their cart until it recovers. For a shopping cart specifically, I'd actually pick availability — a merge conflict on a cart is a minor, recoverable annoyance, but the cart being completely unusable during a network blip would directly cost the business money. I'd contrast that against checkout itself, where I'd flip to consistency, because double-charging or losing a payment is a much more serious failure than a brief unavailability."


15. What is an API Gateway, and what problems does it solve in a microservices architecture?

"An API Gateway is a single entry point that sits in front of a collection of backend microservices and handles cross-cutting concerns before routing a request to the right internal service — authentication, rate limiting, request logging, and sometimes aggregating responses from multiple services into one response for the client. Without it, every single microservice would need to independently reimplement authentication and rate limiting, which is both wasteful and a security risk, since now dozens of services all need to be independently hardened against the public internet. It also means the client — a mobile app, say — doesn't need to know that a single screen is actually being assembled from five different backend services; it just calls the gateway once."


16. What is the difference between synchronous and asynchronous communication between services, and when would you use each?

"Synchronous communication — a direct API call, like REST or gRPC — means the caller waits for the response before continuing. It's simple to reason about and you get an immediate result, but it tightly couples the two services' availability: if the downstream service is slow or down, the caller is stuck waiting too, unless I've added timeouts and circuit breakers. Asynchronous communication — publishing to a queue or event stream — means the caller fires off the request and moves on, and the actual processing happens independently, later. That decouples the two services completely: the caller doesn't care if the downstream consumer is temporarily down, since the message just waits in the queue. I'd use synchronous when I genuinely need the result right now to continue — like checking inventory during checkout — and asynchronous for anything that can happen slightly later without the user noticing, like sending a confirmation email."


17. Explain the Circuit Breaker pattern and why it's needed in a microservices system.

"A circuit breaker sits between a service and one of its dependencies, and it tracks the failure rate of calls to that dependency. Normally it's closed and just lets calls through. If the failure rate crosses some threshold, it trips open, and for a while it immediately fails every subsequent call without even attempting to reach the struggling dependency — that's 'failing fast.' After a cooldown period, it goes half-open and lets a few test requests through to see if the dependency has recovered, and either closes again or goes back to open based on the result. The reason it matters is cascading failure: without it, if one downstream service starts timing out, every upstream caller ends up with a growing pile of threads stuck waiting on it, which can make the caller itself slow or unresponsive, and that spreads up the whole call chain. The circuit breaker stops that spread at the source."


18. What's the difference between a monolith and microservices, and when would you choose one over the other?

"A monolith is one codebase, one deployment, everything running as a single application. Microservices split that into many small, independently deployable services, each usually owning its own database. The honest answer is that a monolith isn't automatically worse — it's simpler to develop locally, there's no network overhead between components, and you get real ACID transactions across your whole data model for free. Microservices solve organizational problems more than technical ones — they let separate teams deploy independently without stepping on each other, and let you scale just the parts of the system that are actually under load. I'd start a small team or early-stage product as a monolith, because the coordination overhead of microservices genuinely isn't worth it yet, and I'd only split it apart once a specific team or a specific component's scaling needs actually justify the added distributed-systems complexity."


19. What is idempotency, and why does it matter for retries in a distributed system?

"An operation is idempotent if doing it once has the exact same effect as doing it multiple times. It matters because networks are unreliable — a client might send a request, the server might process it successfully, but the response gets lost, so the client times out and retries. If that operation isn't idempotent — like 'charge the customer $50' — that retry can cause a real, harmful duplicate side effect, like a double charge. The standard fix is to have the client generate a unique idempotency key for each distinct attempt, and have the server check whether it's already successfully processed that exact key before doing anything — if so, it just returns the original result instead of repeating the action. I'd bring this up any time I'm designing retry logic, because it's genuinely easy to forget until it causes a real production incident."


20. Explain the Saga pattern and how it differs from a traditional distributed (Two-Phase Commit) transaction.

"Two-Phase Commit tries to make a transaction spanning multiple services fully atomic — a coordinator asks every participant 'are you ready to commit,' and only tells everyone to actually commit once all of them say yes. It gives strong consistency, but it's a blocking protocol — if the coordinator crashes after everyone's said yes but before the final commit message, everyone's stuck holding locks, unable to proceed, which is a real availability cost. The Saga pattern instead breaks a multi-step transaction into individually-committed local steps, each with a defined compensating action to undo it if something later fails — so instead of one big atomic operation, you get a sequence of small committed operations that self-corrects if a later step fails, rather than holding everything open and blocked. I'd reach for Sagas in a microservices architecture specifically because they don't block, even though the trade-off is you're accepting eventual consistency — there's a brief window where the overall operation is partially complete."


21. How would you design a rate limiter, and what algorithms could you use?

"For the algorithm itself, I like token bucket — a bucket holds a max number of tokens, refilling at a steady rate, and each request consumes one token; if the bucket's empty, the request gets rejected. It naturally allows short bursts if tokens have built up, while still enforcing a steady average rate over time, which I think matches real traffic patterns better than a naive fixed window, which has an edge case where a client can send double the intended limit right across a window boundary. But the more interesting part of this question, if it's asked as a standalone service rather than just 'name the algorithm,' is how you share state across many application server instances — if each instance tracked its own local counter, a client could exceed the real limit just by hitting different servers. I'd use a fast, centralized store like Redis with atomic increment operations so every instance reads and writes the same shared counter, and if that becomes a bottleneck at very high scale, I'd shard the counters by client key using consistent hashing."


22. What is database indexing, and what's the trade-off of adding more indexes to a table?

"An index is a separate data structure, usually a B-tree, that lets the database find rows matching a query condition without scanning the entire table — it turns an O(n) full table scan into something closer to O(log n). The trade-off is that every index has to be updated on every insert, update, or delete to that table, so more indexes mean slower writes and more storage used. So I wouldn't just index every column — I'd specifically index columns that are actually filtered, sorted, or joined on frequently, and I'd actually check the query's execution plan to confirm an index is being used before assuming it's helping."


23. How do you design a system to handle a "thundering herd" or cache stampede problem?

"This happens when a very popular cache key expires or gets invalidated, and a huge number of concurrent requests all miss the cache at the same instant and hit the database simultaneously trying to recompute the same value. My first fix is locking — the first request that misses acquires a lock and actually goes to the database, while every other concurrent request for that same key waits briefly and then reads the now-freshly-cached value, instead of every single one independently hammering the database. My second option is refresh-ahead — for genuinely hot keys, proactively refresh the cache slightly before it actually expires, so from the requester's point of view it basically never expires at all. I'd also add jitter to TTLs across many keys so they don't all expire at exactly the same moment in the first place."


24. What is back-of-the-envelope estimation, and why do interviewers ask you to do it?

"It's translating a vague scale like 'millions of users' into concrete numbers — requests per second, storage growth per year, bandwidth needed — early in the design, because those numbers directly drive real architectural decisions. If my estimated read QPS is in the low hundreds, I probably don't need a distributed cache at all; if it's in the hundreds of thousands, caching and read replicas become mandatory, not optional. Interviewers ask for this because it shows I'm not just pattern-matching a design from memory — I'm actually reasoning about whether this specific system, at this specific scale, needs the complexity I'm about to introduce. I'd do this early, right after clarifying requirements, and keep the math rough and fast — the goal is order of magnitude, not precision."


25. What are the four "golden signals" or key metrics you'd monitor for a production system?

"Latency — how long requests take, and I'd specifically track p99, not just average, since averages hide the worst experiences. Traffic — how much demand the system is currently under, usually requests per second. Errors — the rate of failed requests. And Saturation — how close a given resource, like CPU or a connection pool, is to its actual limit, which tends to show rising latency well before a system fully falls over. I'd bring this up specifically when asked about observability, because these four signals together give a genuinely complete picture of system health, and they're what I'd put on the main dashboard for any system I'm on call for."
