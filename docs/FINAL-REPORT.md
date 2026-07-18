# FINAL REPORT

# Domain-Agnostic Hallucination Verification Framework (HVE)

**A pluggable verification engine for evidence-grounded answers across any domain**

---

| | |
|---|---|
| **Project** | HVE — Hallucination Verification Engine |
| **Author** | Harsh Kashyap · NYU research project |
| **Status** | Concept proposal for team review |
| **Document type** | Final consolidated report |
| **Date** | 2026-07-18 |
| **Audience** | Research professor, collaborators, engineering team |
| **Reading time** | 30–45 minutes (full) · 10 minutes (executive summary only) |
| **Companion documents** | `research-report.md` (3,000 lines, 250+ sources), `reference-architecture.md` (1,346 lines, v1 detail), `architecture-review-v2.md` (895 lines, committee review), `architecture-brief.md` (359 lines, 10-min brief) |

---

## Reading Map

This document is organized in **four parts** with **22 sections**. The reader can stop at any part.

| Part | Sections | Purpose | Read if... |
|---|---|---|---|
| **I. Foundation** | 1–4 | Why this matters, what's out there, what we learned | You want context |
| **II. The Architecture** | 5–13 | The complete v2 architecture, with diagrams | You want the design |
| **III. Validation** | 14–17 | Committee review, alternatives, novelty claim | You want critical analysis |
| **IV. Forward** | 18–22 | Open questions, risks, roadmap, feedback request | You want to know what's next |

For a 10-minute read: Sections 1, 2, 5, 18.
For a 30-minute read: Sections 1–5, 13, 17, 18, 21.
For the full read: All sections.

---

# PART I — FOUNDATION

## 1. Executive Summary

### 1.1 The Problem

AI systems hallucinate. A factual error in a chatbot is recoverable; a factual error in a medical search engine advising a patient, a legal research tool citing a non-existent precedent, or a financial assistant projecting a tax liability is not. As generative AI moves from "answer questions conversationally" to "answer questions in high-stakes domains," the central question is no longer *"can the model produce a fluent response?"* but *"can we trust every factual statement in the response?"*

### 1.2 The Gap

A domain-independent hallucination verification engine is missing. The surveyed landscape has the following shape:

- **RAG-shaped** tools (RAGAS, DeepEval, TruLens, Patronus) — assume one reference type, single-turn, no agent or multi-evidence
- **Observability-shaped** tools (Phoenix, LangSmith) — capture traces but do not verify
- **Model-eval-shaped** tools (OpenAI Evals, Promptfoo) — test the model, not the answer
- **Specialty** tools (Galileo, Cleanlab) — purpose-built small evaluators, closed weights, no agent eval

None accepts a heterogeneous bundle of evidence (PDFs, web pages, API responses, structured databases) and returns a per-claim verdict with confidence and source attribution, all without domain-specific configuration. None represents multi-evidence cases (a claim supported by three pieces of evidence, two of which contradict) as a first-class concept.

### 1.3 The Proposed Solution

The **Hallucination Verification Engine (HVE)** is a pluggable verification engine that takes three inputs:

- **Question** — the user's query
- **Generated Answer** — the answer to be verified
- **Supporting References** — a heterogeneous bundle of evidence (PDFs, web pages, API responses, structured databases, search results, research papers)

and returns a structured **Verification Report** with per-claim verdicts, confidence scores, evidence pointers, conflict flags, and a cost report.

HVE composes three architectural patterns orthogonally:

- **Layered (Defense-in-Depth)** — a 5-layer judge stack (lexical → embedding → NLI → LLM → jury) with short-circuit on confident verdicts, controlling cost
- **Graph (Bipartite Claim-Evidence)** — claims and evidence are nodes, verdicts are edges, conflicts are explicit graph structures
- **Plugin (Microkernel)** — six plug-in types with formal interface contracts, enabling extensibility across reference types, judges, aggregators

### 1.4 Expected Impact

- **70–85% factual reliability** on organic (real-world) benchmarks across at least five domains
- **Auditability:** every verdict is traceable to the evidence that produced it
- **Domain independence:** zero per-domain configuration
- **Cost viability:** sub-$0.05 per typical verification at production scale
- **First-class conflict detection, bias correction, and confidence calibration** — properties no existing framework provides
- **First formal research framework** that addresses the composition of multi-evidence reasoning, layered judging, and plug-in extensibility for verification

### 1.5 What This Document Is

This is the **consolidated final report** for the HVE project. It synthesizes:

- A 3,000-line research report covering 250+ sources (papers, frameworks, benchmarks, open-source implementations)
- A 1,346-line reference architecture (Version 1)
- A 895-line committee review by 7 simulated professors (Version 2 changes)
- A 359-line 10-minute architecture brief

The reader should finish this document with: (a) why this matters, (b) what existing systems miss, (c) what HVE proposes, (d) why this architecture was chosen, (e) what feedback is requested before implementation.

---

## 2. Why This Matters

### 2.1 The Stakes Have Changed

A factual error in a chatbot is recoverable. A factual error in a medical search engine advising a patient, a financial assistant projecting a tax liability, or a legal research tool quoting a non-existent precedent is not. As generative AI moves from "answer questions conversationally" to "answer questions in high-stakes domains," the cost of a hallucination rises from a UX inconvenience to a regulatory, legal, or safety incident.

### 2.2 Hallucination Is the #1 Reported Production Failure

Across the surveyed industry and academic sources, hallucination is consistently identified as the single largest production failure mode. Microsoft AIRT's 2025 taxonomy, the agentmarketcap.ai production failure breakdown, the futureagi.com 5-category taxonomy, and the Zenml 457+419 case studies converge on the same point: hallucination is reported as the #1 unmet need across AI applications — not just agents.

### 2.3 The Three Research Gaps That Motivate This Project

**Gap 1: Heterogeneous evidence.** Existing systems assume one reference type (RAG chunk, Wikipedia paragraph, or single context). Production applications have *multiple* types: PDFs, web pages, structured DBs, API responses, search results, research papers. The verification engine must handle this heterogeneity without per-domain configuration.

**Gap 2: Multi-reference aggregation.** When two references disagree, what is the verdict? When three references partially support, what is the verdict? Existing systems have no principled aggregation.

**Gap 3: Domain transferability.** A detector trained on Wikipedia (FActScore) does not transfer to medical references. A detector trained on news (HaluEval) does not transfer to legal. A domain-independent engine must work out-of-the-box across domains without per-domain retraining.

### 2.4 The Research Opportunity

The HVE is not just a tool. It is a research framework that enables a research program:

- **Heterogeneous-evidence aggregation:** how to combine verdicts across PDFs, web pages, and structured DBs?
- **Domain transferability:** how does a verification engine's accuracy transfer across Health, Legal, Research, News, Financial?
- **Calibration of verification confidence:** when the engine says "0.8 confidence," how often is it correct?
- **Adversarial robustness:** how does the engine resist prompt injection, evidence poisoning, judge attacks?
- **Compositional reasoning:** when evidence partially supports a claim, how should the engine aggregate?

These questions are publishable. The framework is the platform for the research.

---

## 3. The Current Landscape

### 3.1 The Frameworks Studied

We studied 9 frameworks that span the landscape. The synthesis below is the *insight*, not the full landscape (full coverage is in the companion `research-report.md`).

| Framework | Primary Purpose | Strengths | Limitations | Key Insight We Learned |
|---|---|---|---|---|
| **RAGAS** | RAG evaluation | Industry standard; reference-free; LLM-judge based | RAG-shaped; single-turn; no agent or multi-evidence | The claim-extract → NLI primitive is the right base; the *locus* of measurement is wrong (final answer, not every step) |
| **DeepEval** | Pytest-style LLM evaluation | 50+ metrics; multimodal; conversational; agent metrics | Per-turn; no trajectory-level; visualization gated | Metric-by-metric coverage is high but composition is manual |
| **TruLens** | RAG feedback functions | "RAG Triad"; plug-in feedback functions | RAG-shaped; primary feedback is groundedness, not multi-evidence conflict | Feedback-function pattern is right; lacks formal contract |
| **Phoenix (Arize)** | OTEL-native observability | OpenTelemetry-native; trace UI; eval templates | No verification logic; no claim extraction; no conflict detection | The OTEL GenAI spec is converging; verification engines should consume it |
| **LangSmith** | LangChain trace + eval | Deepest LangGraph integration; rich trace UI | LangChain-coupled; closed source; no verification per se | Run/Trace/Thread primitives are the right granularity for verification |
| **Patronus AI** | Eval-first with Lynx | Purpose-built hallucination detector; open weights | RAG-only; no multi-evidence; no conflict detection | A fine-tuned open-weights detector matches GPT-4 at 100× lower cost |
| **OpenAI Evals** | Registry-based model eval | Standardized; reproducible; widely cited | No observability; no online eval; hosted platform sunsetting Oct 2026 | The Completion Function Protocol is an early agent-eval primitive |
| **Promptfoo** | CLI prompt regression | CLI-first; red-team focus; CI/CD native | No observability; no agent eval | CLI-first DX is winning the developer mindshare |
| **Galileo / Cleanlab** | Sub-200ms runtime guardrails | Purpose-built small evaluator models; runtime protection | Closed weights; expensive at scale; no agent eval | Sub-200ms latency is the bar for inline production guardrails |

### 3.2 What Every Existing System Misses

1. **Heterogeneous evidence.** Every framework assumes one reference type (RAG chunk or single context). Production applications have *many* types.
2. **Multi-evidence reasoning.** Every framework treats each (claim, evidence) pair independently. None represents a claim supported by three pieces of evidence, two of which contradict.
3. **Layered judging with short-circuit.** Every framework uses a single judge (typically GPT-4o). None exploits the cost-quality curve of cheap → expensive layers.
4. **Formal plug-in contracts.** Frameworks allow customization but commit to no contract. Customization is by convention, not by specification.
5. **First-class bias correction, calibration, and conflict detection.** No framework addresses these. Existing systems rely on prompt engineering to mitigate bias; they do not address calibration; they do not detect conflicts.

**HVE addresses all five.**

### 3.3 The Underlying Research Consensus

The 250+ sources we surveyed converge on the following empirical facts:

- **No detector exceeds ~63% F1 on real human-annotated data.** (AuthenHallu: 63.91%. FaithBench: 55% F1-macro.) The ceiling is low because the task is genuinely hard.
- **The "real vs synthetic" gap is severe.** NLI and embedding methods that achieve 0% FPR on synthetic HaluEval data fail with 100% FPR on real RLHF-aligned model outputs (arXiv:2512.15068).
- **Reasoning training amplifies tool hallucination.** arXiv:2510.22977 demonstrates that reinforcement-learning-driven reasoning improvements correlate with *increased* tool fabrication.
- **Best fine-tuned detectors match GPT-4 at 400× lower cost.** MiniCheck (770M Flan-T5), Patronus Lynx, Vectara HHEM.
- **Every LLM judge has 12+ documented biases.** Position, length, self-preference, sycophancy, beauty, concreteness, sentiment, empty-reference, prefix, narrator identity, judge-ambiguity. Mitigation is necessary, not optional.
- **Cascading hallucination is the most under-addressed production need.** Every long-horizon agent is affected. CHARM (arXiv:2606.04435) is the only formal research approach.

These facts inform the HVE design.

---

## 4. The Research Foundation

The HVE design rests on four research sub-areas.

### 4.1 Hallucination Taxonomy

The 2025 agent hallucination survey (arXiv:2509.18970, Peng et al.) proposes a 5-type taxonomy for LLM-based agents: Reasoning, Execution, Perception, Memorization, Communication. The 2026 PING taxonomy (arXiv:2601.22984) groups Deep Research Agent failures into 4 loci: Propagation, Intent, Noise-induced, Grounding. The 2026 CHARM work (arXiv:2606.04435) formalizes 4 cascade patterns: Retrieval Cascade, Inference Cascade, Context Poisoning Cascade, Confidence Inflation Cascade.

For HVE, the takeaway: hallucinations occur at every stage of an agent pipeline, and they cascade. The engine must catch them per-claim and per-edge, not just per-final-answer.

### 4.2 Detection Methods

Detection methods fall into 9 families: NLI-based, LLM-as-judge, self-consistency, knowledge-base verification, fine-tuned small models, uncertainty-based, chain-of-verification, fine-grained, and graph-based. Each has a clear best-use case; no single detector dominates. The canonical pipeline is **decompose-then-verify** (FActScore, FacTool, VeriScore, Safe, Fava, AFEV, Claimify, MiniCheck, CCHD).

For HVE, the takeaway: layered judging is the right pattern. Cheap → expensive escalation with short-circuit is the cost-control mechanism.

### 4.3 LLM-as-a-Judge

Seven paradigms exist: pairwise, rubric, binary, multi-dimensional, constitutional, chain-of-verification, jury. Twelve documented biases must be mitigated. Open-source judges (Prometheus 2, M-Prometheus, PandaLM) match GPT-4 on standard criteria. Bias-correction techniques (swap-averaging, length regression, cross-family voting) are well-studied.

For HVE, the takeaway: bias correction is a first-class cross-cutting concern, not a plug-in. Calibration is required for confidence scores to be interpretable.

### 4.4 Benchmarks and Open Source

40+ benchmarks exist (HaluEval, FaithBench, FActScore, MiniCheck, AgentHallu, Collu-Bench, SQLHD, etc.). Best detectors achieve 22-33% accuracy on code hallucination (Collu-Bench), 69-83% F1 on SQL hallucination (SQLHD), 41.1% step-localization on agent trajectories (AgentHallu). No detector exceeds 63% F1 on real human-annotated data.

For HVE, the takeaway: the evaluation suite must include organic (not synthetic) benchmarks across multiple domains. The targets are realistic, not aspirational.

---

# PART II — THE ARCHITECTURE

## 5. The Composition

### 5.1 The Three Sub-Architectures

HVE is a **Hybrid (Layered + Graph + Plugin)** architecture. Three sub-architectures are composed orthogonally, not stacked:

- **Layered (Defense-in-Depth)** — cost-effective judging: lexical → embedding → NLI → LLM → jury, with short-circuit on confident verdicts
- **Graph (Bipartite Claim-Evidence)** — multi-evidence reasoning: claims and evidence are nodes; verdicts are edges; conflicts are explicit
- **Plugin (Microkernel)** — extensibility: six plug-in types (ReferenceParser, ClaimExtractor, EvidenceRetriever, Verifier, Aggregator, OutputFormatter) with formal contracts

Each sub-architecture addresses a concern the others cannot. Layered without Graph cannot represent multi-evidence. Graph without Layered pays full judge cost on every edge. Plugin without the others has no structure to plug into.

### 5.2 Why Composition, Not Stacking

Stacking sub-architectures (e.g., Pipeline on top of Layered on top of Graph) creates a sequential dependency that compounds latency and complexity. Composing them orthogonally means each sub-architecture is a *concern* of the engine, not a *stage* in a sequence. The graph is the data structure. The layered judging is the per-edge operation. The plugin model is the extensibility surface.

### 5.3 The Composition Diagram

```
                         ┌─────────────────────────────────────┐
                         │  Application Boundary               │
                         │  (the engine, as the user sees it) │
                         └─────────────────┬───────────────────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
   ┌──────────▼──────────┐      ┌──────────▼──────────┐     ┌───────────▼──────────┐
   │ Plugin Shell        │      │ Application Core     │     │ Domain Core          │
   │ (extensibility)     │      │ (orchestration)     │     │ (data structures)    │
   │                    │      │                     │     │                      │
   │ • Plugin Registry  │      │ • Orchestrator      │     │ • Claim              │
   │ • Plugin Loader    │      │ • Workflow Engine   │     │ • Evidence           │
   │ • Six plug-in      │      │ • State Manager     │     │ • Verdict            │
   │   types            │      │                     │     │ • Graph              │
   └────────────────────┘      └──────────┬──────────┘     │ • Trace              │
                                           │                └──────────┬─────────────┘
   ┌───────────────────────────────────────▼──────────────────────────┐ │
   │ Layered Judge Stack (on each graph edge)                       │ │
   │                                                                │ │
   │  Lexical  →  Embedding  →  NLI  →  LLM Judge  →  Jury         │ │
   │  (cheap)     (cheap)      (mid)    (expensive)  (consensus)    │ │
   │       │ short-circuit on confident verdict │                  │ │
   └────────────────────────────────────────────────────────────────┘ │
                                                                   │
   ┌────────────────────────────────────────────────────────────────┘
   │
   ▼
   Graph Layer (data structure)
   • Claim nodes · Evidence nodes · Edges with verdicts
   • Graph propagation · Conflict detection
```

---

## 6. High-Level Architecture

### 6.1 The Five Logical Layers

The HVE is organized into **5 logical layers** with a strict dependency rule (each layer depends only on the one below):

| Layer | Name | Responsibility |
|---|---|---|
| L4 | **Presentation** | Input parsing, output formatting, configuration interface |
| L3 | **Application** | Orchestration, workflow, lifecycle management |
| L2 | **Domain** | Data structures (Claim, Evidence, Verdict, Graph, Trace) and their invariants |
| L1 | **Plugin** | Plugin contracts, plugin registry, plugin loader |
| L0 | **Infrastructure** | Judge pool, embedding index, knowledge store, observability sinks |

The layer count is the minimum that separates concerns cleanly. Each layer is independently testable. The dependency rule prevents the layering from collapsing into a tangle.

### 6.2 The Six Subsystems (24 Modules)

The architecture is organized into **6 subsystems**, each a coherent group of modules that can be reviewed independently.

| Subsystem | Purpose | Key Modules |
|---|---|---|
| **Input** (L4) | Parse input, format output, hold configuration | Input Gate (M1), Output Formatter (M2), Configuration Manager (M3) |
| **Orchestration** (L3) | Drive the workflow, manage state | Core Orchestrator (M4), Workflow Engine (M5), State Manager (M6) |
| **Domain** (L2) | Define data types and invariants | Claim (M7), Evidence (M8), Verdict (M9), Graph (M10), Trace (M11), Conflict Resolver (M12), Confidence Calibrator (M13), **Trainable Aggregator** (M-A1) |
| **Plugin** (L1) | Define extensibility surface | Plugin Registry (M14), Plugin Loader (M15), Interface Specification (M-I1) |
| **Infrastructure** (L0) | Hold runtime resources | Judge Pool (M16), Embedding Index (M17) |
| **Cross-cutting** (X) | Filter observations and post-process verdicts | Observability Sink (X1), Error Handler (X2), Bias Corrector (X3) |

Three new modules in Version 2 (added by the committee review): Trainable Aggregator (M-A1), Interface Specification (M-I1), and the explicit identification of the cross-cutting concerns as first-class (X1, X2, X3).

---

## 7. The Major Components

Every component has a single responsibility, a defined contract, and a defined interaction surface.

### M1. Input Gate

| | |
|---|---|
| **Purpose** | Validate and normalize the user-provided input bundle. |
| **Input** | Raw input: question (String), answer (String), references (List of typed reference objects). |
| **Output** | Validated input bundle with detected language, normalized reference types, and a unique run ID. |

### M2. Output Formatter

| | |
|---|---|
| **Purpose** | Render the structured VerificationReport into the requested output format. |
| **Input** | VerificationReport; output format selection (JSON / JSONL / CSV / Markdown). |
| **Output** | A rendered output document. |

### M3. Configuration Manager

| | |
|---|---|
| **Purpose** | Hold and resolve the engine's runtime configuration. |
| **Input** | Configuration sources: defaults, user-supplied overrides, environment, plug-in profiles. |
| **Output** | A resolved Configuration object passed to the orchestrator. |

### M4. Core Orchestrator

| | |
|---|---|
| **Purpose** | Drive the verification workflow for a single run. |
| **Input** | Validated input bundle; resolved configuration. |
| **Output** | A VerificationReport. |

### M5. Workflow Engine

| | |
|---|---|
| **Purpose** | Sequence the application-level stages. |
| **Input** | A workflow plan; the available plug-ins. |
| **Output** | Stage execution results; stage checkpoints. |

### M6. State Manager

| | |
|---|---|
| **Purpose** | Hold the state of in-flight verifications. |
| **Input** | State updates from M4 and M5. |
| **Output** | State snapshots; resumable state. |

### M7. Claim Model

| | |
|---|---|
| **Purpose** | Define the data structure for an atomic claim. |
| **Input** | Raw text from the answer. |
| **Output** | Claim instances with text, normalized text, type, source span. |

### M8. Evidence Model

| | |
|---|---|
| **Purpose** | Define the data structure for a piece of evidence. |
| **Input** | Raw text or bytes from a reference. |
| **Output** | Evidence instances with text, source ID, source type, span offset, trust signal. |

### M9. Verdict Model

| | |
|---|---|
| **Purpose** | Define the data structure for a verification verdict. |
| **Input** | A (claim, evidence) pair; judge outputs. |
| **Output** | Verdict instances with label, confidence, judge trace. |

### M10. Graph Model

| | |
|---|---|
| **Purpose** | Define the data structure for the claim-evidence bipartite graph. |
| **Input** | Claim nodes, evidence nodes, edge candidates. |
| **Output** | The graph; edge operations; graph queries. |

### M11. Trace Model

| | |
|---|---|
| **Purpose** | Define the data structure for a complete verification trace. |
| **Input** | Trace events from all components. |
| **Output** | A trace object recording every step of the verification. |

### M12. Conflict Resolver

| | |
|---|---|
| **Purpose** | Detect conflicts in the verdict graph. |
| **Input** | The graph (M10). |
| **Output** | A conflict set; per-claim conflict flags. |

### M13. Confidence Calibrator

| | |
|---|---|
| **Purpose** | Convert raw confidence to calibrated confidence. |
| **Input** | Raw verdicts; calibration model. |
| **Output** | Calibrated verdicts. |

### M14. Plugin Registry

| | |
|---|---|
| **Purpose** | Maintain the catalog of available plug-ins. |
| **Input** | Plug-in metadata at startup; plug-in self-registration. |
| **Output** | Plug-in lookups; plug-in lists per category. |

### M15. Plugin Loader

| | |
|---|---|
| **Purpose** | Instantiate and lifecycle-manage plug-ins. |
| **Input** | Plug-in IDs; instantiation context. |
| **Output** | Plug-in instances; plug-in capability exposure. |

### M16. Judge Pool

| | |
|---|---|
| **Purpose** | Hold the available judges (NLI, LLM, etc.). |
| **Input** | Judge descriptors at startup. |
| **Output** | Judge instances; judge selection per layer. |

### M17. Embedding Index

| | |
|---|---|
| **Purpose** | Hold embeddings of evidence and (optionally) claims for retrieval. |
| **Input** | Evidence texts; claim texts. |
| **Output** | Top-K nearest neighbors for a query. |

### M-A1. Trainable Aggregator *(V2 — added by committee review)*

| | |
|---|---|
| **Purpose** | A neural network that takes a set of verdicts with their confidences, edge scores, and judge traces, and produces a per-claim verdict. |
| **Input** | A set of edges with verdicts. |
| **Output** | A per-claim verdict with confidence. |

### Cross-Cutting Concerns

- **X1. Observability Sink:** every component emits trace events. The sink aggregates and exports.
- **X2. Error Handler:** every component has defined failure modes. The handler captures and routes.
- **X3. Bias Corrector:** a filter that post-processes LLM verdicts to mitigate known biases (position, length, self-preference).

---

## 8. The 11-Stage Execution Pipeline

### 8.1 The Pipeline at a Glance

The execution pipeline is the ordered sequence of stages that turns an input bundle into a verification report. The pipeline is **strictly linear at the stage level** (S_i feeds S_{i+1}) but **branching at the component level** (a stage may use multiple components).

```
   INPUT (question, answer, references)
     │
     ▼
  S1  Input Acceptance          (validate schema, detect language)
     │
     ▼
  S2  Configuration Resolution  (load config, select plug-ins, select judges)
     │
     ▼
  S3  Reference Ingestion       (per-type parser → text + provenance)
     │
     ▼
  S4  Reference Indexing        (chunk, embed, lexical index, cache)
     │
     ▼
  S5  Claim Extraction          (decompose answer into atomic claims)
     │
     ▼
  S6  Claim Normalization       (resolve coreferences, filter subjective)
     │
     ▼
  S7  Evidence Matching         (per-claim top-K candidate evidence)
     │
     ▼
  S8  Edge Construction         (build bipartite graph, score edges, filter)
     │
     ▼
  S9  Layered Verification      (5-layer judge stack on each edge)
     │
     ▼
  S10 Conflict Resolution       (flag, do not adjudicate)
     │
     ▼
  S11 Aggregation & Scoring     (per-claim + cross-claim + calibration)
     │
     ▼
   OUTPUT (Verification Report)
```

### 8.2 Pipeline Properties

- **Determinism:** given the same input, configuration, and plug-in versions, the pipeline produces the same output. (LLM stochasticity is contained within S9 and mitigated by bias correction and jury voting.)
- **Resumability:** each stage produces a checkpointed state. If the pipeline fails at S9, it can resume from S9.
- **Parallelism:** S5–S6 and S7–S8 are embarrassingly parallel across claims and evidence.
- **Observability:** each stage emits a trace event.

---

## 9. The Three Lifecycles

The engine has three lifecycles: Claim, Evidence, and Edge. The lifecycles synchronize at the bipartite graph: an edge exists if and only if both sides are in `MATCHED` state; an edge is `RECORDED` if and only if it has been judged.

### 9.1 Claim Lifecycle (9 States)

```
  RAW → EXTRACTED → NORMALIZED → TYPED → ELIGIBLE →
  MATCHED → VERIFIED → AGGREGATED → REPORTED

  (side states: UNVERIFIABLE, UNMATCHED, CONFLICT)
```

| State | Entry condition | Operations |
|---|---|---|
| RAW | Text from the answer, not yet extracted | None (initial state) |
| EXTRACTED | Claim extractor decomposed the answer | Atomic, spans preserved |
| NORMALIZED | Normalizer resolved coreferences | Decontextualized |
| TYPED | Typer assigned type and certainty | Entity type, fact type |
| ELIGIBLE | Filter passed: verifiable, not subjective | Eligible for matching |
| MATCHED | ≥1 evidence candidate found | Edges created |
| VERIFIED | Layered judge stack assigned a verdict | Verdict + confidence |
| AGGREGATED | Per-claim verdict combined with others | Overall score |
| REPORTED | Output formatter produced the report | Final state |

### 9.2 Evidence Lifecycle (8 States)

```
  RAW → PARSED → CHUNKED → INDEXED → MATCHED →
  RELEVANT → JUDGED

  (terminal: REJECTED, CONTRADICT)
```

| State | Entry condition | Operations |
|---|---|---|
| RAW | Bytes or text from a reference | None (initial state) |
| PARSED | Parser extracted text and metadata | Type identified |
| CHUNKED | Chunker split into spans | Sentence / window / paragraph |
| INDEXED | Indexer embedded and indexed | Embedding + lexical index |
| MATCHED | Claim query retrieved this evidence | Candidate for a claim |
| RELEVANT | Above relevance threshold | Edge created |
| JUDGED | Edge judged by the layered stack | Verdict stored on edge |

**Invariant:** a piece of evidence is in `JUDGED` state if and only if it is on a judged edge in the graph.

### 9.3 Edge Lifecycle (Verification Lifecycle)

```
              ┌──────────┐
              │ CREATED  │  (edge exists; relevance known)
              └─────┬────┘
                    ▼
            ┌──────────────┐
            │  JUDGED-L1   │  (lexical judge returned)
            └──────┬───────┘
                   │ short-circuit?
              ┌────┴────┐
             YES       NO
              │         │
              │         ▼
              │   JUDGED-L2  →  JUDGED-L3  →  JUDGED-L4  →  JUDGED-L5
              │   (embed)      (NLI)        (LLM)        (jury)
              ▼
       ┌──────────────┐
       │   RECORDED   │  (verdict + bias-corrected confidence)
       └──────────────┘
```

The per-edge state machine captures the essential property of the layered judge stack: each edge progresses through the layers, short-circuiting when a layer is confident, ending in a recorded verdict.

---

## 10. The 5-Layer Judge Stack

For each (claim, evidence) edge, the engine runs through 5 layers:

1. **Lexical** (cheap, deterministic) — token overlap, substring match, citation URL ping
2. **Embedding** (cheap) — cosine similarity over a small embedding model
3. **NLI** (mid-cost) — premise-hypothesis entailment using a fine-tuned NLI model (DeBERTa / MiniCheck / Lynx)
4. **LLM Judge** (expensive) — frontier LLM with a rubric-based prompt
5. **Multi-Judge Consensus** (jury) — multiple LLM judges with bias correction

Each layer can short-circuit on a confident verdict. Most claims are resolved at the cheap layers; a minority reach the expensive jury.

### 10.1 The Workflow

```
              ┌─────────────────────┐
              │   For each edge     │
              │   (claim, evidence) │
              └──────────┬──────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  L1. Lexical Judge   │
              │  • Token overlap     │
              │  • Substring match   │
              │  • Citation ping     │
              └──────────┬───────────┘
                         │ confident?
                    ┌────┴────┐
                   YES       NO
                    │         │
                    ▼         ▼
            ┌──────────┐  ┌──────────────────┐
            │ short-   │  │ L2. Embedding    │
            │ circuit  │  │ • Cosine         │
            └──────────┘  └────────┬─────────┘
                                   │ confident?
                              ┌────┴────┐
                             YES       NO
                              │         │
                              ▼         ▼
                       ┌──────────┐  ┌──────────────┐
                       │ short-   │  │ L3. NLI      │
                       │ circuit  │  │ • DeBERTa    │
                       └──────────┘  │ • MiniCheck  │
                                    │ • Lynx       │
                                    └──────┬───────┘
                                           │ confident?
                                      ┌────┴────┐
                                     YES       NO
                                      │         │
                                      ▼         ▼
                              ┌──────────┐ ┌──────────────┐
                              │ short-   │ │ L4. LLM Judge│
                              │ circuit  │ │ • Frontier   │
                              └──────────┘ │ • Rubric     │
                                           └──────┬───────┘
                                                  │ confident?
                                             ┌────┴────┐
                                            YES       NO
                                             │         │
                                             ▼         ▼
                                     ┌──────────┐ ┌──────────────┐
                                     │  record  │ │ L5. Jury     │
                                     │  verdict │ │ • Multi-LLM  │
                                     └──────────┘ │ • Bias corr. │
                                                 └──────────────┘
```

### 10.2 The Bias-Correction Filter (Cross-Cutting)

Layer 4 and Layer 5 verdicts are subject to known biases. The Bias Corrector (X3) applies:

- **Swap-averaging:** for pairwise judgments, swap and re-judge, average
- **Length regression:** subtract the estimated length contribution from the confidence
- **Cross-family voting:** if multiple LLM judges are used, vote across model families
- **Constitutional rubric:** apply an explicit rubric and re-judge against it

The bias corrector is a filter, not a stage. It does not produce a verdict; it adjusts the verdict.

---

## 11. Information Flow

### 11.1 The Three Data Shapes

The engine has three canonical data shapes: input, internal, output.

```
   USER-PROVIDED                        INTERNAL                            ENGINE-RETURNED

┌────────────────────┐         ┌────────────────────┐              ┌────────────────────┐
│ question: String    │         │ VerificationRun    │              │ VerificationReport │
│ answer:   String    │ ──────► │  ├─ claim_list     │ ──────────►  │  ├─ overall_score  │
│ references: [       │         │  ├─ evidence_pool  │              │  ├─ per_claim[]    │
│   {                 │         │  ├─ graph          │              │  ├─ conflicts[]    │
│     type: Enum      │         │  └─ verdict_set    │              │  ├─ coverage_ratio │
│     content: Bytes  │         │                    │              │  └─ cost_report    │
│     metadata: Map   │         └────────────────────┘              └────────────────────┘
│   }                 │
│ ]                   │
└────────────────────┘
```

### 11.2 The Five Internal Transformations

Within a single verification, the data flows through 5 transformations, each a pure function of the previous:

1. **Normalization** — heterogeneous inputs become uniform internal types
2. **Claim extraction** — the answer is decomposed into atomic claims
3. **Evidence matching** — for each claim, candidate evidence spans are retrieved
4. **Verification** — for each (claim, evidence) pair, a verdict is produced
5. **Aggregation** — per-claim verdicts are combined into an overall result

The pipeline is **pipeline-correct**: it cannot enter an inconsistent state because each stage is a function of the previous stage's output.

---

## 12. Confidence Estimation and Calibration

### 12.1 The Three Sources of Confidence

The engine produces confidence from three sources:

- **Per-layer confidence:** the confidence reported by the judging layer (e.g., NLI's entailment probability)
- **Cross-edge aggregation:** the per-claim confidence is aggregated from all edges using a configured strategy
- **Calibration:** temperature, Platt, or isotonic scaling (plug-in choice)

### 12.2 The Calibration Module

Raw LLM confidence is poorly calibrated. The calibrator applies:

- **Temperature scaling:** a single temperature parameter learned on a held-out dataset
- **Platt scaling:** a one-parameter logistic model learned on held-out data
- **Isotonic regression:** a non-parametric mapping learned on held-out data

The output report exposes both raw and calibrated confidence so operators can compare.

### 12.3 The Six Aggregation Strategies

| Strategy | Description | When to use |
|---|---|---|
| **Majority** | The most common verdict across edges wins | Multiple references of equal weight |
| **Weighted** | Each edge's verdict is weighted by its source trust score | References with heterogeneous credibility |
| **Source-priority** | Verdicts from higher-priority sources override lower-priority | Domain-specific credibility ordering |
| **Jury** | Verdicts are treated as votes; jury rules apply | Multiple LLM judges |
| **Max-contradiction** | A single contradicting edge flips the claim to contradicted | High-stakes domains (medical, legal) |
| **Min-confidence** | The minimum confidence across edges is the per-claim confidence | Conservative reporting |

### 12.4 The Trainable Aggregator (V2)

The hand-engineered strategies are augmented with a **Trainable Aggregator (M-A1)** — a neural network that takes a set of verdicts with their confidences, edge scores, and judge traces, and produces a per-claim verdict. The aggregator can be a simple linear model or a small Transformer, depending on the research contribution.

---

## 13. Final Reporting

The Verification Report contains:

- **Per-claim verdicts:** label, confidence, evidence pointers, judge trace
- **Conflict flags:** which claims have conflicting evidence
- **Coverage ratio:** fraction of claims that were verifiable
- **Reliability score:** overall factual reliability
- **Cost report:** per-stage and per-judge counts
- **Audit trail:** the complete trace for reproducibility

Output formats: JSON, JSONL, CSV, Markdown. Streaming output supported for long answers.

### 13.1 The Cost Report

The cost report exposes:

- **Per-stage counts:** how many claims, evidence spans, edges, verdicts
- **Per-judge counts:** how many times each judge was invoked
- **Per-layer counts:** how many edges were short-circuited at which layer
- **Token counts:** total tokens consumed (if the judge exposes this)
- **Time breakdown:** time per stage

The cost report is the operator's tool for tuning the engine.

---

# PART III — VALIDATION

## 14. The Committee Review

We subjected the Version 1 architecture to a 7-reviewer committee critique. Each reviewer brought the perspective of their discipline.

### 14.1 Reviewer 1 — AI Research

**Verdict:** *"Sound but under-formalized. A research proposal without a formal model is not a research proposal."*

Key issues:
- No formal definition of the verification function V
- No complexity analysis
- No theoretical justification for the 5-layer ordering
- No claim decomposition theory
- The "evidence" primitive is underspecified
- The short-circuit policy is a heuristic

### 14.2 Reviewer 2 — Information Retrieval

**Verdict:** *"IR-naive. Evidence matching is the heart of the problem, not a step in a pipeline."*

Key issues:
- No dedicated Retriever Module
- No retrieval algorithm choice (BM25, dense, hybrid)
- No query reformulation
- No retrieval evaluation (Recall@K, NDCG)
- No handling of zero-result cases

### 14.3 Reviewer 3 — Distributed Systems

**Verdict:** *"Single-process blueprint. For production credibility, the distributed model must be addressed."*

Key issues:
- No distributed execution model
- No consensus protocol
- No queue or message bus
- No replication or failover
- No backpressure or rate limiting

### 14.4 Reviewer 4 — Software Architecture

**Verdict:** *"Over-specified in some areas, under-specified in others. No interface contracts."*

Key issues:
- 17 modules is at the upper end of what is reviewable
- No interface contracts
- No deployment topology
- No mode of operation (sync / async / streaming)
- No version policy

### 14.5 Reviewer 5 — LLM Evaluation

**Verdict:** *"No human evaluation pipeline. The success metrics can't be measured without one."*

Key issues:
- No human annotation pipeline
- No inter-rater reliability metrics
- No judge prompt versioning
- No cross-lingual judging
- No judge calibration theory

### 14.6 Reviewer 6 — Machine Learning

**Verdict:** *"No trainable components. The most interesting research opportunities are mentioned as plug-ins but not developed."*

Key issues:
- No learned aggregator
- No learned edge scorer
- No training pipeline
- No distribution shift detection
- No model staleness handling

### 14.7 Reviewer 7 — Production AI

**Verdict:** *"Research-grade but not production-credible. Caching, rate limiting, SLAs, integration missing."*

Key issues:
- No SLA / SLO definition
- No caching
- No rate limiting
- No integration patterns
- No multi-tenancy model
- No chaos engineering

### 14.8 The Four Cross-Cutting Themes

Reading all seven reviews, four themes emerged:

1. **Under-formalization.** The architecture is procedurally described but not formally defined.
2. **Missing core components.** No dedicated retriever, no human annotation pipeline, no learned components, no distributed execution, no interface contracts, no caching, no rate limiting.
3. **Unknown theoretical properties.** No complexity bounds, no calibration theory, no falsifiable predictions.
4. **Limited novelty claim.** All components are known patterns; novelty must be in the *research contribution*, not in the *architectural pattern*.

---

## 15. Version 2 — The 12 Changes

Version 2 incorporates the 7 reviewers' critiques. The changes are organized by the cross-cutting themes. Each change is described as: **What changed**, **Why it changed**, **How it improves the framework**.

### 15.1 Added a Theoretical Framework Module

**What:** A new **Theoretical Framework Module (M-T1)** owns the formal definitions of every concept: Claim, Evidence, Verdict, the verification function V, the calibration function C, the aggregation function A.

**Why:** Reviewers 1 and 4 called out the lack of formalization. A research proposal without a formal model is not a research proposal.

**How it improves:** The architecture is now falsifiable, comparable, and teachable.

**V2 Commits To:** A signature for V with monotonicity, calibration, and robustness properties; a signature for C; a signature for A; complexity bounds; at least one falsifiable prediction.

### 15.2 Added a Dedicated Retriever Subsystem

**What:** The vague evidence-matching logic is replaced with a dedicated **Retriever Subsystem (M-R1)** containing Query Reformulator, Lexical Retriever, Semantic Retriever, and Hybrid Ranker.

**Why:** Reviewer 2 called out the lack of a dedicated retrieval module. Evidence matching is the heart of IR; treating it as a step in a pipeline is a category error.

**How it improves:** The architecture now has a single owner for retrieval; the IR literature can be referenced; the retrieval algorithm is plug-in-replaceable.

**V2 Commits To:** Default BM25 + dense with reciprocal rank fusion; sentence-level chunking; top-K=10.

### 15.3 Added a Human Annotation Subsystem

**What:** A new **Human Annotation Subsystem (M-H1)** supports the creation of gold-standard test sets and inter-rater reliability. Contains Annotation Interface, Inter-Rater Reliability Module, Gold-Standard Store, Evaluation-of-Evaluators Module.

**Why:** Reviewers 1 and 5 called out the absence of a human evaluation pipeline. The success metrics require human-annotated ground truth.

**How it improves:** The engine's accuracy can be measured; calibration can be evaluated; failure modes can be characterized.

**V2 Commits To:** Annotation schema aligned with the Verdict Model; ≥1,000 annotated examples per domain; κ ≥ 0.7.

### 15.4 Added a Trainable Aggregator

**What:** A new **Trainable Aggregator (M-A1)** — a neural network that takes a set of verdicts and produces a per-claim verdict.

**Why:** Reviewer 6 called out the absence of trainable components. The most interesting research opportunities are mentioned as plug-ins but not developed.

**How it improves:** The engine has at least one trainable component that is a research contribution; the architecture has a research hypothesis.

**V2 Commits To:** A neural network architecture (2-layer Transformer over edge verdicts); cross-entropy loss; ≥5% F1 improvement over best hand-engineered strategy.

### 15.5 Added a Distributed Execution Subsystem

**What:** A new **Distributed Execution Subsystem (M-D1)** containing Work Queue, Distributed State Store, Consensus Module, Rate Limiter, Idempotency Module.

**Why:** Reviewer 3 called out the absence of a distributed execution model.

**How it improves:** The architecture is now production-credible at the sketch level; the consensus model is defined; rate-limiting and idempotency are addressed.

### 15.6 Added Interface Contracts

**What:** A formal **Interface Contract Specification** for each plug-in type: inputs, outputs, preconditions, postconditions, failure modes, error contract.

**Why:** Reviewer 4 called out the lack of interface contracts.

**How it improves:** Plug-in authors have a clear specification; the architecture can be reasoned about formally.

### 15.7 Added a Versioning and Migration Subsystem

**What:** A new **Versioning and Migration Subsystem (M-V1)** specifies semantic versioning, compatibility matrix, migration path, deprecation policy.

**Why:** Reviewers 4 and 7 called out the absence of a version policy.

**How it improves:** Plug-in authors know what versions to target; operators know how to upgrade; in-flight verifications are preserved.

### 15.8 Added a Caching and Rate-Limiting Subsystem

**What:** A new **Caching and Rate-Limiting Subsystem (M-C1)** contains Verdict Cache, Embedding Cache, Rate Limiter, Backpressure Handler.

**Why:** Reviewer 7 called out the absence of caching and rate limiting.

**How it improves:** Repeated verifications are cheap; downstream judges are protected; the engine degrades gracefully.

### 15.9 Added an Interface Specification Document

**What:** A new **Interface Specification Document (M-I1)** is committed as a research artifact. Not code; a formal specification of every interface, event, and state transition.

**Why:** Reviewer 4 called out the lack of interface contracts.

**How it improves:** The architecture is now formally specified; plug-in authors have a target; the engine is teachable.

### 15.10 Reorganized the Component Map

**What:** The 17 modules + 3 cross-cutting concerns are reorganized into 24 modules + 3 cross-cutting in 6 subsystems.

**Why:** Reviewer 4 called out 17 modules as at the upper end of what is reviewable.

**How it improves:** The architecture is organized into coherent subsystems; each can be reviewed independently.

### 15.11 Added an SLA / SLO Specification

**What:** A new **SLA / SLO Subsystem (M-S1)** specifies availability, latency, error budget, throughput SLOs.

**Why:** Reviewer 7 called out the absence of SLAs.

**How it improves:** Operators have a target; the architecture can be evaluated against the SLO; the evolutionary path is concrete.

### 15.12 Added an Integration Adapter Subsystem

**What:** A new **Integration Adapter Subsystem (M-IA1)** defines adapters for LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, Anthropic Claude Agent SDK, n8n, and a generic adapter.

**Why:** Reviewer 7 called out the absence of integration patterns.

**How it improves:** The engine's framework-agnostic claim is operationalized; practitioners have a clear integration path.

### 15.13 Summary of V2 Changes

| # | Change | Addresses | Type |
|---|---|---|---|
| 1 | Added Theoretical Framework Module | R1, R4 | Formalism |
| 2 | Added Retriever Subsystem | R2 | New module |
| 3 | Added Human Annotation Subsystem | R1, R5 | New module |
| 4 | Added Trainable Aggregator | R6 | New trainable |
| 5 | Added Distributed Execution Subsystem | R3 | New module |
| 6 | Added Interface Contracts | R4 | Formalism |
| 7 | Added Versioning and Migration Subsystem | R4, R7 | New module |
| 8 | Added Caching and Rate-Limiting Subsystem | R7 | New module |
| 9 | Added Interface Specification Document | R4 | Research artifact |
| 10 | Reorganized into 6 subsystems | R4 | Restructuring |
| 11 | Added SLA / SLO Subsystem | R7 | New module |
| 12 | Added Integration Adapter Subsystem | R7 | New module |

---

## 16. Alternative Architectures Considered

We considered 4 alternatives before selecting Hybrid (Layered + Graph + Plugin).

| Alternative | Core Idea | Advantages | Drawbacks | Why Not Selected |
|---|---|---|---|---|
| **Pure LLM-as-a-Judge** | One LLM call per claim; no layered judging | Simple; flexible; one prompt to write | Too expensive at scale; bias-prone; no short-circuit; no multi-evidence | Single judge is uneconomic and inadequate for high-stakes domains |
| **Retrieval-first verification** | Strong retriever first; only verify what retrieval returns | Strong when retrieval is good; rejects bad inputs early | Retrieval failure cascades silently; treats evidence-matching as a black box | Hides the IR problem; cannot detect conflicts; brittle when retrieval fails |
| **Multi-agent verifier** | Multiple LLM agents debate each claim (Council Mode, CSMAD) | Demonstrated 39% reduction on TruthfulQA; multiple paradigms | 3–5× cost; hard to reproduce; not plug-in-extensible | Cost is prohibitive at scale; bias not formally addressed |
| **Plugin-first architecture** | Microkernel + plug-ins, no graph, no layered judging | Maximum extensibility; small core | No multi-evidence reasoning; no cost control via short-circuit; plug-in quality variance | Inadequate for the multi-evidence cases that production requires |
| **Hybrid (Layered + Graph + Plugin)** ← *selected* | All three composed: graph is the data structure, layered judges operate on edges, plug-ins extend every boundary | Multi-evidence native; cost-effective; extensible; vendor-neutral | Higher conceptual complexity; longer ramp-up | The composition is the only one that addresses all five gaps from Section 3.2 |

**When alternatives may be preferable:**
- *Pure LLM-as-a-Judge* is preferable for low-volume, high-stakes single-claim evaluation (e.g., a one-off legal brief review).
- *Retrieval-first* is preferable when the retrieval system is already very strong and the cost of verification is dominated by retrieval.
- *Multi-agent verifier* is preferable for novel claim types where no single judge is reliable.
- *Plugin-first* is preferable for a small team that wants to ship quickly without a graph layer.

---

## 17. Conceptual Novelty vs Existing Frameworks

### 17.1 The Five Dimensions of Novelty

The novelty is not in any single component (each component is a known pattern). The novelty is in the **composition** and in the **research contribution**.

**Dimension 1 — Domain Independence with Heterogeneous Evidence.** HVE accepts a (question, answer, references) input where references are a heterogeneous bundle. No existing framework treats heterogeneous evidence as a first-class concern.

**Dimension 2 — Multi-Evidence Graph Representation.** HVE constructs a bipartite claim-evidence graph and detects conflicts. No existing framework represents multi-evidence cases as a graph.

**Dimension 3 — Layered Judge Stack with Short-Circuit.** HVE's verification workflow passes each edge through 5 layers with short-circuit logic. No existing framework implements this cost-control pattern.

**Dimension 4 — Plugin-Based Extensibility with Formal Contracts.** HVE has six plug-in types with formal interface contracts. No existing framework commits to formal contracts.

**Dimension 5 — First-Class Bias Correction, Calibration, and Conflict Detection.** HVE has cross-cutting components for all three. No existing framework has any of them.

### 17.2 The Comparison Table

| Dimension | HVE | RAGAS | DeepEval | TruLens | Phoenix | LangSmith | Patronus | OpenAI Evals |
|---|---|---|---|---|---|---|---|---|
| Domain independence | ✓ | ✗ | ✗ | ✗ | N/A | ✗ | ✗ | ✗ |
| Heterogeneous evidence | ✓ | ✗ | ✗ | ✗ | N/A | N/A | ✗ | ✗ |
| Multi-evidence graph | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Layered judge stack | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Formal plug-in contracts | ✓ | ✗ | partial | partial | partial | ✗ | partial | partial |
| First-class bias correction | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| First-class calibration | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| First-class conflict detection | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

**The novelty is not in any single component. It is in the composition of these eight properties. No existing framework has all eight.**

---

# PART IV — FORWARD

## 18. Open Research Questions

1. **What is the upper bound of verification accuracy?** Given perfect retrieval, perfect NLI, and a perfect LLM, what is the achievable accuracy?
2. **What is the optimal layer ordering?** Is lexical → embedding → NLI → LLM → jury optimal? Under what conditions?
3. **Can the calibration function be learned end-to-end?** A jointly-trained calibrator may outperform a post-hoc transform.
4. **How does the engine behave under LLM version drift?** When a pinned judge model is upgraded, how is accuracy affected?
5. **What is the minimum number of judges in a jury for stable verdicts?** A theoretical question with practical implications.
6. **Can the engine be improved by active learning?** Selectively annotating the most uncertain verdicts.
7. **How does the engine handle cross-lingual evidence?** Multilingual references require cross-lingual retrieval and judging.
8. **What is the formal relationship between the engine and the cascade-hallucination literature (CHARM, AgentHallu)?**

---

## 19. Risks and Assumptions

### 19.1 Potential Risks

- **Calibration drift over time** — the calibrator is trained on a held-out dataset; over time, the distribution shifts.
- **LLM version drift** — pinned judge versions become deprecated; reproducibility breaks.
- **Retrieval failure cascade** — if the retriever misses the relevant evidence, the verifier has nothing to verify against.
- **Adversarial references** — poisoned evidence (a webpage designed to look like a citation) could mislead the engine.
- **Conceptual complexity** — the Hybrid composition is harder to explain than a single pattern; community adoption may be slower.
- **Concept drift in evaluation suites** — benchmarks go stale; the engine's accuracy on stale benchmarks may not reflect current accuracy.

### 19.2 Assumptions

- References are provided by the application; the engine does not fetch or search.
- A judge model (LLM or NLI) is available; self-hosted open-weight judges are supported.
- Factual claims are extractable from the answer; subjective language is reported as `unverifiable`.
- The application's response-time budget includes verification (~30s for typical workloads).
- The engine's accuracy target (70-85%) is sufficient for the application domain; higher accuracy requires domain-specific customization.

---

## 20. Roadmap and Milestones

| Milestone | Duration | Deliverable |
|---|---|---|
| **M1: Theoretical Framework** | Months 0–2 | Formal definitions of Claim, Evidence, Verdict, V, C, A; complexity bounds; ≥1 falsifiable prediction |
| **M2: Evaluation Suite** | Months 2–4 | Organic benchmarks across 5 domains (Health, Legal, Research, News, Financial) with ≥1,000 annotated examples per domain; κ ≥ 0.7 |
| **M3: Reference Implementation** | Months 4–8 | Engine with 4 critical plug-in types (ReferenceParser, ClaimExtractor, Verifier, Aggregator); baseline accuracy on the evaluation suite |
| **M4: Open-Source Release** | Months 8–12 | Public release; community plug-in contributions; production case studies in ≥3 domains |
| **M5: Trainable Components** | Months 12–18 | Trainable Aggregator; learned edge scorer; calibration benchmarks |
| **M6: Production Hardening** | Months 18–24 | Distributed execution; caching; rate limiting; SLA monitoring; production deployments |

---

## 21. Feedback Requested

Before implementation, the team would value feedback on:

1. **Scope of the theoretical framework.** Is the formal model the right level of formalism for the research program?
2. **Choice of evaluation domains.** Are Health, Legal, Research, News, Financial the right starting set? Should any be replaced or added?
3. **Composition vs decomposition.** Is the Hybrid (Layered + Graph + Plugin) composition the right architectural choice, or should we commit to a single pattern (e.g., Graph-only)?
4. **Trainable aggregator.** Is the trainable aggregator the right place to invest research effort, or should the trainable component be elsewhere (e.g., learned edge scorer)?
5. **Calibration theory.** Should calibration be a first-class research contribution, or is "post-hoc temperature scaling" sufficient?
6. **Ethical and societal risks.** What are the ethical considerations for a verification engine that may be used in high-stakes domains? How should we address bias in the engine's own outputs?
7. **Publication strategy.** Is a 24-month publication plan appropriate, or should we compress or extend? Which venues are the right targets (ACL, EMNLP, ICLR, NeurIPS, or others)?
8. **Resource allocation.** Is the proposed 6-subsystem, 24-module structure appropriately scoped for a research team of [N] people, or should some modules be deferred to v2?

---

## 22. References

The full 250+ source reference list is in the companion document `research-report.md`. The most-cited primary sources are:

### Surveys

- Es, S., James, J., Espinosa-Anke, L., Schockaert, S. "RAGAS: Automated Evaluation of Retrieval Augmented Generation." arXiv:2309.15217, 2023.
- Peng, H., et al. "LLM-based Agents Suffer from Hallucinations: A Survey of Taxonomy, Methods, and Directions." arXiv:2509.18970, 2025.
- "Large Language Models Hallucination: A Comprehensive Survey." arXiv:2510.06265, 2025.
- "A Survey of Automatic Hallucination Evaluation on Natural Language Generation." arXiv:2404.12041, 2024-2025.

### Detection Methods

- Manakul, P., Liusie, A., Gales, M.J.F. "SelfCheckGPT: Zero-Resource Black-Box Hallucination Detection." arXiv:2303.08896, 2023.
- Min, S., et al. "FActScore: Fine-grained Atomic Evaluation of Factual Precision." arXiv:2305.14251, 2023.
- Tang, L., Laban, P., Durrett, G. "MiniCheck: Efficient Fact-Checking of LLMs on Grounding Documents." arXiv:2404.10774, EMNLP 2024.
- Dhuliawala, S., et al. "Chain-of-Verification Reduces Hallucination in Large Language Models." ACL Findings 2024.

### Agent-Specific

- "Why Your Deep Research Agent Fails? On Hallucination in Deep Research Agents (PING Taxonomy)." arXiv:2601.22984, 2026.
- "Cascading Hallucination in Agentic RAG (CHARM)." arXiv:2606.04435, 2026.
- "How Enhancing LLM Reasoning Amplifies Tool Hallucination." arXiv:2510.22977, 2025.
- "AgentHallu: Benchmarking Automated Hallucination Attribution of LLM-based Agents." arXiv:2601.06818, 2026.

### Benchmarks

- Li, J., et al. "HaluEval." arXiv:2305.11747, EMNLP 2023.
- Hong, F., et al. "FaithBench." arXiv:2410.13210, 2024.
- "HalluLens: LLM Hallucination Benchmark." arXiv:2504.17550, ACL 2025.
- "A Benchmark for Predicting Language Model Hallucinations in Code (Collu-Bench)." arXiv:2410.09997, 2024.
- "Hallucination Detection for LLM-based Text-to-SQL Generation via Meta-Review (SQLHD)." arXiv:2512.22250, 2025.

### Judge Models

- Kim, S., et al. "Prometheus: Inducing Fine-grained Evaluation Capability in Language Models." arXiv:2310.08491, 2023.
- Kim, S., et al. "Prometheus 2." arXiv:2405.01535, 2024.
- M-Prometheus. arXiv:2504.04953, 2025.

### Industry Frameworks (Selected)

- DeepEval: github.com/confident-ai/deepeval (Apache 2.0)
- TruLens: github.com/truera/trulens (MIT, Snowflake)
- Phoenix: github.com/Arize-ai/phoenix (Elastic 2.0)
- LangSmith: smith.langchain.com (Closed source)
- Patronus AI: github.com/patronus-ai (Partial OSS)
- Vectara HHEM: huggingface.co/vectara/hallucination_evaluation_model
- RAGAS: docs.ragas.io (Apache 2.0)

### Methodological

- "Re-evaluating Hallucination Detection in LLMs." arXiv:2508.08285, 2025.
- "The Semantic Illusion: Certified Limits of Embedding-Based Hallucination Detection." arXiv:2512.15068, 2025.
- "The Mirage of Hallucination Detection." ACL Findings 2025.

### Open-Source Implementations

- guardrails-ai/guardrails
- LettuceDetect: github.com/KRLabsOrg/LettuceDetect
- Prometheus-Eval: github.com/prometheus-eval/prometheus-eval
- Cleanlab: github.com/cleanlab/cleanlab
- SelfCheckGPT: github.com/potsawee/selfcheckgpt
- MiniCheck: github.com/Liyan06/MiniCheck
- Exa hallucination detector: github.com/exa-labs/exa-hallucination-detector

### Companion Documents in This Project

- `research-report.md` — Full 10x research synthesis (3,063 lines, 250+ sources)
- `reference-architecture.md` — Full v1 reference architecture (1,346 lines, 17 modules)
- `architecture-review-v2.md` — Committee review and v2 changes (895 lines)
- `architecture-brief.md` — Concise 10-minute brief (359 lines)
- `FINAL-REPORT.md` — This document (consolidated, ~22 pages)

---

## Closing Note

This document is a **concept proposal for discussion, not approval**. The implementation will follow after the feedback is incorporated.

The research program enabled by HVE is the research program the field needs: a domain-independent, evidence-grounded, multi-evidence-aware, bias-corrected, calibrated, conflict-detecting, plug-in-extensible, formally specified verification engine. No existing system provides this. The composition is the contribution.

**The ask is feedback. The next deliverable is a research-grade theoretical framework, an evaluation suite, a reference implementation, and a publication plan. The work begins after the feedback.**

*— End of FINAL REPORT —*
