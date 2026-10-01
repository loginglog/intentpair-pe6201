# Data Guide

## Purpose and freeze policy

IntentPair uses keyword pairs and frozen Google result snapshots to evaluate whether a text-only classifier can predict Same, Different, or Uncertain search intent. SERP records create evaluation labels only; they are never inserted into the LLM prompt.

The initial snapshot, derived labels, development/test split, and LLM responses are committed so the reported results can be reproduced without new network calls. A fresh SERP collection must be stored separately and must not overwrite the frozen evaluation labels.

## Data sources

- **Seed keywords:** 82 B2B SaaS SEO keywords listed in Step 1 of the notebook.
- **Adversarial variants:** 25 variants created once with `anthropic/claude-haiku-4.5`, a model different from the classifier, then frozen in `data/adv_pairs.json`.
- **SERP evidence:** Google top-10 organic results collected through Serper.dev with Singapore and English fixed as `gl=sg` and `hl=en`.
- **LLM responses:** cached outputs from `openai/gpt-4o-mini` through OpenRouter.

## File catalogue

| File | Records | Purpose |
|---|---:|---|
| `data/adv_pairs.json` | 25 | Base keyword and adversarial variant pairs |
| `data/pairs.csv` | 140 | Reproducibly generated sampled and adversarial pairs |
| `data/serp_snapshot.json` | 107 keywords | Frozen initial top-10 SERP records |
| `data/serp_meta.json` | 1 object | Initial snapshot date, locale, and freeze policy |
| `data/ground_truth.csv` | 140 | Overlap scores, labels, and development/test split |
| `data/llm_cache.json` | 280 entries | Frozen v1 and v2 model responses |
| `data/drift_snapshot.json` | 30 keywords | Fresh SERPs for the fixed 20-pair audit sample |
| `data/drift_meta.json` | 1 object | Drift date, locale, seed, and sample size |
| `results/predictions.csv` | 120 | V1 test predictions and reasons |
| `results/predictions_v2.csv` | 120 | V2 labels, confidence, reasons, and routed result |
| `results/drift_results.csv` | 20 | Old/new overlap and label movement |

The seed list is directly checked into the notebook, while `pairs.csv` contains every pair evaluated.

## Main schemas

### Pair and ground-truth data

| Field | Meaning |
|---|---|
| `pair_id` | Stable integer identifier |
| `kw_a`, `kw_b` | Keyword strings |
| `jaccard` | Raw token-Jaccard similarity |
| `source` | `sampled` or `adversarial` |
| `serp_overlap` | Shared normalised URLs divided by the smaller result-set size |
| `truth` | Different, Uncertain, or Same |
| `split` | `dev` or `test` |

The frozen label rule is:

- overlap = 0 → Different;
- 0 < overlap < 0.3 → Uncertain;
- overlap ≥ 0.3 → Same.

### V1 results

`predictions.csv` adds `jac_syn`, `pred`, `best_guess`, and `reason`. The `best_guess` field supports the forced-choice error analysis for cases where v1 predicts Uncertain.

### V2 results

`predictions_v2.csv` adds `v2_label`, `v2_conf`, `v2_reason`, and `pred_v2`. The final system output `pred_v2` becomes Uncertain when `v2_conf < 0.85`.

### Drift results

`drift_results.csv` contains the frozen overlap and label, the fresh overlap and label, and `label_moved`. It does not change `ground_truth.csv`.

## Provenance and timestamps

The frozen initial SERP artifact was saved on 2026-09-26. Only the date can be supported reliably; the original exact query time was not retained. The earlier notebook logic rewrote `fetched_at` during a cached rerun, so that field could describe notebook execution rather than collection. The corrected `serp_meta.json` records the supported date and the notebook now preserves snapshot creation metadata instead of rewriting it.

The drift audit was collected on 2026-09-29 with random seed 42. It uses 20 fixed pairs containing 30 unique keywords. Four labels moved and overlap MAE was 0.030. All changes were between Different and Uncertain.

## Reproduction rules

1. Use the committed files for the reported evaluation.
2. Run the notebook from top to bottom without deleting the caches.
3. Do not change the label thresholds after inspecting LLM test results.
4. Do not replace `ground_truth.csv` with drift labels.
5. For a new study, write new snapshots and results under new filenames and record a new collection timestamp.
6. Never commit `SERPER_API_KEY` or `OPENROUTER_API_KEY`.

## Known data limitations

SERP overlap is a time-sensitive proxy for search intent. The 140 pairs are too small for narrow confidence intervals, and the 20-pair drift sample provides only a low-cost stability bound. Jaccard-stratified sampling also gives the lexical baseline a structural advantage because the sampling feature resembles its prediction feature. Finally, Uncertain represents an overlap band rather than a directly observed semantic class.

