# IntentPair — one page or two?

PE6201 End-of-Course Project by Jiayao Ge (Section B).

IntentPair evaluates whether an LLM can decide, from two keyword strings alone, whether an SEO team should target them with one page or two. Google top-10 SERP overlap is used only to create frozen evaluation labels and never enters the LLM prompt.

## Headline results

| System | Three-class macro-F1 (test n=120) |
|---|---:|
| Majority-class baseline | 0.234 |
| Tuned lexical-rule baseline | **0.591** |
| LLM v1 | 0.398 |
| LLM v2 with confidence-threshold abstention | 0.438 |

The pre-specified target of 0.700 was not met. The most useful observed signal was selective: among the 91 pairs with definite ground truth, answered pairs had a 3.5% forced-choice error rate, compared with 41.2% among abstained pairs.

## Submission-week drift audit

The SERPs for a fixed 20-pair audit sample were collected again on 2026-09-29. Four labels moved (20%) and overlap MAE was 0.030. All movements were between Different and Uncertain; none moved to or from Same. The frozen labels used for evaluation were not modified.

## Repository contents

```text
data/
  adv_pairs.json
  drift_snapshot.json
  ground_truth.csv
  llm_cache.json
  pairs.csv
  serp_meta.json
  serp_snapshot.json
notebooks/
  IntentPair_Colab_English.ipynb
results/
  drift_results.csv
  predictions.csv
  predictions_v2.csv
REPORT.md
requirements.txt
```

## Reproduce the analysis

The notebook contains the complete pipeline and saved outputs. Frozen SERP and LLM caches are included so the reported analysis can be reproduced without making new paid API calls.

1. Clone this repository or download it as a ZIP.
2. Open `notebooks/IntentPair_Colab_English.ipynb` in Google Colab.
3. Place the repository's `data/` and `results/` folders in `MyDrive/IntentPair/`, or set `USE_DRIVE = False` and run from the repository root.
4. Run the cells from top to bottom.

Serper and OpenRouter credentials are needed only when a required cache entry is absent or when deliberately collecting a fresh snapshot. Store credentials in Colab Secrets as `SERPER_API_KEY` and `OPENROUTER_API_KEY`; never commit them to Git.

## Evaluation protocol

- 82 real B2B SaaS seed keywords and 25 adversarial variants.
- 140 keyword pairs, split into 20 development and 120 held-out test pairs.
- Labels frozen before LLM evaluation: overlap 0 = Different; overlap between 0 and 0.3 = Uncertain; overlap at least 0.3 = Same.
- LLM prompt receives keyword text only; SERP data is excluded to prevent leakage.
- Primary metric: three-class macro-F1, with per-class support reported first.

See `REPORT.md` for the business and technical trade-off analysis.
