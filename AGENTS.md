<!-- mavis-knowledge: auto-load pointer -->
## Knowledge graph (auto)

This repo is indexed by `mavis-knowledge`. Before answering code questions:

1. Read `.claude/knowledge/repo_map.md` (1-page summary of the repo).
2. For specific questions, run `mavis-knowledge search "<query>"` to pull the most relevant chunks from the index.
3. For symbol lookups, run `mavis-knowledge symbol <name>`.
4. For "what breaks if I change X", run `mavis-knowledge impact <file>`.

If the index looks stale, run `mavis-knowledge refresh`. To see the current state, run `mavis-knowledge status`.

---

## Project context (read this first)

- **Project:** Hallucination Verification Engine (HVE) — research prototype for verifying LLM-generated answers against provided references.
- **Mentor:** Professor Dennis Shasha (NYU, db shasha @ nyu). Reviews the HLD via email; provides design feedback and the validation protocol.
- **Teammate:** Taranum Wasu. Owns the implementation (separate repo). Communicates with the professor and Harsh in the same email thread.
- **Status (2026-07-23):** HLD approved with three design overrides. Implementation built. Validation via permutation test in progress.
- **Read `docs/STATUS.md` first** — it captures the post-approval state, the three design decisions that override the V2 architecture, and the validation methodology.
- **The 11-stage pipeline in `reference-architecture.md` is the V2 design.** The teammate's implementation uses a 4-stage merged pipeline (extract+match combined). The V2 docs are still the design-of-record until a V3 is written.
- **External systems** in the professor's vocabulary = expertise-based systems outside the nerd family (e.g., other LLM literature-review tools, OpenDeepResearch). Used as test cases for HVE.

## Key decisions that override the docs

1. **Merge claim extraction + reference matching** into one step (professor's framing: "each claim in the synopsis should end with one or more citations").
2. **"Ask the document directly"** verification — prompt the source document: "Do you agree with the claim X?" Not the V2 rubric-based LLM-as-judge.
3. **HVE is standalone** for now. Integration with CustomNerd is conditional on reducing CustomNerd's hallucinations.

## Validation protocol (permutation test, professor's exact words)

> *"We can take a synopsis that makes N citations and randomly permute N/2 citations. The N/2 that weren't permuted should be declared ok and the N/2 that are permuted should be declared not ok. This will not be perfect, but the difference should be big enough. We do this for several synopses and also for several related work sections of papers."*

- Taranum is building this now.
- Test cases: arXiv Related Work sections (real citations, varied domains) + OpenDeepResearch outputs (agent-generated) + permutation-perturbed versions of both.

## Repository boundaries

- **This repo** = design layer. V2 architecture docs, 10x research, HLD diagram. Authoritative for "what the system is supposed to be."
- **Taranum's repo** = implementation layer. The merged-pipeline version. Not in this repo. When stable, the implementation should be linked (or ported) here.
