# Architecture Brief

## Domain-Agnostic Hallucination Verification Framework

*A design discussion for the research professor and team. Read in ≤10 minutes.*

**Status:** Concept proposal — feedback requested before implementation.
**Author:** Harsh Kashyap · NYU research project
**Date:** 2026-07-18

---

## 1. Executive Summary

### Problem
AI systems hallucinate. A factual error in a chatbot is recoverable; a factual error in a medical search engine, a legal research tool, or a financial assistant is not. As generative AI moves into high-stakes domains, the question is no longer *"can the model produce a fluent response?"* but *"can we trust every factual statement in the response?"*

### Motivation
A domain-independent verification engine is missing. Existing systems are RAG-shaped (RAGAS, DeepEval, TruLens, Patronus), observability-shaped (Phoenix, LangSmith), or model-eval-shaped (OpenAI Evals). None accepts a heterogeneous bundle of evidence (PDFs, web pages, API responses, structured databases) and returns a per-claim verdict with confidence and source attribution, all without domain-specific configuration.

### Proposed Solution
The **Hallucination Verification Engine (HVE)** — a pluggable verification engine that takes three inputs (question, generated answer, supporting references) and returns a structured per-claim verification report. The engine composes three architectural patterns: a **layered judge stack** for cost-effective verification, a **bipartite claim-evidence graph** for multi-evidence reasoning, and a **plugin-based microkernel** for extensibility.

### Expected Impact
- **70–85% factual reliability** on organic (real-world) benchmarks across at least five domains
- **Auditability:** every verdict is traceable to the evidence that produced it
- **Domain independence:** zero per-domain configuration
- **Cost viability:** sub-$0.05 per typical verification at production scale
- **Research contribution:** first formal framework that addresses the composition of multi-evidence reasoning, layered judging, and plug-in extensibility for verification

### What This Document Is Not
Not a research paper. Not an implementation plan. Not a literature review. It is a design proposal for discussion.

---

## 2. Current Landscape

We studied 9 frameworks. All solve a piece of the problem. None solves the whole problem.

### Comparison Table

| Framework | Primary Purpose | Strengths | Limitations | Key Insight We Learned |
|---|---|---|---|---|
| **RAGAS** | RAG evaluation | Industry standard; reference-free; LLM-judge based | RAG-shaped (question, context, answer); single-turn; no agent or multi-evidence | The claim-extract → NLI primitive is the right base; the locus of measurement is wrong (final answer, not every step) |
| **DeepEval** | Pytest-style LLM evaluation | 50+ metrics; multimodal; conversational; agent metrics | Per-turn; no trajectory-level; visualization gated | Metric-by-metric coverage is high but composition is manual |
| **TruLens** | RAG feedback functions | "RAG Triad"; plug-in feedback functions | RAG-shaped; primary feedback is groundedness, not multi-evidence conflict | Feedback-function pattern is right; lacks formal contract |
| **Phoenix (Arize)** | OTEL-native observability | OpenTelemetry-native; trace UI; eval templates | No verification logic; no claim extraction; no conflict detection | The OTEL GenAI spec is converging; verification engines should consume it |
| **LangSmith** | LangChain trace + eval | Deepest LangGraph integration; rich trace UI | LangChain-coupled; closed source; no verification per se | Run/Trace/Thread primitives are the right granularity for verification |
| **Patronus AI** | Eval-first with Lynx | Purpose-built hallucination detector (Lynx); open weights | RAG-only; no multi-evidence; no conflict detection | A fine-tuned open-weights detector (Lynx) matches GPT-4 at 100× lower cost |
| **OpenAI Evals** | Registry-based model eval | Standardized; reproducible; widely cited | No observability; no online eval; hosted platform sunsetting Oct 2026 | The Completion Function Protocol is an early agent-eval primitive |
| **Promptfoo** | CLI prompt regression | CLI-first; red-team focus; CI/CD native | No observability; no agent eval | CLI-first DX is winning the developer mindshare |
| **Galileo / Cleanlab** | Sub-200ms runtime guardrails | Purpose-built small evaluator models; runtime protection | Closed weights; expensive at scale; no agent eval | Sub-200ms latency is the bar for inline production guardrails |

### What Every Existing System Misses

1. **Heterogeneous evidence.** Every framework assumes one reference type (RAG chunk or single context). Production applications have *many* types.
2. **Multi-evidence reasoning.** Every framework treats each (claim, evidence) pair independently. None represents a claim supported by three pieces of evidence, two of which contradict.
3. **Layered judging with short-circuit.** Every framework uses a single judge (typically GPT-4o). None exploits the cost-quality curve of cheap → expensive layers.
4. **Formal plug-in contracts.** Frameworks allow customization but commit to no contract. Customization is by convention, not by specification.
5. **First-class bias correction, calibration, and conflict detection.** No framework addresses these. Existing systems rely on prompt engineering to mitigate bias; they do not address calibration; they do not detect conflicts.

**HVE addresses all five.**

---

## 3. Proposed Architecture

### 3.1 The Composition

HVE is a **Hybrid (Layered + Graph + Plugin)** architecture. Three sub-architectures are composed orthogonally, not stacked:

- **Layered (Defense-in-Depth)** — cost-effective judging: lexical → embedding → NLI → LLM → jury, with short-circuit on confident verdicts
- **Graph (Bipartite Claim-Evidence)** — multi-evidence reasoning: claims and evidence are nodes; verdicts are edges; conflicts are explicit
- **Plugin (Microkernel)** — extensibility: six plug-in types (ReferenceParser, ClaimExtractor, EvidenceRetriever, Verifier, Aggregator, OutputFormatter) with formal contracts

Each sub-architecture addresses a concern the others cannot. Layered without Graph cannot represent multi-evidence. Graph without Layered pays full judge cost on every edge. Plugin without the others has no structure to plug into.

### 3.2 High-Level Architecture Diagram

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

### 3.3 The 6 Subsystems (24 modules)

The architecture is organized into **6 subsystems**. Each is a coherent group of modules that can be reviewed independently.

| Subsystem | Purpose | Key Modules |
|---|---|---|
| **Input** (L4) | Parse input, format output, hold configuration | Input Gate, Output Formatter, Configuration Manager |
| **Orchestration** (L3) | Drive the workflow, manage state | Core Orchestrator, Workflow Engine, State Manager |
| **Domain** (L2) | Define the engine's data types and invariants | Claim, Evidence, Verdict, Graph, Trace, Conflict Resolver, Confidence Calibrator, **Trainable Aggregator** |
| **Plugin** (L1) | Define extensibility surface | Plugin Registry, Plugin Loader, Interface Specification |
| **Infrastructure** (L0) | Hold runtime resources | Judge Pool, Embedding Index |
| **Cross-cutting** (X) | Filter observations and post-process verdicts | Observability Sink, Error Handler, Bias Corrector |

### 3.4 Information Flow

```
  Question ─┐
           │
  Answer ───┼──►  [HVE Verification Engine]  ──►  [Verification Report]
           │                                     • per-claim verdicts
  References ┘                                   • confidence scores
           │                                     • evidence pointers
           │                                     • conflict flags
           │                                     • cost report
           │
      ┌────┴────┐
      │         │
   heterogeneous: PDFs, web pages, API responses,
   structured DB rows, search results, research papers
```

The engine treats all reference types uniformly. The output is a single structured report, regardless of input heterogeneity.

### 3.5 End-to-End Verification Pipeline (11 Stages)

```
   INPUT
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

### 3.6 Claim Lifecycle (9 States)

```
  RAW → EXTRACTED → NORMALIZED → TYPED → ELIGIBLE →
  MATCHED → VERIFIED → AGGREGATED → REPORTED

  (side states: UNVERIFIABLE, UNMATCHED, CONFLICT)
```

- **RAW:** text from the answer, not yet extracted
- **EXTRACTED:** decomposed; atomic; spans preserved
- **NORMALIZED:** coreferences resolved; decontextualized
- **TYPED:** entity type, fact type, certainty assigned
- **ELIGIBLE:** verifiable, not subjective
- **MATCHED:** ≥1 evidence candidate found
- **VERIFIED:** verdict + confidence assigned
- **AGGREGATED:** combined with other claims for overall score
- **REPORTED:** present in the output report

### 3.7 Evidence Lifecycle (8 States)

```
  RAW → PARSED → CHUNKED → INDEXED → MATCHED →
  RELEVANT → JUDGED

  (terminal: REJECTED, CONTRADICT)
```

- **RAW:** bytes or text from a reference
- **PARSED:** text extracted, type identified, metadata captured
- **CHUNKED:** split into spans (sentence / window / paragraph)
- **INDEXED:** embedded, lexical index, cached
- **MATCHED:** retrieved as candidate for a claim
- **RELEVANT:** above relevance threshold; edge created
- **JUDGED:** verdict assigned; stored on edge

### 3.8 Verification Lifecycle (5-State Edge Machine)

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

**Key invariant:** a piece of evidence is in `JUDGED` state if and only if it is on a `RECORDED` edge. The graph is the source of truth.

### 3.9 The 5-Layer Judge Stack

For each (claim, evidence) edge, the engine runs through:

1. **Lexical** (cheap, deterministic) — token overlap, substring match, citation URL ping
2. **Embedding** (cheap) — cosine similarity over a small embedding model
3. **NLI** (mid-cost) — premise-hypothesis entailment using a fine-tuned NLI model (DeBERTa / MiniCheck / Lynx)
4. **LLM Judge** (expensive) — frontier LLM with a rubric-based prompt
5. **Multi-Judge Consensus** (jury) — multiple LLM judges with bias correction

Each layer can short-circuit on a confident verdict. The Bias Corrector (X3) post-processes Layer 4 and Layer 5 verdicts to mitigate known biases (position, length, self-preference).

### 3.10 Confidence Estimation

Raw confidence is poorly calibrated. The engine applies:
- **Per-layer confidence:** the confidence reported by the judging layer
- **Cross-edge aggregation:** the per-claim confidence is aggregated from all edges
- **Calibration:** temperature, Platt, or isotonic scaling (plug-in choice)
- **Bias correction:** swap-averaging, length regression, cross-family voting (cross-cutting filter)

The output report exposes both raw and calibrated confidence, so operators can compare.

### 3.11 Final Reporting

The Verification Report contains:
- **Per-claim verdicts:** label, confidence, evidence pointers, judge trace
- **Conflict flags:** which claims have conflicting evidence
- **Coverage ratio:** fraction of claims that were verifiable
- **Reliability score:** overall factual reliability
- **Cost report:** per-stage and per-judge counts
- **Audit trail:** the complete trace for reproducibility

Output formats: JSON, JSONL, CSV, Markdown. Streaming output supported for long answers.

---

## 4. Alternative Architectures Considered

We considered 4 alternatives before selecting Hybrid (Layered + Graph + Plugin).

| Alternative | Core Idea | Advantages | Drawbacks | Why Not Selected |
|---|---|---|---|---|
| **Pure LLM-as-a-Judge** | One LLM call per claim; no layered judging | Simple; flexible; one prompt to write | Too expensive at scale; bias-prone; no short-circuit; no multi-evidence | Single judge is uneconomic and inadequate for high-stakes domains |
| **Retrieval-first verification** | Strong retriever first; only verify what retrieval returns | Strong when retrieval is good; rejects bad inputs early | Retrieval failure cascades silently; treats evidence-matching as a black box | Hides the IR problem; cannot detect conflicts; brittle when retrieval fails |
| **Multi-agent verifier** | Multiple LLM agents debate each claim (Council Mode, CSMAD) | Demonstrated 39% reduction on TruthfulQA; multiple paradigms | 3–5× cost; hard to reproduce; not plug-in-extensible | Cost is prohibitive at scale; bias not formally addressed |
| **Plugin-first architecture** | Microkernel + plug-ins, no graph, no layered judging | Maximum extensibility; small core | No multi-evidence reasoning; no cost control via short-circuit; plug-in quality variance | Inadequate for the multi-evidence cases that production requires |
| **Hybrid (Layered + Graph + Plugin)** ← *selected* | All three composed: graph is the data structure, layered judges operate on edges, plug-ins extend every boundary | Multi-evidence native; cost-effective; extensible; vendor-neutral | Higher conceptual complexity; longer ramp-up | The composition is the only one that addresses all five gaps from Section 2 |

**When alternatives may be preferable:**
- *Pure LLM-as-a-Judge* is preferable for low-volume, high-stakes single-claim evaluation (e.g., a one-off legal brief review).
- *Retrieval-first* is preferable when the retrieval system is already very strong and the cost of verification is dominated by retrieval.
- *Multi-agent verifier* is preferable for novel claim types where no single judge is reliable.
- *Plugin-first* is preferable for a small team that wants to ship quickly without a graph layer.

---

## 5. Open Questions, Risks, and Next Steps

### Open Research Questions

1. **What is the upper bound of verification accuracy?** Given perfect retrieval, perfect NLI, and a perfect LLM, what is the achievable accuracy? The architecture does not bound this.
2. **What is the optimal layer ordering?** The architecture assumes lexical → embedding → NLI → LLM → jury. Is this optimal? Under what conditions?
3. **Can the calibration function be learned end-to-end?** A jointly-trained calibrator may outperform a post-hoc transform.
4. **How does the engine behave under LLM version drift?** When a pinned judge model is upgraded, how is accuracy affected?
5. **What is the minimum number of judges in a jury for stable verdicts?** A theoretical question with practical implications.
6. **Can the engine be improved by active learning?** Selectively annotating the most uncertain verdicts.

### Potential Risks

- **Calibration drift over time** — the calibrator is trained on a held-out dataset; over time, the distribution shifts.
- **LLM version drift** — pinned judge versions become deprecated; reproducibility breaks.
- **Retrieval failure cascade** — if the retriever misses the relevant evidence, the verifier has nothing to verify against.
- **Adversarial references** — poisoned evidence (a webpage designed to look like a citation) could mislead the engine.
- **Conceptual complexity** — the Hybrid composition is harder to explain than a single pattern; community adoption may be slower.

### Assumptions

- References are provided by the application; the engine does not fetch or search.
- A judge model (LLM or NLI) is available; self-hosted open-weight judges are supported.
- Factual claims are extractable from the answer; subjective language is reported as `unverifiable`.
- The application's response-time budget includes verification (~30s for typical workloads).

### Immediate Next Milestones

1. **Month 0–2:** Theoretical framework document — formal definitions of Claim, Evidence, Verdict, the verification function V, complexity bounds, at least one falsifiable prediction.
2. **Month 2–4:** Evaluation suite — organic benchmarks across 5 domains (Health, Legal, Research, News, Financial) with ≥1,000 annotated examples per domain; inter-rater reliability κ ≥ 0.7.
3. **Month 4–8:** Reference implementation of the engine with the 4 most critical plug-in types (ReferenceParser, ClaimExtractor, Verifier, Aggregator); baseline accuracy on the evaluation suite.
4. **Month 8–12:** Open-source release; community plug-in contributions; production case studies in at least 3 domains.

### Feedback Requested From the Professor

Before implementation, we would value the professor's input on:

1. **Scope of the theoretical framework.** Is the formal model the right level of formalism for the research program?
2. **Choice of evaluation domains.** Are Health, Legal, Research, News, Financial the right starting set? Should any be replaced or added?
3. **Composition vs decomposition.** Is the Hybrid (Layered + Graph + Plugin) composition the right architectural choice, or should we commit to a single pattern (e.g., Graph-only)?
4. **Trainable aggregator.** Is the trainable aggregator the right place to invest research effort, or should the trainable component be elsewhere (e.g., learned edge scorer)?
5. **Calibration theory.** Should calibration be a first-class research contribution, or is "post-hoc temperature scaling" sufficient?
6. **Ethical and societal risks.** What are the ethical considerations for a verification engine that may be used in high-stakes domains (medical, legal)? How should we address bias in the engine's own outputs?
7. **Publication strategy.** Is a 24-month publication plan appropriate, or should we compress or extend? Which venues are the right targets (ACL, EMNLP, ICLR, NeurIPS, or others)?

---

## Closing Note

This document is a concept proposal for discussion. The full reference architecture is in `reference-architecture.md`; the committee review and Version 2 changes are in `architecture-review-v2.md`; the underlying research synthesis is in `research-report.md`. All documents are in the `hallucinationNerd/` directory.

**The ask is feedback, not approval. The implementation will follow after the feedback is incorporated.**

*— End of Architecture Brief —*
