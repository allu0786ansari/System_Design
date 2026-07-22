SYSTEM DESIGN INTERVIEW — CANDIDATE ANSWERS
PART 5: SECTION E — LLM / GENERATIVE AI SYSTEM DESIGN (96–106)
AND SECTION F — TRADE-OFF AND JUDGMENT QUESTIONS (107–110)


96. Design the serving infrastructure for a large language model API that must handle thousands of concurrent requests with variable output length.

"I'd immediately reframe the problem the way I think interviewers actually want it framed — this isn't about how the model works internally, it's a systems question about serving latency, GPU utilization, and graceful behavior under load.

Because individual inferences take seconds and produce variable-length output, I wouldn't handle requests one at a time per GPU — I'd use continuous batching, where multiple in-flight requests share the same GPU forward passes, and requests that finish early free up their slot for a new request to join mid-batch, rather than waiting for the whole batch to complete together, which would waste enormous GPU capacity on requests that finished generating early.

I'd stream tokens back to the client as they're generated, using something like Server-Sent Events, both for a much better perceived latency and because it lets the client start rendering long before the full response is complete.

For handling load beyond current capacity, I'd use a request queue with an explicit admission control policy rather than letting requests queue unboundedly — reject or defer new requests once queue depth crosses a threshold, with clear backpressure signaled to the caller, rather than accepting everything and letting latency silently degrade for everyone.

I'd also explicitly plan for GPU capacity as a real, first-class constraint — unlike a typical stateless web service, I can't just spin up cheap new instances instantly, so I'd think about a mix of reserved and spot capacity, and a fallback to a smaller, faster model under sustained heavy load rather than letting the primary model's queue grow unboundedly."


97. Design an LLM-based chat system (ChatGPT-style), including token streaming and conversation context management.

"For token streaming, I'd use Server-Sent Events specifically, since this is a one-directional, server-to-client streaming need — the model generates tokens, and the client just needs to receive and render them incrementally, without needing to send anything back over that same channel mid-generation.

For conversation context, each new message needs the relevant prior conversation history included so the model has continuity — but the model has a fixed maximum context length, so I'd design an explicit strategy for long conversations rather than letting them silently break once the limit is hit. I'd either summarize older parts of the conversation into a shorter form once we're approaching the limit, or simply drop the oldest messages, depending on whether losing early context is acceptable for this specific product.

For serving at scale, I'd use the same continuous-batching approach I just described, since this system has the exact same variable-length, GPU-bound characteristics.

I'd also design explicit input and output safety filtering as a required part of the pipeline, not an optional add-on — checking user input before it reaches the model, and checking the model's generated output before it's shown to the user, since a production chat system genuinely needs both, not just one."


98. Design a Retrieval-Augmented Generation (RAG) system for answering questions over a company's internal documents.

"I'd split this clearly into an offline ingestion pipeline and an online query pipeline, since they run on completely different cadences.

For ingestion, source documents get chunked into reasonably sized pieces, since a full document is usually too long to use as one retrieval unit, converted into vector embeddings using an embedding model, and stored in a vector database. This runs initially for all existing documents and then incrementally as documents are added or updated.

For the query path, the user's question is converted into an embedding using that same embedding model, and I search the vector database for the most similar stored chunks using approximate nearest-neighbor search, since exact search over potentially millions of vectors would be too slow. Those retrieved chunks get included directly in the prompt alongside the user's question, and the model generates its answer grounded in that specific retrieved context.

I'd explicitly call out why I'm reaching for RAG instead of fine-tuning here — RAG lets me update the underlying knowledge just by updating the vector database, which is fast and cheap, rather than needing to retrain the model every time a document changes, which is slower and more expensive.

I'd also insist on citing sources in the final answer, carrying each retrieved chunk's source metadata through to the response, and I'd specifically evaluate retrieval quality separately from generation quality, since a wrong answer could stem from either failing to retrieve the right document, or retrieving it correctly but the model still generating something wrong — those are two different bugs with two different fixes."


99. Design an inference-batching system for a single GPU that can handle up to 100 inputs per batch while users wait synchronously.

"Since users are waiting synchronously, I have a real latency budget I can't exceed, so I'd use dynamic batching with a bounded wait — collect incoming requests into a batch, but cap how long I'll wait for the batch to fill before running it, rather than always waiting for a full 100 inputs, since that could mean a very fast, low-traffic request waits an unacceptably long time behind an unfilled batch.

Concretely, I'd trigger the batch to run either once it reaches the max batch size, or once a maximum wait time elapses, whichever happens first — that gives me good GPU utilization during high traffic, when batches fill quickly, while still bounding worst-case latency during low traffic.

I'd also group requests by similar expected computation cost where possible — for example, batching similarly-sized inputs together — since padding wildly different-sized inputs into one batch wastes compute on the shorter ones.

For monitoring, I'd track GPU utilization and the batch-fill rate as key operational metrics, since if batches are consistently small, I'm underutilizing expensive GPU capacity and might want a longer max wait time, and if the wait time is consistently maxing out, I likely need either more capacity or a shorter max batch size to protect latency."


100. Design a system to distribute one enormous file (e.g., model weights) to thousands of machines over a constrained network link.

"The naive approach — every machine pulling the file directly from one central source — would completely saturate that source and the constrained link, so I'd design this explicitly to avoid a single bottleneck.

I'd use a peer-to-peer, BitTorrent-style distribution approach: the file is split into chunks, and once a machine has downloaded a given chunk, it becomes a source for that chunk for other machines too, rather than every machine independently pulling every chunk from the original source. This means download bandwidth scales with the number of participating machines instead of being capped by one source's outbound bandwidth.

I'd organize this in a tree or hierarchical topology matching the actual network topology where possible — machines in the same rack or same data center preferentially pull from each other rather than crossing the constrained link repeatedly for the same data, since intra-datacenter bandwidth is typically far cheaper and faster than the constrained link itself.

I'd also checksum each chunk so a machine can detect and re-request a corrupted chunk from a different peer, rather than a single bad transfer silently corrupting model weights on that machine, and I'd track overall rollout progress centrally so I know when the distribution across all thousands of machines has actually completed successfully."


101. Design safeguards and guardrails for an AI agent that can take real actions on a user's behalf.

"I'd start from the principle that giving an autonomous, LLM-driven agent the ability to actually execute real actions is a meaningfully higher-stakes system than a purely conversational one, so I'd design the guardrails as genuinely non-optional infrastructure, not an afterthought layered on top later.

For the tool-calling interface itself, I'd enforce a strict, explicit allowlist of exactly what actions the agent is permitted to take, with each action's parameters validated before execution, rather than letting the agent construct and run arbitrary, unconstrained commands.

For higher-risk or irreversible actions — anything involving money, deleting data, or sending communications externally — I'd require explicit human confirmation before execution, rather than letting the agent act fully autonomously, and I'd draw that line based on how reversible and how consequential the action actually is.

I'd wrap every individual tool call with the same resilience patterns I'd use for any unreliable dependency — timeouts, retries with backoff, and a circuit breaker that stops the agent from repeatedly retrying an action that's clearly, persistently failing.

I'd also log every single action the agent takes, along with its reasoning at that step, both for auditability after the fact and so a human can review and understand exactly why a particular action was taken if something goes wrong — and I'd design a clear, simple way for a human to intervene mid-task and halt the agent immediately if its behavior looks wrong."


102. Design a multi-agent AI platform where several specialized agents collaborate on a complex task.

"I'd choose between orchestration and choreography based mainly on how many steps and how much visibility I need. For anything beyond two or three coordinated steps, I'd lean toward orchestration — a central orchestrator explicitly directs which specialized agent handles each step and in what order — since that gives me one single place to monitor, trace, and debug the overall flow, which matters a lot given how unpredictable individual agent steps can be.

For shared state, I'd use a common data store that all agents read from and write to as they progressively work through a task together, alongside the same conversation-context-management approach I'd use in a single-agent chat system for each individual agent's own working memory.

Given that language models are fundamentally probabilistic, not deterministic, I'd wrap every agent-to-agent and agent-to-tool interaction with the same resilience toolkit as the guardrails question — timeouts, retries with backoff, and circuit breakers — specifically so one confused or malfunctioning agent step can't derail or stall the entire multi-step task.

I'd also invest heavily in tracing across the whole multi-agent flow, since debugging this is genuinely harder than tracing a normal deterministic microservices call chain — the same input can legitimately produce different execution paths on different runs — and I'd define a clear fallback strategy, like escalating to a human reviewer, for when an individual step repeatedly fails or produces low-confidence output."


103. Design an evaluation framework for judging the quality of LLM-generated outputs at scale.

"I'd say upfront that this is genuinely one of the hardest parts of building any LLM-powered system, since unlike a classification model's clean right-or-wrong answer, most LLM outputs are open-ended and don't have one single correct answer to check against.

I'd combine three approaches. First, automated metrics where they genuinely apply — things like checking factual claims against retrieved source documents in a RAG system, or simple structural checks like whether the output is valid, well-formed JSON if that's the required format.

Second, using a separate 'judge' LLM to score outputs against a defined rubric — this scales much better than pure human review, though I'd validate the judge model's own scores against a smaller set of genuine human-labeled examples periodically, to confirm the judge itself is actually reliable and not just confidently wrong in a consistent way.

Third, targeted human review for a sampled subset of outputs, especially for anything high-stakes or where the automated and judge-based signals disagree or show low confidence.

I'd track this evaluation continuously in production, not just once before launch, since model behavior can shift as the underlying model provider updates it, or as real user queries drift from what I originally tested against, and I'd treat sustained evaluation-score degradation as a genuine, actionable alert, the same way I'd treat degrading accuracy in a traditional ML monitoring system."


104. Design a system to detect and filter harmful or policy-violating AI-generated content in real time.

"I'd apply filtering at two points — before the model generates anything, checking the input prompt for clearly malicious intent, and after generation, checking the actual output before it's ever shown to the user, since relying on only one of these leaves a real gap.

For the filtering mechanism itself, I'd use a layered approach: fast, cheap checks first — a classifier or keyword/pattern matching for the most clear-cut violations — catching obvious cases with minimal added latency, and a more careful, slower check only for genuinely ambiguous content, balancing the real latency cost of safety filtering against the thoroughness of the check.

Given that this needs to run in the live request path, I'd care a lot about the added latency cost specifically, and I'd look at running the input check and the actual generation concurrently where possible, rather than strictly sequentially, to avoid needlessly adding the full input-check latency on top of the generation latency for every single request.

I'd also treat this as needing the exact same ongoing monitoring and retraining discipline as fraud detection — people actively probing for ways around content filters is a real, continuous adversarial dynamic, not a one-time problem that's solved and then stays solved, so I'd track filter bypass attempts as an explicit signal feeding back into improving the filters over time."


105. Design a prompt-management and versioning system for an organization running many LLM-powered features.

"I'd treat prompts as a genuine first-class artifact, versioned and managed with the same discipline as code or a trained model, not as a string hardcoded inside application logic, since that makes changes risky and completely untraceable.

I'd store each prompt template with an explicit version number, and have application code reference prompts by name and version rather than embedding the actual prompt text directly, so a prompt can be updated centrally without needing a full code deployment.

For rolling out prompt changes safely, I'd apply the exact same gradual-rollout discipline I'd use for a model change — test a new prompt version against a held-out evaluation set first, then roll it out to a small percentage of live traffic, monitoring the same evaluation-framework signals I described earlier, before fully replacing the previous version.

I'd also track which specific prompt version was used for every single generated output, alongside the output itself, so if a bug or quality regression is later reported, I can trace it back to exactly which prompt version was actually responsible, the same way I'd want to trace a bug back to a specific model or code version in a more traditional system."


106. Design a semantic/vector search service to power a RAG application.

"For ingestion, I'd chunk source documents — I'd specifically think through chunk size and overlap here, since chunks that are too large dilute the specific relevant information with irrelevant surrounding text, while chunks that are too small can lose important context, so I'd typically use a few hundred tokens per chunk with a modest overlap between consecutive chunks so relevant information isn't awkwardly split right at a chunk boundary.

Each chunk gets converted into an embedding using an embedding model, and stored in a vector database, using an index structure like HNSW for efficient approximate nearest-neighbor search at query time rather than exact search, which wouldn't scale to a large document collection at real query latency.

For retrieval quality specifically, I'd go beyond pure vector similarity — I'd combine dense vector search with traditional keyword-based search, since dense embeddings alone can miss exact-term matches that clearly matter, like a specific product code or an exact name; combining both, sometimes called hybrid search, tends to retrieve better results than either alone. I'd also add a re-ranking step using a more precise, heavier cross-encoder model on just the top candidates from the initial retrieval, trading a bit of extra latency for meaningfully better final ranking quality.

I'd evaluate this retrieval layer independently from the downstream generation step, using a held-out set of queries with known relevant documents, specifically so I can tell whether a poor final answer is a retrieval problem or a generation problem, rather than only ever being able to judge the system's final output as one opaque whole."


SECTION F — TRADE-OFF AND JUDGMENT QUESTIONS (107–110)


107. "Your system just got 10x more traffic overnight — walk me through what breaks first, and how you'd fix it."

"I'd walk through my own design in the order things would actually fail, not randomly. The first thing to break is almost always the database, specifically its write capacity or connection pool, since that's usually the least horizontally scalable piece of any design I've proposed — my application servers and cache are typically already designed to scale out easily, but the primary database often isn't.

My first fix would be read replicas, if I haven't already got them, to offload read traffic from the primary — that's the cheapest, fastest lever. If writes themselves are the bottleneck, that's when I'd reach for sharding, accepting the added complexity of cross-shard queries becoming harder in exchange for real write scalability.

I'd also expect my cache to come under much heavier pressure, and I'd specifically worry about cache stampede risk if a hot key expires under this much higher load — so I'd add jittered TTLs and a locking mechanism around cache repopulation if I hadn't already.

I'd explicitly say that I wouldn't try to redesign the whole system reactively under this pressure — I'd first add capacity and caching at the current bottleneck to buy time, then plan the more structural fix, like sharding, as a proper follow-up rather than a panic response."


108. "An entire AWS region just went down — what happens to your system, and how would you have designed for this?"

"I'd first be honest about what my original design actually supports — if I designed for a single region with just multi-AZ redundancy, then a full regional outage means real downtime, and I'd say that plainly rather than pretending my design handles something it doesn't.

If disaster recovery genuinely matters for this system, I'd have set up at least a warm standby in a second region — a scaled-down but functional copy of production, kept continuously updated via asynchronous cross-region replication. On a regional failure, I'd fail over by updating DNS or a global load balancer to redirect traffic to the standby region, then scale that region's capacity up to handle full production load.

I'd be explicit about the real cost of this: asynchronous replication means some very recent writes — whatever hadn't replicated yet — could be lost in the failover, so I'd state my actual RPO honestly rather than claiming zero data loss, and I'd only justify the added cost and complexity of an active-active, near-zero-RPO setup if the business genuinely cannot tolerate that small window of potential loss, like in the banking system we discussed earlier.

I'd also mention that this failover process needs to actually be tested periodically, not just designed and left untested, since an untested failover plan is one I genuinely wouldn't trust to work correctly under real pressure."


109. "What's the actual cost profile of this design, and where would you cut costs if you had to reduce your cloud bill by 50%?"

"I'd first identify where the cost is actually concentrated in my design, rather than guessing — usually that's either compute capacity provisioned for peak load that sits mostly idle off-peak, data transfer and storage costs, or, if this is an ML system, GPU inference costs specifically.

For idle peak capacity, I'd lean harder on auto-scaling and, where the workload tolerates it, spot or preemptible instances for anything that isn't strictly latency-critical, rather than provisioning fixed capacity for worst-case load at all times.

For storage, I'd apply tiering — moving infrequently accessed data to cheaper cold storage automatically via a lifecycle policy, rather than keeping everything in the expensive, fast tier indefinitely.

For caching, I'd specifically evaluate whether I'm caching aggressively enough — every cache hit is a database or compute cost avoided, so under real cost pressure I'd look hard at increasing cache TTLs and cache coverage for anything that can tolerate slightly staler data.

And if this is an ML-heavy system, I'd look specifically at whether every prediction genuinely needs the full, most expensive model — a smaller, cheaper model for the bulk of easy cases, with the expensive model reserved for genuinely ambiguous cases, the same tiered approach I described for fraud detection, is often a very real, meaningful cost lever that doesn't require sacrificing much overall quality."


110. "You proposed [X]; what would you have done differently five years ago, and what would you do differently five years from now?"

"Five years ago, I'd say I'd likely have leaned more heavily on a single relational database with vertical scaling for longer before reaching for sharding, since a lot of today's managed, easy-to-operate horizontally-scalable databases either didn't exist yet or weren't nearly as mature and battle-tested as they are now — the operational cost of running your own sharded cluster used to be a much bigger deterrent than it is today.

Five years from now, I'd expect the boundary between 'build this yourself' and 'use a managed service' to keep shifting further toward managed services for more and more of what I've described today — a lot of what I'm proposing as custom infrastructure, like parts of the caching layer or the event bus, is increasingly available as a fully managed, serverless offering, and I'd expect that trend to continue, meaningfully lowering the operational burden of designs like this one.

I'd also say that if this system has any AI-powered features at all, I'd expect the specific ML infrastructure choices I'd make today to look meaningfully different in five years, simply because that space is moving so fast right now — so I'd deliberately keep those specific pieces of the design more loosely coupled and easier to swap out than the more stable, foundational infrastructure pieces, precisely because I have less confidence those specific choices will still be the right ones five years out."
