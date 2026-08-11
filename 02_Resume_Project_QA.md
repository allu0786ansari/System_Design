# HyperVerge MLE — Round 1 (Automated AI Interview) Prep
## File 2 of 3: Resume & Project Deep-Dive Q&A

> This round is explicitly **"AI fundamentals, based on resume."** The AI interviewer will very likely pick specific lines off your resume (LexAI, Portfolio Optimization Engine, URL Shortener, the Qclairvoyance quantum internship) and ask you to explain, justify, and defend them. Practice saying these out loud in under 90 seconds each — don't just recognize them, be able to *narrate* them.

---

## Section 1 — AI-Powered Portfolio Optimization Engine

**Q1. Walk me through this project in 60 seconds.**
A: "I built an end-to-end quantitative ML platform that allocates a portfolio across 40 Nifty50 and S&P500 assets using reinforcement learning — PPO and SAC agents — trained to maximize risk-adjusted return. I benchmarked it against classical baselines (Markowitz, Black-Litterman, equal-weight) using walk-forward backtesting, and the RL agent achieved a Sharpe ratio of about 1.4 versus 0.6 for the benchmark, with the outperformance statistically significant at p<0.05 across 11 rolling evaluation windows. I also built a production MLOps pipeline around it — MLflow for experiment tracking, Airflow for scheduled retraining, Prometheus for monitoring, and a FastAPI serving layer with sub-200ms latency."

**Q2. Why reinforcement learning instead of a supervised model or classical optimization for portfolio allocation?**
A: Classical mean-variance optimization (Markowitz) assumes you can accurately estimate expected returns and covariance ahead of time, which is notoriously unstable and error-prone in real markets, and it's a static, single-period optimization. RL instead learns a *dynamic policy* — it directly optimizes a long-horizon risk-adjusted reward through trial-and-error interaction with historical market data, and can adapt allocation as the state (recent price/volatility/momentum features) changes, without needing to explicitly forecast returns.

**Q3. Why Sortino instead of Sharpe as the RL reward signal?**
A: Sharpe penalizes *all* volatility symmetrically, including large positive days — which perversely discourages the agent from letting winning trades run, since a big up-day increases standard deviation and shrinks the Sharpe ratio. Sortino only penalizes downside deviation, so occasional large gains aren't punished — a much better fit for what we actually want a trading policy to do. Making this switch gave roughly 40% more stable training and about +0.3 validation Sharpe. (Precise detail if pushed: the per-step training reward is actually the day's log return minus a transaction-cost penalty — simpler and lower-noise, which suits PPO's need for a dense immediate signal — while full episode-level Sortino is used to evaluate finished episodes and decide "champion" model promotion.)

**Q4. Why compare against Markowitz/Black-Litterman baselines at all?**
A: To prove the RL approach earns its complexity. A sophisticated model is only valuable if it beats a much simpler, well-understood baseline — otherwise you're adding risk, latency, and interpretability cost for nothing. Showing p<0.05 statistically significant outperformance across 11 rolling windows, not just one lucky backtest, is what makes the claim credible rather than anecdotal.

**Q5. What is walk-forward validation and why not standard k-fold cross-validation here?**
A: Walk-forward validation trains on a historical window, tests on the next chronological period, then rolls the window forward and repeats — always testing on data that comes *after* training data in time. Standard k-fold shuffles data randomly across folds, which for time series would let the model "see the future" (train on data from after the test period), leaking information and producing an unrealistically optimistic backtest.

**Q6. How did you align Nifty50 (NSE) and S&P500 (US) data given different trading calendars?**
A: I computed the intersection of trading dates across all 40 tickers — only kept dates where every single asset had a valid price — since a joint portfolio return literally can't be computed if one leg is missing a price for that day (e.g. a US holiday when Indian markets are open). Naively concatenating the two calendars would create rows with missing values that either crash the pipeline or silently corrupt the return calculation.

**Q7. Why is your Airflow layer described as "lightweight" — what does it actually contain?**
A: Airflow contains zero ML code. Every DAG task is just an HTTP POST to endpoints on the FastAPI service (`/retrain/fetch-data`, `/retrain/train`, `/retrain/evaluate-champion`) — Airflow's job is purely scheduling/sequencing/retry logic, not running PyTorch itself. This keeps the Airflow container's image small (no PyTorch/Stable-Baselines3/Gymnasium bundled in), avoids dependency conflicts with Airflow's own requirements, and keeps the scheduler reliable since it's never competing for resources with genuinely heavy training jobs.

**Q8. Describe your model serving strategy — how do you get <200ms latency with zero downtime on retraining?**
A: A FastAPI service loads the current MLflow "champion" model at startup on a background thread (so `/health` responds immediately without waiting on model load), while a separate `/ready` endpoint returns 503 until the model has actually finished loading. For predictions, the service serves from a model file already cached in memory/on local disk rather than re-fetching from MLflow on every request. A background thread polls MLflow every 5 minutes to check if the "champion" alias points to a new version; if so, it loads the new model and atomically swaps it in under a lock, so in-flight requests always complete against a consistent model version — no request ever hits a half-swapped state, and no manual restart is needed when a new model is promoted.

**Q9. What statistical test did you use to validate outperformance, and why?**
A: A two-sided t-test on monthly excess returns (RL strategy return minus benchmark return) to check whether the mean excess return is significantly different from zero, giving p<0.05 significance — a standard, defensible way to argue the outperformance isn't just noise from one lucky sample path.

**Q10. If your interviewer probes on exact numbers and they differ slightly from what you say, how do you handle it?**
A: Be upfront: "Depending on the specific backtest run and configuration, Sharpe came in around 1.4 to 1.47 for the RL agent versus roughly 0.74 to 0.77 for Markowitz — I'd want to pull the exact run before quoting one fixed number, since I tested across a few configurations." This demonstrates real understanding rather than a memorized flashcard number.

---

## Section 2 — LexAI (RAG Legal Chatbot)

**Q11. Walk me through LexAI in 60 seconds.**
A: "LexAI is a production-grade Retrieval-Augmented Generation system for Indian legal question-answering. I built the ingestion pipeline — semantic chunking, embedding, and parallel FAISS + BM25 indexing over 18+ Indian legal statutes and Supreme Court judgments. At query time it uses HyDE query rewriting, hybrid dense+sparse retrieval merged with Reciprocal Rank Fusion, and CrossEncoder reranking before the LLM (Google Gemini via LangChain LCEL) generates an answer grounded only in the retrieved context. The backend is async FastAPI with Server-Sent Events streaming and session-aware memory, and the frontend is React with real-time streaming and interactive citation panels."

**Q12. Why RAG instead of just fine-tuning an LLM on Indian law?**
A: Fine-tuning is expensive, slow to update whenever a law is amended, doesn't give per-answer citations, and can still hallucinate facts it half-learned during training. RAG keeps the base LLM frozen and grounds every answer in retrieved source text, so updating the knowledge base is as cheap as re-running ingestion on new documents, and every answer can point back to the exact statute/judgment it came from — critical for a legal-accuracy use case where an unverifiable answer is actively dangerous.

**Q13. Why hybrid (dense + sparse) retrieval instead of just embeddings?**
A: Dense/embedding retrieval is great at semantic matching ("penalty for stealing" ≈ "punishment for theft") but can miss exact keyword hits like a specific section number (e.g. "Section 302 IPC"), because embeddings compress away exact-token information. BM25 (sparse/lexical) is excellent at exact keyword matching but misses paraphrases. Combining both via Reciprocal Rank Fusion gets the best of both — essential in law, where both precise terminology and conceptual meaning matter.

**Q14. Why normalize vectors and use inner-product FAISS index instead of cosine similarity directly?**
A: Cosine similarity = dot product divided by the product of vector magnitudes — it isolates the angle (semantic direction) between vectors, ignoring length. If you normalize every vector to unit length *before* indexing, the magnitudes become 1 and cancel out of that formula, so dot product and cosine similarity become mathematically identical. That lets me use FAISS's plain inner-product index (`IndexFlatIP`), which is the faster, simpler primitive FAISS supports natively, while still getting cosine-similarity search behavior.

**Q15. What is HyDE and why use it here specifically?**
A: HyDE (Hypothetical Document Embeddings) has the LLM first generate a hypothetical answer to the user's question, then embeds *that* generated answer for retrieval instead of the raw (often short/ambiguous) query. A plausible-sounding answer tends to be semantically closer, in embedding space, to the real relevant legal passages than a terse question is — improving retrieval quality, especially for vague or conversationally-phrased legal questions.

**Q16. Why rerank with a CrossEncoder after hybrid retrieval instead of just trusting the top-k from retrieval?**
A: Initial retrieval (dense+sparse) has to score against the entire corpus, so it uses cheap, pre-computable similarity (independent vector comparisons) — fast but approximate. A CrossEncoder reranker instead jointly encodes the (query, passage) pair together, letting it model fine-grained interaction between the two texts — much more accurate, but too expensive to run against the whole corpus. So the pipeline uses fast approximate retrieval to narrow thousands of chunks down to a small candidate set, then applies the expensive, precise reranker only to that small set — getting both speed and accuracy.

**Q17. What is semantic chunking and why not just fixed-size chunking?**
A: Fixed-size chunking splits documents every N characters/tokens regardless of content, which can cut a sentence, clause, or legal provision mid-thought — destroying context needed for accurate retrieval and generation. Semantic chunking splits at natural topic/meaning boundaries so each chunk is self-contained and coherent, which matters a lot in legal text where a clause's meaning can depend on staying intact.

**Q18. How does LexAI prevent hallucination specifically?**
A: The system prompt explicitly instructs the LLM to "answer using ONLY the retrieved context below... do not fabricate section numbers, case names, or citations that are not in the context" — combined with the retrieval+rerank pipeline supplying genuinely relevant grounded text, and `Citation` objects that trace every claim back to its source chunk, shown interactively in the frontend's citation panel so a user can verify the answer against the primary source themselves.

**Q19. How would you evaluate whether LexAI's answers are actually good?**
A: Along faithfulness (is the answer actually supported by retrieved context, not hallucinated), answer relevance (does it address the question asked), and retrieval quality (context precision/recall — did we retrieve the right passages at all). In practice using a framework like RAGAS to score these automatically, and gating deployment (CI) so a change that measurably regresses answer quality doesn't get shipped.

**Q20. Why an async FastAPI backend with SSE streaming instead of a simple synchronous request-response?**
A: LLM generation is slow (seconds) and I/O-bound (waiting on the LLM API, waiting on vector search) — a synchronous blocking backend would freeze the whole server on every single request, unable to serve other users while one answer streams. Async lets the server hold many requests concurrently, each paused at its own `await` point, while streaming partial tokens back to the frontend via Server-Sent Events as they're generated — so the user sees the answer appear progressively instead of waiting for the entire response to finish.

**Q21. Why test with dummy/mocked classes (e.g. forcing an embedding "quota" error) instead of just testing against the real APIs?**
A: Deliberately triggering a real API quota failure on demand is impractical, costly, slow, and non-reproducible. Minimal dummy classes that implement only the exact interface the real code calls (`add_documents`, `save_local`, etc.) let me deterministically force a specific failure scenario — like "embedding call raises RuntimeError('quota')" — and assert the ingestion pipeline still degrades gracefully and reports correct stats, in milliseconds, every time, without depending on any real external service.

---

## Section 3 — URL Shortener (Production-Grade Distributed System)

**Q22. Walk me through this project in 60 seconds.**
A: "A horizontally scalable URL shortener built with FastAPI, PostgreSQL, and Redis caching. It's fully async end-to-end, decouples click-analytics recording from the redirect request path using Celery + Redis so analytics volume never affects redirect latency, and is instrumented with Prometheus/Grafana and structured JSON logging with request-ID tracing. I verified reliability with 90+ unit, integration, and end-to-end tests, including load-testing correct behavior under 50 concurrent requests, and automated the whole test/deploy flow with GitHub Actions using real Postgres and Redis containers in CI."

**Q23. Why does the redirect endpoint use a 307 status code instead of a standard 301 redirect?**
A: 301 (Moved Permanently) gets cached forever by browsers, meaning the browser stops even asking the server after the first visit — which breaks two things this project needs: click-analytics would undercount (server never sees repeat visits), and link management/deactivation would silently fail for anyone whose browser already cached the old redirect. 307 (Temporary Redirect) tells the browser "this might change, ask me again," so every click still hits the server, keeping analytics accurate and letting deactivated/updated links take effect immediately.

**Q24. Why decouple click analytics from the redirect path using Celery + Redis?**
A: If analytics writes happened synchronously inside the redirect request, redirect latency would depend on analytics-database write speed — under heavy traffic that write becomes a bottleneck slowing down the one thing (the redirect) that actually needs to be fast. Pushing the analytics write onto a Celery task queue backed by Redis makes it fire-and-forget from the redirect's perspective: the user gets redirected immediately, and analytics gets recorded asynchronously in the background, independent of request volume.

**Q25. Why async (async/await) throughout the whole backend?**
A: Every request involves I/O waits — a Postgres query, a Redis lookup. Synchronous code blocks the entire single-threaded process during each wait, so one slow database call freezes every other user's request too. Async lets the event loop pause a request at its `await` point and immediately go serve another request, then resume the first one once its I/O completes — so a single process can hold hundreds of requests "in flight" concurrently. Important: async doesn't make one query faster, it lets many slow waits happen concurrently instead of one at a time.

**Q26. You mention SQLite for unit tests but Postgres for integration — you found a bug this way. Explain.**
A: The fast unit-test suite runs against SQLite for speed, but SQLite doesn't enforce foreign-key constraints by default. An earlier version of the click-tracking logic could record a click referencing a link ID that didn't actually exist, and the SQLite-backed unit tests passed silently since SQLite never checked the constraint. Only the smaller, slower suite running against a real Postgres database caught it, because Postgres does enforce that foreign key and rejected the bad insert. This is exactly the class of bug a testing pyramid's small top layer of "real infrastructure" tests exists to catch — a fast test suite alone wasn't sufficient.

**Q27. How did you handle rate limiting and prevent abuse?**
A: Enhanced security with rate limiting (429 Too Many Requests on excessive calls) and custom alias support with validation, alongside standard input validation (e.g. rejecting self-referential or malformed URLs with 400/422 responses) to prevent obvious abuse vectors like redirect loops.

**Q28. Why Docker Compose with `condition: service_healthy` instead of just `depends_on`?**
A: Plain `depends_on` only guarantees container *start order*, not that the dependency has actually finished booting and is ready to accept connections — Postgres can be "started" but still mid-initialization. `condition: service_healthy` waits for an actual healthcheck to pass first, preventing the API container from starting and immediately failing because Postgres/Redis, while technically running, weren't truly ready yet.

**Q29. Why is CI path-filtered in this project?**
A: A change confined to, say, the dashboard service doesn't need to spin up Postgres/Redis containers and run the full API test suite — that's purely wasted time/CI resources, not a correctness issue. Path-filtering skips unrelated test suites for a given change, speeding up feedback without changing what's ultimately verified before merge.

---

## Section 4 — Qclairvoyance Quantum Labs Internship (QUBO / Quantum Optimization)

**Q30. Walk me through your quantum optimization internship.**
A: "I formulated Quadratic Unconstrained Binary Optimization (QUBO) models for molecular docking of a specific protein structure (1Y6R), using Grid Point Matching and Feature Atom Matching to encode spatial/geometric docking constraints as quadratic penalty terms. I benchmarked docking performance across multiple solvers — SCIP, CPLEX, Hercules, and Qubolite — on a 675-variable optimization problem, achieving a minimum binding energy of -8.57. I also evaluated exact vs. heuristic QUBO solvers using Jaccard similarity and bitstring overlap metrics, finding 83.3% structural agreement between exact solvers, while heuristic approaches diverged more — showing that comparing raw objective value alone isn't sufficient to judge QUBO solution quality in computational drug discovery."

**Q31. What is QUBO and why formulate molecular docking as a QUBO problem?**
A: QUBO (Quadratic Unconstrained Binary Optimization) expresses a problem as minimizing a quadratic function over binary variables, with no explicit constraints — constraints instead get encoded as quadratic *penalty terms* added into the objective, so violating them costs the objective function. This formulation is exactly what quantum annealers and many classical/heuristic QUBO solvers are built to optimize, so casting molecular docking (which normally involves discrete spatial-matching constraints) into this binary quadratic form makes it solvable by that whole family of solvers, including quantum approaches.

**Q32. Why compare exact solvers (SCIP, CPLEX) against heuristic ones, and why measure agreement, not just final objective value?**
A: Exact solvers guarantee (given enough time) the true optimum; heuristic solvers trade optimality guarantees for speed on large problems. Two solutions can report similar objective/binding-energy values while actually corresponding to structurally different docking poses (different bitstring assignments) — so I used Jaccard similarity and bitstring overlap specifically to check whether solvers agree on the *actual solution structure*, not just the score. Finding 83.3% agreement between exact solvers but more divergence among heuristics demonstrated that objective value alone can mask real structural disagreement — an important caution for anyone using heuristic QUBO solvers in drug discovery, where the actual molecular pose matters, not just a number.

**Q33. Why does this quantum internship matter for an ML Engineer role — how is it related?**
A: It's the same underlying skill set as ML engineering: formulating a real-world problem as a rigorous mathematical optimization, benchmarking multiple solving approaches quantitatively rather than trusting one method, and being careful about *what* metric actually reflects solution quality — the same discipline behind comparing baselines in the Portfolio project or validating retrieval quality in LexAI. It also shows I'm comfortable with the math/optimization theory (constraint encoding, combinatorial search) underlying advanced ML techniques.

---

## Section 5 — General Resume / Fit Questions

**Q34. Why these three projects together — what story do they tell about you as an ML engineer?**
A: Each project deliberately targets a different core ML engineering competency: Portfolio Optimization is reinforcement learning + rigorous statistical evaluation + MLOps; LexAI is modern LLM/RAG systems engineering with retrieval and grounding; URL Shortener is production backend/distributed-systems fundamentals (async, caching, testing, observability) that any ML system ultimately has to be served on top of. Together they cover the full stack from ML/RL modeling, to LLM application engineering, to the production backend engineering that makes any of it deployable and reliable.

**Q35. What was the hardest technical decision across these projects, and why?**
A: (Pick genuinely — good candidates: choosing Sortino over Sharpe as the RL reward and having to separate the dense per-step reward from the episode-level evaluation metric; or choosing hybrid retrieval + RRF fusion in LexAI after realizing pure embeddings missed exact section-number lookups; or catching the SQLite-vs-Postgres foreign-key bug and restructuring the test pyramid because of it.) Frame as: what the "obvious" choice would have been, why it fell short, what you measured that changed your mind, and the outcome.

**Q36. If you had more time/resources, what would you improve in your strongest project?**
A: Reasonable, honest answers: for Portfolio Optimization — extending beyond backtesting to a genuinely live paper-trading deployment to validate the drift-detection/retraining loop under real conditions (be honest that current results are from backtesting/simulation, not live trading — this "honesty as a strength" framing is explicitly a good look in interviews). For LexAI — expanding the corpus beyond 18 statutes/judgments and adding a more rigorous, continuously-run RAGAS evaluation gate in CI.
