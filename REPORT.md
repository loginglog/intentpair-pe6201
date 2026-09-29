# IntentPair — Business and Technical Trade-off Analysis

**Jiayao Ge · PE6201 Section B · End-of-Course Project · SERP snapshot 2026-09-26 · submission-week drift audit 2026-09-29**

## 0. Headline

I built IntentPair to test one claim: that an LLM, given only two keyword strings, can decide
whether Google would serve them with largely the same top-10 results, and therefore whether an
SEO team should write one page or two.

**The claim failed on my primary metric, which was pre-specified in the Problem Statement.**

| System | 3-class macro-F1 (n=120) |
|---|---|
| B1 — majority class | 0.234 |
| **B2 — tuned lexical rule** | **0.591** |
| LLM v1 — model self-declares Uncertain | 0.398 |
| LLM v2 — forced binary + confidence threshold | 0.438 |
| *Target committed in my Problem Statement* | *0.700* |

A ten-line token-overlap rule beats the selected compact LLM by 0.153 macro-F1. I did not hit my target and I
am not going to restate the target to make it look like I did.

What the experiment *did* produce is a usable result of a different kind. The LLM's **confidence is
highly informative even though its labels are not**: on the 91 pairs with an unambiguous ground
truth, the pairs it chose to answer were **96.5% correct**, while the pairs it chose to abstain on
would have been **41.2% wrong** had I forced an answer — an **11.7× error lift**. That makes
IntentPair potentially useful as a selective triage system rather than a fully automatic
classifier. Section 5 shows that its economic viability depends on the assumed cost of an error.

---

## 1. What the system does

```
two keyword strings  →  one LLM call  →  {Same | Different | Uncertain} + confidence + one-line reason
```

No SERP data, search volume, or ranking feature ever enters the prompt. That is a deliberate
constraint: SERP overlap *is* my ground truth, so letting it reach the model would make the
evaluation meaningless. The model sees exactly what a content strategist sees when they look at a
keyword spreadsheet — the words themselves.

## 2. Data and ground truth

| | |
|---|---|
| Seed keywords | **82**, taken from my employer's live SEO keyword list (B2B lead-generation SaaS: "email finder", "linkedin scraper", …). Generic product-category terms; no confidential or personal data. |
| Adversarial keywords | **25 variants** authored by `anthropic/claude-haiku-4.5`; 23 appeared in the held-out test set. The generator is deliberately different from the classifier, so the test set is not written by the system under test. |
| Evaluation pairs | **140** — stratified by token-set Jaccard (45 high ≥0.5 / 40 mid / 30 low) + 25 adversarial. `seed=42`. |
| Ground truth | Serper.dev Google top-10 organic URLs (`gl=sg`, `hl=en`), normalised (host minus `www`, path minus trailing slash). Overlap = \|A∩B\| / min(\|A\|,\|B\|). |
| Label rule (frozen *before* any LLM call) | `overlap = 0 → Different` · `0 < overlap < 0.3 → Uncertain` · `overlap ≥ 0.3 → Same` |
| Class counts | all 140: Different 76 / Uncertain 34 / Same 30 — test (n=120): **65 / 29 / 26** — dev n=20 |

**On the thresholds.** `Different = 0` because sharing zero of ten URLs is not a borderline call —
Google is stating the intents differ. `Same ≥ 0.3` because three shared URLs out of ten is the
conventional cut-off in commercial SERP-clustering tools. I committed to these values on the first
snapshot, and when a runtime loss forced me to re-pull the SERPs a month later I applied the *same*
two numbers to the new distribution rather than re-reading the histogram. Threshold shopping after
seeing results is the single easiest way to fake a good number in this kind of study; not doing it
is why the 0.591 below is allowed to embarrass me.

**Submission-week drift check.** On 2026-09-29, I re-collected the 30 unique keywords contained in
the same 20-pair audit sample, using the original `gl=sg` and `hl=en` settings. Four of the 20 labels
changed (20%), while the mean absolute change in SERP overlap was 0.030. Three pairs moved from
Uncertain to Different as their overlap fell from 0.1 to 0.0, and one moved from Different to
Uncertain as its overlap rose from 0.0 to 0.1. No sampled pair moved to or from Same. The observed
drift was therefore confined to the zero-overlap boundary separating Different from the Uncertain
measurement band. I retain the original frozen labels for evaluation and treat this as a
temporal-sensitivity audit rather than a relabelling step. The three-class scores and
abstention-based ROI should consequently be interpreted as snapshot-specific.

## 3. Why the LLM lost — and why it is not a tuning problem

### 3.1 v1: the model never abstains

Asked to output `Same | Different | Uncertain`, `gpt-4o-mini` chose **Uncertain 0 times out of 120**.
That behaviour forces the Uncertain-class F1 to zero and substantially depresses the three-class
macro-F1. I verified the non-abstention on the **dev** split first, so the decision to redesign was
not informed by test data.

### 3.2 v2: make abstention the system's decision, not the model's

v2 forces a binary commitment plus a separate confidence number, and the *system* overrides it to
`Uncertain` when `confidence < τ`. τ was grid-searched on dev only (τ = 0.85) and test was scored
once. It worked mechanically — Uncertain F1 went **0.000 → 0.297** — but macro-F1 rose only to
0.438, because the same threshold destroyed the Same class:

```
v2, τ=0.85, test n=120
              precision  recall  f1
Different         0.718   0.785  0.750
Uncertain         0.244   0.379  0.297
Same              1.000   0.154  0.267     ← 20 of 26 true-Same pairs were abstained away
```

### 3.3 Threshold choice does not rescue performance within the tested grid

If the problem were a badly chosen τ on a small dev set, some other τ would rescue it. It does not:

```
      τ     dev F1   test F1
  0.55–0.80  0.475    0.446
  0.85–0.90  0.630    0.438   ← selected on dev
  0.95       0.133    0.130

  dev-selected τ=0.85 → test 0.438
  oracle       τ=0.55 → test 0.446   (post-hoc; not usable as a result)
```

The best post-hoc τ in this grid scores **0.446** — still 0.145 below the lexical rule and 0.254
below my target. It is only 0.008 above the dev-selected result. This diagnostic cannot replace the
pre-specified result, but it suggests that threshold selection was not the main source of failure.

### 3.4 The real cause: construct mismatch

Ground-truth `Uncertain` means *"the SERP overlap landed in the measurement band (0, 0.3)"*. It is a
statement about **my instrument**, not about the keywords. Model confidence means *"how sure am I
about the intent"*. These are different quantities. Keyword surface form provides, at best, indirect evidence about
whether the observed URL count will fall inside this narrow band. I asked the model to predict a
property of my measuring device and then scored it as if I had asked only about intent.

This suggests a broader evaluation risk: when a benchmark derives classes by thresholding a
continuous proxy, a middle class may partly reflect the measurement rule rather than a distinct
semantic construct.

### 3.5 An honest discount on the baseline's win

I sampled pairs by stratifying on token-set Jaccard, and the lexical rule's only feature *is* token
Jaccard. Over all 140 pairs, Spearman(jac_syn, serp_overlap) = **0.471** (p = 4.2e-9), and mean
`jac_syn` rises monotonically across classes (Different 0.255 → Uncertain 0.367 → Same 0.544). The
rule therefore enjoys a structural advantage: its input shares a source with my sampling variable.

The moderate positive association confirms that the lexical baseline benefits from the sampling
design. It softens the interpretation of the margin, although the observed gap remains substantial.

## 4. What the LLM is actually good at

### 4.1 On labels: practically tied with the rule in this sample

On the binary task (the 91 pairs whose truth is not the measurement band), with the rule's threshold
re-tuned on dev for fairness:

| System | binary macro-F1 (n=91) |
|---|---|
| B1 majority | 0.417 |
| B2 lexical rule (`jac_syn ≥ 0.55`) | 0.741 |
| LLM v2 forced binary | **0.744** |

A 0.002 gap on 91 samples is a tie. **The LLM buys no accuracy over ten lines of Python.** I report
this as a tie and not as a win, and I note that the binary view is a *secondary* analysis — my
pre-specified primary metric is the three-class macro-F1 in Section 0, which failed.

### 4.2 On knowing when to stop: a 11.7× error lift

| | n | error rate |
|---|---|---|
| Pairs the system answered | 57 | **3.5%** |
| Pairs the system abstained on (forced-guess error) | 34 | **41.2%** |

The confidence signal separates easy from hard cases by a factor of **11.7**. Mean confidence was
0.873 on correct calls versus 0.812 on incorrect ones — a small gap, but the model's confidence
output is lumpy enough that a single cut at 0.85 separates them sharply.

I did not implement or calibrate a comparable abstention signal for the lexical rule. Raw Jaccard
was used as a prediction feature, not validated as confidence, so whether the rule can reproduce
the LLM's selective-prediction lift remains an open question. In this experiment, confidence-based
triage is the LLM's main observed advantage.

### 4.3 The expensive error never happened

In SEO, the two error directions are not symmetric:

- **Wrong merge** (truth Different, predicted Same) → two intents crammed onto one page, which
  ranks for neither. Costs a rewrite plus lost traffic, and is hard to detect because the page
  looks fine.
- **Wrong split** (truth Same, predicted Different) → one redundant article. Costs its production
  budget and risks cannibalisation.

**Wrong merges: 0 out of 65.** However, v2 predicted Same only four times, so its observed precision
of **1.000 (4/4)** is too small a sample to establish zero deployment risk. The definite errors
observed in this test were wrong splits rather than wrong merges.

### 4.4 The adversarial set earned its keep

| pair source | n | v2 macro-F1 |
|---|---|---|
| adversarial (written by a different model) | 23 | **0.181** |
| stratified sample from real keywords | 97 | 0.365 |

Performance halves on the adversarial pairs. The generator found genuine failure modes rather than
padding the set with easy cases — which is the whole point of having a *different* model write them.

## 5. Business case: it breaks even, and volume will not save it

Assumptions, stated so they can be attacked: junior SEO writer at **$25/hr** (SE-Asia market rate),
**5 min** to check one pair manually in Google, **100 pairs/month**.

| | |
|---|---|
| All-manual today | 8.33 hr/mo → **$208.33** |
| With IntentPair (37.5% deferred to a human) | 3.13 hr/mo → **$78.13** |
| **Labour saved** | 5.21 hr/mo → **$130.21** |
| LLM inference (`gpt-4o-mini`, ≈$0.0004/pair) | **$0.04/mo** — negligible |
| Expected wrong splits on auto-approved pairs | 62.5 × 76% definite × 3.5% ≈ **1.67/mo** |

Inference cost is a rounding error. **Error cost is not:**

| Cost of one redundant article | Error cost/mo | **Net saving/mo** |
|---|---|---|
| $25 | $41.67 | **+$88.50** |
| $50 | $83.33 | +$46.83 |
| $75 | $125.00 | +$5.17 |
| **$78.10** | $130.17 | **$0 — break-even** |
| $100 | $166.67 | −$36.50 |
| $150 | $250.00 | −$119.83 |

A redundant article at $75 — three hours of a $25/hr writer — puts the system at **+$5/month**. That
is zero.

**And scaling the volume does not help.** Labour saved and expected error cost are both linear in
pairs/month, so the break-even article cost of **$78** is volume-independent. Doubling to 200
pairs/month doubles both sides and the sign never changes. This is the part of the analysis I would
have got wrong if I had stopped at "5.2 hours and $130 saved": the naive ROI is positive only
because it silently prices errors at zero.

**What would actually change the sign**, in order of expected value:

1. **Test agreement gating.** Auto-approve only where the LLM and lexical rule agree, then measure
   coverage and error on a fresh holdout. The current experiment does not establish whether their
   errors are sufficiently independent for this to help.
2. **Asymmetric thresholds.** The Same class is where the cost sits, and v2's Same precision is
   already 1.000. Requiring high confidence only for Same→Different flips would keep recall without
   adding wrong splits.
3. **Cheaper pages before cheaper labour.** The economics are dominated by production cost, not
   review cost. A team on $25 templated pages gets $89/month; a team on $150 long-form loses money
   and should not deploy this at all.

**My recommendation.** Do not deploy IntentPair as an unconditional classifier. Treat it as
**selective decision support**: auto-approve high-confidence cases and route the remaining 37.5%
to manual review. This preserves the measured labor-saving mechanism, but the small test set is not
sufficient to claim zero wrong-merge risk.

## 6. Technical trade-offs I chose, and what I gave up

| Decision | Why | What it cost |
|---|---|---|
| SERP overlap as ground truth | Operationally relevant, programmatically obtainable, and independent of my own judgement. I did not identify a suitable public keyword-pair intent dataset for this task. | Introduced the measurement band that broke the task (§3.4). A submission-week audit found that 4/20 labels moved, all across the Different–Uncertain boundary; no movement to or from Same was observed. |
| `gpt-4o-mini`, temperature 0 | Approximately $0.0004/pair; temperature 0 reduces run-to-run variation, while frozen caches preserve the reported outputs. | A stronger model might close the 0.153 gap, but selecting one after observing this test set would require a new holdout for a fair comparison. |
| Thresholds and labels frozen before any LLM call | The only defence against retrofitting thresholds to results. | Forced me to report a failed target. |
| τ tuned on dev only, test scored once | Prevents the headline number from being an optimistic artefact. | The post-hoc test-grid optimum was 0.008 higher (§3.3), suggesting limited threshold-selection loss within this grid. |
| Adversarial cases by a different model | Stops the system grading its own homework. | Halved measured performance (§4.4). Correctly. |
| 140 pairs | Fits a free Serper tier (107 unique keyword queries, reused across pairs) and a course timeline. | dev n=20 is thin; wide CIs. The 0.002 binary gap is not resolvable at this sample size. |
| Pairs stratified on token Jaccard | Guarantees hard and easy cases instead of 3,321 mostly-unrelated pairs. | Handed the lexical baseline a structural advantage (§3.5). Random sampling would have been fairer to the LLM but would have produced almost no Same pairs. |

## 7. What I would do next

1. **Replace the three-class target with selective-prediction metrics** — risk–coverage curve, and
   accuracy at fixed coverage. That is what the system is actually for, and unlike macro-F1 over a
   thresholded proxy it is a construct the data can support. (Stated as redesign for a *next*
   study, not as a post-hoc swap of this study's metric.)
2. **Agreement gating** (§5.1) and an error-aware ROI re-run.
3. **Regress overlap as a continuous variable** instead of classifying bands — removes the artefact
   entirely and lets the model be scored on something keyword text might plausibly predict.
4. **Repeat the snapshot weekly for a month** to turn the single drift figure into a variance
   estimate.

## 8. Reproducing this

All frozen assets are committed: `data/serp_snapshot.json` (+ `serp_meta.json` with the UTC pull
time), `data/drift_snapshot.json`, `data/adv_pairs.json`, `data/pairs.csv`,
`data/ground_truth.csv`, `data/llm_cache.json`, `results/drift_results.csv`,
`results/predictions.csv`, and `results/predictions_v2.csv`. With those present the notebook
reproduces every number in this report without spending an API credit; the LLM and SERP caches are
keyed so that re-running cannot silently re-label anything. See `README.md` for the run order.
