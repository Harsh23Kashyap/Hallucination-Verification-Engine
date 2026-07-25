# OpenDeepResearch — Hallucination Verification Report

## Summary

| Metric | Value |
|--------|-------|
| Total claims | 151 |
| Verifiable | 45 |
| Supported | 38 (84.4%) |
| Not supported (verified) | 7 (15.6%) |
| Hallucination rate | **15.6%** |

**Note:** Only includes the 7 hallucination cases manually verified live in-browser. All ambiguous cases removed.

## Verified Hallucinations (7 cases)

### Case 1: ODR-Q1 [[2]]

**Claim:** A relative dosage of 0.1-0.14 g/kg/day of creatine may be beneficial for older adults to enhance muscle strength and bone health.

**Verdict:** NOT_SUPPORTED (confidence: 0.9)

**Reasoning:** The content discusses creatine supplementation, including loading phases and general benefits for muscle mass and strength, but it does not mention specific dosage recommendations for older adults or any effects on bone health. The claim about a relative dosage of 0.1-0.14 g/kg/day for older adults is not addressed in the provided content.

### Case 2: ODR-Q2 [[17]]

**Claim:** Studies showed an average fat mass decrease of 6% while preserving fat-free mass for individuals on the 16/8 method.

**Verdict:** NOT_SUPPORTED (confidence: 0.9)

**Reasoning:** The content provided discusses alternate-day fasting (ADF) and its effects on weight loss and body composition, but it does not mention the 16/8 method or provide specific data about a 6% decrease in fat mass while preserving fat-free mass. The claim about the 16/8 method is not supported by the content available, which focuses on ADF and not the specific intermittent fasting method mentioned in the claim.

### Case 3: ODR-Q2 [[22]]

**Claim:** Some studies reported concerns about muscle loss occurring during fasting days with Alternate-Day Fasting.

**Verdict:** NOT_SUPPORTED (confidence: 1.0)

**Reasoning:** The content provided does not contain any information related to intermittent fasting, weight loss, or muscle loss. It appears to be about membrane protein purification, which is unrelated to the claim about fasting and muscle loss.

### Case 4: ODR-Q2 [[24]]

**Claim:** Some users of Alternate-Day Fasting cite adherence challenges and the potential for overeating on non-fasting days.

**Verdict:** NOT_SUPPORTED (confidence: 0.9)

**Reasoning:** The content provided does not contain any information about Alternate-Day Fasting, adherence challenges, or potential overeating on non-fasting days. It only briefly mentions the flexibility and adherence challenges of the 5:2 diet, which is a different form of intermittent fasting. Therefore, the claim is not supported by the content given.

### Case 5: ODR-Q3 [[1, 3]]

**Claim:** A review highlighted that vitamin D supplementation does not significantly reduce depressive symptoms.

**Verdict:** CONTRADICTED (confidence: 0.9)

**Reasoning:** The content explicitly states that analyses of vitamin D supplementation have demonstrated a beneficial impact on depressive symptom severity, which directly contradicts the claim that vitamin D supplementation does not significantly reduce depressive symptoms.

**Evidence from source (contradiction):** "Correspondingly, analyses of vitD supplementation have demonstrated a beneficial impact on depressive symptom severity"

### Case 6: ODR-Q3 [[4, 10, 12]]

**Claim:** Health practitioners are encouraged to incorporate routine vitamin D screening and recommendations for supplementation, especially in high-risk populations.

**Verdict:** CONTRADICTED (confidence: 1.0)

**Reasoning:** The content explicitly states that the evidence is not strong enough to recommend universal vitamin D supplementation in depression, which contradicts the claim that health practitioners are encouraged to incorporate routine vitamin D screening and recommendations for supplementation.

**Evidence from source (contradiction):** "the evidence is not strong enough to recommend universal supplementation in depression."

### Case 7: ODR-Q4 [[2]]

**Claim:** A systematic review in Frontiers in Nutrition revealed average reductions of 1.19 mmHg in systolic and 0.91 mmHg in diastolic pressure from omega-3 supplementation.

**Verdict:** NOT_SUPPORTED (confidence: 1.0)

**Reasoning:** The content provided does not contain any information about omega-3 supplementation or its effects on blood pressure or cardiovascular health. The claim references a systematic review in 'Frontiers in Nutrition,' but the content provided is from 'Frontiers in Pharmacology,' which does not mention omega-3 or blood pressure. Therefore, the claim is not supported by the content given.

## Supported Claims (sample)

- **The standard recommended dosage for creatine supplementation...** → SUPPORTED (evidence: "evidence-based research shows that creatine supplementation is relatively well t...")
- **An alternative approach to creatine supplementation includes...** → SUPPORTED (evidence: "resistance-trained males who received creatine at a dose of 0.3 g/kg lean body m...")
- **Higher loading doses of creatine lead to rapid increases in ...** → SUPPORTED (evidence: "A creatine loading phase uses 20 to 25 grams daily for 5 to 7 days to quickly sa...")
