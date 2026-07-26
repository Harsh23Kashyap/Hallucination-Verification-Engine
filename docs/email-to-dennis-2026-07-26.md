# Email to Dennis — 2026-07-26 (draft)

**To:** Dennis Shasha, Taranum Wasu
**Cc:** Daksh, Jacob
**Subject:** HVE round-1 results — permutation test passes

---

Dear Professor,

Taranum shared the first-round results on Saturday night. I went through the data this morning and want to share what I found before the statistical-proof step.

The headline: the arxiv_permutation test passes. On 22 SWAPPED citations, HVE flagged 19 as not_supported (sensitivity 95.5%, the other 3 went to unverifiable rather than wrong approvals). On 30 UNSWAPPED citations, HVE approved 24 as ok (specificity 80.0%, the other 6 went to unverifiable rather than false rejections). The gap between the two rates is 75.5 percentage points, well above the big enough threshold you mentioned.

The 47.6% headline on the dietnerd_permutation run is harder to read. The summary report does not separate SWAPPED from UNSWAPPED claims in the raw data, so the aggregate cannot be turned into a permutation-test gap without a re-run. Taranum confirmed the SWAP is happening in the perturbation step, but the flag was not propagated into the per-claim records. A 5-line change to the perturbation script would fix this, and I would suggest making that part of round 2.

Two more findings worth flagging: the format_test passes cleanly (7 claims, 0 hallucinations), and the OpenDeepResearch run caught 7 manually-verified hallucinations in the agent output — including two cases where the agent invented specific numeric values (a creatine dosage range for older adults, and a blood-pressure effect size for omega-3) that do not appear in the cited sources. No false positives in the manual re-check.

For round 2, my priorities would be:

1. Re-run dietnerd_permutation with the SWAP flag on each claim, so we can measure the gap
2. Scale up the arxiv run 5–10x to put error bars on the 95.5% / 80.0% rates
3. Add a calibration check on the confidence scores, since they are currently just labels

I have written up the full analysis at `docs/RESULTS.md` in our repo and committed it this morning. Happy to walk through the numbers on a call if that would be useful.

Warmly,
Harsh
