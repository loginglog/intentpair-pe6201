# IntentPair Business and Technical Trade-off Analysis

**Jiayao Ge · Section B · End-of-Course Project**

**Initial SERP snapshot 2026-09-26 · Submission-week drift audit 2026-09-29**

## 1 Executive decision

IntentPair was designed to help an SEO writer decide whether two similar keywords should be served by one page or two. The project tested whether a low-cost LLM could replace most five-minute manual SERP comparisons while routing uncertain cases to a writer.

The system should **not** be deployed as an unconditional classifier. It missed both its pre-specified technical target and its business target:

| Measure | Target in the Problem Statement | Actual result | Outcome |
|---|---:|---:|---|
| Three-class macro-F1 | At least 0.700 | 0.438 | Not met |
| Improvement over the stronger baseline | At least +0.100 | −0.153 versus the lexical rule | Not met |
| Manual-review rate | At most 25% | 37.5% | Not met |
| Manual effort per 100 pairs | About 2.1 hours | 3.13 hours | Not met |
| Estimated evaluation-run LLM cost | Below $0.10 | About $0.05 | Met |

The tuned lexical rule scored 0.591 macro-F1, compared with 0.438 for the final LLM design. The LLM nevertheless produced one potentially useful signal: among the 91 test pairs with definite ground truth, its answered cases had a 3.5% forced-choice error rate, while its abstained cases would have been wrong 41.2% of the time. This 11.7× error lift supports a limited selective-review pilot, but not automatic production deployment.

My decision is therefore to retain IntentPair as an experimental decision-support tool. A safer pilot should automatically accept only high-confidence Different decisions, require writer confirmation for every Same decision, and route Uncertain decisions to full manual review. Deployment should proceed only if a new holdout confirms the error rate and measured labor savings remain positive after content-error costs.

## 2 Business problem and current workflow

The primary user is an SEO writer in a B2B SaaS business without a dedicated SERP-clustering budget. The writer reviews approximately 40 pre-grouped keyword sets, represented in this analysis as about 100 pair decisions per month. For each pair, the writer must decide whether one page can rank for both keywords or whether separate pages are required.

The current process is a manual Google comparison:

1. Search both keywords using the target locale and language.
2. Compare the top-ranking URLs and infer whether Google treats the intents as equivalent.
3. Decide whether to merge the keywords into one page or split them across two pages.

At five minutes per pair, 100 decisions require approximately 8.33 hours each month. At the assumed labor rate of $25 per hour, the direct review cost is $208.33 per month. The larger cost is a wrong content decision:

- A **wrong merge** places two different intents on one page. It can require a rewrite and may lose ranking opportunities. This is the higher-risk, less visible error.
- A **wrong split** creates an unnecessary article and may cause keyword cannibalisation. Its production cost is easier to observe and estimate.

IntentPair attempts to reduce this effort by predicting Same, Different, or Uncertain from keyword text alone and returning a short reason. SERP data is used to construct evaluation labels but is excluded from the LLM prompt to prevent leakage. Whole-site clustering, non-English keywords, content generation, search-volume forecasting, and production deployment are outside the project scope.

## 3 Manual vs buy vs build options

The business decision is not simply whether the LLM works. It is whether building IntentPair is preferable to the available manual, commercial, and simpler coded alternatives.

| Option | Direct cost | Human effort | Main benefit | Main trade-off |
|---|---:|---:|---|---|
| Manual Google review | About $208.33 in labor per 100 pairs | 8.33 hours | Uses current SERPs and human context | Slow, repetitive, and dependent on individual judgement |
| Buy a commercial SERP-clustering tool | Subscription cost not evaluated | Low after setup | Mature workflow using live ranking data | Recurring vendor cost, less control, and product features were not independently benchmarked in this project |
| Build the lexical rule | Negligible compute cost | Depends on the review policy | Best primary macro-F1 in this experiment and fully transparent logic | No calibrated abstention signal was tested; performance benefits from the Jaccard-based sampling design |
| Build IntentPair with an LLM | About $0.04 inference per 100 pairs under the project estimate | 3.13 hours under the evaluated policy | Produces a reason and identifies a subset of harder cases | Lower primary macro-F1 than the rule, third-party API dependency, and error cost can remove the labor saving |

The experiment does not establish that the LLM is better than the lexical rule. On the binary Same versus Different view, they were practically tied: 0.744 macro-F1 for the LLM and 0.741 for the rule. The LLM's only demonstrated advantage was its selective confidence signal. However, I did not calibrate an equivalent distance-from-threshold or abstention mechanism for the lexical rule, so that advantage remains provisional.

This result changes the original answer to "Why AI?" AI is not justified by classification accuracy because the simpler rule performed better on the primary metric. Continued use of the LLM would be justified only if a further pilot confirms that its confidence and explanations improve human review beyond what a calibrated rule can provide.

The build-versus-buy conclusion is therefore conditional. A commercial live-SERP tool is likely the stronger operational choice when the organisation can justify its subscription and needs fresh ranking evidence. A simple lexical rule is the strongest low-cost technical baseline. IntentPair is justified only as a small pilot when explainable pair-level triage is valuable and the organisation accepts the validation and monitoring burden.

## 4 Business targets and actual outcomes

### Workflow and labor outcome

The evaluated v2 policy sends every Uncertain prediction to a writer and accepts every non-Uncertain prediction automatically. It abstained on 45 of 120 test pairs, giving a 37.5% review rate.

| Policy | Cases sent to a writer | Review hours per 100 pairs | Hours saved | Labor and LLM cost per month | Saving before error costs |
|---|---:|---:|---:|---:|---:|
| All-manual process | 100% | 8.33 | 0.00 | $208.33 | $0.00 |
| Evaluated policy: review Uncertain only | 37.5% | 3.13 | 5.21 | $78.17 | $130.17 |
| Safer pilot: review Uncertain and confirm every Same | 40.8% | 3.40 | 4.93 | $85.11 | $123.22 |

The safer policy reflects the Problem Statement's responsible-use commitment to writer confirmation for every merge. The calculation conservatively assigns the full five-minute review time to Same confirmations. A production pilot should measure the actual confirmation time rather than assume it.

### Error-aware return on investment

The time-only saving is an upper bound because automated errors have business costs. On the evaluated test set, the system produced two definite errors among 57 answered pairs, both wrong splits. Applied to 100 monthly pairs using the observed coverage and class mix, this corresponds to approximately 1.67 wrong splits per month.

| Assumed cost of one wrong split | Monthly error cost | Net monthly saving under the evaluated policy |
|---:|---:|---:|
| $25 | $41.67 | +$88.50 |
| $50 | $83.33 | +$46.83 |
| $75 | $125.00 | +$5.17 |
| $78.10 | $130.17 | Break-even |
| $100 | $166.67 | −$36.50 |
| $150 | $250.00 | −$119.83 |

At a $75 article cost, the evaluated policy saves only about $5 per month. Increasing volume does not solve this problem because labor savings and expected error costs both scale linearly with the number of pairs. Business viability depends mainly on reducing the error rate or limiting the cost of an incorrect content decision.

The error-cost estimate is itself uncertain. It extrapolates from two observed errors in a small test set and excludes lost traffic, management time, and the opportunity cost of delayed content. It should be treated as a sensitivity analysis, not as a forecast.

### Business outcome

The project demonstrated a reduction in expected review effort, but it did not achieve the promised maximum of 25 manual checks per 100 pairs. It also did not establish that the LLM creates more value than the lexical rule. The appropriate business outcome is therefore a limited pilot decision, not a deployment decision.

## 5 Technical design and evaluation

### System design

The implementation contains the following components:

- 82 B2B SaaS seed keywords.
- 25 adversarial variants generated by a model different from the classifier.
- A seeded Python pair generator producing 140 pairs: 20 development and 120 held-out test pairs.
- A frozen Serper.dev snapshot of Google top-10 organic results at `gl=sg` and `hl=en`.
- A lexical baseline using synonym normalisation and token Jaccard.
- An LLM classifier using `openai/gpt-4o-mini` through OpenRouter.
- Frozen caches, structured outputs, per-class metrics, confusion matrices, and ROI calculations.

I own the pair generator, thresholds, prompt design, evaluation logic, logging, and business analysis. I rent LLM inference from OpenRouter and SERP collection from Serper.dev. This keeps implementation small and inexpensive, but transfers availability, pricing, and version-control risks to third parties.

The final implemented label rule was:

- overlap = 0 → Different
- 0 < overlap < 0.3 → Uncertain
- overlap ≥ 0.3 → Same

This differs from the Week 3 Problem Statement, which described 0–1 shared URLs as Different and exactly two as Uncertain. The final rule treats any positive overlap below 0.3 as Uncertain. It was frozen before LLM evaluation and increases manual review in exchange for a more cautious boundary around low-overlap cases.

### Leakage control and evaluation protocol

SERP results never enter the LLM prompt. Labels and thresholds were frozen before LLM evaluation. The 20 development pairs were used for few-shot examples and threshold selection; the 120 test pairs were scored once. The majority and lexical baselines were evaluated on the same held-out test set, with baseline thresholds selected on development data.

The v1 prompt allowed Same, Different, or Uncertain, but the model predicted Uncertain zero times on both development and test data. The v2 design therefore forced a Same or Different choice and returned a separate confidence value. The system converted predictions below the development-selected threshold of 0.85 to Uncertain.

### Evaluation results

| System | Three-class macro-F1 on test n=120 |
|---|---:|
| Majority-class baseline | 0.234 |
| Tuned lexical rule | **0.591** |
| LLM v1 | 0.398 |
| LLM v2 | 0.438 |
| Pre-specified target | 0.700 |

The v2 per-class results were:

| Class | Precision | Recall | F1 | Support |
|---|---:|---:|---:|---:|
| Different | 0.718 | 0.785 | 0.750 | 65 |
| Uncertain | 0.244 | 0.379 | 0.297 | 29 |
| Same | 1.000 | 0.154 | 0.267 | 26 |

The Same precision of 1.000 is based on only four Same predictions. It is insufficient evidence of zero wrong-merge risk, particularly because Same recall was only 0.154.

On the 91 pairs with definite Same or Different ground truth, the LLM's forced binary macro-F1 was 0.744 and the re-tuned lexical rule scored 0.741. This 0.002 difference should be treated as a practical tie, not an LLM win.

The code, frozen assets, results, and notebook are available in the GitHub repository: `https://github.com/loginglog/intentpair-pe6201`.

## 6 Technical trade-offs and limitations

| Technical decision | Benefit | Cost or limitation |
|---|---|---|
| SERP overlap as ground truth | Operationally relevant and independent of my subjective label | Time-sensitive proxy; the Uncertain class partly reflects a measurement band rather than a semantic category |
| Text-only LLM prompt | Cheap and avoids label leakage | Cannot directly observe current ranking behaviour or exact URL overlap |
| Compact LLM at temperature 0 | Low estimated inference cost and reduced run-to-run variation | Model alias and provider behaviour may change; frozen caches reproduce reported outputs but do not guarantee identical fresh calls |
| Development-only threshold selection | Protects the held-out headline result | Development n=20 is small and confidence values are coarse |
| Adversarial variants from a different model | Tests difficult, non-obvious cases | Adversarial test macro-F1 was only 0.181, and the subgroup contained just 23 test pairs |
| Jaccard-stratified pair sampling | Ensures the dataset contains both similar and dissimilar pairs | Gives the lexical baseline a structural advantage because its feature resembles the sampling variable |
| 140-pair scope | Fits the course timeline and free SERP quota | Confidence intervals are wide and rare error directions are poorly estimated |
| One-line LLM reason | Potentially supports writer trust and review | Reason quality and persuasiveness were not evaluated by users |

The most important construct limitation concerns Uncertain. Ground-truth Uncertain means that observed SERP overlap fell inside a numeric band, while model confidence represents subjective certainty about semantic intent. These are different quantities. The v2 confidence threshold improved Uncertain F1 from 0 to 0.297 but reduced Same recall to 0.154. Testing other thresholds did not close the gap with the lexical baseline within the evaluated grid.

The submission-week drift audit provides a second limitation. On 2026-09-29, the system re-collected the 30 unique keywords contained in a fixed 20-pair audit sample. Four of 20 labels changed and overlap MAE was 0.030. Three moved from Uncertain to Different and one from Different to Uncertain; none moved to or from Same. The original labels remain frozen, so all evaluation and ROI results should be interpreted as specific to the original snapshot.

## 7 Operational risks and governance

| Risk | Business consequence | Control for a pilot |
|---|---|---|
| Wrong merge | Missed ranking opportunity, rewrite cost, and difficult-to-detect content failure | Require writer confirmation for every Same decision; monitor Same precision and recall |
| Wrong split | Redundant article cost and cannibalisation | Track downstream article creation and rework; include error cost in ROI |
| SERP and label drift | Decisions and measured performance may change over time | Repeat a fixed drift sample monthly and investigate movement involving Same |
| Model or provider change | Fresh predictions may differ from the evaluated system | Version prompts, log the provider and model alias, preserve frozen responses, and revalidate before upgrades |
| API availability and pricing | Workflow interruption or changing operating cost | Retain the lexical rule and manual process as fallbacks; monitor cost per pair |
| Misleading explanation | A fluent reason may persuade a writer to accept a wrong decision | Present reasons as supporting evidence rather than proof; sample and review reason quality before deployment |
| Automation bias | Writers may stop challenging confident outputs | Display confidence and source limitations; retain accountable human ownership of merge decisions |

For a pilot, the SEO writer remains the decision owner. The system may prioritise work and automate low-risk routing, but it should not silently publish content or merge keyword groups. Logs should retain the input pair, model and prompt version, confidence, prediction, writer override, and final page decision. These records are necessary to measure actual time savings, override rates, wrong merges, wrong splits, and model drift.

## 8 Final recommendation

IntentPair should proceed only as a small, monitored decision-support pilot. It should not replace manual SEO judgement or be presented as superior to the lexical rule.

The pilot workflow should be:

1. Run the lexical rule and LLM on each candidate pair.
2. Automatically accept only high-confidence Different decisions under a documented threshold.
3. Require writer confirmation for every Same decision.
4. Send every Uncertain decision to full SERP review.
5. Record writer overrides, review time, article outcomes, and error costs.

Before wider deployment, the project should meet four gates on a new holdout and real workflow sample:

- Demonstrate that selective automation reduces measured review time, not only estimated time.
- Evaluate a calibrated abstention policy for the lexical rule and compare it fairly with LLM confidence.
- Establish an acceptable upper bound for wrong-merge risk with substantially more Same predictions.
- Show positive error-aware ROI under the organisation's actual content-production cost.

If these gates are not met, the organisation should use either the lexical rule as a transparent prioritisation aid or continue manual review. The current evidence supports learning and a controlled pilot, but not production automation.
