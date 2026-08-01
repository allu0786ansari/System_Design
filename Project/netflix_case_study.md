# Netflix as a Full-Stack Case Study
### One system, every layer — a map for interview prep

Why Netflix: it's public (they blog about internals), it has a real ML system (not a toy recommender), and it forces you to touch OS, networking, backend, databases, system design, and MLOps in one coherent story. Once you can narrate "user presses play → what happens," you have a skeleton you can hang almost any interview question on.

---

## 0. The 30-second story (memorize this first)

You open Netflix and press play on a show. In order:

1. **DNS + CDN routing** finds the nearest server to stream video from.
2. **Client app** (frontend) renders the UI, talks to Netflix's API layer.
3. **API Gateway** (Zuul) routes your request to the right **microservice**.
4. Microservices (Java/Spring Boot) fetch data from **databases** — Cassandra for viewing history, MySQL for billing, EVCache/Redis for fast lookups.
5. A **recommendation ML system** decided what's on your homepage before you even searched.
6. The actual video bytes come from **Open Connect**, Netflix's own CDN — not the microservices at all.
7. Everything is monitored, and if something fails, **circuit breakers** and **chaos engineering** practices keep the rest of the app working.

Everything below is zooming into one of these 6 steps.

---

## 1. Frontend Layer

| Concept | Netflix's approach | What to say in interview |
|---|---|---|
| Client diversity | Same product on TV, web, mobile, game consoles — huge device fragmentation | "UI logic needs to be decoupled from device-specific rendering" |
| Rendering | React.js for web (Netflix co-created React Server-Side rendering patterns) | SSR reduces time-to-first-paint on low-power devices |
| BFF (Backend-for-Frontend) | A thin Node.js layer per device type, aggregating calls to microservices | Avoids one giant API that every client must adapt to |
| A/B testing at UI level | Every visual element (thumbnail, row order) is testable | Frontend isn't just UI — it's an experimentation surface |

**Your bridge:** Your Streamlit dashboard is a simplified BFF — it aggregates FastAPI responses into one view for a specific "client" (a human looking at a browser).

---

## 2. Networking Layer

This is the layer freshers usually understand weakest — worth over-indexing here.

**DNS → CDN → Load Balancer → API Gateway**, in order, for every request:

1. **DNS resolution**: `netflix.com` → IP address, routed to the nearest AWS region.
2. **CDN (Open Connect)**: Netflix built its *own* CDN — thousands of servers embedded directly inside ISPs worldwide, caching video files close to users. This is why video streaming doesn't hit their main backend at all — it's a separate delivery network.
3. **TLS handshake**: HTTPS everywhere; TLS terminates at the edge (load balancer) to reduce compute cost on app servers.
4. **HTTP/2 (and QUIC/HTTP3 increasingly)**: multiplexed requests over one connection — critical because a single page load fires dozens of API calls.
5. **Load Balancer (AWS ELB)**: spreads traffic across many instances of a service.
6. **API Gateway (Zuul)**: the single entry point that does routing, auth, rate limiting, and request filtering before traffic reaches microservices.

**Key networking concepts to be able to explain cold:**
- TCP handshake (SYN/SYN-ACK/ACK) vs UDP (used in QUIC for speed)
- Why CDNs reduce latency (physical distance = speed of light delay)
- Load balancing algorithms: round robin, least-connections, consistent hashing
- Rate limiting algorithms: token bucket, leaky bucket, sliding window

**Your bridge:** Your URL shortener's rate limiter is a token-bucket implementation — literally the same primitive Zuul uses at Netflix's edge, just at a smaller scale.

---

## 3. Backend / Microservices Layer

Netflix runs 700+ microservices. Key architectural ideas, not the count:

| Principle | Why it matters |
|---|---|
| **Service per business capability** | "Billing service," "Recommendation service," "Playback service" — each owned, deployed, scaled independently |
| **Loose coupling via REST/gRPC** | Services don't share databases; they call APIs |
| **Statelessness** | Any request can hit any instance — enables horizontal scaling |
| **Circuit breaker pattern (Hystrix)** | If "Recommendations" service is slow/down, the request fails fast and shows a fallback (generic rows) instead of hanging and cascading failure |
| **Service discovery (Eureka)** | Services register themselves; others find them dynamically instead of hardcoded IPs |
| **Async messaging (Kafka)** | Events like "user finished watching X" are published to Kafka, consumed by multiple downstream systems (recs, analytics, billing) without direct coupling |

**The one diagram to be able to draw in an interview:**

```
Client → API Gateway (Zuul) → [Service A] → [Service B]
                                    ↓
                              Circuit Breaker
                                    ↓
                              Fallback response
```

**Your bridge:** Your Airflow-calling-FastAPI-endpoints decision (orchestration separate from execution) is exactly the microservice principle of separation of concerns — you independently arrived at a pattern Netflix uses deliberately.

---

## 4. Database Layer

This is where "SQL vs NoSQL" stops being theoretical and becomes "which database for which access pattern."

| Data | Database used | Why |
|---|---|---|
| Viewing history, user activity (huge write volume, needs to scale horizontally) | **Cassandra** (NoSQL, wide-column) | No single point of failure, tunable consistency, scales linearly by adding nodes |
| Billing, subscriptions (needs strong consistency, transactions) | **MySQL** | ACID guarantees matter for money |
| Caching hot data (session, config, frequently-read data) | **EVCache** (built on Memcached) | Sub-millisecond reads, reduces DB load |
| Search (titles, metadata) | **Elasticsearch** | Full-text search, ranking |

**Concepts to master:**
- **CAP theorem**: Cassandra picks Availability + Partition tolerance over strict Consistency (eventual consistency) — this is *why* your "continue watching" position might lag by a second across devices.
- **Sharding/partitioning**: how Cassandra distributes rows via consistent hashing on a partition key.
- **Replication**: multiple copies of data across data centers for fault tolerance.
- **Cache-aside pattern**: check cache first, fall back to DB, populate cache on miss.

**Your bridge:** Your URL shortener's Redis cache-aside pattern (storing `{"id": int, "url": str}` instead of bare strings) is precisely this pattern — you already have a live example to walk through in an interview instead of reciting theory.

---

## 5. System Design Principles (the "why" layer)

This is what separates "I used Docker" from "I understand distributed systems." Interviewers weight this heavily.

- **Eventual consistency vs strong consistency** — tradeoff decision, not a default.
- **Idempotency** — retrying a failed request shouldn't double-charge or double-write. (Your Celery workers should already think about this.)
- **Chaos Engineering (Chaos Monkey)** — Netflix randomly kills production instances *on purpose* to verify the system survives. This taught the industry that resilience must be tested, not assumed.
- **Graceful degradation** — better to show a stale recommendation list than an error page.
- **Backpressure** — a slow downstream service should signal upstream to slow down, not get flooded.
- **Horizontal vs vertical scaling** — Netflix scales out (more machines), rarely up (bigger machines), because it's cheaper and more fault-tolerant at their scale.

**Interview framing to practice:** "Design a video streaming service" → CDN for delivery, microservices for control plane, Cassandra for scale-heavy writes, circuit breakers for resilience, and a recommendation ML pipeline for personalization. That single sentence, expanded, is basically this whole document.

---

## 6. The ML System (Recommendations)

This is the part most relevant to your track, so go deeper here.

**End-to-end pipeline:**

```
User events (clicks, watches, ratings)
        ↓  (Kafka streaming)
Feature Store (offline batch + online real-time features)
        ↓
Model Training (offline, on Spark/large clusters)
        ↓
Model Registry (versioned models)
        ↓
Serving Layer (low-latency inference API)
        ↓
Ranking + Business Rules (diversity, freshness, licensing constraints)
        ↓
A/B Testing framework (measure if new model actually improves engagement)
        ↓
Homepage rows you see
```

**Key ML system concepts (this is what interviewers actually probe):**

- **Feature store**: same features must be computable both in training (batch) and serving (real-time) — a huge source of production bugs is "training/serving skew."
- **Multiple models stacked**: Netflix doesn't use one recommender — there's row selection, ranking within a row, artwork personalization (yes, even thumbnails are personalized via a separate model), and search ranking, each a distinct model.
- **Offline metrics vs online metrics**: a model can improve offline accuracy (RMSE, precision@k) but *not* improve real engagement — this is why A/B testing in production is the real ground truth, not the offline eval.
- **Explore vs exploit**: showing only "safe" popular content overfits to popularity; some traffic is used to explore under-shown content to avoid feedback loops.
- **Cold start problem**: new users/new titles have no history — solved with metadata-based models until enough interaction data exists.

**Your bridge:** Your Portfolio Optimization Engine's RL agents (PPO/SAC) and LSTM forecasting map directly onto this: your FastAPI `/retrain/*` endpoints are your "serving layer triggering retraining," MLflow is your model registry, and your SB3 zip-file local inference decision mirrors Netflix's choice to avoid per-request heavy remote calls (you optimized for latency the same way they did — that's a great interview story to tell explicitly).

---

## 7. OS & Infrastructure Layer

- **Linux everywhere**: virtually the entire backend runs on Linux (process isolation, cgroups, namespaces — the primitives Docker is built from).
- **Containers**: Netflix built **Titus**, their own container orchestration platform (conceptually similar to Kubernetes) to run and schedule containers across their fleet, integrated with AWS.
- **AWS as the substrate**: Netflix runs almost entirely on AWS (EC2 for compute, S3 for storage) — they don't run their own data centers for compute (only Open Connect CDN boxes are their own hardware).
- **Immutable infrastructure**: new deployments = new container/VM images, not patching servers in place. Rollbacks are just "redeploy the old image."

**OS concepts worth being fluent in:**
- Processes vs threads, and why containers are "lightweight VMs" (namespaces + cgroups, not full hardware virtualization)
- How a Docker container achieves isolation (PID namespace, network namespace, filesystem overlay)
- Why multi-stage Docker builds reduce image size (you've already done this — be ready to explain *why* it matters: smaller attack surface, faster cold starts, cheaper storage/transfer)

**Your bridge:** Your multi-stage Docker builds and per-service `requirements/*.txt` split IS this principle — smaller, single-purpose containers instead of one bloated image. Say exactly this in an interview.

---

## 8. Observability & DevOps

- **CI/CD**: Netflix built **Spinnaker**, an open-source continuous delivery platform, for automated canary deployments (roll out to 1% of traffic, watch metrics, then expand).
- **Monitoring**: **Atlas** is their in-house metrics platform (conceptually your Prometheus/Grafana stack at smaller scale).
- **Canary analysis**: automated statistical comparison of new-version metrics vs old-version before a full rollout — this is deployment risk management, not just "did the build pass."
- **Logging/tracing**: distributed tracing matters enormously once you have hundreds of services — a single user request can touch dozens of services, and you need to reconstruct the full path when something breaks.

**Your bridge:** Your GitHub Actions CI/CD pipelines and Prometheus/Grafana monitoring on both projects are literally junior-scale versions of Spinnaker + Atlas. Frame it that way — "I'm using the same category of tooling as Netflix, at a scale appropriate to my project."

---

## 9. The Interview Cheat-Sheet

When asked "walk me through a system you understand well," structure your answer exactly like this document, top to bottom:

1. **Entry point** — DNS/CDN/Load Balancer/API Gateway
2. **Service layer** — microservices, how they talk (REST/gRPC/events), resilience patterns
3. **Data layer** — which database for which access pattern, and *why* (this is the #1 differentiator between junior and mid-level answers)
4. **Failure handling** — what happens when something goes down (circuit breakers, fallbacks, retries, idempotency)
5. **If ML is relevant** — feature store, training/serving split, offline vs online evaluation
6. **Scale mechanism** — horizontal scaling, caching, sharding
7. **Ops** — CI/CD, monitoring, deployment safety (canaries)

If you can answer "why this database and not another" and "what happens if this service dies" for *any* system — Netflix or your own projects — you're demonstrating system design maturity, which is what separates fresher answers from mid-level ones.

---

## 10. Practice Prompt

Try narrating this out loud, using your own two projects as the concrete example instead of Netflix, in under 3 minutes:

> "In my Portfolio Optimization Engine, a user request would hit FastAPI [API layer], which either serves cached SB3 model inference [caching layer] or, on a schedule via Airflow, triggers retraining by calling a `/retrain` endpoint [orchestration separate from execution — a microservices principle]. Models are versioned through MLflow [model registry], metrics are exposed via Prometheus and visualized in Grafana [observability], and the whole thing is containerized per-service to keep images small and deploys fast [infra]."

That's a fresher-level answer that sounds like someone who's actually internalized system design — because you're not reciting Netflix, you're mapping the same mental model onto something you built and can defend under follow-up questions.
