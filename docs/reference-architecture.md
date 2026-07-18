# Hallucination Verification Engine (HVE) — Reference Architecture

*A research-grade high-level architecture for domain-independent, evidence-grounded verification of generated answers.*

**Style note for the reader.** This document is intentionally implementation-free. There is no code, no language choice, no API signature, no library reference. Every construct is described as a *role* with a *contract* and an *interaction pattern*. The document is suitable for a research-committee review, a PhD thesis chapter, or a funding proposal.

---

## Table of Contents

1. [Overall Architecture](#1-overall-architecture)
2. [Layered Architecture](#2-layered-architecture)
3. [Component Diagram](#3-component-diagram)
4. [Data Flow](#4-data-flow)
5. [Execution Pipeline](#5-execution-pipeline)
6. [Verification Workflow](#6-verification-workflow)
7. [Claim Lifecycle](#7-claim-lifecycle)
8. [Evidence Lifecycle](#8-evidence-lifecycle)
9. [Verification Lifecycle](#9-verification-lifecycle)
10. [Final Scoring Workflow](#10-final-scoring-workflow)
11. [Module Reference](#11-module-reference)

---

## 1. Overall Architecture

### 1.1 The Composition Principle

The HVE architecture is the **Hybrid (Layered + Graph + Plugin)** architecture. It is composed of three sub-architectures, each of which addresses a concern the others cannot:

| Sub-architecture | Concern it addresses | Why the others cannot |
|---|---|---|
| **Layered (Defense-in-Depth)** | Cost-effective judging | Graph and Plugin do not by themselves provide escalation; without it, every claim pays the full cost of the most expensive judge |
| **Graph (Bipartite Claim-Evidence)** | Multi-evidence reasoning and conflict detection | Layered and Plugin treat each (claim, evidence) pair independently; they cannot represent a claim supported by three pieces of evidence, two of which contradict |
| **Plugin (Microkernel)** | Extensibility across reference types, judges, aggregators | Layered and Graph bake in their components; they cannot accommodate new reference types or new judges without re-engineering |

These three sub-architectures are not stacked or pipelined. They are **composed orthogonally**: the **graph** is the data structure, the **layered judge stack** is the per-edge operation, and the **plugin model** is the extensibility surface.

### 1.2 The Composition Diagram

```
                    ┌────────────────────────────────────┐
                    │  Application Boundary              │
                    │  (the engine, as the user sees it) │
                    └─────────────────┬──────────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            │                         │                         │
   ┌────────▼─────────┐    ┌──────────▼───────────┐   ┌─────────▼──────────┐
   │ Plugin Shell     │    │ Application Core     │   │ Domain Core        │
   │ (extensibility)  │    │ (orchestration)      │   │ (data structures)  │
   │                  │    │                      │   │                    │
   │ • Plugin Reg.    │    │ • Orchestrator       │   │ • Claim            │
   │ • Plugin Loader  │    │ • Workflow Engine    │   │ • Evidence         │
   │ • Plugin Index   │    │ • Config Mgr        │   │ • Verdict          │
   │                  │    │ • State Manager     │   │ • Graph            │
   │  ┌────────────┐  │    │ • Pipeline Driver   │   │ • Trace            │
   │  │ Plug-ins   │  │    │ • Lifecycle Hooks   │   │                    │
   │  │            │  │    │                      │   │                    │
   │  │  Parsers   │  │    └──────────┬───────────┘   └─────────┬──────────┘
   │  │  Judges    │  │               │                         │
   │  │  Aggreg.   │  │               │                         │
   │  │  Format.   │  │               │                         │
   │  └────────────┘  │               │                         │
   └──────────────────┘               │                         │
                                      │                         │
   ┌──────────────────────────────────▼─────────────────────────▼──────┐
   │ Layered Judge Stack (on each graph edge)                          │
   │                                                                   │
   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────┐ │
   │  │ Lexical  │─►│Embedding │─►│   NLI    │─►│  LLM     │─►│ Jury │ │
   │  │ (cheap)  │  │ (cheap)  │  │ (mid)    │  │ (expensive) │      │ │
   │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────┘ │
   │       │             │             │             │            │    │
   │       └─short-circuit on confident verdict─┘             │    │
   │                                                          │    │
   └──────────────────────────────────────────────────────────┘    │
                                                                   │
   ┌──────────────────────────────────────────────────────────────┘
   │
   ▼
   Graph Layer (data structure)
   • Claim nodes     • Evidence nodes     • Edges with verdicts
   • Graph propagation    • Conflict detection
```

The visual emphasizes three things:
1. The **graph** is the substrate; everything else operates on it.
2. The **plugin shell** surrounds the application core, providing extensibility at every boundary.
3. The **layered judge stack** operates on each graph edge, with short-circuiting.

### 1.3 Why This Composition

Each sub-architecture alone fails on a critical property the problem demands:

- **Layered alone** is fast and cost-effective but cannot represent multi-evidence cases or detect conflicts.
- **Graph alone** captures multi-evidence reasoning but is expensive without escalation (every edge evaluated by the most expensive judge).
- **Plugin alone** is extensible but does not address the structural needs of the problem.

The composition makes each sub-architecture address a concern the others cannot, and lets them share the data structure (the graph) and the extensibility surface (the plugin registry).

---

## 2. Layered Architecture

### 2.1 The Five Logical Layers

The HVE is organized into five logical layers of abstraction, orthogonal to the three sub-architectures. Each layer has a strict responsibility and a strict dependency rule: a layer may depend only on the layer directly below it.

| Layer | Name | Responsibility | May depend on |
|---|---|---|---|
| L4 | Presentation | Input parsing, output formatting, configuration interface | L3 |
| L3 | Application | Orchestration, workflow, lifecycle management | L2, L4 |
| L2 | Domain | Data structures (Claim, Evidence, Verdict, Graph, Trace) and their invariants | L1 |
| L1 | Plugin | Plugin contracts, plugin registry, plugin loader | L0 |
| L0 | Infrastructure | Judge pool, embedding index, knowledge store, observability sinks | — |

### 2.2 The Layer Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│ L4 — PRESENTATION                                                  │
│                                                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│   │ Input Gate   │  │ Output Gate  │  │ Config Portal│              │
│   └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                     │
│   Purpose: parse the user's input; format the engine's output;     │
│            expose the configuration surface.                        │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ depends on L3
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ L3 — APPLICATION                                                   │
│                                                                     │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│   │ Core Orchestrator│  │ Workflow Engine  │  │ State Manager    │  │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                     │
│   Purpose: drive the verification workflow; manage the state of    │
│            in-flight verifications; enforce lifecycle invariants.   │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ depends on L2
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ L2 — DOMAIN                                                        │
│                                                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────┐  │
│   │ Claim    │  │ Evidence │  │ Verdict  │  │ Graph    │  │Trace │  │
│   │ Model    │  │ Model    │  │ Model    │  │ Model    │  │Model │  │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────┘  │
│                                                                     │
│   Purpose: define the engine's data types and their invariants.     │
│            No I/O. No orchestration. Pure data and rules.           │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ depends on L1
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ L1 — PLUGIN                                                        │
│                                                                     │
│   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │
│   │ Plugin Contract│  │ Plugin Registry│  │ Plugin Loader  │        │
│   └────────────────┘  └────────────────┘  └────────────────┘        │
│                                                                     │
│   Purpose: define the boundaries at which plug-ins attach;          │
│            manage plug-in discovery, versioning, lifecycle.         │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ depends on L0
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ L0 — INFRASTRUCTURE                                                │
│                                                                     │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│   │ Judge Pool │  │ Embed Index│  │ Knowledge  │  │ Observab.  │    │
│   │            │  │            │  │ Store      │  │ Sinks      │    │
│   └────────────┘  └────────────┘  └────────────┘  └────────────┘    │
│                                                                     │
│   Purpose: provide the runtime resources the engine depends on;     │
│            no business logic.                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Why These Five Layers

The layer count is the minimum that separates concerns cleanly:

- **L4 (Presentation)** is where the engine meets the outside world. Without it, the engine has no I/O.
- **L3 (Application)** is where workflows are sequenced. Without it, no orchestration is possible.
- **L2 (Domain)** is where the engine's "language" lives (Claim, Evidence, Verdict, Graph). Without it, the engine has no semantics.
- **L1 (Plugin)** is where extensibility is governed. Without it, the engine is monolithic.
- **L0 (Infrastructure)** is where resources are managed. Without it, the engine has no runtime.

Each layer is independently testable. The **dependency rule** (each layer depends only on the one below) prevents the layering from collapsing into a tangle.

---

## 3. Component Diagram

### 3.1 The Full Component Map

The engine has **17 named components** organized into the 5 layers, plus **3 cross-cutting concerns**. Every component has a single responsibility, a defined contract, and a defined interaction surface.

```
                                  LAYER 4 — PRESENTATION
   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                 │
   │   ┌────────────────┐   ┌─────────────────┐   ┌──────────────┐   │
   │   │ Input Gate     │   │ Output Formatter│   │ Configuration│   │
   │   │ (M1)           │   │ (M2)            │   │ Manager (M3) │   │
   │   └───────┬────────┘   └────────▲────────┘   └──────┬───────┘   │
   │           │                     │                   │           │
   └───────────┼─────────────────────┼───────────────────┼───────────┘
               │                     │                   │
                                  LAYER 3 — APPLICATION
   ┌───────────▼─────────────────────▼───────────────────▼───────────┐
   │                                                                 │
   │   ┌────────────────┐   ┌─────────────────┐   ┌──────────────┐    │
   │   │   Core         │   │ Workflow        │   │   State      │    │
   │   │ Orchestrator   │◄──┤   Engine        ├──►│   Manager    │    │
   │   │ (M4)           │   │ (M5)            │   │   (M6)       │    │
   │   └───────┬────────┘   └────────┬────────┘   └──────┬───────┘    │
   │           │                     │                   │            │
   └───────────┼─────────────────────┼───────────────────┼────────────┘
               │                     │                   │
                                  LAYER 2 — DOMAIN
   ┌───────────▼─────────────────────▼───────────────────▼───────────┐
   │                                                                 │
   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
   │   │ Claim    │  │ Evidence │  │ Verdict  │  │ Graph    │       │
   │   │ Model    │  │ Model    │  │ Model    │  │ Model    │       │
   │   │ (M7)     │  │ (M8)     │  │ (M9)     │  │ (M10)    │       │
   │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
   │        │             │             │             │              │
   │        └─────────────┴─────────────┴─────────────┘              │
   │                              │                                  │
   │   ┌──────────────┐  ┌─────────▼────────┐  ┌──────────────┐      │
   │   │  Trace       │  │  Conflict        │  │  Confidence  │      │
   │   │  Model (M11) │  │  Resolver (M12)  │  │  Calibrator  │      │
   │   │              │  │                  │  │  (M13)       │      │
   │   └──────────────┘  └──────────────────┘  └──────────────┘      │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘
               │                     │                   │
                                  LAYER 1 — PLUGIN
   ┌───────────▼─────────────────────▼───────────────────▼───────────┐
   │                                                                 │
   │   ┌────────────────┐                                             │
   │   │ Plugin         │         ┌────────────────┐                  │
   │   │ Registry (M14) │◄───────►│ Plugin Loader  │                  │
   │   │                │         │ (M15)          │                  │
   │   └────────────────┘         └────────────────┘                  │
   │            │                                                    │
   └────────────┼────────────────────────────────────────────────────┘
                │
                                  LAYER 0 — INFRASTRUCTURE
   ┌───────────▼─────────────────────────────────────────────────────┐
   │                                                                 │
   │   ┌────────────┐  ┌─────────────┐  ┌────────────┐                │
   │   │ Judge Pool │  │ Embedding   │  │ Knowledge  │                │
   │   │ (M16)      │  │ Index (M17) │  │ Store (ext)│                │
   │   └────────────┘  └─────────────┘  └────────────┘                │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │  CROSS-CUTTING CONCERNS                                          │
   │                                                                 │
   │  • Observability: every component emits trace events            │
   │  • Error Handling: every component has defined failure modes    │
   │  • Bias Correction: a filter that post-processes LLM verdicts   │
   └─────────────────────────────────────────────────────────────────┘
```

### 3.2 Component Inventory (17 modules + 3 cross-cutting)

| ID | Module | Layer | Primary role |
|---|---|---|---|
| M1 | Input Gate | L4 | Parse and validate the (question, answer, references) input |
| M2 | Output Formatter | L4 | Render the structured result into the requested output format |
| M3 | Configuration Manager | L4 | Hold the engine's runtime configuration |
| M4 | Core Orchestrator | L3 | Drive the verification workflow |
| M5 | Workflow Engine | L3 | Sequence the application-level steps |
| M6 | State Manager | L3 | Hold the state of in-flight verifications |
| M7 | Claim Model | L2 | The data structure for an atomic claim |
| M8 | Evidence Model | L2 | The data structure for a piece of evidence |
| M9 | Verdict Model | L2 | The data structure for a verification verdict |
| M10 | Graph Model | L2 | The data structure for the claim-evidence bipartite graph |
| M11 | Trace Model | L2 | The data structure for a complete verification trace |
| M12 | Conflict Resolver | L2 | Detect and resolve conflicts in the verdict graph |
| M13 | Confidence Calibrator | L2 | Convert raw confidence into calibrated confidence |
| M14 | Plugin Registry | L1 | Maintain the catalog of available plug-ins |
| M15 | Plugin Loader | L1 | Instantiate and lifecycle-manage plug-ins |
| M16 | Judge Pool | L0 | Hold the available judges (NLI, LLM, etc.) |
| M17 | Embedding Index | L0 | Hold embeddings of evidence and claims for retrieval |
| X1 | Observability Sink (cross-cutting) | — | Receive trace events from all components |
| X2 | Error Handler (cross-cutting) | — | Capture and route component failures |
| X3 | Bias Corrector (cross-cutting) | — | Post-process judge verdicts to mitigate known biases |

### 3.3 Why This Component Decomposition

The 17 modules + 3 cross-cutting concerns is the minimum that satisfies the principle of **single responsibility per component** while keeping the dependency graph shallow.

- **Three input/output components (M1, M2, M3)** isolate the engine from the outside world.
- **Three orchestration components (M4, M5, M6)** separate "what to do" (orchestrator), "how to do it" (workflow), and "what's been done" (state).
- **Seven domain components (M7–M13)** capture the engine's vocabulary: claims, evidence, verdicts, graphs, traces, conflicts, calibration.
- **Two plug-in components (M14, M15)** isolate the engine's extensibility surface.
- **Two infrastructure components (M16, M17)** separate the engine from its runtime resources.
- **Three cross-cutting concerns** are explicitly not modules: they are filters that observe or post-process events from all components. Making them modules would create a tangled web of dependencies; making them cross-cutting concerns keeps the dependency graph acyclic.

---

## 4. Data Flow

### 4.1 The Three Data Shapes

The engine has three canonical data shapes:

- **Input shape** — what the user provides
- **Internal shape** — what the engine operates on
- **Output shape** — what the engine returns

```
   USER-PROVIDED                        INTERNAL                            ENGINE-RETURNED
   (per F1.1)                           (domain types)                      (per F3.x)

┌────────────────────┐         ┌────────────────────┐              ┌────────────────────┐
│ question: String    │         │ VerificationRun    │              │ VerificationReport │
│ answer:   String    │ ──────► │  ├─ claim_list     │ ──────────►  │  ├─ overall_score  │
│ references: [       │         │  ├─ evidence_pool  │              │  ├─ per_claim[]    │
│   {                 │         │  ├─ graph          │              │  ├─ conflicts[]    │
│     type: Enum      │         │  └─ verdict_set    │              │  ├─ coverage_ratio │
│     content: Bytes  │         │                    │              │  └─ cost_report    │
│     metadata: Map   │         └────────────────────┘              └────────────────────┘
│   }                 │                  │
│ ]                   │                  │
└────────────────────┘                  │
                                        ▼
                              ┌────────────────────┐
                              │ Claim (M7)         │
                              │  ├─ id             │
                              │  ├─ text           │
                              │  ├─ normalized     │
                              │  ├─ type           │
                              │  └─ source_span    │
                              └────────────────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │ Evidence (M8)      │
                              │  ├─ id             │
                              │  ├─ text           │
                              │  ├─ source_id      │
                              │  ├─ source_type    │
                              │  ├─ span_offset    │
                              │  └─ trust_signal   │
                              └────────────────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │ Edge               │
                              │  ├─ claim_id       │
                              │  ├─ evidence_id    │
                              │  ├─ relevance      │
                              │  └─ verdict_id     │
                              └────────────────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │ Verdict (M9)       │
                              │  ├─ label: Enum    │
                              │  ├─ confidence: F  │
                              │  └─ judge_trace    │
                              └────────────────────┘
```

### 4.2 The Sequence Diagram — End-to-End

This sequence diagram shows the interaction between modules for a single verification request. Each arrow is a method-level interaction (the implementation is irrelevant; the role of the interaction is what matters).

```
  User       M1         M3          M4           M5           M6
   │          │          │           │            │            │
   │ ──────►  │ verify() │           │            │            │
   │          │ ──────►  │ load_cfg() │           │            │
   │          │          │ ──────►   │            │            │
   │          │          │ cfg       │            │            │
   │          │ ◄──────  │           │            │            │
   │          │ ────────►│           │            │            │
   │          │          │ start_run()           │            │
   │          │          │ ──────►  │            │            │
   │          │          │           │ plan()     │            │
   │          │          │           │ ──────►   │            │
   │          │          │           │ workflow  │            │
   │          │          │           │ ◄──────   │            │
   │          │          │           │ execute()  │            │
   │          │          │           │ ──────►   │            │
   │          │          │           │            │ save_state()│
   │          │          │           │            │ ──────►   │
   │          │          │           │ ◄──────   │ state      │
   │          │          │           │            │            │
   │          │          │           │  (workflow loop:        │
   │          │          │           │   parse → extract →     │
   │          │          │           │   match → judge → agg)  │
   │          │          │           │            │            │
   │          │          │           │ ──────►   │            │
   │          │          │           │ finish()  │            │
   │          │          │           │ ──────►   │            │
   │          │          │           │            │            │
   │ ◄────── │ ────────►│           │ ──────►  │            │
   │ report  │          │           │           │            │
   │          │          │           │            │            │
```

The orchestrator (M4) is the **single owner of the workflow loop**. The state manager (M6) persists the run state at every checkpoint. The workflow engine (M5) sequences the stages. The orchestrator is the only module that holds the run's lifecycle state.

### 4.3 The Internal Data Flow Within a Verification

Within a single verification, the data flows through five transformations:

```
1. INPUT
   question, answer, references
            │
            ▼
2. NORMALIZATION
   references: List[Reference] (typed, parsed, metadata-extracted)
            │
            ▼
3. CLAIM EXTRACTION
   claims: List[Claim] (atomic, normalized, typed)
            │
            ▼
4. EVIDENCE MATCHING
   edges: List[(Claim, Evidence, relevance)] (relevance-scored)
            │
            ▼
5. VERIFICATION
   verdicts: List[Verdict] (per edge, with confidence)
            │
            ▼
6. AGGREGATION
   final: Verdict (per claim + overall)
            │
            ▼
7. OUTPUT
   report: VerificationReport (formatted)
```

Each transformation is a **pure function** over the previous shape. No transformation depends on the input shape directly; each depends only on the previous transformation's output. This is why the engine is **pipeline-correct**: it cannot enter an inconsistent state because each stage is a function of the previous stage's output.

### 4.4 Why This Data Flow

The five-transformation flow is the minimum that separates concerns:

- **Normalization** is needed because inputs are heterogeneous.
- **Claim extraction** is needed because the answer is not the unit of verification; the claim is.
- **Evidence matching** is needed because verification is a per-(claim, evidence) operation.
- **Verification** is the unit of judgment.
- **Aggregation** is needed because verdicts must be combined into an overall result.

Skipping any transformation makes the engine either incorrect (verifying the answer as a whole instead of claim-by-claim) or coupled (one stage depending on input details).

---

## 5. Execution Pipeline

### 5.1 The Pipeline at a Glance

The execution pipeline is the **ordered sequence of stages** that turns an input bundle into a verification report. The pipeline has 11 stages. Each stage is a component (or a composition of components) and has a defined entry and exit condition.

```
┌──────────────────────────────────────────────────────────────────────┐
│ EXECUTION PIPELINE                                                  │
│                                                                      │
│  Stage 1: Input Acceptance          [M1 → M3]                        │
│  Stage 2: Configuration Resolution  [M3 → M4]                        │
│  Stage 3: Reference Ingestion      [M4 → plug-in → M10]             │
│  Stage 4: Reference Indexing       [M10 → M17]                       │
│  Stage 5: Claim Extraction         [M4 → plug-in → M7]              │
│  Stage 6: Claim Normalization      [M7]                             │
│  Stage 7: Evidence Matching        [M4 → plug-in → M10]             │
│  Stage 8: Edge Construction        [M10]                            │
│  Stage 9: Layered Verification     [M10 → M16 → plug-ins → M9]      │
│  Stage 10: Conflict Resolution     [M10 → M12]                       │
│  Stage 11: Aggregation & Scoring   [M9, M10 → M13 → M2]             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 The Workflow Diagram

```
                          ┌──────────────┐
                          │   Input      │
                          │   Bundle     │
                          └──────┬───────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S1. Input Acceptance         │
                  │ • Validate schema            │
                  │ • Detect language            │
                  │ • Reject malformed inputs    │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S2. Configuration Resolution │
                  │ • Load runtime config        │
                  │ • Select plug-ins            │
                  │ • Select judges              │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S3. Reference Ingestion      │
                  │ • Per-type reference parser  │
                  │ • Extract text from PDF,     │
                  │   web, API response, etc.    │
                  │ • Preserve provenance        │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S4. Reference Indexing       │
                  │ • Chunk by sentence / window │
                  │ • Embed (M17)                 │
                  │ • Build lexical index         │
                  │ • Cache for retrieval        │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S5. Claim Extraction         │
                  │ • Decompose answer           │
                  │ • Atomicity / decontextualize│
                  │ • Type each claim            │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S6. Claim Normalization      │
                  │ • Resolve coreferences       │
                  │ • Filter unverifiable        │
                  │ • Tag subjective claims      │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S7. Evidence Matching        │
                  │ • For each claim, retrieve   │
                  │   top-K candidate evidence   │
                  │   spans via lexical, embed,  │
                  │   or hybrid                  │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S8. Edge Construction        │
                  │ • Build bipartite graph      │
                  │ • Score edge relevance       │
                  │ • Filter by relevance floor  │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S9. Layered Verification     │
                  │ • For each edge, run judge    │
                  │   stack: lexical → embed →   │
                  │   NLI → LLM → jury           │
                  │ • Short-circuit on confident │
                  │ • Emit per-edge verdict      │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S10. Conflict Resolution     │
                  │ • Detect claim-level         │
                  │   conflicts                  │
                  │ • Detect edge-level          │
                  │   contradictions            │
                  │ • Flag, do not adjudicate     │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ S11. Aggregation & Scoring   │
                  │ • Per-claim aggregation       │
                                  │ • Cross-claim aggregation    │
                  │ • Confidence calibration      │
                  │ • Format output              │
                                  ▼
                          ┌──────────────┐
                          │  Output      │
                          │  Report      │
                          └──────────────┘
```

### 5.3 Why This Pipeline

The 11 stages are the minimum that captures the work without conflating concerns:

- **S1–S2** are the input boundary: validate, configure.
- **S3–S4** are the reference side: ingest and index.
- **S5–S6** are the claim side: extract and normalize.
- **S7–S8** are the matching side: connect claims to evidence.
- **S9–S10** are the judgment side: verify and resolve conflicts.
- **S11** is the output side: aggregate and format.

The pipeline is **strictly linear at the stage level** (S_i feeds S_{i+1}) but **branching at the component level** (a stage may use multiple components). This linear-at-stage, branching-at-component structure is what makes the pipeline both simple to reason about and rich in internal behavior.

### 5.4 Pipeline Properties

- **Determinism:** Given the same input, the same configuration, and the same plug-in versions, the pipeline produces the same output. (LLM stochasticity is contained within Stage 9 and is mitigated by bias correction and jury voting.)
- **Resumability:** Each stage produces a checkpointed state. If the pipeline fails at Stage 9, it can resume from Stage 9.
- **Parallelism:** Stages 5–6 and 7–8 are embarrassingly parallel across claims and evidence. The orchestrator (M4) decides the parallelism policy.
- **Observability:** Each stage emits a trace event. The end-to-end trace is reconstructable from the events.

---

## 6. Verification Workflow

### 6.1 The Verification Workflow in Detail

The verification workflow is the per-(claim, evidence) sub-workflow. It is the heart of the engine. The pipeline (Section 5) is the macro-flow; the verification workflow is the micro-flow.

```
                    ┌─────────────────────┐
                    │   For each edge     │
                    │   (claim, evidence) │
                    └──────────┬──────────┘
                               │
                               ▼
                ┌──────────────────────────────┐
                │  L1. Lexical Judge           │
                │  • Token overlap             │
                │  • Substring match           │
                │  • Citation URL ping         │
                │  • Verdict: matched/unmatched│
                │  • Confidence: high if       │
                │    exact match; else low     │
                └──────────────┬───────────────┘
                               │
                       confident?
                       ┌───────┴───────┐
                       │               │
                      YES             NO
                       │               │
                       ▼               ▼
                ┌──────────┐  ┌──────────────────────┐
                │ short-   │  │  L2. Embedding Judge │
                │ circuit  │  │  • Cosine similarity │
                │          │  │  • Threshold check   │
                └──────────┘  └──────────┬───────────┘
                                          │
                                  confident?
                                  ┌───────┴───────┐
                                  │               │
                                 YES             NO
                                  │               │
                                  ▼               ▼
                           ┌──────────┐  ┌──────────────────┐
                           │ short-   │  │  L3. NLI Judge   │
                           │ circuit  │  │  • Premise-      │
                           └──────────┘  │    hypothesis    │
                                          │    entailment    │
                                          │  • DeBERTa /     │
                                          │    MiniCheck /   │
                                          │    Lynx          │
                                          └────────┬─────────┘
                                                   │
                                           confident?
                                           ┌────┴────┐
                                           │         │
                                          YES       NO
                                           │         │
                                           ▼         ▼
                                    ┌──────────┐ ┌──────────────────┐
                                    │ short-   │ │  L4. LLM Judge   │
                                    │ circuit  │ │  • Frontier LLM  │
                                    └──────────┘ │  • Rubric-based  │
                                                │  • Self-critique │
                                                └────────┬─────────┘
                                                         │
                                                  confident?
                                                  ┌────┴────┐
                                                  │         │
                                                 YES       NO
                                                  │         │
                                                  ▼         ▼
                                          ┌──────────┐ ┌──────────────────┐
                                          │  record  │ │ L5. Multi-Judge  │
                                          │  verdict │ │   Consensus     │
                                          └──────────┘ │  • Jury of LLMs  │
                                                      │  • Bias correct  │
                                                      │  • Confidence    │
                                                      │    aggregation   │
                                                      └────────┬─────────┘
                                                               │
                                                               ▼
                                                       ┌────────────┐
                                                       │  Verdict   │
                                                       │  (M9)      │
                                                       └────────────┘
```

### 6.2 The Workflow Properties

The verification workflow is **layered, short-circuiting, and progressive**. Three properties matter:

1. **Layered.** Each layer is more expensive and more accurate than the previous. The architecture exploits this by short-circuiting when a layer is confident.
2. **Short-circuiting.** If a layer returns a verdict with confidence above a threshold, the workflow stops. This bounds the cost.
3. **Progressive.** The layers build on each other. A lexical match may be a quick reject, but a "support" verdict from Layer 1 is provisional until confirmed by a higher layer.

### 6.3 The Bias-Correction Filter (Cross-Cutting)

Layer 4 (LLM Judge) and Layer 5 (Jury) are subject to known biases (position, length, self-preference). Before their verdicts are recorded, the **Bias Corrector (X3)** applies:

- **Swap-averaging:** for pairwise judgments, swap and re-judge, average.
- **Length regression:** subtract the estimated length contribution from the confidence.
- **Cross-family voting:** if multiple LLM judges are used, vote across model families.
- **Constitutional rubric:** apply an explicit rubric (e.g., "the verdict must cite the evidence span") and re-judge against it.

The bias corrector is a **filter** in the workflow, not a stage. It does not produce a verdict; it adjusts the verdict produced by Layer 4 or 5.

### 6.4 Why This Workflow

The layered workflow is the canonical **defense-in-depth** pattern adapted to verification. Each layer has a different cost and accuracy profile, and the short-circuit logic ensures that the engine pays the cost of the most expensive layer only when necessary.

This pattern is well-validated in the research (the synthesis identified "Layered detection pipeline" as a 5/5-confidence established pattern). What the engine adds is the **integration with the graph**: each layer operates on a specific edge, and the verdict is stored as an attribute of the edge.

---

## 7. Claim Lifecycle

### 7.1 The Claim States

A claim moves through **9 states** during its lifetime. Each state has a defined entry condition, a defined set of operations, and a defined exit condition.

```
                    ┌──────────┐
                    │  RAW     │  (text from answer, not yet extracted)
                    └─────┬────┘
                          │ claim extractor
                          ▼
                    ┌──────────┐
                    │ EXTRACTED│  (decomposed; atomic; spans preserved)
                    └─────┬────┘
                          │ normalizer
                          ▼
                    ┌──────────┐
                    │NORMALIZED│  (decontextualized; coreferences resolved)
                    └─────┬────┘
                          │ typer
                          ▼
                    ┌──────────┐
                    │  TYPED   │  (entity type, fact type, certainty)
                    └─────┬────┘
                          │ filter
                          ▼
                    ┌──────────┐
                    │ ELIGIBLE │  (verifiable; not subjective)
                    └─────┬────┘
                          │ matcher
                          ▼
                    ┌──────────┐
                    │ MATCHED  │  (≥1 evidence candidate found)
                    └─────┬────┘
                          │ judge stack
                          ▼
                    ┌──────────┐
                    │ VERIFIED │  (verdict assigned; confidence recorded)
                    └─────┬────┘
                          │ aggregator
                          ▼
                    ┌──────────┐
                    │AGGREGATED│  (combined with other claims for overall score)
                    └─────┬────┘
                          │ formatter
                          ▼
                    ┌──────────┐
                    │ REPORTED │  (in the output report)
                    └──────────┘

   (side states)

                    ┌──────────┐
                    │UNVERIFI- │
                    │  ABLE    │  (subjective or too vague; flagged, not verified)
                    └──────────┘

                    ┌──────────┐
                    │ UNMATCHED│  (no evidence found; verdict = not_enough_info)
                    └──────────┘

                    ┌──────────┐
                    │ CONFLICT │  (conflicting evidence found; flagged)
                    └──────────┘
```

### 7.2 The Lifecycle Properties

The claim lifecycle has three properties:

- **State-machine formal.** A claim is in exactly one state at a time. Transitions are guarded by conditions. The lifecycle is implementable as a finite state machine.
- **Side states are first-class.** `UNVERIFIABLE`, `UNMATCHED`, and `CONFLICT` are not failures; they are legitimate outcomes. The lifecycle treats them as terminal states that are reported to the user.
- **Aggregation is downstream.** A claim is `VERIFIED` before it is `AGGREGATED`. The aggregation step operates on already-verified claims and combines them into an overall score.

### 7.3 Why This Lifecycle

The 9 states are the minimum that captures the claim's journey without conflating concerns:

- **RAW → EXTRACTED:** the unit of verification is the claim, not the answer. The claim extractor makes this transition explicit.
- **EXTRACTED → NORMALIZED:** coreferences ("he," "it," "this") must be resolved for the claim to be independently verifiable.
- **NORMALIZED → TYPED:** different types (date, person, number, definition) may use different evidence matchers.
- **TYPED → ELIGIBLE:** subjective or vague claims are filtered before they consume evidence-matching resources.
- **ELIGIBLE → MATCHED:** the evidence-matcher produces candidates.
- **MATCHED → VERIFIED:** the layered judge stack produces a verdict.
- **VERIFIED → AGGREGATED:** the per-claim verdict is combined with other per-claim verdicts.
- **AGGREGATED → REPORTED:** the final report is formatted.

Skipping any state makes the engine either incorrect (verifying without normalizing) or wasteful (matching evidence for unverifiable claims).

---

## 8. Evidence Lifecycle

### 8.1 The Evidence States

Evidence follows **8 states** during its lifetime. The states are distinct from claim states because evidence enters the engine from a different side.

```
                    ┌──────────┐
                    │  RAW     │  (bytes or text from a reference; not yet ingested)
                    └─────┬────┘
                          │ reference parser
                          ▼
                    ┌──────────┐
                    │ PARSED  │  (text extracted; type identified; metadata)
                    └─────┬────┘
                          │ chunker
                          ▼
                    ┌──────────┐
                    │ CHUNKED │  (split into spans; sentence/window/paragraph)
                    └─────┬────┘
                          │ indexer
                          ▼
                    ┌──────────┐
                    │ INDEXED │  (embedded; lexical index; cached)
                    └─────┬────┘
                          │ matcher (via claim query)
                          ▼
                    ┌──────────┐
                    │ MATCHED │  (retrieved as candidate for a claim)
                    └─────┬────┘
                          │ relevance filter
                          ▼
                    ┌──────────┐
                    │ RELEVANT│  (above relevance threshold; edge created)
                    └─────┬────┘
                          │ judge stack
                          ▼
                    ┌──────────┐
                    │ JUDGED  │  (verdict assigned; stored on edge)
                    └──────────┘

   (terminal states)

                    ┌──────────┐
                    │ REJECTED │  (below relevance floor; not used)
                    └──────────┘

                    ┌──────────┐
                    │CONTRADICT│  (verdict = contradicted; reported)
                    └──────────┘
```

### 8.2 The Lifecycle Properties

The evidence lifecycle has three properties:

- **Independent from claim lifecycle.** Evidence is indexed before any claim is matched. The two lifecycles synchronize at the "edge" — the bipartite graph's edge is created when both sides are in their `MATCHED` state.
- **Relevance is a first-class filter.** An evidence span that is matched but not relevant is rejected. This is what keeps the graph sparse and the verification cost bounded.
- **The judge stack runs on the edge, not on the evidence.** The evidence's JUDGED state is the result of being on an edge that was judged. Evidence that is not on any edge is in `REJECTED` state.

### 8.3 Why This Lifecycle

The 8 states mirror the claim lifecycle but on the other side of the graph:

- **RAW → PARSED:** heterogeneous references must be normalized.
- **PARSED → CHUNKED:** verification is per-claim, so evidence must be at the claim's granularity.
- **CHUNKED → INDEXED:** the matcher needs an index to search.
- **INDEXED → MATCHED:** a claim's query retrieves the evidence.
- **MATCHED → RELEVANT:** not every match is relevant; filter.
- **RELEVANT → JUDGED:** the edge is judged.
- **JUDGED → CONTRADICT or RELEVANT (post-judgment):** the verdict determines the evidence's role.

The key invariant: **a piece of evidence is in JUDGED state if and only if it is on a judged edge in the graph.** This invariant is what makes the graph the source of truth.

---

## 9. Verification Lifecycle

### 9.1 The Per-Edge Verification State Machine

The verification lifecycle operates on **edges** of the claim-evidence graph. An edge has 5 states.

```
                    ┌──────────┐
                    │ CREATED  │  (edge exists; claim + evidence; relevance known)
                    └─────┬────┘
                          │ judge stack: lexical
                          ▼
                    ┌──────────┐
                    │ JUDGED-L1│  (lexical judge returned)
                    └─────┬────┘
                          │ short-circuit?
                    ┌─────┴─────┐
                    │           │
                   YES         NO
                    │           │
                    │           ▼
                    │    ┌──────────┐
                    │    │JUDGED-L2 │  (embedding judge returned)
                    │    └─────┬────┘
                    │          │ short-circuit?
                    │    ┌─────┴─────┐
                    │    │           │
                    │   YES         NO
                    │    │           │
                    │    │           ▼
                    │    │    ┌──────────┐
                    │    │    │JUDGED-L3 │  (NLI judge returned)
                    │    │    └─────┬────┘
                    │    │          │ short-circuit?
                    │    │    ┌─────┴─────┐
                    │    │    │           │
                    │    │   YES         NO
                    │    │    │           │
                    │    │    │           ▼
                    │    │    │    ┌──────────┐
                    │    │    │    │JUDGED-L4 │  (LLM judge returned)
                    │    │    │    └─────┬────┘
                    │    │    │          │ short-circuit?
                    │    │    │    ┌─────┴─────┐
                    │    │    │    │           │
                    │    │    │   YES         NO
                    │    │    │    │           │
                    │    │    │    │           ▼
                    │    │    │    │    ┌──────────┐
                    │    │    │    │    │JUDGED-L5 │  (jury returned)
                    │    │    │    │    └─────┬────┘
                    │    │    │    │          │
                    │    │    │    │          │
                    ▼    ▼    ▼    ▼          ▼
                    ┌──────────────────────┐
                    │      RECORDED        │
                    │ (verdict committed to │
                    │  the graph; bias      │
                    │  correction applied)  │
                    └──────────────────────┘

   (post-verdict side states)

                    ┌──────────┐
                    │CONTRADICT│  (verdict = contradicted; cross-edge check)
                    └──────────┘

                    ┌──────────┐
                    │ CONFLICT │  (other edges of same claim disagree)
                    └──────────┘

                    ┌──────────┐
                    │ UNKNOWN  │  (all layers below confidence threshold)
                    └──────────┘
```

### 9.2 The Bias-Correction Step

After the layer that returns the verdict commits it, the **Bias Corrector (X3)** is invoked. The corrector may:

- Adjust the confidence by an estimated bias factor (length, position, self-preference).
- Trigger a re-judgment at a higher layer if bias is detected as a confound.
- Apply a constitutional rubric and re-judge.

Only after bias correction does the verdict become `RECORDED`.

### 9.3 The Cross-Edge Consistency Check

After the edge is `RECORDED`, the engine checks other edges of the same claim. If a claim has multiple evidence spans, and they disagree, the edge enters the `CONFLICT` state. The conflict is reported but not adjudicated (per the non-goal of adjudicating expert disagreement).

### 9.4 Why This Lifecycle

The per-edge state machine captures the essential property of the layered judge stack: **each edge progresses through the layers, short-circuiting when a layer is confident, and ending in a recorded verdict**. The 5-layer state space is the minimum that expresses this progression without conflating layers.

The cross-edge consistency check is what makes the graph-based architecture more powerful than a simple pipeline: it detects conflicts that no single-edge verifier can see.

---

## 10. Final Scoring Workflow

### 10.1 The Scoring Pipeline

The final scoring workflow takes all `RECORDED` verdicts and produces the report.

```
┌──────────────────────────────────────────────────────────────────────┐
│ FINAL SCORING WORKFLOW                                              │
│                                                                      │
│   For each claim:                                                    │
│     1. Collect all edges                                             │
│     2. Aggregate verdicts across edges (M12)                         │
│     3. Apply confidence calibration (M13)                            │
│     4. Compute per-claim score                                       │
│                                                                      │
│   Across all claims:                                                 │
│     5. Compute coverage ratio                                        │
│     6. Compute conflict ratio                                        │
│     7. Compute overall reliability score                             │
│     8. Compute domain-level summary                                  │
│                                                                      │
│   Output:                                                            │
│     9. Per-claim verdicts with confidence                            │
│     10. Overall report with cost                                     │
│     11. Audit trail                                                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 10.2 The Scoring Workflow Diagram

```
   Recorded Verdicts (M9)
            │
            ▼
   ┌────────────────────────────────────────┐
   │ Per-claim aggregation (M12)            │
   │  • For each claim, collect all edges   │
   │  • Combine verdicts using the          │
   │    configured aggregation strategy     │
   │    (majority / weighted / source-      │
   │    priority / jury)                    │
   │  • Detect conflicts                    │
   └────────────────┬───────────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────────┐
   │ Confidence calibration (M13)           │
   │  • Apply temperature scaling           │
   │  • Apply isotonic / Platt scaling      │
   │  • Output calibrated confidence       │
   └────────────────┬───────────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────────┐
   │ Per-claim score                        │
   │  • Label (supported/contradicted/...)  │
   │  • Confidence                          │
   │  • Evidence pointers                   │
   │  • Conflict flag                       │
   └────────────────┬───────────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────────┐
   │ Cross-claim aggregation                │
   │  • Coverage ratio                      │
   │  • Reliability score                   │
   │  • Domain-level summary                │
   └────────────────┬───────────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────────┐
   │ Output rendering (M2)                  │
   │  • JSON / JSONL / CSV / Markdown       │
   │  • Cost report                         │
   │  • Audit trail                         │
   └────────────────┬───────────────────────┘
                    │
                    ▼
            ┌──────────────┐
            │   Report     │
            └──────────────┘
```

### 10.3 The Aggregation Strategies

The aggregator (M12) supports multiple strategies. The choice is a configuration option, not a hard-coded behavior.

| Strategy | Description | When to use |
|---|---|---|
| **Majority** | The most common verdict across edges wins | Multiple references of equal weight |
| **Weighted** | Each edge's verdict is weighted by its source trust score | References with heterogeneous credibility |
| **Source-priority** | Verdicts from higher-priority sources override lower-priority | Domain-specific credibility ordering |
| **Jury** | Verdicts are treated as votes; jury rules apply | When multiple LLM judges are used |
| **Max-contradiction** | A single contradicting edge flips the claim to contradicted | High-stakes domains (medical, legal) |
| **Min-confidence** | The minimum confidence across edges is the per-claim confidence | Conservative reporting |

### 10.4 The Confidence Calibration

The confidence calibration step (M13) addresses the **calibration gap** identified in the research: raw LLM confidence scores are poorly calibrated. The calibrator applies:

- **Temperature scaling:** a single temperature parameter learned on a held-out dataset.
- **Platt scaling:** a one-parameter logistic model learned on held-out data.
- **Isotonic regression:** a non-parametric mapping learned on held-out data.

The calibrator is a plug-in (per the Plugin model). The default is temperature scaling; alternatives are plug-in options.

### 10.5 The Cost Report

The output report includes a cost report:

- **Per-stage counts:** how many claims, evidence spans, edges, and verdicts.
- **Per-judge counts:** how many times each judge was invoked.
- **Per-layer counts:** how many edges were short-circuited at which layer.
- **Token counts:** total tokens consumed (if the judge exposes this).
- **Time breakdown:** time per stage.

The cost report is the operator's tool for tuning the engine.

### 10.6 Why This Scoring Workflow

The scoring workflow is the minimum that produces a calibrated, auditable, cost-transparent report:

- **Per-claim aggregation** is needed because the unit of verification is the claim, but the unit of judgment is the edge.
- **Confidence calibration** is needed because raw confidence is poorly calibrated.
- **Cross-claim aggregation** is needed because the user wants an overall reliability score, not just per-claim verdicts.
- **Output rendering** is needed because the user has chosen a format.
- **Cost report** is needed because the operator must tune the engine.

Skipping any step makes the engine either inaccurate (no calibration) or unobservable (no cost report).

---

## 11. Module Reference

For every module: purpose, inputs, outputs, responsibilities, interactions.

### M1. Input Gate

| | |
|---|---|
| **Purpose** | Validate and normalize the user-provided input bundle. |
| **Inputs** | Raw input: question (String), answer (String), references (List of typed reference objects). |
| **Outputs** | Validated input bundle with detected language, normalized reference types, and a unique run ID. |
| **Responsibilities** | Schema validation; language detection; reference type validation; run ID generation; rejection of malformed inputs with structured error. |
| **Interactions** | M3 (Configuration Manager) — to apply input-level configuration; M4 (Core Orchestrator) — to start the run. |

### M2. Output Formatter

| | |
|---|---|
| **Purpose** | Render the structured VerificationReport into the requested output format. |
| **Inputs** | VerificationReport (structured); output format selection (JSON / JSONL / CSV / Markdown). |
| **Outputs** | A rendered output document. |
| **Responsibilities** | Format selection; schema projection; serialization; human-readable rendering. |
| **Interactions** | M13 (Confidence Calibrator) — receives calibrated verdicts; M4 (Core Orchestrator) — returns the final report. |

### M3. Configuration Manager

| | |
|---|---|
| **Purpose** | Hold and resolve the engine's runtime configuration. |
| **Inputs** | Configuration sources: defaults, user-supplied overrides, environment, plug-in profiles. |
| **Outputs** | A resolved Configuration object passed to the orchestrator. |
| **Responsibilities** | Layered configuration resolution (defaults → overrides → env → profiles); plug-in selection per use case; confidence thresholds per layer; aggregation strategy; output format. |
| **Interactions** | M14 (Plugin Registry) — to enumerate available plug-ins; M16 (Judge Pool) — to enumerate available judges; M4 (Core Orchestrator) — to receive the resolved configuration. |

### M4. Core Orchestrator

| | |
|---|---|
| **Purpose** | Drive the verification workflow for a single run. |
| **Inputs** | Validated input bundle (from M1); resolved configuration (from M3). |
| **Outputs** | A VerificationReport (delegated to M2). |
| **Responsibilities** | Plan the workflow; dispatch stages to the Workflow Engine; track run state; handle errors; emit trace events. |
| **Interactions** | M1 (input); M3 (config); M5 (workflow); M6 (state); X1 (observability); X2 (error handler). |

### M5. Workflow Engine

| | |
|---|---|
| **Purpose** | Sequence the application-level stages. |
| **Inputs** | A workflow plan (from M4); the available plug-ins (from M15). |
| **Outputs** | Stage execution results; stage checkpoints. |
| **Responsibilities** | Stage sequencing; parallelism policy; stage-to-component dispatch; checkpoint emission. |
| **Interactions** | M4 (orchestrator); M6 (state); all plug-in entry points; the Domain models (M7–M13). |

### M6. State Manager

| | |
|---|---|
| **Purpose** | Hold the state of in-flight verifications. |
| **Inputs** | State updates from M4 and M5. |
| **Outputs** | State snapshots; resumable state. |
| **Responsibilities** | Checkpoint persistence; resumability; consistency invariants. |
| **Interactions** | M4 (orchestrator); M5 (workflow). |

### M7. Claim Model

| | |
|---|---|
| **Purpose** | Define the data structure for an atomic claim. |
| **Inputs** | Raw text (from the answer). |
| **Outputs** | Claim instances with text, normalized text, type, source span in the answer. |
| **Responsibilities** | Atomicity; decontextualization; type assignment; span preservation. |
| **Interactions** | M5 (workflow); M10 (graph) — the claim is a node type. |

### M8. Evidence Model

| | |
|---|---|
| **Purpose** | Define the data structure for a piece of evidence. |
| **Inputs** | Raw text or bytes (from a reference). |
| **Outputs** | Evidence instances with text, source ID, source type, span offset, trust signal. |
| **Responsibilities** | Provenance preservation; trust signal handling; span addressing. |
| **Interactions** | M5 (workflow); M10 (graph) — evidence is a node type; M17 (embedding index). |

### M9. Verdict Model

| | |
|---|---|
| **Purpose** | Define the data structure for a verification verdict. |
| **Inputs** | A (claim, evidence) pair; judge outputs. |
| **Outputs** | Verdict instances with label, confidence, judge trace. |
| **Responsibilities** | Verdict taxonomy enforcement (5 labels: supported, contradicted, partially_supported, not_enough_info, unverifiable); confidence range enforcement; judge trace recording. |
| **Interactions** | M5 (workflow); M10 (graph) — verdict is an edge attribute; M12 (conflict resolver); M13 (calibrator). |

### M10. Graph Model

| | |
|---|---|
| **Purpose** | Define the data structure for the claim-evidence bipartite graph. |
| **Inputs** | Claim nodes (M7), evidence nodes (M8), edge candidates. |
| **Outputs** | The graph; edge operations; graph queries. |
| **Responsibilities** | Bipartite graph construction; edge creation; edge scoring; graph traversal; subgraph extraction. |
| **Interactions** | M5 (workflow); M7, M8 (nodes); M9 (edge attributes); M12 (conflict detection). |

### M11. Trace Model

| | |
|---|---|
| **Purpose** | Define the data structure for a complete verification trace. |
| **Inputs** | Trace events from all components. |
| **Outputs** | A trace object recording every step of the verification. |
| **Responsibilities** | Event recording; trace structure; reproducibility. |
| **Interactions** | X1 (observability sink); M4 (orchestrator). |

### M12. Conflict Resolver

| | |
|---|---|
| **Purpose** | Detect conflicts in the verdict graph. |
| **Inputs** | The graph (M10). |
| **Outputs** | A conflict set; per-claim conflict flags. |
| **Responsibilities** | Detect claim-level conflicts (one claim, contradicting verdicts); detect edge-level conflicts (same claim, different evidence); flag but do not adjudicate. |
| **Interactions** | M10 (graph); M5 (workflow). |

### M13. Confidence Calibrator

| | |
|---|---|
| **Purpose** | Convert raw confidence to calibrated confidence. |
| **Inputs** | Raw verdicts (M9); calibration model. |
| **Outputs** | Calibrated verdicts. |
| **Responsibilities** | Apply temperature / Platt / isotonic scaling; report calibration error; plug-in selection. |
| **Interactions** | M5 (workflow); X1 (observability). |

### M14. Plugin Registry

| | |
|---|---|
| **Purpose** | Maintain the catalog of available plug-ins. |
| **Inputs** | Plug-in metadata at startup; plug-in self-registration. |
| **Outputs** | Plug-in lookups; plug-in lists per category. |
| **Responsibilities** | Plug-in discovery; version tracking; metadata storage; query interface. |
| **Interactions** | M3 (configuration); M15 (loader). |

### M15. Plugin Loader

| | |
|---|---|
| **Purpose** | Instantiate and lifecycle-manage plug-ins. |
| **Inputs** | Plug-in IDs (from M14); instantiation context. |
| **Outputs** | Plug-in instances; plug-in capability exposure. |
| **Responsibilities** | Plug-in instantiation; lifecycle (init, run, shutdown); capability advertisement. |
| **Interactions** | M14 (registry); M5 (workflow). |

### M16. Judge Pool

| | |
|---|---|
| **Purpose** | Hold the available judges (NLI, LLM, etc.). |
| **Inputs** | Judge descriptors at startup. |
| **Outputs** | Judge instances; judge selection per layer. |
| **Responsibilities** | Judge registration; judge selection by layer; judge versioning; cost tracking. |
| **Interactions** | M5 (workflow); X3 (bias corrector). |

### M17. Embedding Index

| | |
|---|---|
| **Purpose** | Hold embeddings of evidence and (optionally) claims for retrieval. |
| **Inputs** | Evidence texts (M8); claim texts (M7). |
| **Outputs** | Top-K nearest neighbors for a query. |
| **Responsibilities** | Embedding generation; vector storage; similarity search; index refresh. |
| **Interactions** | M5 (workflow); M8 (evidence); M7 (claim). |

### X1. Observability Sink (Cross-Cutting)

| | |
|---|---|
| **Purpose** | Receive trace events from all components. |
| **Inputs** | Trace events. |
| **Outputs** | Trace logs, metrics, structured traces. |
| **Responsibilities** | Receive events; aggregate; export. |
| **Interactions** | All components emit events. |

### X2. Error Handler (Cross-Cutting)

| | |
|---|---|
| **Purpose** | Capture and route component failures. |
| **Inputs** | Errors from any component. |
| **Outputs** | Structured error reports; partial results; fallback verdicts. |
| **Responsibilities** | Error capture; classification; routing; partial-result reporting. |
| **Interactions** | All components. |

### X3. Bias Corrector (Cross-Cutting)

| | |
|---|---|
| **Purpose** | Mitigate known biases in LLM judges. |
| **Inputs** | LLM verdicts (M9). |
| **Outputs** | Bias-corrected verdicts. |
| **Responsibilities** | Apply swap-averaging, length regression, cross-family voting, constitutional rubric. |
| **Interactions** | Layer 4 and Layer 5 of the verification workflow; M16 (judge pool). |

---

## Closing Note

This document describes the **shape** of the Hallucination Verification Engine, not its implementation. The 17 modules + 3 cross-cutting concerns, the 5 logical layers, the 11-stage execution pipeline, the 9-state claim lifecycle, the 8-state evidence lifecycle, the 5-state edge lifecycle, and the 11-step final scoring workflow are the architectural commitments of the project.

The next document in the series is the **theoretical framework** — the formal definitions of Claim, Evidence, Verdict, the verification function V, the calibration function, and the aggregation function. After that, the **evaluation suite** — the organic benchmarks and the metrics that will be used to validate the architecture.

*End of reference architecture.*
