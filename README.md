# Hallucination Verification Engine

A research prototype for verifying LLM-generated answers against a provided set of references. Takes a question, an answer, and the references the answer is grounded in. Returns a per-claim verdict (supported, contradicted, not enough info) with a confidence score, an evidence pointer, and an overall reliability assessment.

## What's here

```
.
├── docs/                             written content
│   ├── STATUS.md                     current project state + decisions (read first)
│   ├── RESULTS.md                    first-round validation results analysis
│   ├── FINAL-REPORT.md               consolidated deliverable
│   ├── FINAL-REPORT-v3.html          print-ready HTML
│   ├── research-report.md            underlying 10x research synthesis
│   ├── reference-architecture.md     v1 architecture
│   ├── architecture-review-v2.md     committee review + v2
│   ├── architecture-brief.md         10-min read for the professor
│   └── archive/
│       └── FINAL-REPORT.html         older 33-page version
├── diagrams/
│   ├── HVE-architecture.drawio       editable architecture (draw.io)
│   └── HVE-architecture.drawio.png   high-res PNG export
└── results/                          Taranum's first-round validation results (2026-07-25)
    ├── arxiv_permutation/            arXiv Related Work + permutation test
    ├── dietnerd_permutation/         CustomNerd-style diet/nutrition + permutation
    ├── format_test/                  format coverage (PDF / web / text)
    └── open_deep_research_tavily/    ODR agent output hallucination hunt
```

## Reading order

1. `docs/STATUS.md` — current state of the project (read first; explains the design overrides from the professor's feedback)
2. `docs/RESULTS.md` — interpretation of the first-round validation results (permutation test pass/fail per experiment)
3. `diagrams/HVE-architecture.drawio.png` — the high-level design, one page (V2; the implementation in the teammate's repo uses the merged 4-stage variant)
4. `docs/architecture-brief.md` — 10-minute read on what the system is and why
5. `docs/reference-architecture.md` — the full architecture with the 24-module decomposition
6. `docs/architecture-review-v2.md` — 7-reviewer critique and the v2 changes
7. `docs/FINAL-REPORT.md` — the consolidated deliverable
8. `docs/research-report.md` — the 10x research synthesis (250+ sources) underlying it all

## What the system does

1. **Extract claims** — split the answer into atomic factual claims
2. **Match references** — for each claim, find the most relevant spans in the references (BM25 + dense retrieval, fused with RRF)
3. **Verify** — for each (claim, reference) pair, an NLI model pre-filters the easy ones, an LLM judge handles the rest
4. **Aggregate** — combine per-claim verdicts into an overall reliability assessment

Output: per-claim verdict + confidence + evidence pointer + overall verdict.

## Status

**Approved, validation in progress** (as of 2026-07-26). The HLD was approved by Professor Dennis Shasha (NYU) on 2026-07-22 with three design overrides: (1) merge claim extraction + reference matching into one step, (2) use the "ask the document directly" framing for verification, (3) ship as a standalone tool (not CustomNerd-integrated). Initial implementation is being built by Taranum Wasu (separate repo). Validation methodology defined by the professor: **permutation test** (N citations → permute N/2 → HVE should detect the perturbations) + arXiv Related Work + OpenDeepResearch outputs.

**Round-1 results (2026-07-25, Taranum):** the arxiv_permutation test passes with a 75.5 pp gap between the SWAPPED and UNSWAPPED subsets (sensitivity 95.5%, specificity 80.0%). The dietnerd_permutation data does not contain an explicit SWAP split and needs a re-run before it can be evaluated as a permutation test. The format_test passes cleanly. The ODR experiment caught 7/7 manually-verified hallucinations in agent output. Full breakdown in `docs/RESULTS.md`. **Next: statistical proof** (significance testing on the perturbation gap, larger sample size).

## License

MIT
