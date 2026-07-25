# Project Status

*Live state of the Hallucination-Verification-Engine project as of 2026-07-25. The historical design docs (reference-architecture.md, architecture-review-v2.md, architecture-brief.md, FINAL-REPORT.md) describe the V2 design as proposed. This file tracks what actually happened after the proposal was sent to the professor.*

---

## Current state (2026-07-25)

- **HLD approved** by Professor Dennis Shasha (NYU) on 2026-07-22.
- **Initial implementation built** by Taranum Wasu (project teammate). The implementation is **NOT in this repo** — it lives separately. This repo remains the design/proposal layer.
- **Validation methodology defined** by the professor: permutation test + arXiv real citations + OpenDeepResearch outputs. Taranum is executing this in parallel.
- **Latest from Taranum (2026-07-25 21:33):** first round of permutation tests, ODR runs with verified hallucinations, format coverage, and the docs are complete. Taranum is now moving to **prove the result statistically** — the next milestone is significance testing on the permutation test results.
- **First-round results analyzed (2026-07-26).** Full breakdown in `docs/RESULTS.md`. Headline: the arxiv_permutation test **passes** with a 75.5 pp gap between the SWAPPED and UNSWAPPED subsets (sensitivity 95.5%, specificity 80.0%). The dietnerd_permutation data does not contain an explicit SWAP split — that needs a re-run. The format_test passes cleanly. The ODR experiment found 7/7 manually-verified hallucinations (no false positives in the manual re-check). The aggregate "43.2% / 47.6% hallucination rate" headlines are misleading as written — they conflate the two populations that the permutation test is designed to keep separate.
- **Integration decision pending** — HVE will be **standalone** for now. The professor will only integrate it into CustomNerd if it measurably reduces CustomNerd's hallucinations.

## Email thread (summarized)

- **Tue 21 Jul 00:36** — Harsh sent the HLD email with the simplified 4-stage architecture + tech stack. Three specific questions for the professor.
- **Wed 22 Jul 09:17** — Professor replied. Key feedback:
  - On the HLD as a whole: *"Perfect. That's what we want."*
  - On 4-stage split (extract / match / verify / aggregate): *"Each claim in the synopsis should end with one or more citations, so merging makes more sense to me but I don't feel strongly about it."* — **merge** claim extraction + reference matching.
  - On NLI-pre-filter-then-LLM ordering: *"I have no good intuition about this question."* — this needs empirical validation, not a priori.
  - On tech stack: *"Ideally when looking at the source document we'd like to ask that document 'Do you agree with the claim \"blah-blah-blah\" ?'"* — **"ask the document directly"** framing for verification.
- **Tue 21 Jul 18:04** — Harsh forwarded to Taranum with green light: *"Please go ahead with this."*
- **Wed 22 Jul 20:38** — Taranum shared the implementation. Reported 92% precision and 0% hallucination on 5 test questions. Asked clarification on (a) the "external systems" evaluation and (b) standalone vs CustomNerd integration.
- **Wed 22 Jul 22:00** — Professor clarified: external systems = expertise-based systems outside the nerd family. Use arXiv Related Work sections + OpenDeepResearch outputs as test cases. HVE is **standalone**; integration only if it reduces CustomNerd hallucinations.
- **Wed 22 Jul 23:27** — Harsh asked professor for more detail on the verification mechanism.
- **Thu 23 Jul 06:22** — Professor specified the **permutation test**: take a synopsis with N citations, randomly permute N/2, run HVE, expect N/2 unswapped = "ok" and N/2 swapped = "not ok". Apply to multiple synopses + arXiv Related Work sections.
- **Thu 23 Jul 14:56** — Taranum confirmed understanding. Already tested on arXiv. Now building the permutation test + OpenDeepResearch evaluation.
- **Fri 24 Jul 00:07** — Professor confirmed: "Correct" on the permutation test. Reinforced: *"do the same thing for arxiv papers as you are doing for synopses"* — the permutation test applies to **both** synopses and arXiv Related Work sections.
- **Sat 25 Jul 10:58** — Harsh confirmed understanding to the professor.
- **Sat 25 Jul 19:21** — Professor: *"Great. Thanks."*
- **Sat 25 Jul 21:33** — Taranum shared first results: permutation tests done, ODR runs with verified hallucinations done, format coverage done, docs done. **Next: statistical proof.**

## Design decisions (overrides the V2 architecture)

These three decisions came from the professor's feedback and **supersede the V2 architecture where they conflict**.

### 1. Merge claim extraction + reference matching

- **V2 (proposed):** separate stages for Extract (S5) and Match (S7) with Normalize (S6) and Edge Construction (S8) in between.
- **Decided:** extract and match collapse into one step. For each claim, the system directly retrieves the evidence spans it needs. The professor's framing: *"each claim in the synopsis should end with one or more citations, so merging makes more sense"*.
- **Implication for the HLD:** the 11-stage pipeline (S1–S11) should be redrawn as a 4-stage pipeline (extract+match / verify / aggregate / report) or 5 stages if conflict resolution stays separate. The HVE-architecture.drawio in this repo is the V2 version; the implementation in the teammate's repo uses the merged version.
- **Open:** where does claim normalization (coreference resolution, decontextualization) live? Inside the merged step, or in a pre-pass?

### 2. "Ask the document directly" verification framing

- **V2 (proposed):** the judge stack uses a rubric-based prompt with self-critique. The LLM judge reads claim + evidence span and renders one of 5 verdicts.
- **Decided:** the verification step phrases the question to the source document directly: *"Do you agree with the claim 'X'?"*. The source document is the judge; the LLM is the conduit, not the judge. This is closer to entailment-as-grounding than to LLM-as-judge.
- **Implication for the HLD:** the L4 layer in the judge stack should be re-imagined as a question-to-document prompt, not a rubric-based verdict prompt. NLI (L3) and Jury (L5) may still apply, but L4's prompt structure changes. The 12-bias correction (X3) is still relevant because the LLM carrying the prompt is still biased — but the question is whether the framing change reduces bias.
- **Open:** does the "ask the document" prompt still go through the LLM, or is it sent to the document via something more direct (e.g., a fine-tuned NLI head)? Taranum's implementation will tell us.

### 3. HVE is standalone (not CustomNerd-integrated) for now

- **Decided:** HVE ships as a standalone tool. Integration with CustomNerd is conditional: *"If this reduces the hallucinations of customnerd, then we'll include it."* (Dennis, 2026-07-22 22:00)
- **Implication for the HLD:** the Integration Adapter Subsystem (M-IA1) moves from V2-critical to V2-optional. The primary deployment mode (in-process or sidecar) becomes the default. The 99.9% availability / 100 verif/min/instance SLO is internal, not CustomNerd-bound.

## Validation methodology (defined by the professor)

### The permutation test (primary)

The exact protocol from Dennis Shasha (2026-07-23 06:22):

> *"We can take a synopsis that makes N citations and randomly permute N/2 citations. The N/2 that weren't permuted should be declared ok and the N/2 that are permuted should be declared not ok. This will not be perfect, but the difference should be big enough. We do this for several synopses and also for several related work sections of papers."*

Reinforced on 2026-07-24 00:07: *"do the same thing for arxiv papers as you are doing for synopses"* — the permutation test applies identically to **both** synopses and arXiv Related Work sections.

**Operationalized:**
- Input: a synopsis or arXiv Related Work section with N citations.
- Perturbation: randomly permute N/2 citations (swap their referent with another citation in the same document).
- HVE prediction: the N/2 unperturbed citations → "ok"; the N/2 perturbed citations → "not ok".
- Metric: agreement with the ground-truth perturbation mask. Not accuracy — agreement. "Ok" + "not ok" labels are independent per-citation.
- Statistical power: "the difference should be big enough" — i.e., HVE's agreement rate on perturbed vs unperturbed should be statistically separable from 50% (random).
- **Statistical proof (next milestone, 2026-07-25):** Taranum is now computing the significance of the gap between perturbed and unperturbed agreement rates.
- Repeat: several synopses + several Related Work sections.

**Why this is a strong test:** it isolates the verification step from generation. A model that just memorizes the input would score near 100% on unperturbed and near 0% on perturbed (high agreement). A model that doesn't actually verify would score ~50% on both (no signal). The gap between HVE's score on the two subsets is the verification signal.

### Test cases

1. **arXiv Related Work sections** — real citations, real papers, varied domains. This is the canonical test set per the professor.
2. **OpenDeepResearch outputs** — agent-generated synopses with citations. Tests the "agent hallucination" scenario, not just "human-author hallucination."
3. **Permutation-perturbed versions of both** — synthetic ground truth.

## Taranum's reported initial metrics (2026-07-22 20:38, pre-permutation-test)

- **Scope:** 5 test questions.
- **Precision:** 92%.
- **Hallucination rate:** 0% (i.e., HVE did not output a "supported" verdict for any citation that was actually perturbed or absent).
- **Supported reference types:** PDF, web pages, plain text.
- **Output format:** per-claim verdict + confidence + evidence span pointer + overall reliability verdict, in JSON and Markdown.
- **Caveat:** n=5. Not statistically significant. The permutation test is the real evaluation.

## Taranum's progress as of 2026-07-25 21:33

- **Permutation test:** complete (raw results, not yet with statistical proof).
- **OpenDeepResearch runs with verified hallucinations:** complete.
- **Format coverage:** complete.
- **Docs:** complete.
- **Next:** prove the result statistically (compute significance of the perturbed-vs-unperturbed gap).

## Round-1 results — first look (2026-07-26)

Taranum shared 17 files across 4 experiment directories. The data is now in `results/` in this repo. Full analysis in `docs/RESULTS.md`. The three-line summary:

1. **arxiv_permutation: PASS.** 22 SWAPPED claims, 19 flagged `NOT_SUPPORTED` (sensitivity 95.5%, with 2 unverifiable, 0 false approvals). 30 UNSWAPPED claims, 24 `OK` (specificity 80.0%, with 6 unverifiable, 0 false rejections). **75.5 pp gap.**
2. **dietnerd_permutation: aggregate only.** 50 `NOT_SUPPORTED`, 1 `CONTRADICTED`, 71 `UNVERIFIABLE` out of 176 claims. The raw data does not flag which claims were SWAPPED — the summary's 47.6% number cannot be converted to a permutation-test gap without re-running with an explicit SWAP marker.
3. **ODR + format test: pass.** 7/7 manually-verified hallucinations in ODR output (no false positives in the re-check). Format test: 7/7 verifiable, 0 hallucinations.

## Architectural implications still to resolve

1. **Pipeline re-draw.** The 4-stage pipeline is the right shape now. The HVE-architecture.drawio in this repo shows the 11-stage V2 version. The teammate's implementation follows the merged shape. The drawio should be re-exported to match.
2. **Normalization placement.** Where does coreference resolution + decontextualization happen? Pre-pass before the merged extract+match, or inside it?
3. **Layer 4 prompt redesign.** The LLM judge prompt changes from "rubric + self-critique" to "do you agree with the claim X?". The 4 X3 bias-correction techniques may still apply, but their cost-effectiveness on the new framing is an open question.
4. **M-H1 (Human Annotation Subsystem) repurposing.** The V2 plan for M-H1 was gold-standard annotation by humans. The permutation test is a *synthetic* gold standard. M-H1 may no longer be the primary evaluation method. Worth keeping the gold-standard path open for "external systems" evaluation.
5. **External systems evaluation.** The professor's "external systems" mention is ambiguous. Likely meaning: HVE is run against the output of other expertise-based systems (e.g., other literature-review tools, other LLM agents) to see if HVE catches the hallucinations those systems make. The arXiv Related Work test is one example. OpenDeepResearch is another. Worth clarifying with the professor before M-H1 is finalized.
6. **Standalone packaging.** With HVE standalone, the in-process deployment mode is primary. The sidecar and service modes are future work. M-D1's distributed execution design may be deprioritized for V3.

## Open questions for the professor (carry-over from FINAL-REPORT §21)

- **Q3 (composition):** Resolved by the merge decision. The composition is Layered + Graph + Plugin; the pipeline is now 4 stages, not 11.
- **Q4 (trainable aggregator):** Open. The professor hasn't weighed in. The permutation test result may inform whether M-A1 is worth building.
- **Q5 (calibration):** Open. Same — no opinion from the professor.
- **Q6 (ethics):** Open. HVE is now standalone, so medical/legal deployment is further away. The 70–85% accuracy target is still a constraint.
- **New question (permutation test as evaluation):** Does the professor want M-H1 to remain the gold-standard annotation path, or is the permutation test sufficient? Likely "permutation test is primary, gold standard is fallback."

## What lives where

| Where | What's there | Status |
|---|---|---|
| **This repo** | Design docs (V2 architecture), 10x research, HLD diagram | **Design layer.** Authoritative for "what the system is supposed to be." |
| **Teammate's repo (Taranum)** | Initial implementation (merged extract+match, ask-the-document framing) | **Implementation layer.** The merged-pipeline version that the professor approved. Not in this repo. |
| **Email thread** | Approval, design decisions, validation protocol | **Decisions layer.** Captured here in this STATUS.md. |
| **Future commit back to this repo** | The HVE-architecture.drawio should be re-exported to reflect the 4-stage merged pipeline. The README's status section should be updated. The architecture-brief and FINAL-REPORT should have an addendum pointing to the merged version. | **Pending** — depends on Taranum's implementation stabilizing. |

## Action items

1. Re-export `diagrams/HVE-architecture.drawio` with the 4-stage merged pipeline once Taranum's implementation is stable.
2. Update `README.md` status section: "Approved, implementation in progress (Taranum), validation via permutation test."
3. Add an addendum to `architecture-brief.md` and `FINAL-REPORT.md` flagging the three V2-override decisions (merge, ask-the-document, standalone).
4. After the permutation test results come in: update Taranum's reported metrics with statistical significance.
5. After the professor confirms "it reduces hallucinations of customnerd": move M-IA1 from optional to required and re-plan M-D1.
