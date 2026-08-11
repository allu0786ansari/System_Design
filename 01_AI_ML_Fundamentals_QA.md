# HyperVerge MLE — Round 1 (Automated AI Interview) Prep
## File 1 of 3: Core AI/ML & DL Fundamentals — Q&A

> HyperVerge's Round 1 is an **automated, AI-driven interview focused on "AI fundamentals"**, personalized against your resume. Based on candidate reports, it tests: core ML/DL theory, math intuition (not derivations), and your ability to explain concepts simply and confidently — similar to a spoken viva, not a written exam. Expect follow-ups ("why not X instead?") since it's conversational. Answer in 30–90 second spoken chunks: definition → intuition → example → (if relevant) how you used it in your projects.

---

## A. Supervised vs Unsupervised vs Reinforcement Learning

**Q1. What is the difference between supervised, unsupervised, and reinforcement learning?**
A: Supervised learning trains on labeled data (input→output pairs) to predict an output — e.g. classification or regression. Unsupervised learning works on unlabeled data to find structure — clustering, dimensionality reduction. Reinforcement learning has an agent that learns by interacting with an environment, taking actions, and receiving rewards, optimizing long-term cumulative reward rather than matching a fixed label.
*Tie to your work:* "In my Portfolio Optimization project, I used RL (PPO and SAC) — the agent learns an allocation policy by trial and error, rewarded by portfolio performance (Sortino ratio), not by being shown 'correct' allocations."

**Q2. What's the difference between classification and regression?**
A: Classification predicts a discrete/categorical label (spam/not spam); regression predicts a continuous value (price, temperature). Same underlying ML pipeline, different loss functions and output layers (e.g. softmax + cross-entropy vs. linear output + MSE).

**Q3. What is semi-supervised learning and when would you use it?**
A: Uses a small amount of labeled data plus a large amount of unlabeled data — useful when labeling is expensive (e.g. medical imaging). The model learns structure from unlabeled data and refines using the few labels.

---

## B. Bias-Variance, Overfitting, Regularization

**Q4. Explain the bias-variance tradeoff.**
A: Bias is error from overly simplistic assumptions (underfitting — model too rigid to capture patterns). Variance is error from sensitivity to the specific training data (overfitting — model memorizes noise). Total error ≈ bias² + variance + irreducible error. Increasing model complexity typically lowers bias but raises variance; the goal is the sweet spot that minimizes total generalization error.

**Q5. What is overfitting? How do you detect and prevent it?**
A: Overfitting is when a model learns noise/idiosyncrasies of training data rather than the general pattern, so training accuracy is high but validation/test accuracy is much lower. Detect via a train-vs-validation loss/accuracy gap. Prevent with: more data, regularization (L1/L2, dropout), early stopping, cross-validation, simpler architecture, data augmentation.
*Tie to your work:* "In the Portfolio Optimization engine's LSTM forecaster, I used dropout and early stopping against a validation set to avoid overfitting to historical price noise."

**Q6. L1 vs L2 regularization — difference and effect?**
A: L1 (Lasso) adds sum of absolute weights to the loss, which drives some weights exactly to zero → sparse models, implicit feature selection. L2 (Ridge) adds sum of squared weights, shrinking weights smoothly toward zero without eliminating them → handles multicollinearity, generally smoother solutions. L1's gradient is constant near zero which pushes weights to exactly 0; L2's gradient shrinks proportionally to the weight so it rarely hits exactly 0.

**Q7. What is dropout and why does it work?**
A: Dropout randomly deactivates a fraction of neurons during each training forward pass, forcing the network to not rely on any single neuron/co-adapted feature. It approximates training an ensemble of many thinner networks whose predictions get averaged at test time, improving generalization.

**Q8. What is cross-validation and why use k-fold?**
A: Cross-validation splits data into k folds, training on k-1 folds and validating on the remaining fold, rotating through all folds. It gives a more robust estimate of generalization performance than a single train/val split, especially with limited data, and reduces the risk that one lucky/unlucky split skews your evaluation.
*Tie to your work:* "In the Portfolio project I used walk-forward validation instead of standard k-fold — for time series, random k-fold would leak future information into training, so I used rolling time-ordered windows instead."

---

## C. Evaluation Metrics

**Q9. Precision vs Recall vs F1 — when do you optimize for which?**
A: Precision = TP/(TP+FP) — of predicted positives, how many are correct (minimizes false alarms). Recall = TP/(TP+FN) — of actual positives, how many were caught (minimizes misses). F1 is their harmonic mean, useful when you need a balance. Optimize for precision when false positives are costly (e.g. flagging legitimate transactions as fraud); optimize for recall when false negatives are costly (e.g. missing an actual fraud case, or missing cancer in diagnosis).

**Q10. What is a ROC curve and AUC?**
A: ROC plots True Positive Rate vs False Positive Rate at varying classification thresholds. AUC (area under curve) summarizes overall separability of classes across all thresholds — 0.5 is random, 1.0 is perfect. Useful for comparing classifiers independent of a chosen threshold, though it can be misleading under heavy class imbalance (precision-recall curves are often better there).

**Q11. How do you handle imbalanced datasets?**
A: Techniques: resampling (oversample minority via SMOTE, undersample majority), class-weighted loss functions, choosing appropriate metrics (F1/PR-AUC instead of accuracy), ensemble methods, anomaly-detection framing for extreme imbalance, and threshold tuning post-training.

**Q12. Why is accuracy a misleading metric sometimes?**
A: With imbalanced classes (e.g. 95% negative, 5% positive), a model that always predicts "negative" scores 95% accuracy while being useless. Metrics like precision, recall, F1, or PR-AUC better reflect performance on the minority class that usually matters most.

**Q13. What is the Sharpe ratio / Sortino ratio (finance-adjacent metric, relevant to your project)?**
A: Sharpe ratio = (portfolio return − risk-free rate) / standard deviation of returns — risk-adjusted return, but penalizes upside volatility equally with downside. Sortino ratio only penalizes downside deviation, so it doesn't punish large positive days — a better fit as an RL reward signal since you don't want to discourage big wins.

---

## D. Core Algorithms

**Q14. Explain how gradient descent works.**
A: Gradient descent iteratively updates model parameters in the direction that reduces the loss function fastest — the negative gradient. Parameter update: `θ = θ − η∇L(θ)`, where η is the learning rate. Variants: batch (full dataset per step, stable but slow), stochastic (one sample per step, noisy but fast), mini-batch (a middle ground, most common in practice).

**Q15. What is the difference between SGD, Adam, and RMSProp optimizers?**
A: SGD updates using raw gradients (optionally with momentum to smooth updates and escape local minima/saddle points). RMSProp adapts the learning rate per-parameter using a moving average of squared gradients, so parameters with consistently large gradients get smaller effective steps. Adam combines momentum (first moment) with RMSProp-style adaptive learning rates (second moment) — generally the default choice for deep learning because it converges quickly with little tuning.

**Q16. What is backpropagation?**
A: The algorithm for computing gradients of the loss with respect to every weight in a neural network, using the chain rule, propagating error backward from the output layer to the input layer. Each layer's gradient depends on the gradient of the layer after it, which is why deep networks can suffer vanishing/exploding gradients.

**Q17. Vanishing/exploding gradients — cause and fixes?**
A: In deep networks, gradients are products of many layer-wise derivatives via the chain rule. If those derivatives are consistently <1 (e.g. sigmoid/tanh saturating), gradients shrink toward zero in early layers (vanishing) — those layers stop learning. If derivatives are consistently >1, gradients blow up (exploding). Fixes: ReLU-family activations, careful weight initialization (Xavier/He), batch normalization, residual/skip connections, gradient clipping, LSTM/GRU gating for sequence models.

**Q18. Explain decision trees and how they split.**
A: A decision tree recursively splits data on feature thresholds to create purer subsets. Splits are chosen to maximize information gain (reduction in entropy) or minimize Gini impurity. Prone to overfitting if grown too deep — controlled via max depth, min samples per leaf, or pruning.

**Q19. What is ensemble learning — bagging vs boosting?**
A: Bagging (e.g. Random Forest) trains many models independently and in parallel on bootstrapped samples, then averages/votes — reduces variance. Boosting (e.g. XGBoost, AdaBoost) trains models sequentially, where each new model focuses on correcting the errors of the previous ones — reduces bias, often yields higher accuracy but is more overfitting-prone and harder to parallelize.

**Q20. What's the difference between generative and discriminative models?**
A: Discriminative models learn the decision boundary / P(y|x) directly (logistic regression, SVM, most classifiers) — good for prediction. Generative models learn the joint distribution P(x,y) or P(x) (Naive Bayes, GANs, VAEs, LLMs) — can generate new samples and also do classification via Bayes' rule, but usually need more data to model well.

---

## E. Deep Learning Architectures

**Q21. ANN vs CNN vs RNN — when to use which?** *(explicitly asked in past HyperVerge GD rounds)*
A: ANN (fully connected / MLP) treats inputs as flat vectors with no structural assumption — fine for tabular data but doesn't exploit spatial or sequential structure. CNN uses convolutional filters that exploit spatial locality and translation invariance — ideal for images (and increasingly 1D sequences/text). RNN processes sequences step by step, maintaining a hidden state that carries information from previous steps — suited to sequential/temporal data like text or time series, though largely superseded by Transformers for long sequences due to vanishing gradients over long time horizons.

**Q22. How does a CNN work — convolution, pooling, filters?**
A: A convolutional layer slides small learnable filters (kernels) across the input, computing dot products to produce feature maps that detect local patterns (edges, textures, and in deeper layers, more abstract shapes). Pooling (max/average) downsamples feature maps, reducing spatial size and providing some translation invariance while cutting computation. Stacking conv+pool layers builds a hierarchy from low-level to high-level features.

**Q23. What is a Transformer and why did it replace RNNs for NLP?**
A: A Transformer processes an entire sequence in parallel using self-attention, letting every token directly attend to every other token regardless of distance — avoiding the sequential bottleneck and vanishing-gradient issues of RNNs over long sequences, and enabling much better parallelization/training speed on GPUs. Core components: multi-head self-attention, positional encodings (since attention alone is order-agnostic), feed-forward layers, layer norm, and residual connections.

**Q24. Explain self-attention / the attention mechanism in simple terms.**
A: For each token, attention computes a Query, Key, and Value vector. The similarity (dot product) between a token's Query and every other token's Key determines how much "attention weight" that other token gets; those weights are used to compute a weighted sum of Value vectors — so each token's new representation is a context-aware blend of the whole sequence, not just its own embedding. "It's like each word asking 'which other words in this sentence are relevant to understanding me?' and blending their information in proportion to relevance."

**Q25. What is a Vision Transformer (ViT)?** *(explicitly asked in HyperVerge GD)*
A: ViT applies the Transformer architecture to images by splitting an image into fixed-size patches, flattening and linearly embedding each patch (treating patches like "tokens"), adding positional encodings, and feeding the sequence through standard Transformer encoder blocks. Unlike CNNs, ViT has no built-in inductive bias for locality/translation invariance, so it typically needs more training data (or pretraining) to match CNN performance, but scales very well with data and compute.

**Q26. What are the drawbacks of Transformers/ViT vs CNNs?** *(explicitly asked in HyperVerge GD: "drawbacks")*
A: Self-attention has O(n²) compute/memory complexity in sequence length — expensive for long sequences or high-resolution images. Transformers lack CNNs' built-in inductive biases (locality, translation invariance), so they generally need more data or pretraining to generalize well from scratch. They're also more compute/memory hungry to train and often less interpretable in intermediate layers.

**Q27. What is batch normalization and why does it help?**
A: BatchNorm normalizes layer activations (zero mean, unit variance) across a mini-batch during training, then applies learnable scale/shift parameters. It stabilizes and speeds up training by reducing internal covariate shift, allows higher learning rates, and has a mild regularizing effect.

**Q28. Explain activation functions: sigmoid, tanh, ReLU, softmax — pros/cons.**
A: Sigmoid squashes to (0,1), useful for binary probabilities but saturates and causes vanishing gradients. Tanh is zero-centered (-1,1), same saturation issue. ReLU (max(0,x)) is cheap, avoids saturation for positive inputs, and is the default for hidden layers, but can "die" (always output 0) if a neuron's weights push it permanently negative — Leaky ReLU/ELU address this. Softmax converts a vector of logits into a probability distribution (sums to 1), used at the output layer for multi-class classification.

---

## F. NLP / RAG / LLM Fundamentals
*(Directly relevant since your flagship project, LexAI, is a RAG system — expect deeper questioning here.)*

**Q29. What is Retrieval-Augmented Generation (RAG) and why does it exist?**
A: RAG splits question-answering into two stages: retrieve relevant passages from a trusted document corpus, then feed those passages as context to an LLM which generates an answer grounded only in that context. It exists to fix three core LLM weaknesses: hallucination (answers aren't verifiable against a source), staleness (the model's knowledge is frozen at training time), and lack of domain specificity — RAG lets you update the *document corpus* without retraining the model, and lets you cite sources.
*Tie to your work:* "In LexAI I built exactly this — a legal QA system where the LLM answers Indian law questions using only retrieved statute/judgment text, never from its own memory, and every answer includes citations."

**Q30. What are embeddings, and cosine similarity vs dot product?**
A: An embedding model maps text to a fixed-length vector such that semantically similar text lands close together in vector space, regardless of exact wording. Dot product is sensitive to vector magnitude; cosine similarity divides by the magnitudes, measuring only the angle (pure semantic similarity). If vectors are pre-normalized to unit length, dot product and cosine similarity become mathematically identical — which is why many vector search systems normalize vectors and use the cheaper inner-product operation.

**Q31. What is hybrid search (dense + sparse retrieval) and why combine them?**
A: Dense retrieval (embeddings via FAISS) captures semantic similarity but can miss exact keyword/entity matches (e.g. a specific section number). Sparse retrieval (BM25) is strong at exact lexical/keyword matching but misses paraphrases and synonyms. Combining both and merging results (e.g. via Reciprocal Rank Fusion) captures the strengths of each — critical in a legal domain where exact section numbers matter as much as conceptual meaning.

**Q32. What is reranking, and why rerank after retrieval?**
A: Initial retrieval (dense/sparse/hybrid) is optimized for speed over a large corpus and can return "roughly relevant" results. A reranker (e.g. a CrossEncoder) takes the smaller candidate set and scores each (query, passage) pair jointly and much more precisely (since it isn't constrained to a pre-computed vector), reordering results so the truly best passages surface at the top before being passed to the LLM.

**Q33. What is HyDE (Hypothetical Document Embeddings)?**
A: Instead of embedding the raw user query (which can be short/ambiguous), HyDE prompts an LLM to first generate a hypothetical answer to the question, then embeds *that* hypothetical answer for retrieval — since a plausible answer tends to be semantically closer to the real relevant documents than a short question is.

**Q34. Why chunk documents, and what is semantic chunking?**
A: Chunking splits long documents into smaller pieces because embedding models and LLM context windows have size limits, and retrieval precision improves when each chunk is topically coherent. Naive fixed-size chunking can cut a sentence or legal clause mid-thought; semantic chunking splits at natural topic/meaning boundaries, preserving coherent context per chunk.

**Q35. How would you evaluate a RAG system's answer quality?**
A: Key axes: faithfulness (is the answer actually grounded in the retrieved context, not hallucinated?), answer relevance (does it address the question?), context precision/recall (did retrieval surface the right passages?). Frameworks like RAGAS automate scoring these. In practice you'd also gate deployments — fail a build if a quality metric regresses versus a baseline.

**Q36. What's the difference between fine-tuning an LLM and using RAG?**
A: Fine-tuning bakes new knowledge/behavior into model weights — expensive, slow to update, doesn't inherently give per-answer citations, and the model can still hallucinate facts it "half-learned." RAG keeps the base model frozen and injects fresh, verifiable knowledge at query time via retrieval — cheap to update (just re-index documents), and naturally supports citations. They're complementary: fine-tuning can teach a model a style/format, RAG supplies facts.

**Q37. What is tokenization, and why does it matter for LLMs?**
A: Tokenization breaks text into subword units (not always whole words) that the model actually operates on — using subwords lets a fixed vocabulary handle rare words, typos, and multiple languages without an explosion in vocabulary size. Common algorithms: BPE (Byte Pair Encoding), WordPiece, SentencePiece.

**Q38. What is prompt engineering and why does system prompt design matter in RAG?**
A: Prompt engineering is deliberately structuring the instructions and context given to an LLM to steer its output — role/system framing, explicit constraints, few-shot examples, output format. In RAG, the system prompt is what forces the "answer using ONLY the provided context, do not fabricate citations" behavior — without it, the LLM defaults to answering from its own (possibly wrong, unverifiable) training knowledge even when good context is retrieved.

---

## G. Reinforcement Learning (relevant to your Portfolio project)

**Q39. What are the core components of an RL problem?**
A: Agent, environment, state, action, reward, and policy (the agent's strategy mapping states to actions). The agent interacts with the environment over time steps, receiving a reward signal, and learns a policy that maximizes expected cumulative (discounted) reward.

**Q40. What is the exploration-exploitation tradeoff?**
A: The agent must balance exploiting the best-known action so far (to earn reward now) against exploring less-tried actions (which might reveal an even better long-term strategy). Too much exploitation risks getting stuck in a suboptimal policy; too much exploration wastes reward on trying clearly bad actions.

**Q41. What is PPO (Proximal Policy Optimization) at a high level?**
A: PPO is a policy-gradient RL algorithm that updates the policy to increase the probability of actions that led to higher-than-expected reward, but *clips* the size of each policy update so the new policy doesn't diverge too far from the old one in a single step — this stabilizes training versus older policy-gradient methods, which could take a bad large step and collapse performance.

**Q42. Why might you choose SAC over PPO, or vice versa?**
A: SAC (Soft Actor-Critic) is off-policy and maximizes reward plus an entropy bonus (encouraging exploration/randomness in the policy), typically more sample-efficient for continuous control tasks. PPO is on-policy, simpler to tune, and often more stable, at the cost of needing more fresh interaction data per update. Comparing both (as you did) is a legitimate way to check whether the extra sample efficiency of SAC translates into better real performance on your specific environment.

---

## H. Practical / System-Level ML Questions

**Q43. Walk me through the ML lifecycle end-to-end.**
A: Problem framing → data collection/labeling → EDA & cleaning → feature engineering → train/val/test split → model selection & training → evaluation against a baseline → hyperparameter tuning → deployment (serving) → monitoring for drift/degradation → retraining pipeline.

**Q44. What is data/model drift, and how do you detect it in production?**
A: Data drift is when the statistical distribution of incoming production data diverges from the training distribution (e.g. new market regime, new user behavior). Model/concept drift is when the relationship between inputs and the correct output changes over time even if input distribution is stable. Detect via monitoring input feature distributions, prediction distributions, and live accuracy/proxy metrics against a baseline, with automated alerts when a threshold is crossed — triggering retraining.

**Q45. How do you decide between a simple model and a complex/deep model?**
A: Start with the simplest model that could plausibly work (a strong baseline like logistic regression or Markowitz-style optimization) to establish a performance floor and sanity-check the pipeline; only add complexity (deep learning, RL) if it earns a statistically meaningful improvement, since complexity costs interpretability, latency, and maintenance. This mirrors your Portfolio project: you built classical Markowitz/Black-Litterman baselines *before* justifying the RL agent with a statistically significant Sharpe improvement.

**Q46. What is MLOps and why does it matter?**
A: MLOps applies DevOps discipline (automation, versioning, CI/CD, monitoring) to the ML lifecycle — experiment tracking, reproducible training, automated retraining pipelines, drift detection, and zero-downtime model serving/rollback. It matters because ML systems degrade silently (unlike traditional software bugs) as the world changes, so without monitoring and automated retraining, a model that worked at launch can quietly become wrong.

---

## I. Additional AI Fundamentals — Probability & Statistics

**Q47. What is Bayes' Theorem and why does it matter in ML?**
A: Bayes' Theorem relates conditional probabilities: P(A|B) = P(B|A)·P(A) / P(B) — it lets you update a prior belief P(A) into a posterior belief P(A|B) once you observe evidence B. It underlies Naive Bayes classifiers, Bayesian optimization for hyperparameter tuning, and the general idea of updating model beliefs as new data arrives.

**Q48. What's the difference between correlation and causation, and why does it matter for feature engineering?**
A: Correlation means two variables move together statistically; causation means one variable's change directly produces a change in the other. A model can find a highly correlated feature (e.g. ice cream sales and drowning rates, both driven by summer heat) without it being causal — using such a feature can still help prediction, but it's dangerous for decisions that assume intervention (changing the feature won't change the outcome), and can encode spurious patterns that don't generalize if the underlying confound shifts.

**Q49. What is the Central Limit Theorem and why does it matter in ML/statistics?**
A: It states that the sampling distribution of the mean of a large number of independent, identically distributed random variables approaches a normal distribution, regardless of the underlying population's distribution. It's why many statistical tests (like the t-test used to validate your Portfolio Optimization project's outperformance) are valid even when the raw data isn't normally distributed, as long as the sample size is reasonably large.

**Q50. What is a p-value, and what does p<0.05 actually mean?**
A: A p-value is the probability of observing a result at least as extreme as what you got, *assuming the null hypothesis is true* (e.g. "there is no real outperformance"). p<0.05 means there's less than a 5% chance of seeing this result by random chance alone if there were truly no effect — conventionally treated as "statistically significant," though it does NOT mean "95% probability the effect is real," which is a common misinterpretation worth being precise about if asked.

**Q51. What is a confusion matrix?**
A: A table breaking down predictions into True Positive, False Positive, True Negative, False Negative counts for a classifier — the raw basis from which precision, recall, F1, and accuracy are all computed. Useful because it shows exactly *what kind* of error a model makes, not just an aggregate score.

**Q52. Explain Type I vs Type II error.**
A: Type I error is a false positive — rejecting a true null hypothesis (e.g. flagging a legitimate transaction as fraud). Type II error is a false negative — failing to reject a false null hypothesis (e.g. missing an actual fraud case). There's typically a tradeoff between the two controlled by your decision threshold.

---

## J. Additional AI Fundamentals — Feature Engineering & Data

**Q53. What is feature scaling, and why do some models need it while others don't?**
A: Feature scaling (standardization/normalization) rescales features to a common range or distribution. Distance-based and gradient-based models (KNN, SVM, neural networks, gradient descent-based linear/logistic regression) are sensitive to feature scale — a feature ranging 0–1,000,000 can dominate one ranging 0–1 purely due to magnitude, not actual importance, and it slows/destabilizes gradient descent convergence. Tree-based models (decision trees, random forests, gradient boosting) split on thresholds per feature independently, so they're scale-invariant and don't need it.

**Q54. How do you handle missing data?**
A: Depends on the mechanism and amount: drop rows/columns if missingness is minimal and random; impute with mean/median/mode for simple cases; use model-based imputation (KNN imputer, regression imputation) for more structure-aware filling; or, for time series, forward-fill/interpolate. It's important to first understand *why* data is missing (missing completely at random vs. missing not at random) since blind imputation can introduce bias if missingness itself is informative.

**Q55. What is one-hot encoding vs label encoding, and when would you use each?**
A: Label encoding maps categories to integers (0,1,2,...) — implies a false ordinal relationship unless the categories are genuinely ordinal (e.g. low/medium/high). One-hot encoding creates a binary column per category, avoiding any false ordering — standard for nominal categorical features, though it can blow up dimensionality with high-cardinality categories (mitigated via target encoding or embeddings in that case).

**Q56. What is data leakage and how do you prevent it?**
A: Data leakage is when information from outside the training set (often, indirectly, from the future or from the target itself) leaks into the training process, producing unrealistically good validation metrics that don't hold up in production. Common causes: scaling/imputing using statistics computed on the full dataset before splitting, including a feature that's a proxy for the target, or (as in your Portfolio project) using random k-fold on time series data so training folds contain information from after the test period. Prevention: always fit preprocessing steps (scalers, imputers) only on the training fold, and use time-respecting splits for temporal data.

**Q57. What is the curse of dimensionality?**
A: As the number of features grows, the volume of the feature space grows exponentially, so data points become sparse and distances between points become less meaningful — nearest-neighbor-style methods degrade, and models need exponentially more data to maintain the same sample density. Mitigations: dimensionality reduction (PCA), feature selection, or models less sensitive to sparsity (tree ensembles, regularized linear models).

**Q58. What is PCA (Principal Component Analysis) and how does it work?**
A: PCA finds new orthogonal axes (principal components) along which the data has maximum variance, ordered by how much variance each explains, letting you reduce dimensionality by keeping only the top components while retaining most of the information. It's computed via eigendecomposition of the covariance matrix (or SVD of the data matrix) — useful for visualization, noise reduction, and speeding up downstream models, though the resulting components are linear combinations of original features and lose direct interpretability.

---

## K. Additional AI Fundamentals — Deep Learning & Training Practice

**Q59. What is transfer learning, and why is it useful?**
A: Transfer learning takes a model pretrained on a large, general dataset (e.g. ImageNet for vision, or a large text corpus for language models) and fine-tunes it on a smaller, task-specific dataset — leveraging general features/representations already learned rather than training from scratch. Especially valuable when your own labeled dataset is small, since training a deep network from scratch on limited data would badly overfit.

**Q60. What is the difference between an epoch, a batch, and an iteration?**
A: An epoch is one full pass through the entire training dataset. A batch is a subset of the data processed together in one forward/backward pass (mini-batch gradient descent). An iteration is one such batch update — so if you have 10,000 samples and a batch size of 100, one epoch = 100 iterations.

**Q61. How do you choose a learning rate, and what happens if it's too high or too low?**
A: Too high a learning rate causes the loss to oscillate or diverge, overshooting minima. Too low makes training extremely slow and can get stuck in shallow local minima/plateaus. Common practice: start with a reasonable default (e.g. 1e-3 for Adam), use a learning rate scheduler (decay over time, or warmup then decay), or run a learning-rate range test/finder to pick a good starting point empirically.

**Q62. What is early stopping?**
A: Monitor validation loss/metric during training and stop once it stops improving (or starts worsening) for a set number of epochs (patience), even if training loss keeps decreasing — a simple, effective regularization technique that prevents overfitting by not letting the model over-train on the training set.

**Q63. What's the difference between a parametric and non-parametric model?**
A: Parametric models assume a fixed functional form with a fixed number of parameters regardless of data size (linear regression, neural networks with a fixed architecture) — faster to train/predict but constrained by that assumed form. Non-parametric models grow in complexity with the data (KNN, decision trees, kernel methods) — more flexible, can fit more complex patterns, but often need more data and can be slower at inference.

**Q64. What is knowledge distillation?**
A: Training a smaller "student" model to mimic the outputs (soft probabilities, not just hard labels) of a larger, more accurate "teacher" model — the student learns richer information from the teacher's full output distribution than from ground-truth labels alone, letting you deploy a much smaller, faster model with performance closer to the large teacher than training the small model from scratch would achieve.

**Q65. What is model quantization/pruning, and why do they matter for deployment?**
A: Quantization reduces the numerical precision of model weights (e.g. 32-bit floats → 8-bit integers), shrinking model size and speeding up inference with a usually small accuracy cost. Pruning removes redundant/low-importance weights or neurons. Both matter for latency-sensitive or resource-constrained production serving — directly relevant to your Portfolio project's <200ms serving latency goal, and to any edge/mobile deployment scenario (relevant to HyperVerge's on-device document/ID verification use cases).

**Q66. What is the difference between online learning and batch learning?**
A: Batch learning trains on the full available dataset at once, then deploys a static model until manually retrained. Online learning updates the model incrementally as new data arrives, one sample or mini-batch at a time — useful when data arrives as a continuous stream or the underlying distribution shifts frequently, though it's more prone to instability from noisy individual updates.

---

## L. Additional AI Fundamentals — Rapid-Fire Conceptual Comparisons
*(Common automated-interview format: "what's the difference between X and Y?")*

**Q67. Parameter vs Hyperparameter?**
A: Parameters are learned from data during training (weights, biases). Hyperparameters are set before training and control the learning process itself (learning rate, batch size, number of layers, regularization strength) — typically tuned via grid search, random search, or Bayesian optimization.

**Q68. Epoch vs Iteration vs Batch size?** — see Q60 above.

**Q69. Model vs Algorithm?**
A: An algorithm is the general learning procedure (e.g. gradient boosting, backpropagation). A model is the specific trained artifact — the algorithm applied to specific data producing specific learned parameters.

**Q70. Loss function vs Cost function vs Objective function?**
A: A loss function measures error for a single training example. A cost function is typically the average (or sum) of the loss over the full dataset/batch. Objective function is the more general term for whatever quantity is being optimized (could include the cost function plus a regularization term).

**Q71. AI vs ML vs Deep Learning — how do they relate?**
A: AI is the broad field of building systems that perform tasks requiring intelligence. Machine Learning is a subset of AI where systems learn patterns from data rather than being explicitly programmed with rules. Deep Learning is a subset of ML using multi-layered neural networks specifically, which excel at learning hierarchical representations from large amounts of raw data (images, text, audio) without manual feature engineering.

**Q72. Feature extraction vs Feature engineering vs Feature selection?**
A: Feature engineering is creating new, informative features from raw data (e.g. computing rolling volatility from price data). Feature extraction is automatically deriving a smaller set of representative features from raw/high-dimensional data (PCA, or a CNN's learned convolutional features). Feature selection is choosing a subset of existing features that are most predictive, discarding the rest (e.g. via L1 regularization, or statistical tests).
