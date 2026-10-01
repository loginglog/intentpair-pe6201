# Evaluation Guide

## Evaluation question

The primary question is whether a low-cost text-only LLM can classify keyword pairs as Same, Different, or Uncertain more accurately than simple baselines while reducing manual SEO review. The evaluation also tests whether confidence can route difficult cases to a writer and whether the resulting labor saving remains positive after error costs.

## Protocol

The dataset contains 140 frozen keyword pairs. Twenty pairs form the development set and 120 form the held-out test set. Development data supplies few-shot examples and selects baseline and confidence thresholds. The test set is used for the final reported score.

SERP results are excluded from every LLM prompt. They are used only to compute the frozen ground-truth label:

- overlap = 0 → Different;
- 0 < overlap < 0.3 → Uncertain;
- overlap ≥ 0.3 → Same.

This separation prevents direct leakage from the scoring evidence into the classifier.

## Systems evaluated

1. **Majority baseline:** always predicts the most frequent development class.
2. **Lexical baseline:** synonym-normalised token Jaccard with thresholds selected on development data.
3. **LLM v1:** directly predicts Same, Different, or Uncertain and returns a forced best guess.
4. **LLM v2:** predicts Same or Different plus confidence. The system maps confidence below the development-selected threshold of 0.85 to Uncertain.

V1 predicted Uncertain zero times. V2 was introduced to make abstention thresholded and measurable rather than dependent on the model voluntarily selecting an uncertainty class.

## Metrics and targets

The primary metric is three-class macro-F1 because all three classes matter even when their counts differ. Per-class support and metrics are reported before interpreting the aggregate.

| Measure | Target | Actual |
|---|---:|---:|
| Three-class macro-F1 | At least 0.700 | 0.438 |
| Gain over stronger baseline | At least +0.100 | −0.153 |
| Manual-review rate | At most 25% | 37.5% |
| Manual effort per 100 pairs | About 2.1 hours | 3.13 hours |
| Evaluation-run LLM cost | Below US$0.10 | About US$0.05 |

## Headline results

| System | Test macro-F1 |
|---|---:|
| Majority baseline | 0.234 |
| Tuned lexical rule | **0.591** |
| LLM v1 | 0.398 |
| LLM v2 | 0.438 |

V2 per-class results are:

| Class | Precision | Recall | F1 | Support |
|---|---:|---:|---:|---:|
| Different | 0.718 | 0.785 | 0.750 | 65 |
| Uncertain | 0.244 | 0.379 | 0.297 | 29 |
| Same | 1.000 | 0.154 | 0.267 | 26 |

The Same precision of 1.000 is based on four predictions and is not evidence of zero wrong-merge risk.

## Secondary diagnostics

On the 91 test pairs with definite Same or Different ground truth, forced-binary macro-F1 was 0.744 for the LLM and 0.741 for the re-tuned lexical rule. The 0.002 difference is treated as a practical tie.

Among those definite pairs, answered cases had a 3.5% forced-choice error rate, while abstained cases would have been wrong 41.2% of the time. This 11.7-times lift suggests that confidence identifies harder cases. It does not prove that the LLM is better than a calibrated lexical abstention policy, which was not evaluated.

The 20-pair drift audit found four moved labels and overlap MAE of 0.030. No pair moved to or from Same.

## Business evaluation

The ROI section assumes 100 pairs per month, five minutes per manual check, and labor at US$25 per hour. Reviewing only Uncertain predictions reduces estimated work from 8.33 to 3.13 hours and saves US$130.17 before error costs.

Two wrong splits were observed among 57 answered definite cases. Extrapolation gives about 1.67 wrong splits per 100 pairs. At US$75 per wrong split, net saving is about US$5.17; US$78.10 is the break-even error cost. This is a sensitivity analysis, not a forecast.

## Reproduce the evaluation

Run these notebook sections in order:

| Notebook section | Evaluation responsibility | Saved output |
|---|---|---|
| Step 3c | Freeze labels and split | `data/ground_truth.csv` |
| Step 4 | Fit and score baselines | Notebook output |
| Steps 5–6 | Run and score v1 | `results/predictions.csv` |
| Steps 5b–6b | Select v2 threshold on development data and score test | `results/predictions_v2.csv` |
| Step 6c | Binary robustness and abstention diagnostics | Notebook output |
| Step 3d | Repeat the fixed drift sample | `results/drift_results.csv` |
| Step 7 | Time, error-cost, and break-even analysis | Notebook output |

Expected checks after a frozen-cache run:

- test size = 120;
- lexical macro-F1 = 0.591;
- LLM v2 macro-F1 = 0.438;
- v2 review rate = 37.5%;
- drift labels moved = 4/20.

## Evaluation limitations

- Ground-truth Uncertain is a SERP-overlap band, while model confidence is subjective semantic certainty.
- Development `n=20` is small for selecting thresholds.
- The Jaccard-stratified sampling design benefits the lexical baseline.
- The 140-pair dataset gives weak estimates for rare error directions.
- The reasons were not evaluated with users for correctness or persuasiveness.
- No calibrated lexical-rule abstention policy was compared with LLM confidence.
- SERP drift means all headline results are conditional on the frozen snapshot.

The evaluation therefore supports a monitored decision-support pilot, not production automation.

