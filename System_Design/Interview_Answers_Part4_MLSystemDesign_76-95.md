SYSTEM DESIGN INTERVIEW — CANDIDATE ANSWERS
PART 4: SECTION D — MACHINE LEARNING SYSTEM DESIGN QUESTIONS (76–95)

Each of these follows the same general shape a strong ML system design answer takes: clarify the actual business objective and ML task type, describe the data and features, choose a model approach and justify it, describe training/evaluation, then describe serving and monitoring. I've kept each answer focused on what's genuinely distinctive about that specific system rather than repeating the full framework every time.


76. Design a recommendation system for [a video/music/e-commerce/social platform].

"First I'd pin down the actual objective — are we optimizing for engagement, revenue, or long-term retention, since that changes what I'm actually training the model to predict. I'll assume engagement, framed as predicting the probability a user interacts with a given item.

For data, I'd use implicit signals — clicks, watch time, purchases — rather than relying on explicit ratings, since most users never rate anything but generate implicit signal constantly just by using the product.

Given the catalog could be millions of items, I wouldn't score every item for every user directly — I'd use a two-stage architecture: a fast candidate generation stage, using something like collaborative filtering or a two-tower embedding model, narrows millions of items down to a few hundred plausible candidates; a heavier ranking model then scores just those candidates using richer features, including recency and predicted engagement.

For the cold-start problem — new users or new items with no history — I'd fall back to content-based signals, like item metadata or a new user's stated preferences at signup, blended with globally popular items until enough behavioral data accumulates.

For serving, candidate generation and much of the scoring would run in a batch job, precomputed and cached, with a lighter real-time re-ranking pass at request time incorporating the user's most recent activity. And I'd close the loop with an online A/B test before fully rolling out any new model version, since offline accuracy improvements don't always translate to real engagement gains."


77. Design YouTube's or TikTok's video recommendation / "For You" feed.

"This is the general recommendation architecture I just described, with two things that make it specifically distinctive here. First, the feedback signal is richer than a simple click — watch time, completion rate, and rewatches are much stronger engagement signals than a click alone, so I'd weight the model's training labels accordingly rather than treating every click as equally meaningful.

Second, this needs to be much more real-time than something like an e-commerce recommendation — TikTok's feed specifically needs to adapt within a session, based on what a user just watched a minute ago, not just their historical behavior from days prior. So I'd add an online, session-level feature layer on top of the offline candidate generation — recent watch history from just this session gets folded into the ranking pass at request time, not just the precomputed candidates.

I'd also explicitly design for exploration, not just pure exploitation of what the model currently believes a user likes — mixing in a small percentage of higher-uncertainty or diverse content so the system can keep learning and doesn't collapse a user's feed into an overly narrow bubble based on early signal."


78. Design a news feed ranking system (e.g., Facebook News Feed ranking model).

"I'd frame this as a ranking problem over candidate posts from accounts and pages a user follows, rather than a pure recommendation problem over the whole catalog, since the candidate set here is naturally already constrained to what the user follows.

For features, I'd include post-level signals like recency and current engagement velocity, user-post relationship signals like how often this user interacts with this specific poster, and user-level signals like their general engagement patterns.

I'd frame the label as predicted engagement probability — likely to like, comment, or share — and note that different engagement types probably deserve different weights, since a comment is a stronger signal of genuine interest than a passive like.

For serving, I'd separate the assembly of candidate posts, using the fan-out approach from the systems-design side of feed design, from the ranking of those candidates, which happens at request time using the features above. I'd also flag fairness and diversity as a real, explicit design constraint here, not just an engagement-maximization problem — a pure engagement-optimized feed tends to over-favor high-arousal, controversial content, so I'd deliberately incorporate diversity and integrity signals into the ranking objective, not just raw predicted engagement."


79. Design a search-ranking system for a large-scale search engine.

"I'd separate this into retrieval and ranking, similar to the recommendation architecture. Retrieval uses the inverted index and something like BM25 to quickly narrow the full corpus down to a candidate set of documents that plausibly match the query at all.

Ranking then takes that candidate set and applies a learned model — a 'learning to rank' approach — using features like the BM25 score itself, query-document text similarity via embeddings, document quality signals like backlink count or freshness, and historical click-through rate for that document on similar past queries.

For training data, I'd use click logs, but I'd explicitly account for position bias — a result in position 1 gets clicked far more than an equally relevant result in position 5, simply because of where it's shown, so I wouldn't treat raw click-through rate as unbiased ground truth without correcting for this.

For evaluation, I'd use offline ranking metrics like NDCG, but I'd treat those as a necessary but not sufficient check — the real validation is an online experiment measuring actual user satisfaction signals, like whether users click a result and then don't immediately return to search again, which is a much stronger relevance signal than the click itself."


80. Design an ads-ranking or ads-relevance system.

"I'd frame the core prediction as click-through rate, but I'd immediately point out that CTR alone isn't the actual optimization target — the real auction typically ranks by expected value, meaning predicted CTR multiplied by the bid amount, since a lower-relevance ad with a much higher bid can still be the right one to show.

For features, I'd include the usual user and ad content signals, but I'd specifically call out that ads have a much faster-changing candidate pool than organic content — new campaigns launch and old ones exhaust budget constantly — so the candidate generation stage needs to refresh much more frequently than, say, a video recommendation system's candidate pool.

I'd also flag calibration as unusually important here, more than in most other ranking systems — because the CTR prediction directly feeds into a real-money auction, a systematically overconfident or underconfident model doesn't just hurt engagement, it directly costs the business money by mispricing the auction, so I'd specifically monitor calibration, not just raw accuracy or AUC, as an ongoing production metric."


81. Design a fraud-detection system for financial transactions.

"Because a decision needs to happen within the live transaction path, this requires online inference, typically within a strict tens-of-milliseconds budget, not batch prediction computed ahead of time.

For features, the real predictive power comes overwhelmingly from well-engineered signals — like transaction velocity for this specific card over the past hour, or how much this transaction deviates from the user's own typical spending pattern — rather than from the specific choice of model architecture. These need to be looked up or incrementally maintained with very low latency at the moment of the transaction, so I'd use a fast in-memory store with continuously updating counters rather than querying a full historical database live.

I'd explicitly call out concept drift as the single defining operational challenge here — fraudsters actively adapt their tactics specifically to evade whatever detection is currently deployed, so the relationship between 'what a transaction looks like' and 'is it fraud' genuinely changes over time, which means this model needs a much tighter, often event-triggered retraining loop than a typical, more stable ML system.

For the actual decision flow, I'd use a tiered approach — fast, simple rule-based checks catch the most obvious cases immediately, and only genuinely ambiguous cases get passed to the full model, balancing overall latency against detection accuracy."


82. Design a spam or content-moderation classifier for a social platform.

"I'd frame this as multi-label classification, since content can violate multiple distinct policies simultaneously — spam, harassment, misinformation — rather than a single binary spam/not-spam decision.

For data, I'd combine user reports, human moderator labels, and any existing rule-based detections as training signal, being explicit that this labeled data is going to be noisy and imbalanced, since genuinely violating content is a small minority of all content, so I'd need to actively account for that class imbalance during training rather than assuming a naive accuracy metric means anything useful here.

I'd use a cascading pipeline, similar to fraud detection — a fast, cheap first pass, potentially rule-based for the most clear-cut violations, and a heavier model for genuinely ambiguous content, with the most uncertain cases routed to human review rather than automated action, given how costly a wrongful takedown is for user trust.

I'd weight false negatives and false positives very differently depending on severity — for something like content promoting real-world violence, I'd tolerate a meaningfully higher false-positive rate in exchange for near-zero false negatives, whereas for a milder policy violation, I'd tolerate more false negatives to avoid over-removing legitimate content. I'd also flag adversarial robustness explicitly — bad actors will actively try to evade the classifier, so, like fraud, this needs an ongoing retraining and monitoring loop, not a train-once-and-forget model."


83. Design a "People You May Know" or friend-suggestion system.

"I'd frame this fundamentally as a graph problem layered with ML ranking rather than a pure recommendation problem — the social graph itself, who's already connected to whom, is the primary signal, more than content similarity.

For candidate generation, I'd look at second-degree connections — friends of friends who aren't already connected to the target user — since that's both a strong and computationally efficient signal, naturally supported by a graph database's fast relationship-traversal capability. I'd also incorporate other signals like shared workplace, school, or contact list overlap where available.

For ranking those candidates, I'd train a model predicting the probability of a genuine future connection, using features like number of mutual friends, profile similarity, and any shared group or event membership.

I'd specifically flag privacy as a first-class design constraint here, not an afterthought — contact list matching in particular needs very careful, explicit user consent and handling, and I'd design the system so users can clearly understand and control what signals are actually being used to generate their suggestions."


84. Design autocomplete or spell-check for a search box or mobile keyboard.

"For autocomplete specifically, I'd use the trie-based prefix-matching approach from the systems-design side, with completions ranked by historical popularity, computed offline from query logs and refreshed periodically.

For spell-check, I'd frame it as finding indexed terms within a small edit distance of what the user actually typed, using a structure like a BK-tree for efficient approximate matching rather than comparing the typed query against every possible term individually.

For a mobile keyboard specifically, I'd add a language model layer on top of pure edit-distance matching, since keyboard typos have a distinctive pattern — they're often adjacent-key mistakes, not random character substitutions — so I'd bias corrections toward keys physically near the one the user actually pressed, and I'd use the user's own recent typing history and a general language model to prefer contextually likely corrections over ones that are merely close in edit distance but don't fit the sentence.

I'd keep both of these fully on-device where possible for a mobile keyboard specifically, both for latency — this needs to respond within milliseconds on every keystroke — and for privacy, since keystroke-level data is genuinely sensitive and I'd rather not need to send it to a server at all if I can avoid it."


85. Design an ETA-prediction system for a ride-sharing or delivery app (e.g., Uber ETA).

"I'd frame this as a regression problem — predicting trip duration given a route, current traffic conditions, and contextual factors like time of day and weather.

For features, the road network itself, modeled as a graph, gives me a baseline estimate using something like a shortest-path calculation with historical average speeds per road segment. On top of that baseline, I'd layer real-time signals — current traffic conditions derived from other active trips' actual speeds on the same roads right now — since historical averages alone miss things like an accident that just happened five minutes ago.

I'd train the model to predict a correction on top of the baseline route-distance estimate, rather than predicting the absolute duration from scratch, since that tends to generalize better and lets the baseline naturally handle new or rarely-traveled routes reasonably well even with limited direct training data for that exact route.

For evaluation, I'd track mean absolute error in real time against actual completed trip durations, and I'd specifically monitor for systematic bias — like consistently underestimating ETAs during rush hour — since a biased ETA damages user trust in the product even if the average error looks acceptable in aggregate."


86. Design a churn-prediction system for a subscription product.

"I'd frame this as binary classification — will this user cancel or fail to renew within some defined future window, like the next 30 days.

For features, I'd use engagement trend signals more than raw activity level — a declining trend in usage over the last few weeks is a much stronger churn signal than a user's absolute activity level, since some genuinely loyal users are simply lighter users overall. I'd also include support-ticket history, billing issues, and any recent price changes to their plan.

For labels, I'd be careful about the exact definition of churn — a hard cancellation is unambiguous, but a subscription simply lapsing due to an expired card is a genuinely different problem requiring a different intervention, like a payment-retry flow rather than a retention offer, so I might actually model these as separate problems rather than one combined churn label.

For serving, this doesn't need real-time inference — I'd run this as a daily or weekly batch job, scoring the full user base and feeding high-risk users into a retention workflow, like a targeted offer or proactive outreach. And I'd specifically evaluate this by measuring actual retention lift from an intervention informed by the model's predictions in a proper A/B test, not just by the model's offline classification accuracy, since a model can be accurate and still not actually be useful if the resulting intervention doesn't move the needle."


87. Design a demand-forecasting system (e.g., for ride-sharing surge pricing or retail inventory).

"I'd frame this as a time-series regression problem — predicting demand for a given location and time window, based on historical demand patterns, seasonality, day-of-week effects, and external factors like weather or local events.

For the model itself, I'd start with a strong baseline using classical time-series methods that explicitly account for seasonality and trend, and only move to a more complex model, like a gradient-boosted tree with engineered lag features or a neural sequence model, if I have clear evidence the added complexity is actually earning meaningfully better accuracy, since time-series forecasting often has strong, simple baselines that are genuinely hard to beat by much.

For ride-sharing surge specifically, I'd note this needs to run continuously in near-real-time, since supply and demand can shift within minutes, whereas retail inventory forecasting can reasonably run as a daily or weekly batch job, since inventory decisions operate on a much longer time horizon.

I'd evaluate using a rolling backtest — training only on data up to a certain point in time and evaluating against what actually happened afterward, repeated across multiple historical time windows — rather than a single random train/test split, since naively shuffling time-series data risks leaking future information into training in a way that would make offline accuracy look artificially better than real-world performance will actually be."


88. Design a click-through-rate (CTR) prediction model for ads or feed ranking.

"I'd frame this as binary classification predicting the probability of a click, and I'd specifically flag that calibration matters as much as raw discriminative accuracy here, since the predicted probability itself often feeds directly into a downstream auction or ranking formula, not just a binary yes/no decision.

For features, I'd combine user features, ad or content features, and cross features capturing the interaction between the two — a specific user's affinity for a specific ad category tends to be a much stronger signal than either the user's or the ad's features considered in isolation.

For the model architecture, I'd note that this is a genuinely well-studied problem with strong standard approaches — factorization machines or deep learning architectures that explicitly model feature interactions, since CTR prediction historically benefits a lot from capturing interactions between sparse categorical features rather than treating them independently.

I'd train on logged historical impressions and clicks, but I'd explicitly flag position bias and selection bias — the model only ever sees impressions that were actually shown, under whatever ranking policy was live at the time the data was collected, which can bias what the model learns unless I specifically account for it, for example through techniques that correct for the exposure the prior ranking policy actually gave each item."


89. Design a visual search system (e.g., "find similar products from a photo," Pinterest-style).

"I'd convert both the query image and every catalog image into embeddings — dense vector representations produced by a model trained specifically so that visually or semantically similar images end up close together in that vector space.

At query time, I'd search a vector database for the catalog embeddings nearest to the query image's embedding, using approximate nearest-neighbor search, since finding the mathematically exact closest matches among potentially millions of catalog images would be too slow to do exactly at real-time query latency.

For the embedding model itself, I'd likely start from a pretrained vision model and fine-tune it specifically on the target domain — product photos, for a Pinterest or e-commerce use case — using a contrastive training objective that explicitly pulls similar items closer together and pushes dissimilar items apart in the embedding space, rather than relying on a general-purpose off-the-shelf embedding not tuned to this specific catalog.

For evaluation, I'd use both offline retrieval metrics, like whether genuinely similar items rank highly for a held-out set of query images, and online engagement metrics, like click-through rate on the visually similar results actually shown to real users, since offline embedding quality doesn't automatically guarantee the results feel genuinely useful to an actual shopper."


90. Design a duplicate/near-duplicate detection system for images or text at scale.

"For text, I'd use the MinHash and locality-sensitive hashing approach from the systems-design side — breaking content into shingles, generating a compact similarity-preserving signature, and bucketing documents so I only need to compare documents that land in the same bucket rather than every pair.

For images, I'd use a similar high-level strategy but with a perceptual hash or a learned embedding as the underlying similarity signal instead of text shingles — a perceptual hash is specifically designed so that visually similar images, even after minor edits like cropping or recompression, produce similar hash values, unlike a cryptographic hash, which would produce a completely different value from even one changed pixel.

At real scale, I'd run this as an asynchronous pipeline — new content gets hashed and bucketed at ingestion, and a comparison against just its bucket-mates happens shortly after, flagging likely near-duplicates for either automatic handling or human review depending on confidence, rather than trying to compare every new item against the entire existing corpus synchronously, which simply wouldn't scale.

I'd tune the similarity threshold based on the actual cost of each error type — for something like plagiarism detection, I'd rather have a slightly higher false-positive rate and catch genuine cases, with human review as the final check, than risk missing real duplicates outright."


91. Design an ML model monitoring system that detects data drift, concept drift, and performance degradation.

"I'd track three distinct categories of signal, since they catch genuinely different failure modes. First, operational health — latency, throughput, and error rate of the serving infrastructure itself, the same golden signals I'd track for any production system.

Second, input distribution monitoring, to catch data drift — continuously comparing the statistical distribution of live production input features against the distribution they had in the original training data, using something like population stability index, and alerting when a feature's distribution has shifted meaningfully.

Third, and hardest, actual prediction quality — comparing predictions against real ground truth once it eventually becomes available, which often lags by hours or days depending on the use case, to catch genuine concept drift, where the underlying relationship between inputs and the correct outcome has itself changed, not just the inputs' distribution.

I'd feed all three into an alerting system, using symptom-based thresholds — like 'accuracy dropped below X over the trailing week' — rather than purely cause-based ones, and I'd connect meaningful, sustained degradation directly to an automated or semi-automated retraining trigger, rather than relying on someone happening to notice a dashboard degrading."


92. Design a feature store for a company with many ML teams sharing features.

"The core problem I'd solve for is training-serving skew — making sure a feature computed during model training and that same feature computed at live inference time are genuinely, provably identical, since even small discrepancies between the two silently degrade model accuracy in ways that are notoriously hard to detect after the fact.

I'd build this with two components sharing the same underlying feature definitions: an offline store, holding historical feature values for training, typically backed by a data warehouse; and an online store, holding just the current feature values, optimized for very low-latency lookups at serving time, typically backed by something like Redis.

For feature definitions themselves, I'd have teams register a feature once, with its transformation logic clearly specified, and have both the offline and online pipelines derive from that exact same single definition, rather than each team independently reimplementing similar feature logic in two different places, which is exactly how training-serving skew creeps in.

I'd also add feature discovery and versioning — letting teams see and reuse features other teams have already built and validated, and tracking how a feature's definition has changed over time, plus point-in-time correctness for the offline store specifically, so a training dataset only ever uses feature values that were genuinely available at that historical point in time, avoiding subtle data leakage from accidentally using future information during training."


93. Design an experimentation platform for online A/B testing of ML models.

"I'd support traffic splitting at the model-serving layer — a defined percentage of live traffic routes to a challenger model, with the rest continuing on the current production model, using consistent, deterministic bucketing by user ID so the same user reliably sees the same variant throughout the experiment rather than flipping between them.

I'd log every prediction along with which variant served it, and feed downstream outcome events — did the user click, purchase, or convert — into an analytics pipeline that computes both the primary metric the experiment is actually testing and any guardrail metrics that shouldn't regress even if the primary metric improves, like overall latency or an unrelated engagement metric.

I'd build in statistical significance testing with proper handling of sample size and experiment duration, specifically to prevent someone from stopping an experiment early the moment it happens to look favorable, which is a very common and genuinely tempting mistake that produces unreliable results.

I'd also support gradual, staged rollout on top of pure A/B testing — start a new model at a small traffic percentage, similar to a canary deployment, and only ramp it up further once both the statistical results and the operational health metrics look genuinely healthy, rather than jumping straight from a small test directly to 100% traffic."


94. Design an end-to-end ML pipeline: from raw data ingestion through training, deployment, and monitoring, for any given prediction task.

"I'd walk through this as a connected pipeline rather than a list of separate components. Raw data lands from application databases and event streams into a data lake, in its original form. A data pipeline validates and transforms it, checking for missing values and unexpected distribution shifts before it's considered usable.

Feature engineering, ideally backed by a shared feature store, computes the actual model inputs, with the same feature logic reused for both training and serving to avoid skew. A training pipeline pulls prepared data, trains a candidate model, and logs the full configuration and resulting metrics via experiment tracking, so any specific run can be reproduced or compared later.

If the candidate model passes evaluation against a held-out test set, it gets registered in a model registry, then deployed gradually — typically shadow deployment first, to observe its behavior against real traffic with zero user-facing risk, then canary, then full rollout, each gated on the model continuing to look healthy.

In production, monitoring tracks operational health, input drift, and eventual prediction accuracy against ground truth, feeding into a retraining pipeline that's triggered either on a schedule or by detected drift, which closes the loop by kicking off this exact same process again with fresh data."


95. How would you design the offline evaluation strategy vs. the online evaluation strategy for a new ranking model?

"Offline evaluation happens before the model ever sees real traffic — I'd hold out a test set the model never trained on, and evaluate using ranking-specific metrics like NDCG or MAP rather than plain accuracy, since what matters for a ranking task is getting the relative order right, not just individual predictions.

I'd treat offline evaluation as a gate, not a guarantee — it tells me whether a candidate model is even worth testing further, but I wouldn't fully trust it to predict real-world impact, since offline metrics are computed against historical data collected under the OLD model's ranking policy, which introduces exposure bias — the old model only ever showed users certain items, so the historical data is systematically blind to how users would have responded to items the old model rarely surfaced.

Online evaluation is where I'd actually trust the result — a proper A/B test, routing a small percentage of real traffic to the new model and measuring genuine business metrics, like engagement or conversion, against a control group seeing the current production model, with statistical significance testing before drawing any conclusion.

I'd frame the relationship between the two very directly: offline evaluation is cheap, fast, and necessary but insufficient; online evaluation is expensive, slow, but the only evaluation that actually tells me whether this model change is genuinely worth shipping."
