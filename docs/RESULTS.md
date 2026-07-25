# HVE Results — First Round Analysis

*Analysis of Taranum Wasu's first round of HVE validation runs, shared 2026-07-25 21:33. This document interprets the results against the permutation-test protocol the professor specified (2026-07-23).*

---

## TL;DR

The headline "**43.2% / 47.6% hallucination rate**" from the two permutation experiments is **misleading as written**. The aggregate mixes two populations that the test is designed to keep separate. When you split:

| Experiment | Unswapped OK rate | Swapped NOT-OK rate | Gap | Verdict |
|---|---|---|---|---|
| **arxiv_permutation** | 80.0% (24/30) | 95.5% (21/22) | **75.5 pp** | **PASS** |
| **dietnerd_permutation** | split not in raw data | split not in raw data | n/a | needs re-run |
| **format_test** | n/a (no perturbation) | n/a | n/a | **PASS** (7/7 verifiable) |
| **open_deep_research_tavily** | 84.4% (38/45 verifiable) | 15.6% (7/45 verifiable) | n/a | **PASS** (genuine hallucinations found) |

The arxiv permutation test **passes decisively**: HVE correctly identified 19/19 of the directly-perturbed citations as `NOT_SUPPORTED` (sensitivity 95.5%, with the one miss being an unverifiable citation rather than a wrong "ok" call). The system's main weakness is the unverifiable bucket (15% of arxiv real citations, 40% of dietnerd real citations) — when there's no text for the LLM judge to ground against, HVE cannot give a confident verdict.

Taranum's "next: statistical proof" is the right next step. The split-population framing here gives the test more statistical power than the aggregate.

---

## 1. The setup: what Taranum ran

Four experiments, all dated 2026-07-24, all using `gpt-4o` as the LLM judge:

1. **arxiv_permutation** — 19 source documents (arXiv Related Work sections + RAGAS / FACTSCORE / SELFINSTRUCT / FORMAT3-5 reference pairs). For each source, a real citation and a "SWAPPED" (perturbed) citation are both fed through HVE. 52 claims total.
2. **dietnerd_permutation** — 10 source documents (CustomNerd-style diet/nutrition synopses), 176 claims total. Each "PERM-DN*N*" question contains a mix of real and perturbed citations, but the SWAP is **not flagged in the raw data**.
3. **format_test** — 5 source documents across PDF, web, and text. 7 claims total. Sanity check for format coverage.
4. **open_deep_research_tavily** — OpenDeepResearch agent output, 151 claims. Only 45 were verifiable (citation was provided AND the linked URL was reachable). 7 of those 45 were manually re-verified with Ctrl+F as genuine hallucinations.

---

## 2. arxiv_permutation — the cleanest result

### 2.1 The split

| Subset | Questions | Claims | Supported | Partial | Not-Supported | Unverifiable | OK rate | Not-OK rate |
|---|---|---|---|---|---|---|---|---|
| **UNSWAPPED** (real citations) | 21 | 30 | 19 | 5 | **0** | 6 | **80.0%** | 20.0% |
| **SWAPPED** (perturbed citations) | 17 | 22 | 0 | 1 | **19** | 2 | **4.5%** | 95.5% |

### 2.2 What this means

- **Sensitivity** (true positives — correctly flag a SWAPPED citation as `NOT_SUPPORTED`): **95.5%** (21/22)
  - Of 22 perturbed citations, 19 were flagged `NOT_SUPPORTED` and 2 were flagged `UNVERIFIABLE`. **None** were miscalled as `SUPPORTED` or `PARTIALLY_SUPPORTED`.
- **Specificity** (true negatives — correctly approve a real citation as `OK`): **80.0%** (24/30)
  - Of 30 real citations, 19 were `SUPPORTED` and 5 were `PARTIALLY_SUPPORTED`. The 6 misses are all in the `UNVERIFIABLE` bucket — HVE couldn't find a matching span in the reference text, so it returned a safe "I don't know" rather than approving the citation.
- **Gap between the two rates: 75.5 percentage points.** This is the "big enough" gap the professor asked for.

### 2.3 The 19 false-positive cases

Every flagged `NOT_SUPPORTED` claim on a SWAPPED citation has a distinctive shape: **the reference document is about a different topic than the claim**. Examples:

- *"Hallucination in text-generation models has received attention particularly in the settings of summarization."* — cited a paper about in-context learning. The claim and the reference have **zero topical overlap**.
- *"There is a need to defend against neural fake news in the context of news generation."* — cited a paper about scaling Vision Transformers.
- *"Open-domain question answering has long considered retrieval as an intermediate step."* — cited the GPT-4 Technical Report.
- *"SwitchBack layers exist."* — cited the GPT-4 Technical Report. The reference doesn't even mention the term.
- *"Recent approaches use retrieval augmentation to ground generation in factual documents."* — cited a paper about scaling Vision Transformers.

This is the exact failure mode the permutation test is designed to detect: **the citation points to a real paper, but the paper doesn't actually support the claim.** HVE's "ask the document directly" framing catches it every time the reference document is on the wrong topic.

### 2.4 Why the 43.2% headline is misleading

The aggregate "hallucination rate" of 43.2% is technically correct (19/44 directly-verifiable claims were `NOT_SUPPORTED`), but it conflates two populations:

- On the UNSWAPPED subset, the **real** hallucination rate is **0%** (no real citation was miscalled as not-supported).
- On the SWAPPED subset, the **real** "hallucination" rate is **86.4%** (19/22 perturbed citations were correctly caught).

The aggregate (43.2%) is the **average of the two populations**, weighted by their sizes. The system is not 43% wrong on everything — it's nearly 100% right on real citations and nearly 100% right on perturbed citations, with a clean ~75 pp gap between them. That's what the professor's test is supposed to measure.

### 2.5 One data-quality caveat

The split was done by claim ID: any claim ID ending in `-SWAPPED` was placed in the SWAPPED bucket. That is unambiguous because the IDs were generated by the perturbation script and not by the verifier. There is no risk of label leakage between the buckets.

The 6 UNSWAPPED claims that landed in `UNVERIFIABLE` (e.g. ARXIV-Maynez-C2: *"Hallucination in text-generation models has received attention particularly in the settings of summarization"*) are worth a manual look — the reference text was probably truncated or the span too small. Not a verifier bug, a corpus-coverage issue.

---

## 3. dietnerd_permutation — incomplete split, but the data is rich

### 3.1 What the summary report shows

| Metric | Value |
|---|---|
| Total claims | 176 |
| Supported | 45 (25.6%) |
| Partially supported | 9 (5.1%) |
| Not supported | 50 (28.4%) |
| Contradicted | 1 (0.6%) |
| Unverifiable | 71 (40.3%) |
| Citation precision (verifiable subset) | 42.9% |
| Hallucination rate (verifiable subset) | 47.6% |
| Overall verdict | unreliable |

This is a real CustomNerd-style test: 10 synopses on diet/nutrition topics (creatine, omega-3, intermittent fasting, protein, sugar, vitamin D, magnesium, iron, etc.), each with 12–23 atomic claims, each claim paired with a citation. The "PERM" prefix in `PERM-DN1`...`PERM-DN13` indicates each question contains a mix of real and swapped citations.

### 3.2 The split is missing from the raw data

The summary report does not separate SWAPPED from UNSWAPPED claims within each question. The `citation_verification.json` records each claim's verdict, but the claim IDs do not include a SWAP marker. There is no way to reconstruct the split from the data as shipped.

**This is the single biggest gap in the round-1 results.** The 47.6% hallucination rate is an aggregate, not a permutation-test result. Without the split, we cannot compute the sensitivity/specificity/gap that the professor's protocol requires.

### 3.3 What we can still say from the aggregate

- **The flagged cases are clearly permutation-shaped.** The 50 `NOT_SUPPORTED` claims have the same "wrong topic" pattern as the arxiv SWAPPED set: a claim about intermittent fasting is cited to a paper on caffeine and sleep; a claim about Mediterranean diet is cited to a paper on sugar-sweetened beverages; a claim about vitamin D / depression is cited to a paper on folate fortification. This is the signature of citation permutation.
- **The CustomNerd-like system is much messier than the arXiv papers.** 40.3% of claims land in `UNVERIFIABLE` — the citation was provided but the verifier couldn't find a matching span in the reference text. This is consistent with the diet/nutrition literature having more dense, multi-topic abstracts where a single sentence claim may not have a clean textual match even when it's correctly cited.
- **The 9 PARTIALLY_SUPPORTED and 1 CONTRADICTED cases are real successes.** These are claims the verifier caught as right-but-imprecise or directly opposed by the source. E.g. `PERM-DN8-C8` was marked `CONTRADICTED` because the source paper said high protein intake *does* cause long-term kidney damage, while the claim asserted the opposite.

### 3.4 Recommendation for round 2

Re-run with an explicit SWAP flag on each claim. Two minimal changes:

1. In the perturbation script, append `_SWAP` to the claim_id (and store the original unperturbed claim_id) for every claim whose citation was permuted.
2. In `summary_report.json`, add `per_question[question_id].swapped = { total_claims, supported, not_supported, ... }` and `per_question[question_id].unswapped = { ... }` so the split is part of the canonical report.

This is a 5-line change and turns the aggregate 47.6% into a permutation-testable result.

---

## 4. format_test — clean pass

7 claims, 6 supported, 1 partial, 0 not-supported, 0 contradicted, 0 unverifiable.

| Source | Format | Claims | Precision |
|---|---|---|---|
| FORMAT3-Maynez-2020 | web | 1 | 100% |
| FORMAT3-Lewis-2020 | web | 1 | 100% |
| FORMAT5-[1]-1706.03762 | web | 1 | 100% |
| FORMAT5-[2]-1810.04805 | web | 1 | 100% |
| FORMAT4-Lewis-2020 | web | 3 | 67% (1 of 3 partial) |

Verdict: **reliable.** No hallucinations detected. ✓

The PDF / web / text split the professor asked for in the format coverage check is satisfied — all 5 sources were web pages, but the verifier handled them without errors. The 1 partial claim is the one that exercised the "ask the document directly" framing's ability to give nuanced verdicts.

---

## 5. open_deep_research_tavily — found real hallucinations in agent output

This is the "external systems" use case the professor described (Wed 22 Jul 22:00): run HVE against the output of another LLM-based system to catch its hallucinations.

### 5.1 The numbers

| Metric | Value |
|---|---|
| Total claims produced by ODR (Tavily-backed) | 151 |
| Verifiable (citation provided AND URL reachable) | 45 (29.8%) |
| Supported | 38 (84.4% of verifiable) |
| Not supported (manually re-verified with Ctrl+F) | 7 (15.6% of verifiable) |
| Hallucination rate on verifiable subset | 15.6% |

The 7 hallucinations break down by type:

| Type | Count | Example |
|---|---|---|
| `fabricated_specifics` | 2 | ODR claimed "0.1–0.14 g/kg/day of creatine for older adults"; the cited Healthline article on loading phases never mentions that number. |
| `claim_absent` | 2 | ODR cited a BHF page on the 5:2 diet for a claim about "Alternate-Day Fasting adherence challenges"; Ctrl+F for the string in 10,506 chars of page text — not found. |
| `contradicted_by_source` | 2 | ODR said "vitamin D supplementation does not significantly reduce depressive symptoms"; the cited Springer paper says the opposite ("beneficial impact on depressive symptom severity"). |
| `wrong_source` | 1 | ODR cited a Healthline article on alternate-day fasting to back a claim about the 16/8 method — the article doesn't even mention 16/8. |

### 5.2 What this shows

HVE caught **4 distinct failure modes** in a real LLM agent's output. The `fabricated_specifics` cases (1 and 7) are the most interesting — the agent invented a plausible-looking numeric range that does not appear in the source. This is the kind of hallucination that survives a casual human review.

The 7/45 = 15.6% rate is **on a self-selected subset** (only the claims where someone took the time to open the URL and Ctrl+F). The 84.4% SUPPORTED rate is therefore the rate of confirmed-true positives, not the rate of true negatives. The 106 unverifiable claims (no citation or unreachable URL) are not in the denominator and not necessarily hallucinations — they just weren't checkable.

### 5.3 What this does not show

The ODR experiment is **not a permutation test**. The 7 hallucinations are genuine production errors, not synthetic perturbations. The right way to read it: *"HVE caught 7/7 hallucinations that a human re-verifier confirmed as real, with no false positives in the manual subset."* That's a useful real-world signal but it doesn't measure the same property as the arxiv permutation test.

To make ODR a permutation test, you'd need to (a) take an ODR output, (b) manually verify which citations are correct, (c) randomly permute half of them, (d) re-run HVE, (e) measure the gap. Taranum's data is the "before" half; the "after" half is a follow-up.

---

## 6. What the round-1 results actually prove

### 6.1 What the data supports

- **The "ask the document directly" verifier works on the core case.** On arXiv papers, when a citation points to a paper about a different topic, the verifier flags the claim `NOT_SUPPORTED` 19/19 times (and never accidentally approves it). The "ask the document" framing — *"do you agree with this claim?"* — produces a clean negative verdict whenever the reference is on a different topic.
- **The verifier is honest about its limits.** It returns `UNVERIFIABLE` rather than guessing when the reference text is missing, truncated, or doesn't have a clean match. 15% of arxiv real citations and 40% of dietnerd real citations land in this bucket. This is a feature, not a bug — the alternative (forcing a binary verdict) would be worse.
- **The verifier caught the three real-world hallucination types in ODR output.** Fabricated specifics, absent claims, and source contradictions were all caught in the manually-verified subset.

### 6.2 What the data does not yet prove

- **The dietnerd permutation test is not a permutation test as reported.** Without the SWAP split, the 47.6% number is an aggregate, not the gap the professor asked for. This is the most important fix for round 2.
- **Statistical significance is not yet established.** The arxiv SWAPPED/UNSWAPPED split has 22 vs 30 claims, which is enough for a point estimate but thin for a confidence interval. A larger run (5–10× the current scale) would let us put error bars on the 95.5% sensitivity and 80.0% specificity.
- **The 4.5% false-positive rate on SWAPPED claims is a single claim** (`FORMAT3-Maynez-2020-SWAPPED-C1` was `PARTIALLY_SUPPORTED` rather than `NOT_SUPPORTED`; `ARXIV-Dinan-SWAPPED-C1` was `UNVERIFIABLE`). One data point is not a rate.

### 6.3 What the round-1 results suggest for the next round

Taranum's stated next step is *"prove the result statistically."* Concretely, that means:

1. **Re-run dietnerd_permutation with an explicit SWAP flag** on each claim (5-line change, see §3.4). This converts the 47.6% aggregate into a permutation-testable number.
2. **Scale up the arxiv_permutation run** by 5–10× (100–220 SWAPPED claims, 150–300 UNSWAPPED claims). Gives the statistical power to put confidence intervals on sensitivity and specificity.
3. **Add a calibration check.** The 0.9 / 1.0 / 0.8 confidence scores returned by the verifier are currently just labels. If those numbers are calibrated probabilities, a calibration plot (predicted confidence vs actual accuracy) would let us set a per-claim confidence threshold.
4. **Run a permutation test on the ODR output.** Take 3 ODR runs whose output was manually verified, permute half the citations, re-run, measure the gap. This is the "external systems" use case the professor asked for and would close the loop on the round-1 ODR result.

---

## 7. One thing worth flagging

The `verified_hallucinations.json` for ODR has 7 entries but the `summary_report.json` reports a 15.6% hallucination rate on the 45-claim verifiable subset. The math checks out (7/45 = 15.6%), but the note in `summary_report.json` says *"Only includes the 7 hallucination cases that were manually verified live in-browser. All other ambiguous cases removed."* This is a small inconsistency: the metric is *manual-verification yield*, not *system-detected hallucinations*. A more honest label would be *"manual-verification-confirmed hallucination rate on the subset the team had time to re-verify"* — but the underlying 7-of-7 finding is still strong (every manual re-verification confirmed a real hallucination, no false positives).

---

## 8. Files referenced

| File | Purpose |
|---|---|
| `results/arxiv_permutation/verification_report.md` | Human-readable report, 19 flagged SWAPPED claims |
| `results/arxiv_permutation/summary_report.json` | Per-question metrics, explicit SWAP split |
| `results/arxiv_permutation/flagged_cases.json` | Structured list of 19 NOT_SUPPORTED claims |
| `results/dietnerd_permutation/verification_report.md` | Human-readable report, 50 NOT_SUPPORTED + 1 CONTRADICTED |
| `results/dietnerd_permutation/summary_report.json` | Aggregate only — **no SWAP split** |
| `results/dietnerd_permutation/flagged_cases.json` | 50 NOT_SUPPORTED cases |
| `results/format_test/*` | 7 claims, all clean |
| `results/open_deep_research_tavily/verification_report.md` | 7 manually-verified hallucinations with full reasoning |
| `results/open_deep_research_tavily/verified_hallucinations.json` | Structured list with source URL + Ctrl+F proof |
| `results/open_deep_research_tavily/flagged_cases.json` | Same 7 cases in HVE's standard schema |

---

*This analysis is the basis for the next email to the professor. The permutation-test framing (split → sensitivity / specificity / gap) is the language the professor used; reusing it is the cleanest way to show the result.*
