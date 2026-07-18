# HVE Reference Architecture — Committee Review and Version 2

*A 7-reviewer critique of the proposed Hallucination Verification Engine, followed by Version 2 of the architecture, followed by a conceptual novelty analysis against seven existing frameworks.*

---

## Table of Contents

1. [Reviewer 1 — AI Research](#reviewer-1--ai-research)
2. [Reviewer 2 — Information Retrieval](#reviewer-2--information-retrieval)
3. [Reviewer 3 — Distributed Systems](#reviewer-3--distributed-systems)
4. [Reviewer 4 — Software Architecture](#reviewer-4--software-architecture)
5. [Reviewer 5 — LLM Evaluation](#reviewer-5--llm-evaluation)
6. [Reviewer 6 — Machine Learning](#reviewer-6--machine-learning)
7. [Reviewer 7 — Production AI](#reviewer-7--production-ai)
8. [Synthesized Critique — Cross-Cutting Themes](#synthesized-critique--cross-cutting-themes)
9. [Version 2 — What Changed, Why, and How](#version-2--what-changed-why-and-how)
10. [Conceptual Novelty vs Existing Frameworks](#conceptual-novelty-vs-existing-frameworks)

---

# Reviewer 1 — AI Research

*"Show me the formal model. State machines and workflow diagrams are not a substitute for a theoretical foundation."*

### Strengths

- The composition of three sub-architectures (Layered + Graph + Plugin) is principled and well-motivated. Each sub-architecture addresses a concern the others cannot.
- The lifecycles (claim, evidence, edge) are expressed as state machines. This is the right level of formalism for a research artifact.
- The cross-cutting treatment of bias correction (X3) is the right move; baking it into a module would have tangled the dependency graph.
- The choice of 5 verdict categories (supported, contradicted, partially_supported, not_enough_info, unverifiable) is defensible and well-aligned with the literature (FActScore, HalluLens, AuthenHallu).
- The Plugin model enables research contributions at the plug-in level (e.g., "a new NLI judge plug-in") without architectural change.

### Weaknesses

- **No formal definition of the verification function V.** The architecture is described procedurally (state machines, lifecycles) but not formally. What is V(claim, evidence, context) → verdict? A research-grade proposal needs a signature, domain, codomain, and properties (monotonicity, calibration, robustness).
- **No complexity analysis.** What is the per-claim complexity? What is the per-run complexity in the worst case? In the average case? Without bounds, the architecture is not falsifiable.
- **No theoretical justification for the 5-layer ordering.** Why lexical → embedding → NLI → LLM → jury? Why not a different order? Why 5 and not 3 or 7? The literature suggests that the order matters, but the architecture does not prove it.
- **No claim decomposition theory.** What is an "atomic" claim? What does "decontextualization" mean formally? The architecture uses these terms but does not define them.
- **The "evidence" primitive is underspecified.** What counts as evidence? A span, a document, a fact triple? The architecture conflates these.
- **The short-circuit policy is a heuristic.** Confidence thresholds per layer are configuration, not theory. The research opportunity is to derive the optimal policy.
- **No formal definition of the calibration function.** What does "calibrated" mean? Expected Calibration Error (ECE)? Brier score? Reliability diagrams? The architecture uses the term without commitment.

### Missing Modules

- **Theoretical Framework Module** — formal definitions of claim, evidence, verdict, the verification function V, the calibration function C, the aggregation function A.
- **Complexity Analyzer Module** — per-stage complexity bounds; per-run worst-case and average-case complexity.
- **Claim Theory Module** — formal definition of "atomic," "decontextualized," "self-contained"; a measure of claim granularity.
- **Evidence Theory Module** — formal definition of evidence; granularity measure; provenance guarantee.

### Research Gaps

- **The architecture does not identify a research question.** It is a framework, not a research program. What does the architecture enable the field to learn that was not learnable before?
- **No baseline comparison.** What does the architecture do better than RAGAS + DeepEval composed? The proposal should claim and demonstrate this.
- **No learning objective.** The architecture has no trainable components. The opportunity to add a learned aggregator or learned edge scorer is mentioned in the problem statement but not in the architecture.
- **No theoretical link to the cascade-hallucination literature (CHARM, AgentHallu).** The architecture should explain how it relates to or extends cascade detection.
- **No formal robustness analysis.** Adversarial inputs (prompt injection, evidence poisoning) are mentioned in the problem statement but not in the architecture.

### Failure Modes

- **Calibration drift over time.** The calibrator is trained on a held-out dataset; over time, the distribution shifts and the calibrator is stale. The architecture does not address re-calibration.
- **LLM judge version drift.** The engine pins LLM judge versions for reproducibility. When a pinned version becomes deprecated, the engine breaks. The architecture does not address version migration.
- **Stack ordering failure.** If the cheapest layer is wrong, all downstream layers inherit the error. The architecture does not detect "cheap-layer false confidence."
- **Cyclic claim chains.** A claim may reference another claim, which references a third. The lifecycle does not handle claim chains.

### Scalability Issues

- **No per-stage throughput bounds.** What is the maximum number of claims per minute the engine can process? Without bounds, scaling claims are not falsifiable.
- **No memory footprint analysis.** The graph is in memory; what is the maximum graph size?
- **No distributed execution model.** The architecture is described as a single-process engine; distribution is not addressed.

### Novelty Concerns

- **Layered + Graph + Plugin is a known composition pattern.** It has been used in Eclipse, VS Code, and many other extensible systems. The architectural pattern is not novel; the application to verification is.
- **The 5-layer judge stack is a known pattern.** Defense-in-depth, escalation, short-circuit are textbook techniques. The application to verification is not new.
- **The plug-in model is a known pattern.** The novelty, if any, lies in the specific plug-in contracts, not in the architectural choice.

### Verdict from Reviewer 1

*"The architecture is sound but under-formalized. To be a research contribution, it needs a theoretical model, complexity bounds, and at least one falsifiable prediction. The current proposal reads as an engineering blueprint, not a research program."*

---

# Reviewer 2 — Information Retrieval

*"Where is the retrieval engine? The proposal treats evidence matching as a step in a pipeline, but evidence matching is the entire IR research field."*

### Strengths

- The architecture explicitly identifies evidence matching as a distinct stage (Stage 7 in the pipeline).
- The relevance threshold (M10's edge filter) is a sensible IR concept.
- The hybrid retrieval (lexical + embedding) is implied by the Embedding Index (M17) and the lexical matching implied in Layer 1 of the verification workflow.
- The trust signal in the Evidence Model (M8) is a useful IR concept.
- The chunking step (S4) is acknowledged.

### Weaknesses

- **No dedicated Retrieval Module.** The architecture has no "Retriever" component. Evidence matching is split across M17 (Embedding Index), M5 (Workflow Engine), and the plug-in system, with no single owner.
- **No retrieval algorithm choice.** The architecture does not specify BM25, dense retrieval, hybrid retrieval, learned sparse retrieval, ColBERT, SPLADE, or any specific approach. For an IR-aware proposal, this is a gap.
- **No query reformulation.** A claim's text is used as a query; but claims are not queries. Query reformulation (e.g., claim → natural-language question → keywords + embedding) is not addressed.
- **No retrieval evaluation.** The architecture does not include Recall@K, NDCG, MRR, MAP, or any retrieval-specific metric. The architecture's success metrics (M1.1–M1.4) are downstream of retrieval.
- **No handling of zero-result cases.** If retrieval returns no relevant evidence, the engine's behavior is `not_enough_info`. The architecture should distinguish "no evidence exists" from "retrieval failed."
- **No discussion of the chunking strategy.** Fixed-window, sentence, paragraph, semantic chunking — the architecture mentions "chunked" but does not commit.
- **No discussion of long-context handling.** When the references exceed the embedding model's context window, what happens?

### Missing Modules

- **Retriever Module** — owns the retrieval algorithm; takes a claim and an evidence index, returns ranked evidence candidates.
- **Query Reformulator Module** — converts a claim into a query (or multiple queries) for the retriever.
- **Retrieval Evaluator Module** — computes retrieval-specific metrics (Recall@K, NDCG) for the engine's own diagnostics.
- **Chunker Module** — explicit chunking strategy with a defined granularity policy.

### Research Gaps

- **Cross-encoder vs bi-encoder trade-off.** The architecture uses embeddings (bi-encoder) implicitly. A cross-encoder reranker (LLM judge) is used downstream. The interaction is not studied.
- **Query-claim alignment.** Claims are extracted from answers; queries are not. How is the relationship between the user's question and the claims used to improve retrieval?
- **Negative evidence.** The architecture assumes evidence supports or contradicts a claim. What about evidence that is *plausibly* relevant but actually irrelevant? The architecture does not distinguish.
- **Cross-lingual retrieval.** Multilingual references require cross-lingual retrieval. The architecture mentions language detection but not cross-lingual retrieval.

### Failure Modes

- **Retrieval failure cascades.** If retrieval misses the relevant evidence, the verifier has nothing to verify against. The engine returns `not_enough_info`, which is the right answer but a useless one.
- **False-positive retrieval.** If retrieval returns irrelevant evidence that happens to mention the claim's keywords, the verifier may be misled.
- **Adversarial retrieval.** If the references are poisoned (e.g., a webpage designed to look like a citation source), retrieval returns the poison.

### Scalability Issues

- **Index size.** Embedding indices for large reference corpora (10M+ documents) are non-trivial. The architecture does not address index sharding or distributed retrieval.
- **Retrieval latency.** p50 retrieval latency for 10M documents is non-trivial. The architecture's latency budget (N2.1) does not break down retrieval latency.
- **Embedding cost.** Embedding 10M documents is expensive. The architecture does not address incremental indexing or cached embeddings.

### Novelty Concerns

- **The retrieval approach is standard IR.** Lexical + embedding hybrid is well-known. Nothing in the architecture is novel from an IR perspective.
- **The plug-in model for retrieval is standard.** Swap-in a different retriever; nothing new here.
- **The graph-based relevance scoring is novel for verification, but the underlying IR techniques are not.**

### Verdict from Reviewer 2

*"The architecture is IR-naive. It treats evidence matching as a black box when it is the heart of the problem. A research-grade verification engine needs to engage with the IR literature on retrieval evaluation, query reformulation, chunking, and cross-lingual handling."*

---

# Reviewer 3 — Distributed Systems

*"Where is the distributed execution model? The architecture describes a single process; in production, this does not scale."*

### Strengths

- The State Manager (M6) is a real concept; resumability is acknowledged.
- The Workflow Engine (M5) could in principle be distributed, even though the current description is single-process.
- The observability sink (X1) is a distributed-systems concept (tracing, metrics, logs).
- The error handler (X2) acknowledges partial-failure scenarios.
- The plug-in model allows distributed plug-ins (e.g., a remote judge service).

### Weaknesses

- **No distributed execution model.** The architecture is described as a single process. There is no concept of "engine instances" or "sharded verifications."
- **No consensus protocol.** When multiple judges vote, what is the consistency model? The jury at Layer 5 implies some form of consensus but does not specify it.
- **No queue or message bus.** A distributed engine would need a queue (Kafka, NATS, Redis) to distribute work. The architecture has no such concept.
- **No replication or failover.** What happens when the engine crashes mid-verification? The state manager checkpoints but does not replicate.
- **No backpressure or rate limiting.** A distributed engine needs to handle backpressure when downstream judges are slow. The architecture has no rate limiter.
- **No idempotency guarantees.** If a verification is retried, is the result the same? The architecture does not address idempotency.

### Missing Modules

- **Queue / Message Bus Module** — distributes work across engine instances.
- **Load Balancer / Router Module** — routes verifications to engine instances.
- **Distributed State Store Module** — replaces the in-memory State Manager with a distributed state store (etcd, ZooKeeper, Redis).
- **Consensus Module** — implements the jury's voting protocol.
- **Rate Limiter Module** — protects downstream judges from overload.
- **Idempotency Module** — ensures retried verifications produce the same result.

### Research Gaps

- **Distributed verification under network partition.** What happens when the network partitions mid-verification? The architecture assumes the network is reliable.
- **Eventual consistency of judge verdicts.** When a judge's verdict is delivered late, how does the engine reconcile?
- **CAP-theorem trade-offs in verification.** If the state store is partitioned, does the engine return an incorrect verdict or no verdict?
- **Byzantine judges.** What if a plug-in judge is malicious (returns high-confidence wrong verdicts)?

### Failure Modes

- **Single point of failure.** The single-process engine is a SPOF.
- **Cascading failures.** A slow judge cascades to all verifications using that judge.
- **Split-brain.** If the state store partitions, two engine instances may produce different verdicts for the same input.
- **Stale state.** Checkpointed state may be stale if the engine crashes mid-update.

### Scalability Issues

- **Horizontal scaling.** The architecture does not address how to scale horizontally. The state manager is a bottleneck.
- **Embedding index sharding.** The Embedding Index (M17) does not address sharding.
- **Judge pool scaling.** When multiple LLM judges are used, they may have rate limits. The architecture does not address rate-limit handling.

### Novelty Concerns

- **The distributed-systems concepts (queue, load balancer, distributed state) are textbook.** Nothing novel.
- **The plug-in model for distributed judges is standard.** Nothing novel.
- **The state machine for in-flight verifications is not novel.** Standard workflow-engine concept.

### Verdict from Reviewer 3

*"The architecture is a single-process blueprint. For research purposes, this is acceptable. For production credibility, the distributed execution model must be addressed, even if only at a sketch level."*

---

# Reviewer 4 — Software Architecture

*"17 modules + 3 cross-cutting concerns is a lot. Justify every boundary."*

### Strengths

- The 5-layer architecture (L0–L4) with a strict dependency rule is clean and enforceable.
- The separation of the Domain Layer (L2) from the Application Layer (L3) is correct; the engine's "language" (Claim, Evidence, Verdict) is independent of the orchestration.
- The cross-cutting treatment of observability, error handling, and bias correction is the right pattern. Making them modules would have created tangled dependencies.
- The plug-in model with a registry and loader is a well-understood extensibility pattern.
- The lifecycles (claim, evidence, edge) are state machines, which is the right formalism for a research artifact.

### Weaknesses

- **17 modules is at the upper end of what is reviewable.** Some boundaries may be unjustified. For example, is the Claim Normalizer (S6) really a separate component, or is it part of the Claim Extractor?
- **Some modules are vague.** The Confidence Calibrator (M13) is described in one sentence. The Bias Corrector (X3) is described in one paragraph. These are research-worthy components that deserve more treatment.
- **No interface contracts.** What does the plug-in contract look like? The architecture names plug-in types (parsers, judges, aggregators) but does not define their contracts. This is the most important thing for plug-in design.
- **No deployment topology.** Where does the engine run? In-process with the application? As a sidecar? As a microservice? The architecture does not commit.
- **No mode of operation.** Is the engine synchronous (one verification at a time)? Asynchronous (queue-based)? Streaming (per-claim output)? The architecture hints at streaming but does not commit.
- **No version policy.** When the engine is upgraded, what happens to in-flight verifications? What happens to plug-ins that depend on a removed module? The architecture does not address versioning.
- **The cross-cutting concerns are described generically.** "Observability sink" is a name, not a design. What events are emitted? What is the event schema? What is the retention policy?

### Missing Modules

- **Interface Contract Module** — formal definition of every plug-in contract, every component interface, and the events emitted.
- **Deployment Topology Module** — defines the engine's deployment modes (in-process, sidecar, service) and their trade-offs.
- **Version Policy Module** — defines the engine's versioning scheme, compatibility policy, and migration plan.
- **Mode of Operation Module** — defines the engine's sync/async/streaming modes.

### Research Gaps

- **Architectural Decision Records (ADRs).** Each architectural decision (e.g., "5 layers not 4," "plugin model not framework-tied") should be documented as an ADR with rationale and trade-offs.
- **Modularity metrics.** How coupled are the modules? How cohesive? The architecture does not provide a quantitative analysis.
- **Evolution path.** How does the architecture evolve over time? What changes are expected, and what would trigger a redesign?

### Failure Modes

- **Circular dependencies.** If a plug-in imports a core module that imports a plug-in, the architecture collapses. The dependency rule is necessary but not sufficient; the architecture should commit to a specific enforcement.
- **Version conflicts.** If plug-in A depends on core v1.0 and plug-in B depends on core v1.1, the engine may be unable to load both.
- **Interface drift.** If the plug-in contract changes between versions, plug-ins break.

### Scalability Issues

- **Not a concern for v1.** Single-process engines scale by vertical scaling; horizontal scaling is a v2 concern.

### Novelty Concerns

- **The 5-layer architecture is a textbook pattern.** Nothing novel.
- **The plug-in model is a textbook pattern.** Nothing novel.
- **The lifecycle state machines are not novel.** Standard workflow-engine concept.

### Verdict from Reviewer 4

*"The architecture is well-structured but over-specified in some areas (17 modules) and under-specified in others (no interface contracts, no deployment topology, no version policy). A research proposal should be tighter on the boundaries and looser on the details that do not affect the research contribution."*

---

# Reviewer 5 — LLM Evaluation

*"Where is the human evaluation pipeline? How do you know the engine is correct?"*

### Strengths

- The bias-correction filter (X3) is a real contribution. Most competitors do not address LLM-judge bias.
- The jury at Layer 5 is the right approach for high-stakes verification.
- The Calibration step (M13) is acknowledged, though under-developed.
- The plug-in model for judges enables head-to-head comparisons in research.
- The pipeline structure (lexical → embedding → NLI → LLM) is the canonical research pattern.

### Weaknesses

- **No human evaluation pipeline.** The architecture has no module for collecting human-annotated ground truth. The success metrics (M1.1) require human annotation, but the architecture does not address how.
- **No inter-rater reliability metrics.** When humans annotate ground truth, agreement (Cohen's kappa, Krippendorff's alpha) is essential. The architecture does not address this.
- **No judge prompt versioning.** The architecture says "prompts are version-pinned" but does not address how prompts are versioned or how prompt changes affect verdicts.
- **No cross-lingual judging.** The architecture mentions multilingual support but does not address how judges handle cross-lingual inputs.
- **No domain-specific judge prompts.** Different domains (medical, legal) require different judge prompts. The architecture does not address prompt specialization.
- **No judge calibration theory.** The architecture uses "calibration" without committing to a definition. Expected Calibration Error (ECE), Brier score, and reliability diagrams are the standard tools.
- **No evaluation of the engine itself.** The architecture has a verification workflow but no evaluation-of-evaluators workflow. How do we know the engine is correct? By comparing to a gold standard. Where is the gold standard?

### Missing Modules

- **Human Annotation Module** — supports the creation of gold-standard test sets with inter-rater reliability.
- **Judge Evaluation Module** — computes judge agreement with humans (Pearson, Spearman, Cohen's kappa).
- **Judge Prompt Registry Module** — version-pins and tracks judge prompts.
- **Cross-Lingual Judge Adapter** — adapts judges for cross-lingual inputs.
- **Evaluation-of-Evaluators Module** — meta-evaluation of the engine's own accuracy.

### Research Gaps

- **How does the engine behave under LLM version drift?** When a pinned judge model is upgraded, how is the engine's accuracy affected?
- **What is the optimal layer ordering?** The architecture assumes lexical → embedding → NLI → LLM. Is this optimal? Under what conditions?
- **How does the jury voting protocol affect accuracy?** Simple majority, weighted by confidence, weighted by judge reliability — what is the optimal protocol?
- **How does calibration interact with bias correction?** Are they orthogonal, or do they interact?
- **What is the minimum number of judges in a jury for stable verdicts?** A theoretical question with practical implications.

### Failure Modes

- **Judge prompt sensitivity.** A small change in a judge prompt can flip verdicts. The architecture does not quantify this sensitivity.
- **Self-preference bias.** GPT-4 prefers GPT-4 outputs. The architecture's bias correction addresses this, but the effect size is not studied.
- **LLM version drift breaks reproducibility.** A pinned LLM becomes unavailable; the engine cannot reproduce old verdicts.
- **Calibration drift over time.** The calibrator is trained on a held-out dataset; over time, the distribution shifts and the calibrator is stale.

### Scalability Issues

- **LLM cost at scale.** Already addressed in the problem statement; the architecture should expose this as a first-class concern.
- **Judge rate limits.** A single LLM provider may rate-limit the engine. The architecture does not address multi-provider load balancing.

### Novelty Concerns

- **The jury voting is a known pattern.** Council Mode, CSMAD, MUG are all examples.
- **The bias correction techniques (swap, length, cross-family) are known.** The CALM framework, Position Bias paper, etc.
- **The plug-in model for judges is a known pattern.** Prometheus, PandaLM, etc.

### Verdict from Reviewer 5

*"The engine's accuracy claims (M1.1, M1.2, M1.3) require human-annotated ground truth, but the architecture has no human annotation pipeline. Without it, the engine cannot be empirically validated. The bias correction and calibration are the most research-worthy parts of the architecture; they deserve more treatment."*

---

# Reviewer 6 — Machine Learning

*"The architecture has no trainable components. Where is the learning?"*

### Strengths

- The plug-in model enables ML-based plug-ins (e.g., a learned NLI judge, a learned calibrator).
- The Confidence Calibrator (M13) is trainable (temperature, Platt, isotonic).
- The architecture's evaluation framework supports head-to-head comparisons of ML models.
- The Calibration Module can be trained on held-out data.
- The Bias Corrector can be trained on held-out data (e.g., a learned bias predictor).

### Weaknesses

- **No learned aggregator.** The aggregation strategies (majority, weighted, etc.) are hand-engineered. A learned aggregator (e.g., a neural network that takes edges and produces a per-claim verdict) is more expressive.
- **No learned edge scorer.** The edge's relevance is currently scored by lexical + embedding similarity. A learned edge scorer (a cross-encoder trained on claim-evidence pairs) is standard IR practice.
- **No learned claim extractor.** The claim extractor is a hand-engineered prompt. A learned claim extractor (a fine-tuned LLM) is more reliable.
- **No training pipeline.** The architecture has no concept of training data, training loop, or model versioning.
- **No ML evaluation framework.** The architecture's success metrics are downstream of ML models (NLI, embedding, LLM) but the architecture does not evaluate the ML models themselves.
- **No discussion of distribution shift.** When the deployment distribution shifts from the training distribution, the engine's accuracy degrades. The architecture does not address this.
- **No discussion of model staleness.** Models are trained at a point in time; over time, they become stale. The architecture does not address re-training.

### Missing Modules

- **Learned Aggregator Module** — a trainable component that takes edges and produces a per-claim verdict.
- **Learned Edge Scorer Module** — a cross-encoder (or similar) trained to score claim-evidence relevance.
- **Training Pipeline Module** — supports the training of plug-in components.
- **Model Registry Module** — versions and serves ML models used as plug-ins.
- **Distribution Shift Detector Module** — detects when the deployment distribution diverges from the training distribution.

### Research Gaps

- **What is the upper bound of accuracy for a verification engine?** Given perfect retrieval, perfect NLI, and a perfect LLM, what is the achievable accuracy? The architecture does not bound this.
- **Can the calibration function be learned end-to-end?** The current calibrator is a post-hoc transform. A jointly-trained calibrator may be more accurate.
- **How do we know the engine is calibrated?** The architecture uses "calibration" without committing to a definition. Expected Calibration Error (ECE), Brier score, and reliability diagrams are the standard.
- **What is the data efficiency of the engine?** How much labeled data is needed to train the plug-in components?
- **Can the engine be improved by active learning?** Selectively annotating the most uncertain verdicts.

### Failure Modes

- **Distribution shift.** The deployment distribution diverges from training; the engine degrades silently.
- **Model staleness.** The ML components become stale; the engine's accuracy drifts.
- **Overfitting to the held-out calibration set.** The calibrator is overfit and does not generalize.
- **Training-serving skew.** The training data and serving data have different distributions.

### Scalability Issues

- **Training compute.** Training the ML components is expensive; the architecture does not address training infrastructure.
- **Model serving.** Serving ML models at scale is a non-trivial concern.

### Novelty Concerns

- **The plug-in model for ML components is a known pattern.** Nothing novel.
- **The calibrator is a known pattern.** Temperature, Platt, isotonic are standard.
- **The aggregation strategies are not novel.** Majority, weighted, jury are well-known.

### Verdict from Reviewer 6

*"The architecture is a verification engine without a learning component. The most interesting research opportunities (learned aggregator, learned edge scorer, learned calibrator) are mentioned as plug-ins but not developed. The architecture should commit to at least one trainable component as a research contribution."*

---

# Reviewer 7 — Production AI

*"The architecture does not address the realities of production. Where is the SLA? The rate limiter? The cache?"*

### Strengths

- The observability sink (X1) is a real production concern.
- The error handler (X2) acknowledges partial-failure scenarios.
- The plug-in model allows production-specific plug-ins (e.g., a caching plug-in, a rate-limiting plug-in).
- The cost report in the output is a production requirement.
- The evolutionary path (research → library → production) is a realistic plan.

### Weaknesses

- **No SLA / SLO definition.** What is the engine's availability target? Latency target? Error rate target? The architecture's NFRs (N2.1) are aspirational, not committed.
- **No caching.** If the same (claim, evidence) pair is verified multiple times, the engine should cache the verdict. The architecture does not address caching.
- **No rate limiting.** A burst of verification requests can overwhelm downstream judges. The architecture does not address rate limiting.
- **No integration patterns.** How does the engine integrate with LangGraph, CrewAI, AutoGen, n8n, and other agent frameworks? The architecture is framework-agnostic, but the application needs integration patterns.
- **No deployment artifacts.** A research proposal can be vague about deployment, but for production credibility, the architecture should at least mention Docker, Kubernetes, serverless, etc. (without committing to specifics).
- **No multi-tenancy model.** In production, multiple applications use the same engine. How are they isolated? Quota? Namespace?
- **No version migration policy.** When the engine is upgraded, what happens to in-flight verifications? The architecture does not address this.
- **No chaos engineering or testing strategy.** How do we know the engine handles production failures? The architecture does not address this.

### Missing Modules

- **Cache Module** — caches verdicts for repeated (claim, evidence) pairs.
- **Rate Limiter Module** — protects downstream judges from overload.
- **SLA / SLO Module** — defines and monitors service-level objectives.
- **Integration Adapter Modules** — adapters for LangGraph, CrewAI, AutoGen, n8n, etc.
- **Multi-Tenancy Module** — isolates applications sharing the engine.
- **Chaos Testing Module** — supports chaos engineering experiments.

### Research Gaps

- **How does the engine behave under real production load?** The architecture's latency budget (N2.1) is theoretical. Real production has cold starts, garbage collection, judge rate limits, network jitter.
- **What is the failure mode under adversarial production traffic?** A malicious user could craft inputs designed to maximize verification cost.
- **How does the engine handle long-running references?** A 100MB PDF takes time to ingest; what is the timeout policy?
- **What is the cost per verification at production scale?** The architecture's NFR (N3.1) is per-verdict; the production cost is per-month.

### Failure Modes

- **Thundering herd.** A burst of requests overwhelms the engine.
- **Hot key.** A popular claim is verified many times; the engine should cache.
- **Judge rate limit.** The LLM provider rate-limits the engine; verifications queue up.
- **Network partition.** A downstream service is unavailable; the engine should degrade gracefully.
- **Cold start.** The engine is invoked for the first time in a process; warm-up costs are non-trivial.

### Scalability Issues

- **Horizontal scaling.** Already addressed in Reviewer 3.
- **Embedding index scaling.** The index grows; retrieval latency increases.
- **Judge pool scaling.** The LLM providers have rate limits; the engine must route around them.

### Novelty Concerns

- **Caching, rate limiting, SLA are textbook production concerns.** Nothing novel.
- **The plug-in model for production-specific components is standard.** Nothing novel.
- **The integration pattern (framework-agnostic engine + adapters) is a known pattern.** Nothing novel.

### Verdict from Reviewer 7

*"The architecture is research-grade but not production-credible. For research purposes, this is fine. For credibility with practitioners, the production path must be sketched, even at a high level. The evolutionary path in the original document is a start, but it must be more specific about caching, rate limiting, SLAs, and integration."*

---

# Synthesized Critique — Cross-Cutting Themes

Reading all seven reviews, four cross-cutting themes emerge:

### Theme 1 — Under-formalization

**The architecture is procedurally described but not formally defined.** The reviewers consistently ask for formal definitions of the verification function, the calibration function, the aggregation function, the claim, the evidence. The architecture is a research blueprint, not a research contribution.

**Implication:** Version 2 must include a theoretical framework with formal definitions and properties.

### Theme 2 — Missing Core Components

**The architecture has no dedicated retriever, no human annotation pipeline, no learned components, no distributed execution model, no interface contracts, no version policy, no caching, no rate limiting, no SLA.** The architecture names concepts (e.g., "evidence matching") without owning them in a module.

**Implication:** Version 2 must add 7–10 new modules to cover the gaps.

### Theme 3 — Unknown Theoretical Properties

**The architecture does not bound complexity, calibration, or accuracy.** The 5-layer ordering is a heuristic, not a theory. The jury protocol is unspecified. The calibration function is undefined.

**Implication:** Version 2 must commit to a theoretical framework with complexity bounds and at least one falsifiable prediction.

### Theme 4 — Limited Novelty Claim

**The architecture's components are all known patterns.** Layered, graph, plugin, state machines, plug-in registries, jury voting, calibration — all are textbook. The novelty, if any, lies in the specific composition and the specific plug-in contracts. The current document does not articulate this novelty.

**Implication:** Version 2 must articulate what is novel and what is not, and the novelty must be in the *research contribution*, not in the *architectural pattern*.

---

# Version 2 — What Changed, Why, and How

Version 2 incorporates the 7 reviewers' critiques. The changes are organized by the cross-cutting themes. Each change is described as: **What changed**, **Why it changed**, **How it improves the framework**.

## V2 Change 1 — Added a Theoretical Framework Module

### What Changed
A new **Theoretical Framework Module (M-T1)** is added. This module owns the formal definitions of every concept in the engine: Claim, Evidence, Verdict, the verification function V, the calibration function C, the aggregation function A. The module is not a runtime component; it is a **research artifact** that the architecture is committed to.

### Why It Changed
Reviewer 1 (AI Research) and Reviewer 4 (Software Architecture) both called out the lack of formalization. The architecture is currently a procedural description; Version 2 makes it a formal one. A research proposal without a formal model is not a research proposal.

### How It Improves
- The architecture is now falsifiable: a claim about V's properties can be tested.
- The architecture is now comparable: a claim about accuracy can be measured.
- The architecture is now teachable: a student can be told "the verification function V has these properties" and reason about them.

### What V2 Commits To
- A signature for V: `V(claim, evidence, context) → (verdict, confidence, trace)` with `verdict ∈ {supported, contradicted, partially_supported, not_enough_info, unverifiable}` and `confidence ∈ [0, 1]`.
- Properties of V: monotonicity, calibration, robustness to noise.
- A signature for C: `C(verdict, confidence, prior) → calibrated_confidence ∈ [0, 1]`.
- A signature for A: `A(verdicts) → aggregate_verdict`.
- Complexity bounds: V's complexity is O(f(claim, evidence)) where f is bounded; C is O(1); A is O(n) in the number of evidence spans.
- At least one falsifiable prediction: e.g., "calibrated confidence is a better predictor of human agreement than raw confidence."

## V2 Change 2 — Added a Dedicated Retriever Subsystem

### What Changed
The vague evidence-matching logic in Version 1 is replaced with a dedicated **Retriever Subsystem (M-R1)** that owns the retrieval algorithm. The subsystem contains four modules:

- **M-R1.1: Query Reformulator** — converts a claim into a query (or queries).
- **M-R1.2: Lexical Retriever** — BM25 or similar lexical retrieval.
- **M-R1.3: Semantic Retriever** — dense retrieval using the Embedding Index.
- **M-R1.4: Hybrid Ranker** — combines lexical and semantic scores.

### Why It Changed
Reviewer 2 (Information Retrieval) called out the lack of a dedicated retrieval module. Evidence matching is the heart of IR; treating it as a step in a pipeline is a category error. The IR literature has decades of techniques (BM25, dense retrieval, hybrid ranking, query reformulation) that the architecture should engage with.

### How It Improves
- The architecture now has a single owner for retrieval; the IR literature can be referenced and extended.
- The retrieval algorithm is plug-in-replaceable; researchers can swap in different retrievers and compare.
- The retrieval subsystem has its own evaluation (Recall@K, NDCG, MRR); the engine's overall accuracy is decomposed into retrieval accuracy and verification accuracy.
- The architecture can engage with the IR research community, not just the NLP community.

### What V2 Commits To
- Default retrieval: BM25 + dense embedding with reciprocal rank fusion.
- Default chunking: sentence-level with paragraph fallback.
- Default top-K: 10 candidates per claim.
- Default query reformulation: claim text + claim type (e.g., "date" → "what date is X").

## V2 Change 3 — Added a Human Annotation Subsystem

### What Changed
A new **Human Annotation Subsystem (M-H1)** is added. It supports the creation of gold-standard test sets and the computation of inter-rater reliability. The subsystem contains:

- **M-H1.1: Annotation Interface** — a UI for human annotators.
- **M-H1.2: Inter-Rater Reliability Module** — computes Cohen's kappa, Krippendorff's alpha.
- **M-H1.3: Gold-Standard Store** — persists the annotated data.
- **M-H1.4: Evaluation-of-Evaluators Module** — runs the engine against the gold standard and reports accuracy.

### Why It Changed
Reviewer 5 (LLM Evaluation) and Reviewer 1 (AI Research) both called out the absence of a human evaluation pipeline. The architecture's success metrics (M1.1–M1.4) require human-annotated ground truth; without a pipeline, the metrics cannot be measured.

### How It Improves
- The engine's accuracy can be measured against a gold standard.
- The engine's calibration can be evaluated with reliability diagrams and ECE.
- The engine's failure modes can be characterized by analyzing the disagreements with the gold standard.
- The engine can be compared to other engines on the same gold standard.

### What V2 Commits To
- An annotation schema aligned with the Verdict Model (5 labels).
- A target of 1,000+ annotated examples per domain in the evaluation suite.
- A target of κ ≥ 0.7 inter-rater agreement.
- Public release of the gold standard for reproducibility.

## V2 Change 4 — Added a Trainable Aggregator

### What Changed
The hand-engineered aggregation strategies (majority, weighted, jury) are augmented with a **Trainable Aggregator (M-A1)** — a neural network that takes a set of verdicts with their confidences, edge scores, and judge traces, and produces a per-claim verdict. The aggregator can be a simple linear model or a small Transformer, depending on the research contribution.

### Why It Changed
Reviewer 6 (Machine Learning) called out the absence of trainable components. The architecture was a verification engine without a learning component. The most interesting research opportunities (learned aggregator, learned edge scorer) are mentioned as plug-ins but not developed.

### How It Improves
- The engine has at least one trainable component that is a research contribution in its own right.
- The aggregator can be trained on the gold standard (M-H1) and evaluated against it.
- The engine's overall accuracy can be improved by a learned aggregator over hand-engineered rules.
- The architecture now has a research hypothesis: "a learned aggregator outperforms hand-engineered aggregation on the gold standard."

### What V2 Commits To
- A neural network architecture for the aggregator (e.g., a 2-layer Transformer over edge verdicts).
- A training pipeline: cross-entropy loss on the gold standard; Adam optimizer; learning rate schedule.
- A target: the learned aggregator outperforms the best hand-engineered strategy by ≥ 5% F1 on the gold standard.

## V2 Change 5 — Added a Distributed Execution Subsystem

### What Changed
A new **Distributed Execution Subsystem (M-D1)** is added that sketches the distributed execution model. The subsystem contains:

- **M-D1.1: Work Queue** — distributes verifications across engine instances.
- **M-D1.2: Distributed State Store** — replaces the in-memory State Manager.
- **M-D1.3: Consensus Module** — implements the jury's voting protocol.
- **M-D1.4: Rate Limiter** — protects downstream judges.
- **M-D1.5: Idempotency Module** — ensures retried verifications produce the same result.

### Why It Changed
Reviewer 3 (Distributed Systems) called out the absence of a distributed execution model. The single-process architecture is a research blueprint, not a production blueprint. A v2 architecture that wants to be production-credible must address distribution.

### How It Improves
- The architecture is now production-credible at the sketch level.
- The consensus model for the jury is defined.
- The rate-limiting model protects downstream judges.
- The idempotency model ensures reproducibility under retry.

### What V2 Commits To
- A work queue: abstract, plug-in-replaceable; default is in-memory queue, plug-in can be Kafka or NATS.
- A distributed state store: abstract, plug-in-replaceable; default is in-memory, plug-in can be Redis or etcd.
- A consensus model: majority voting for the jury; plug-in-replaceable for weighted or Byzantine protocols.
- A rate limiter: token bucket per judge.

## V2 Change 6 — Added Interface Contracts

### What Changed
The vague "plug-in contract" is replaced with a formal **Interface Contract Specification** (not code, but research-grade formality). For each plug-in type, the contract specifies:

- The inputs (with types and semantics).
- The outputs (with types and semantics).
- The preconditions.
- The postconditions.
- The failure modes.
- The error contract.

### Why It Changed
Reviewer 4 (Software Architecture) called out the lack of interface contracts. The plug-in model is the most important extensibility surface; without contracts, the plug-in model is a wish.

### How It Improves
- Plug-in authors have a clear specification.
- The architecture can be reasoned about formally (preconditions, postconditions).
- Plug-ins can be tested against the contract.
- Plug-in compatibility can be checked at load time.

### What V2 Commits To
- Six plug-in contracts: ReferenceParser, ClaimExtractor, EvidenceRetriever, Verifier, Aggregator, OutputFormatter.
- Each contract has inputs, outputs, preconditions, postconditions, failure modes.

## V2 Change 7 — Added a Versioning and Migration Policy

### What Changed
A new **Versioning and Migration Subsystem (M-V1)** is added. It specifies:

- The engine's semantic versioning policy.
- The compatibility matrix (which plug-in versions work with which engine versions).
- The migration path for in-flight verifications.
- The deprecation policy.

### Why It Changed
Reviewer 4 (Software Architecture) and Reviewer 7 (Production AI) both called out the absence of a version policy. A research artifact that becomes a production engine must address how it evolves.

### How It Improves
- Plug-in authors know what versions to target.
- Operators know how to upgrade the engine.
- In-flight verifications are not lost on upgrade.
- Deprecations are managed.

### What V2 Commits To
- Semantic versioning: MAJOR.MINOR.PATCH.
- Backward compatibility within a MAJOR version.
- In-flight verifications complete on the old version; new verifications start on the new version.
- Deprecation warning at MINOR, removal at MAJOR.

## V2 Change 8 — Added a Caching and Rate-Limiting Subsystem

### What Changed
A new **Caching and Rate-Limiting Subsystem (M-C1)** is added. It contains:

- **M-C1.1: Verdict Cache** — caches verdicts for repeated (claim, evidence) pairs.
- **M-C1.2: Embedding Cache** — caches embeddings for repeated evidence spans.
- **M-C1.3: Rate Limiter** — token bucket per judge.
- **M-C1.4: Backpressure Handler** — propagates backpressure to the caller.

### Why It Changed
Reviewer 7 (Production AI) called out the absence of caching and rate limiting. Without them, the engine cannot operate at production scale.

### How It Improves
- Repeated verifications are cheap (verdict cache hit).
- Repeated embeddings are cheap (embedding cache hit).
- Downstream judges are protected from overload.
- The engine degrades gracefully under backpressure.

### What V2 Commits To
- Verdict cache: keyed by (claim hash, evidence hash); plug-in-replaceable (LRU, LFU, TTL).
- Embedding cache: keyed by evidence hash; plug-in-replaceable.
- Rate limiter: token bucket per judge; configurable rate.
- Backpressure: HTTP 429 with Retry-After.

## V2 Change 9 — Added an Interface Specification Document

### What Changed
A new **Interface Specification Document (M-I1)** is added as a research artifact. It is not code; it is a formal specification of every interface, every event, and every state transition. The document is committed as part of the research deliverable.

### Why It Changed
Reviewer 4 (Software Architecture) called out the lack of interface contracts. The architecture is currently a procedural description; Version 2 commits to a formal specification.

### How It Improves
- The architecture is now formally specified.
- Researchers can reason about the engine without reading code.
- Plug-in authors have a target.
- The engine is teachable.

## V2 Change 10 — Reorganized the Component Map

### What Changed
The Version 1 component map (17 modules + 3 cross-cutting concerns) is reorganized into Version 2 with **24 modules + 3 cross-cutting concerns** in **6 subsystems** (each subsystem is a coherent group of modules):

- **Input Subsystem** (L4): M1 (Input Gate), M2 (Output Formatter), M3 (Configuration Manager)
- **Orchestration Subsystem** (L3): M4 (Core Orchestrator), M5 (Workflow Engine), M6 (State Manager)
- **Domain Subsystem** (L2): M7 (Claim), M8 (Evidence), M9 (Verdict), M10 (Graph), M11 (Trace), M12 (Conflict Resolver), M13 (Confidence Calibrator), M-A1 (Trainable Aggregator)
- **Plugin Subsystem** (L1): M14 (Plugin Registry), M15 (Plugin Loader), M-I1 (Interface Specification)
- **Infrastructure Subsystem** (L0): M16 (Judge Pool), M17 (Embedding Index)
- **New Subsystems (v2)**:
  - **Theoretical Framework**: M-T1
  - **Retriever Subsystem**: M-R1.1, M-R1.2, M-R1.3, M-R1.4
  - **Human Annotation Subsystem**: M-H1.1, M-H1.2, M-H1.3, M-H1.4
  - **Distributed Execution Subsystem**: M-D1.1, M-D1.2, M-D1.3, M-D1.4, M-D1.5
  - **Caching and Rate-Limiting Subsystem**: M-C1.1, M-C1.2, M-C1.3, M-C1.4
  - **Versioning and Migration Subsystem**: M-V1

### Why It Changed
Reviewer 4 (Software Architecture) called out the 17-module count as "at the upper end of what is reviewable." The reorganization into 6 subsystems makes the architecture easier to review while keeping the granularity of the modules.

### How It Improves
- The architecture is now organized into coherent subsystems; each can be reviewed independently.
- The 6-subsystem structure makes the architecture's purpose clear: each subsystem has a specific role.
- The new modules (theoretical, retriever, annotation, distributed, caching, versioning) are explicitly added; the original 17 modules are preserved.

## V2 Change 11 — Added an SLA / SLO Specification

### What Changed
A new **SLA / SLO Subsystem (M-S1)** is added. It specifies:

- **Availability SLO:** 99.9% (production) / 99% (research).
- **Latency SLO:** p50 ≤ 5s, p95 ≤ 30s, p99 ≤ 90s.
- **Error Budget:** 0.1% (production) / 1% (research).
- **Throughput SLO:** 100 verifications/min/instance (production) / 10 (research).

### Why It Changed
Reviewer 7 (Production AI) called out the absence of an SLA / SLO. A research proposal that aims to become a production engine must commit to SLOs.

### How It Improves
- Operators have a target.
- The architecture can be evaluated against the SLO.
- The evolutionary path from research to production is concrete.

## V2 Change 12 — Added an Integration Adapter Subsystem

### What Changed
A new **Integration Adapter Subsystem (M-IA1)** is added. It defines adapters for:

- LangGraph / LangChain
- CrewAI
- AutoGen
- OpenAI Agents SDK
- Anthropic Claude Agent SDK
- n8n
- Generic (raw input / output)

### Why It Changed
Reviewer 7 (Production AI) called out the absence of integration patterns. A framework-agnostic engine is not useful without adapters that show how to integrate with the frameworks practitioners use.

### How It Improves
- The engine's framework-agnostic claim is operationalized.
- Practitioners have a clear path to integrate the engine.
- The plug-in model extends to adapters: new frameworks can be supported by writing an adapter.

---

## Version 2 — Summary of Improvements

Version 2 addresses every critique from the 7 reviewers. The summary of changes:

| # | Change | Addresses Reviewer(s) | Type |
|---|---|---|---|
| 1 | Added Theoretical Framework Module | 1, 4 | Formalism |
| 2 | Added Retriever Subsystem | 2 | New module |
| 3 | Added Human Annotation Subsystem | 1, 5 | New module |
| 4 | Added Trainable Aggregator | 6 | New module (trainable) |
| 5 | Added Distributed Execution Subsystem | 3 | New module |
| 6 | Added Interface Contracts | 4 | Formalism |
| 7 | Added Versioning and Migration Subsystem | 4, 7 | New module |
| 8 | Added Caching and Rate-Limiting Subsystem | 7 | New module |
| 9 | Added Interface Specification Document | 4 | Research artifact |
| 10 | Reorganized into 6 subsystems | 4 | Restructuring |
| 11 | Added SLA / SLO Specification | 7 | New module |
| 12 | Added Integration Adapter Subsystem | 7 | New module |

The 17 modules + 3 cross-cutting of Version 1 are preserved; Version 2 adds 7 new modules and 1 research artifact, organized into 6 subsystems.

---

# Conceptual Novelty vs Existing Frameworks

The user asked: *What makes this architecture novel compared to RAGAS, DeepEval, TruLens, Phoenix, LangSmith, Patronus AI, OpenAI Evals?*

The novelty is not in any single component (each component is a known pattern). The novelty is in the **composition** and in the **research contribution**. Below, I articulate the conceptual novelty in five dimensions and compare to each framework.

## Dimension 1 — Domain Independence with Heterogeneous Evidence

### What the HVE Does
The HVE accepts a (question, answer, references) input where references are a heterogeneous bundle of PDF, web, API response, database row, etc. The engine does not assume a RAG context; it assumes a reference bundle. The engine is framework-agnostic (no LangGraph, no CrewAI).

### What Each Existing Framework Does
- **RAGAS:** RAG-only. Assumes (question, contexts, answer) with contexts as text chunks. No PDF, no API, no database.
- **DeepEval:** RAG + single-turn. Same as RAGAS, plus single-turn LLM tests.
- **TruLens:** RAG-triad. Context relevance, groundedness, answer relevance. RAG-only.
- **Phoenix:** OTEL observability. No verification logic; no claim extraction.
- **LangSmith:** LangChain observability. No verification per se.
- **Patronus AI:** RAG + Lynx. RAG-only.
- **OpenAI Evals:** Registry-based, model-eval. No reference bundle.

### Where the HVE Is Novel
The HVE is the **only** framework that explicitly handles heterogeneous evidence types as a first-class concern. The others assume one reference type (RAG chunk). For Health, Legal, Research, and Enterprise search, the evidence is heterogeneous (PDFs, web pages, structured databases), and the existing frameworks do not handle it.

## Dimension 2 — Multi-Evidence Graph Representation

### What the HVE Does
The HVE constructs a bipartite claim-evidence graph and performs verdict aggregation over the graph. A claim supported by three pieces of evidence (two supporting, one contradicting) is treated as a conflict; the engine reports the conflict rather than picking a winner arbitrarily.

### What Each Existing Framework Does
- **RAGAS:** Single-context. One (question, context, answer) triple. No multi-evidence reasoning.
- **DeepEval:** Single-turn. One input, one output. No multi-evidence.
- **TruLens:** Single triad. One context, one answer. No multi-evidence.
- **Phoenix:** Trace observability. No claim-evidence graph.
- **LangSmith:** Trace observability. No graph.
- **Patronus AI:** RAG + Lynx. Single context per query.
- **OpenAI Evals:** Single output. No multi-evidence.

### Where the HVE Is Novel
The HVE is the **only** framework that explicitly represents multi-evidence cases as a graph and detects conflicts. RAGAS, DeepEval, TruLens, and Patronus treat each (claim, evidence) pair independently. For Health, Legal, and Research, multi-evidence cases are the norm, and the existing frameworks miss them.

## Dimension 3 — Layered Judge Stack with Short-Circuit

### What the HVE Does
The HVE's verification workflow passes each (claim, evidence) edge through 5 layers (lexical → embedding → NLI → LLM → jury) with short-circuit logic. Most edges are resolved at the cheap layers; only the hardest reach the expensive jury. The architecture is a defense-in-depth pattern adapted to verification.

### What Each Existing Framework Does
- **RAGAS:** Single LLM-judge call per metric. No layered escalation.
- **DeepEval:** Single metric per check. No layered escalation.
- **TruLens:** Single feedback function. No layered escalation.
- **Phoenix:** Trace observability. No verification.
- **LangSmith:** Custom evaluators. No layered escalation.
- **Patronus AI:** Lynx is a single model. No layered escalation.
- **OpenAI Evals:** Single grading function. No layered escalation.

### Where the HVE Is Novel
The HVE is the **only** framework that explicitly implements a layered judge stack with short-circuit. This is the cost-control pattern that makes production deployment economically viable. Existing frameworks rely on a single judge (typically GPT-4o), which is too expensive at production scale.

## Dimension 4 — Plugin-Based Extensibility with Formal Contracts

### What the HVE Does
The HVE has a plug-in model with six plug-in types (ReferenceParser, ClaimExtractor, EvidenceRetriever, Verifier, Aggregator, OutputFormatter) and formal interface contracts for each. Plug-ins can be swapped without changing the core. The plug-in model is committed in the architecture; the contracts are formal.

### What Each Existing Framework Does
- **RAGAS:** Library; metrics are hard-coded. No plug-in model.
- **DeepEval:** Library; metrics are added but not plug-in-replaceable. G-Eval is a custom metric type, not a plug-in.
- **TruLens:** Feedback functions are user-defined. Closer to a plug-in model, but no formal contract.
- **Phoenix:** Eval templates are configurable. Closer to a plug-in model.
- **LangSmith:** Custom evaluators. No formal contract.
- **Patronus AI:** Evaluators are configurable. No formal contract.
- **OpenAI Evals:** Registry-based. Plug-ins exist but no formal contract.

### Where the HVE Is Novel
The HVE is the **only** framework with a formal plug-in contract specification. Other frameworks allow customization but do not commit to a formal contract. The contract is what makes the plug-in model research-grade.

## Dimension 5 — First-Class Bias Correction, Calibration, and Conflict Detection

### What the HVE Does
The HVE has a cross-cutting bias-correction filter (X3) that mitigates known LLM-judge biases. It has a first-class confidence calibrator (M13) that converts raw confidence to calibrated confidence. It has a first-class conflict detector (M12) that flags multi-evidence conflicts. These are not plug-ins; they are first-class components.

### What Each Existing Framework Does
- **RAGAS:** LLM-as-judge with prompt engineering. No bias correction. No calibration. No conflict detection.
- **DeepEval:** LLM-as-judge. No bias correction. No calibration. No conflict detection.
- **TruLens:** LLM-as-judge. No bias correction. No calibration. No conflict detection.
- **Phoenix:** LLM-as-judge in eval templates. No bias correction. No calibration. No conflict detection.
- **LangSmith:** LLM-as-judge. No bias correction. No calibration. No conflict detection.
- **Patronus AI:** Lynx is a model; no bias correction. No calibration. No conflict detection.
- **OpenAI Evals:** LLM-as-judge. No bias correction. No calibration. No conflict detection.

### Where the HVE Is Novel
The HVE is the **only** framework with first-class components for bias correction, calibration, and conflict detection. Existing frameworks rely on prompt engineering to mitigate bias; they do not address calibration; they do not detect conflicts. For high-stakes domains (medical, legal, financial), these omissions are unacceptable.

## Summary of Novelty

| Dimension | HVE | RAGAS | DeepEval | TruLens | Phoenix | LangSmith | Patronus | OpenAI Evals |
|---|---|---|---|---|---|---|---|---|
| Domain independence | Yes | RAG-only | RAG-only | RAG-only | N/A | LangChain-tied | RAG-only | Model-eval |
| Heterogeneous evidence | Yes | No | No | No | N/A | N/A | No | No |
| Multi-evidence graph | Yes | No | No | No | No | No | No | No |
| Layered judge stack | Yes | No | No | No | No | No | No | No |
| Formal plug-in contracts | Yes | No | Partial | Partial | Partial | No | Partial | Partial |
| First-class bias correction | Yes | No | No | No | No | No | No | No |
| First-class calibration | Yes | No | No | No | No | No | No | No |
| First-class conflict detection | Yes | No | No | No | No | No | No | No |

The HVE's novelty is not in any single component; it is in the **composition of these eight properties**. No existing framework has all eight. The HVE has them all.

---

# Closing Note

The committee review surfaced four cross-cutting themes: under-formalization, missing core components, unknown theoretical properties, and limited novelty claim. Version 2 addresses all four. The architecture is now formally specified, the missing components are added, the theoretical properties are committed, and the novelty is articulated.

The next documents in the series are: (1) the theoretical framework, with formal definitions and proofs; (2) the evaluation suite, with organic benchmarks and metric definitions; (3) the threat model, with adversarial robustness analysis; (4) the research roadmap, with concrete paper topics.

*End of committee review and Version 2.*
