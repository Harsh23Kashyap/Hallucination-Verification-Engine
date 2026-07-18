# Hallucination Verification Engine

A research prototype for verifying LLM-generated answers against a provided set of references. Takes a question, an answer, and the references the answer is grounded in. Returns a per-claim verdict (supported, contradicted, not enough info) with a confidence score, an evidence pointer, and an overall reliability assessment.

## What's here

```
.
├── docs/                             written content
│   ├── FINAL-REPORT.md               consolidated deliverable
│   ├── FINAL-REPORT-v3.html          print-ready HTML
│   ├── research-report.md            underlying 10x research synthesis
│   ├── reference-architecture.md     v1 architecture
│   ├── architecture-review-v2.md     committee review + v2
│   ├── architecture-brief.md         10-min read for the professor
│   └── archive/
│       └── FINAL-REPORT.html         older 33-page version
└── diagrams/
    ├── HVE-architecture.drawio       editable architecture (draw.io)
    └── HVE-architecture.drawio.png   high-res PNG export
```

## Reading order

1. `diagrams/HVE-architecture.drawio.png` — the high-level design, one page
2. `docs/architecture-brief.md` — 10-minute read on what the system is and why
3. `docs/reference-architecture.md` — the full architecture with the 24-module decomposition
4. `docs/architecture-review-v2.md` — 7-reviewer critique and the v2 changes
5. `docs/FINAL-REPORT.md` — the consolidated deliverable
6. `docs/research-report.md` — the 10x research synthesis (250+ sources) underlying it all

## What the system does

1. **Extract claims** — split the answer into atomic factual claims
2. **Match references** — for each claim, find the most relevant spans in the references (BM25 + dense retrieval, fused with RRF)
3. **Verify** — for each (claim, reference) pair, an NLI model pre-filters the easy ones, an LLM judge handles the rest
4. **Aggregate** — combine per-claim verdicts into an overall reliability assessment

Output: per-claim verdict + confidence + evidence pointer + overall verdict.

## Status

Concept / research proposal. The HLD is the deliverable for the professor's review. Implementation has not started.

## License

MIT
