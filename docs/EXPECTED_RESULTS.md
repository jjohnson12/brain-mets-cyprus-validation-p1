# Manuscript-locked expected results

These values are validation targets for the audited Cyprus/PROTEAS v3 analysis. Scripts with built-in assertions should stop rather than silently continue when the reconstructed cohort differs.

## Denominators

| Analysis stage | Expected value |
|---|---:|
| Postcontrast T1-weighted studies | 186 |
| Studies with a corresponding source reference mask | 170 |
| Studies without a source reference mask | 16 |
| Edema-only, zero-tumor-core reference studies | 1 |
| Tumor-core-positive cross-sectional studies | 169 |
| Underlying patients | 40 |
| Longitudinal case units | 45 |
| Consecutive available-reference pairs | 125 |
| Strictly adjacent pairs | 116 |

## Cross-sectional endpoint

- Reference lesions: **413**
- Detected: **285**
- Missed: **128**
- False-positive predicted components: **267**, or **1.580 per study**
- Median per-study DSC: **0.7771** (reported as 0.78)

| Reference lesion volume | Detected / total | Sensitivity | Patient-clustered percentile 95% CI |
|---|---:|---:|---:|
| <0.05 mL | 9 / 54 | 16.7% | 7.3%–29.7% |
| 0.05–<0.5 mL | 128 / 193 | 66.3% | 57.5%–73.5% |
| 0.5–<4 mL | 94 / 107 | 87.9% | 79.0%–95.8% |
| ≥4 mL | 54 / 59 | 91.5% | 78.3%–100.0% |

Additional clustered intervals:

- False positives per study: **1.580** (95% CI, **1.222–1.988**)
- Median per-study DSC: **0.7771** (95% CI, **0.7012–0.8484**)

Bootstrap configuration: **10,000** patient-level resamples; seed **20260809**.

## Reference-anchored index-target endpoint

Eligibility flow:

- Measurable index reference lesions: **54**
- Insufficient positive-reference follow-up: **6**
- No matched model track: **4**
- Entry detection failure: **1**
- Matched, evaluable primary trajectories: **43**

Contingency rows are reference categories and columns are model categories in the order improved, stable, progressed:

```text
[[18, 2, 1],
 [ 4, 7, 2],
 [ 0, 0, 9]]
```

Overall agreement: **34/43 = 79.1%** (Wilson 95% CI, **64.8%–88.6%**).

## Whole-burden longitudinal endpoint

- Available-reference pairs: **92/125 = 73.6%** category agreement
- Median absolute total-volume-change error: **0.471 mL** (IQR, **0.157–1.585 mL**)
- Strictly adjacent sensitivity analysis: **87/116 = 75.0%** agreement
- Strictly adjacent median absolute error: **0.516 mL** (IQR, **0.178–1.665 mL**)

## Interpretation boundary

The modified RANO-BM analysis is a lesion-level imaging-response analogue. It is not formal patient-level trial RANO-BM, and the analyses do not establish autonomous clinical performance.
