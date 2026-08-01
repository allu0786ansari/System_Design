# The Netflix Tech Stack — A Study Glossary

For every technology below, the format is always the same:

> **What it is** → plain-language definition
> **What it's used for** → the general problem it solves (true anywhere, not just Netflix)
> **How Netflix uses it** → the concrete application in this case study

Read top to bottom once, then use it as a lookup table before interviews.

---

## Layer 1: Frontend

### React.js
**What it is:** A JavaScript library for building user interfaces out of reusable components. Instead of manually updating the webpage when data changes, you describe *what the UI should look like* for a given state, and React updates the actual page (the DOM) for you.
**What it's used for:** Building interactive, component-based UIs that update efficiently without reloading the whole page.
**How Netflix uses it:** Netflix's web player and browsing UI are built in React. Netflix engineers actually helped pioneer **server-side rendering (SSR)** with React — rendering the initial HTML on the server so low-power devices (older TVs, budget phones) get a fast first paint instead of waiting for a large JavaScript bundle to load and render everything client-side.

### Node.js (as a BFF — Backend-for-Frontend)
**What it is:** A JavaScript runtime that lets you run JS outside a browser, commonly used to build servers.
**What it's used for:** Creating a thin server layer whose only job is to aggregate/shape data for a *specific* client (web vs TV vs mobile), so each device gets exactly the data it needs in one call instead of many.
**How Netflix uses it:** Each device type talks to its own small Node.js layer, which calls multiple backend microservices, combines the results, and sends the frontend one clean payload. This avoids forcing every backend microservice to know about every possible device.

### A/B Testing (frontend-level)
**What it is:** Showing two (or more) versions of something to different user segments and measuring which performs better.
**What it's used for:** Making UI/product decisions based on real user behavior instead of opinion.
**How Netflix uses it:** Almost everything visible — thumbnail images, row order on the homepage, even font size — is a candidate for an A/B test. This turns the frontend into an experimentation surface, not just a display layer.

---

## Layer 2: Networking

### DNS (Domain Name System)
**What it is:** The system that translates a human-readable domain (`netflix.com`) into a machine-usable IP address.
**What it's used for:** Every single request on the internet needs this translation step before it can even begin.
**How Netflix uses it:** DNS resolution for Netflix is also **geo-aware** — it can route you to the nearest available data center/region, which is the first step in reducing latency before your request even leaves your device.

### CDN (Content Delivery Network) — Open Connect
**What it is:** A network of servers distributed geographically, caching content close to where users actually are, so data doesn't have to travel from one central server every time.
**What it's used for:** Reducing latency and bandwidth cost for content that many users request repeatedly (videos, images, static files).
**How Netflix uses it:** Netflix built its **own CDN called Open Connect** — literally placing their own servers physically inside ISP networks around the world. When you press play, the video bytes stream from a nearby Open Connect box, *not* from Netflix's main backend at all. This is why video playback and "browsing the app" are architecturally two almost-separate systems.

### TLS / HTTPS
**What it is:** A protocol that encrypts data in transit between client and server, and verifies the server's identity via certificates.
**What it's used for:** Preventing eavesdropping and tampering on any network request.
**How Netflix uses it:** All traffic is encrypted. TLS is typically **terminated at the edge** (load balancer/gateway level) — meaning the encryption/decryption workload happens once at the front door, not repeated at every internal microservice, saving compute cost deep in the system.

### HTTP/2 (and increasingly HTTP/3 / QUIC)
**What it is:** A version of the HTTP protocol that allows multiplexing — many requests and responses over a single connection at once — instead of HTTP/1.1's one-request-per-connection-at-a-time limitation.
**What it's used for:** Reducing the overhead and latency of loading pages that need dozens of API calls to render (which is basically every modern app).
**How Netflix uses it:** A single Netflix homepage load fires many API calls (rows of recommendations, images, metadata). HTTP/2 lets these all travel efficiently over one connection instead of opening many separate slow connections.

### TCP vs UDP
**What it is:** TCP is a connection-oriented protocol guaranteeing ordered, reliable delivery (via handshake: SYN → SYN-ACK → ACK). UDP is connectionless and doesn't guarantee delivery or order, but is faster since it skips those guarantees.
**What it's used for:** TCP is used when correctness matters more than speed (API calls, page loads). UDP is used when speed matters more than occasional data loss (live video, gaming).
**How Netflix uses it:** Regular API traffic uses TCP (reliability matters — you don't want a corrupted "add to watchlist" request). Newer video delivery protocols increasingly use QUIC (built on UDP) because a dropped video packet is less costly than the delay TCP's retransmission would cause.

### Load Balancer
**What it is:** A component that distributes incoming requests across multiple server instances instead of sending them all to one machine.
**What it's used for:** Preventing any single server from being overwhelmed, and enabling horizontal scaling (add more machines instead of one bigger machine).
**How Netflix uses it:** AWS Elastic Load Balancers sit in front of every service, distributing traffic across many instances of that service. Common algorithms: round-robin (rotate evenly), least-connections (send to the least busy server), consistent hashing (used when you want the *same* client to consistently land on the *same* server, useful for caching).

### API Gateway — Zuul
**What it is:** A single entry point that all external traffic passes through before reaching internal services — it handles routing, authentication, rate limiting, and request filtering.
**What it's used for:** Centralizing cross-cutting concerns (auth, logging, rate limiting) instead of duplicating that logic in every microservice.
**How Netflix uses it:** **Zuul** is Netflix's own API Gateway (open-sourced). Every request from a client is routed through Zuul first, which decides which internal microservice should handle it, applies rate limits, and can reject malicious/malformed requests before they ever reach business logic.

### Rate Limiting
**What it is:** A technique to cap how many requests a client can make in a given time window.
**What it's used for:** Protecting backend services from being overwhelmed (by bugs, abuse, or traffic spikes).
**How Netflix uses it:** Enforced at the Zuul gateway layer using algorithms like **token bucket** (each client has a "bucket" of tokens that refills over time; each request consumes a token; no tokens = request rejected/delayed) or **sliding window** counters.

---

## Layer 3: Backend / Microservices

### Microservices Architecture
**What it is:** An architectural style where an application is split into many small, independently deployable services, each owning one business capability, rather than one large ("monolithic") codebase.
**What it's used for:** Allowing teams to develop, deploy, and scale parts of a system independently, and preventing one bug from taking down the entire application.
**How Netflix uses it:** Netflix runs 700+ microservices — separate services for billing, playback, recommendations, user profiles, etc. Each can be deployed and scaled without touching the others.

### REST and gRPC (service communication)
**What it is:** REST is an API style using HTTP verbs (GET/POST/PUT/DELETE) and typically JSON payloads. gRPC is a faster, binary protocol (using Protocol Buffers) often used for service-to-service communication where performance matters more than human-readability.
**What it's used for:** Letting independent services talk to each other over the network in a standardized way.
**How Netflix uses it:** External client-facing APIs tend to be REST-like (easy for many client types to consume); internal service-to-service calls increasingly use gRPC for lower latency and smaller payloads at very high call volumes.

### Circuit Breaker Pattern — Hystrix
**What it is:** A pattern where a service call is monitored, and if a dependency is failing or too slow, calls to it are automatically "tripped open" — failing fast with a fallback response instead of waiting and potentially cascading the failure upstream.
**What it's used for:** Preventing one failing service from bringing down the entire chain of services that depend on it.
**How Netflix uses it:** Netflix built and open-sourced **Hystrix** for exactly this. If the "Recommendations" service is down or slow, the circuit breaker trips, and the homepage falls back to showing generic/cached rows instead of hanging indefinitely or crashing.

### Service Discovery — Eureka
**What it is:** A system where services register themselves (with their current network address) at startup, and other services look them up dynamically instead of relying on hardcoded IP addresses.
**What it's used for:** Making a system resilient to services being added, removed, or having their addresses change constantly (which happens constantly in cloud environments where instances come and go).
**How Netflix uses it:** **Eureka** is Netflix's own service registry. When a new instance of a service spins up, it registers itself with Eureka; other services query Eureka to find where to send requests.

### Async Messaging — Kafka
**What it is:** A distributed event-streaming platform. Services publish "events" (facts that happened) to topics, and other services subscribe and react, without the publisher needing to know who's listening.
**What it's used for:** Decoupling services from each other in time and structure — the publisher doesn't wait for consumers, and you can add new consumers later without changing the publisher.
**How Netflix uses it:** When you finish watching a show, that event is published to Kafka. Multiple independent systems consume it: the recommendation system (updates your profile), analytics (updates viewing stats), and billing/usage tracking — all without those systems being directly wired to each other.

---

## Layer 4: Databases

### Cassandra (NoSQL, wide-column store)
**What it is:** A distributed, horizontally-scalable NoSQL database designed for very high write throughput and no single point of failure, at the cost of only offering "eventual consistency" by default rather than strict immediate consistency.
**What it's used for:** Storing massive-scale, write-heavy data where availability matters more than instant consistency across every replica.
**How Netflix uses it:** Viewing history, "continue watching" position, and user activity logs — all extremely high write-volume, needed across many data centers — live in Cassandra. It's why your watched-position might take a second to sync between your phone and your TV: that's eventual consistency in action, not a bug.

### MySQL (relational / SQL database)
**What it is:** A relational database using structured tables, rows, and SQL queries, offering ACID guarantees (Atomicity, Consistency, Isolation, Durability).
**What it's used for:** Data where correctness and transactional integrity matter more than raw write throughput — most importantly, anything involving money.
**How Netflix uses it:** Billing and subscription data. You do not want "eventual consistency" on whether someone was charged — this needs strict transactional guarantees.

### EVCache (built on Memcached)
**What it is:** An in-memory key-value cache — data lives in RAM rather than on disk, for extremely fast (sub-millisecond) reads.
**What it's used for:** Reducing load on the primary database by serving frequently-requested data straight from memory.
**How Netflix uses it:** Session data, user configuration, and other frequently-read data is cached in EVCache. The standard pattern is **cache-aside**: check the cache first; on a miss, read from the database, then populate the cache for next time.

### Elasticsearch
**What it is:** A search engine built for fast full-text search and ranking across large datasets, distinct from a regular database's exact-match queries.
**What it's used for:** Powering search bars and any feature needing fuzzy/ranked text matching (typos, partial matches, relevance scoring).
**How Netflix uses it:** Searching titles and metadata — when you type a partial or misspelled show name, Elasticsearch is what finds and ranks the right result.

### CAP Theorem (concept, not a tool)
**What it is:** A principle stating a distributed system can only fully guarantee two of three properties at once: **C**onsistency (every read sees the latest write), **A**vailability (every request gets a response), **P**artition tolerance (the system keeps working even if network communication between nodes breaks).
**What it's used for:** Explaining *why* a database was chosen — every distributed database is making a tradeoff here, not achieving all three.
**How Netflix uses it:** Cassandra prioritizes Availability + Partition tolerance over strict Consistency, which is why viewing history syncs "eventually" rather than instantly — Netflix decided that tradeoff is worth it because a one-second sync delay is much less costly than the service becoming unavailable during a network hiccup.

### Sharding / Partitioning
**What it is:** Splitting a large dataset across multiple machines based on some key (a "partition key"), so no single machine holds all the data.
**What it's used for:** Scaling storage and throughput horizontally — adding more machines instead of needing one impossibly large machine.
**How Netflix uses it:** Cassandra distributes rows across nodes using **consistent hashing** on a partition key (e.g., user ID) — this also makes it easy to add new nodes later without reshuffling all existing data.

### Replication
**What it is:** Keeping multiple copies of the same data on different machines (often in different physical locations).
**What it's used for:** Fault tolerance — if one machine or data center fails, copies elsewhere keep the system running.
**How Netflix uses it:** Data is replicated across multiple AWS availability zones and regions, so a data-center-level outage doesn't take down the whole service.

---

## Layer 5: System Design Principles (concepts, applied everywhere above)

### Idempotency
**What it is:** A property where performing the same operation multiple times has the same effect as performing it once.
**What it's used for:** Safely retrying failed requests (network timeouts, dropped connections) without causing duplicate side effects like double-charging a card.
**How Netflix uses it:** Payment and subscription-update requests are designed so a retried request (e.g., due to a timeout) doesn't double-process the same charge.

### Chaos Engineering — Chaos Monkey
**What it is:** The practice of deliberately injecting failures (killing servers, adding latency, cutting network connections) into a *production* system to verify it survives, rather than assuming it will.
**What it's used for:** Proving resilience empirically instead of theoretically — most systems that "should" handle failure gracefully actually don't, until tested.
**How Netflix uses it:** **Chaos Monkey** randomly terminates production instances during business hours. If the rest of the system doesn't degrade gracefully, that's treated as a bug to fix, not an acceptable risk.

### Graceful Degradation
**What it is:** Designing a system so that when a non-critical part fails, the overall experience gets slightly worse rather than completely breaking.
**What it's used for:** Prioritizing core functionality staying up over every feature being perfect all the time.
**How Netflix uses it:** If the recommendation engine is down, you still see a homepage — just with generic/cached rows instead of personalized ones, rather than an error page.

### Backpressure
**What it is:** A mechanism where a slow or overloaded downstream system can signal upstream systems to slow down, instead of being flooded with more requests than it can handle.
**What it's used for:** Preventing cascading overload in systems made of many connected services.
**How Netflix uses it:** Applied within their streaming/event pipelines (e.g., Kafka consumers) so a temporarily slow consumer doesn't get overwhelmed by a burst of events it can't process fast enough.

### Horizontal vs Vertical Scaling
**What it is:** Vertical scaling = making one machine bigger (more CPU/RAM). Horizontal scaling = adding more machines and distributing load across them.
**What it's used for:** Choosing how a system grows to handle more traffic or data.
**How Netflix uses it:** Netflix scales almost entirely horizontally — more, smaller instances rather than fewer, bigger ones — because it's cheaper at massive scale and more fault-tolerant (losing one of a thousand small machines barely matters; losing one giant machine is catastrophic).

---

## Layer 6: The ML System (Recommendations)

### Feature Store
**What it is:** A centralized system that computes and stores features (input variables for ML models) so the *same* feature values and computation logic are used in both model training and live serving.
**What it's used for:** Preventing "training/serving skew" — a very common real-world bug where a model performs great in training but poorly in production because features were computed slightly differently.
**How Netflix uses it:** Features like "average watch time in last 7 days" are computed once, consistently, and made available both to offline training jobs and the real-time serving layer that ranks your homepage.

### Model Registry
**What it is:** A system for storing, versioning, and tracking trained ML models, including metadata about how each was trained and how it performed.
**What it's used for:** Being able to roll back to a previous model version, compare model versions, and know exactly what's running in production.
**How Netflix uses it:** Every recommendation model version is tracked, so a bad model rollout can be identified and rolled back — conceptually the same role MLflow plays in a smaller-scale project.

### Serving Layer (model inference)
**What it is:** The system that takes a trained model and returns predictions in response to real-time requests, optimized for low latency.
**What it's used for:** Turning an offline-trained model into something that can respond in milliseconds to a live user request.
**How Netflix uses it:** When you open the app, the serving layer ranks and personalizes rows (and even artwork) in real time, within a tight latency budget, using precomputed features from the feature store rather than recomputing everything from scratch.

### Offline vs Online Evaluation
**What it is:** Offline evaluation measures model quality on historical data using metrics like RMSE or precision@k. Online evaluation (A/B testing) measures actual impact on live user behavior.
**What it's used for:** Catching the gap between "looks good on paper" and "actually improves the product" — a model can win offline and lose online, or vice versa.
**How Netflix uses it:** A new recommendation model is only shipped fully after it wins in an A/B test against the current production model on real engagement metrics — not just because it scored better on a held-out dataset.

### Explore vs Exploit
**What it is:** A tradeoff between showing what's already known to perform well ("exploit") versus showing less-certain options to gather more data and avoid narrowing too early ("explore").
**What it's used for:** Preventing feedback loops where a system only ever reinforces what's already popular, starving new or niche content of any chance to be discovered.
**How Netflix uses it:** A portion of recommendation traffic deliberately surfaces less-obvious content, to keep the system's understanding of what people like from stagnating.

### Cold Start Problem
**What it is:** The challenge of making recommendations for a new user (no history) or a new item (no interaction data yet).
**What it's used for:** Understanding why "day one" personalization can't rely purely on collaborative filtering (behavior-based) recommendations.
**How Netflix uses it:** New users are shown recommendations based on onboarding preferences and content metadata (genre, cast, etc.) until enough real viewing behavior accumulates to personalize more precisely.

---

## Layer 7: OS & Infrastructure

### Linux, Processes, and Namespaces/cgroups
**What it is:** Linux provides the underlying primitives — **namespaces** isolate what a process can see (its own filesystem view, network stack, process list), and **cgroups** limit what resources (CPU, memory) a process can consume.
**What it's used for:** These two primitives together are literally what makes containers possible — a container isn't a lightweight VM, it's a regular Linux process with restricted visibility and resource limits.
**How Netflix uses it:** The entire backend runs on Linux. Every microservice, whether in a container or not, is ultimately just Linux processes managed by these isolation primitives.

### Containers (Docker-style)
**What it is:** A packaging format that bundles an application with everything it needs to run (dependencies, runtime, config) into a single portable unit, isolated from the host system using the namespaces/cgroups above.
**What it's used for:** "Works on my machine" problems disappear because the container carries its own environment; also enables fast, consistent deployment across many machines.
**How Netflix uses it:** Services are packaged as containers for consistent deployment across their fleet of machines.

### Container Orchestration — Titus (conceptually like Kubernetes)
**What it is:** A system that decides *where* and *how many* containers to run across a fleet of machines, restarts failed containers, and handles scaling automatically.
**What it's used for:** Removing the need to manually decide which physical/virtual machine runs which container — the orchestrator handles scheduling, health checks, and scaling.
**How Netflix uses it:** **Titus** is Netflix's own container orchestration platform, tightly integrated with AWS, scheduling containers across their compute fleet.

### Cloud Infrastructure — AWS (EC2, S3)
**What it is:** EC2 provides on-demand virtual machines (compute); S3 provides scalable object storage (files/blobs).
**What it's used for:** Renting compute and storage on demand instead of owning physical data centers, and scaling up/down based on real-time need.
**How Netflix uses it:** Almost the entire backend and compute layer runs on AWS. The one major exception: **Open Connect** (their CDN) uses Netflix's own physical hardware, because owning that layer gave them control and cost advantages at their video-delivery scale.

### Immutable Infrastructure
**What it is:** A deployment philosophy where servers/containers are never modified in place after creation — instead, a new image is built and the old one is fully replaced.
**What it's used for:** Eliminating "configuration drift" (servers slowly becoming different from each other due to ad-hoc manual changes) and making rollbacks trivial (just redeploy the previous image).
**How Netflix uses it:** New deployments always ship as new container/AMI images; rolling back a bad deploy is simply redeploying the last known-good image rather than trying to "undo" changes on live servers.

---

## Layer 8: DevOps & Observability

### CI/CD — Spinnaker
**What it is:** Continuous Integration (automatically testing code changes) and Continuous Delivery (automatically deploying tested changes), reducing manual steps in shipping software.
**What it's used for:** Shipping changes safely and frequently instead of large, risky, infrequent releases.
**How Netflix uses it:** **Spinnaker**, Netflix's own open-sourced CD platform, automates deployments including **canary releases** — rolling a new version out to a small percentage of traffic first.

### Canary Deployment
**What it is:** Releasing a new version to a small subset of users/traffic first, comparing its metrics against the old version, and only expanding the rollout if metrics look healthy.
**What it's used for:** Catching bad releases before they affect all users, with an automatic (not just manual) statistical comparison.
**How Netflix uses it:** Spinnaker automates the canary analysis — comparing error rates, latency, etc. between old and new versions — before deciding whether to continue the rollout or automatically roll back.

### Monitoring — Atlas
**What it is:** A metrics platform for collecting, storing, and visualizing time-series data (like request rates, error rates, latency) across a system.
**What it's used for:** Knowing what's happening across hundreds of services in real time, and alerting when something looks wrong.
**How Netflix uses it:** **Atlas** is Netflix's in-house metrics platform, conceptually doing the same job Prometheus + Grafana does at smaller scale — collecting metrics and visualizing them on dashboards.

### Distributed Tracing
**What it is:** A technique for tracking a single request as it flows through multiple services, so you can reconstruct its full path and pinpoint where it slowed down or failed.
**What it's used for:** Debugging systems where one user action triggers a chain of calls across many independent services — without tracing, you can't tell *which* service in the chain caused a slow or failed request.
**How Netflix uses it:** Every request carries a trace ID as it moves through dozens of microservices, letting engineers reconstruct the exact path and timing when diagnosing an incident.

---

## How to Use This Document

1. **First pass:** read start to finish once, just to build the map.
2. **Second pass:** for each term, cover the "How Netflix uses it" column and try to recall it yourself — that's the part interviewers actually probe.
3. **Third pass:** for each term, replace "How Netflix uses it" with "How I used it (or could use it) in my own project" — this is what turns memorized facts into a confident, personal interview answer.
