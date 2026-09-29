# IntentPair Business and Technical Trade-off Analysis

**Jiayao Ge · Section B · End-of-Course Project**

## 1 Executive decision

I designed IntentPair to help an SEO writer decide whether two similar keywords should be served by one page or two. I tested whether a low-cost LLM could replace most five-minute manual SERP comparisons while routing uncertain cases to a writer. Based on my results, I should **not** deploy the system as an unconditional classifier. It missed both my pre-specified technical target and my business target:

| Measure | Target in the Problem Statement | Actual result | Outcome |
|---|---:|---:|---|
| Three-class macro-F1 | At least 0.700 | 0.438 | Not met |
| Improvement over the stronger baseline | At least +0.100 | −0.153 versus the lexical rule | Not met |
| Manual-review rate | At most 25% | 37.5% | Not met |
| Manual effort per 100 pairs | About 2.1 hours | 3.13 hours | Not met |
| Estimated evaluation-run LLM cost | Below $0.10 | About $0.05 | Met |

My tuned lexical rule scored 0.591 macro-F1, compared with 0.438 for my final LLM design. I nevertheless found one potentially useful signal: among the 91 test pairs with definite ground truth, answered cases had a 3.5% forced-choice error rate, while abstained cases would have been wrong 41.2% of the time. I interpret this 11.7× error lift as support for a limited selective-review pilot, but not automatic production deployment.

My decision is therefore to retain IntentPair as an experimental decision-support tool. In a safer pilot, I would automatically accept only high-confidence Different decisions, require writer confirmation for every Same decision, and route Uncertain decisions to full manual review. I would proceed beyond the pilot only if a new holdout confirms the error rate and measured labor savings remain positive after content-error costs.

## 2 Business problem and current workflow

I define the primary user as an SEO writer in a B2B SaaS business without a dedicated SERP-clustering budget. The writer reviews approximately 40 pre-grouped keyword sets, which I represent in this analysis as about 100 pair decisions per month. For each pair, the writer must decide whether one page can rank for both keywords or whether separate pages are required.

The current process is a manual Google comparison:

1. Search both keywords using the target locale and language.
2. Compare the top-ranking URLs and infer whether Google treats the intents as equivalent.
3. Decide whether to merge the keywords into one page or split them across two pages.

At five minutes per pair, 100 decisions require approximately 8.33 hours each month. At the assumed labor rate of $25 per hour, the direct review cost is $208.33 per month. The larger cost is a wrong content decision:

- A **wrong merge** places two different intents on one page. It can require a rewrite and may lose ranking opportunities. This is the higher-risk, less visible error.
- A **wrong split** creates an unnecessary article and may cause keyword cannibalisation. Its production cost is easier to observe and estimate.

I use IntentPair to reduce this effort by predicting Same, Different, or Uncertain from keyword text alone and returning a short reason. I use SERP data to construct evaluation labels but exclude it from the LLM prompt to prevent leakage. I place whole-site clustering, non-English keywords, content generation, search-volume forecasting, and production deployment outside the project scope.

## 3 Manual vs buy vs build options

My business decision is not simply whether the LLM works. I must decide whether building IntentPair is preferable to the available manual, commercial, and simpler coded alternatives.

| Option | Direct cost | Human effort | Main benefit | Main trade-off |
|---|---:|---:|---|---|
| Manual Google review | About $208.33 in labor per 100 pairs | 8.33 hours | Uses current SERPs and human context | Slow, repetitive, and dependent on individual judgement |
| Buy a commercial SERP-clustering tool | Subscription cost not evaluated | Low after setup | Mature workflow using live ranking data | Recurring vendor cost, less control, and product features were not independently benchmarked in this project |
| Build the lexical rule | Negligible compute cost | Depends on the review policy | Best primary macro-F1 in this experiment and fully transparent logic | No calibrated abstention signal was tested; performance benefits from the Jaccard-based sampling design |
| Build IntentPair with an LLM | About $0.04 inference per 100 pairs under the project estimate | 3.13 hours under the evaluated policy | Produces a reason and identifies a subset of harder cases | Lower primary macro-F1 than the rule, third-party API dependency, and error cost can remove the labor saving |

My experiment does not establish that the LLM is better than the lexical rule. On the binary Same versus Different view, they were practically tied: 0.744 macro-F1 for the LLM and 0.741 for the rule. The only LLM advantage I demonstrated was its selective confidence signal. However, I did not calibrate an equivalent distance-from-threshold or abstention mechanism for the lexical rule, so I treat that advantage as provisional.

This result changes my original answer to "Why AI?" I cannot justify AI by classification accuracy because the simpler rule performed better on the primary metric. I would continue using the LLM only if a further pilot confirms that its confidence and explanations improve human review beyond what a calibrated rule can provide.

My build-versus-buy conclusion is therefore conditional. I would prefer a commercial live-SERP tool when the organisation can justify its subscription and needs fresh ranking evidence. I treat the simple lexical rule as the strongest low-cost technical baseline. I would justify IntentPair only as a small pilot when explainable pair-level triage is valuable and the organisation accepts the validation and monitoring burden.

I made the following own-versus-rent decisions across the system stack:

| Layer | Own or rent | Component | Reason for the decision |
|---|---|---|---|
| Interface and serving | Hybrid | I own the notebook-based pair input and result display, while I use Google Colab as the runtime | A notebook is sufficient for the course prototype and avoids the cost of building and hosting a separate application |
| Orchestration | Own | I wrote the Python workflow for pair generation, model calls, threshold routing, caching, and result export | These steps encode the project-specific logic and must remain inspectable and reproducible |
| Model | Rent | I call `openai/gpt-4o-mini` through OpenRouter | Renting a compact model keeps cost and implementation time low while allowing me to compare it with a non-AI baseline |
| Data and retrieval | Hybrid | I own the keyword pairs, frozen labels, and snapshots, while I rent SERP collection from Serper.dev | The sampling and labelling rules define the experiment; live search collection is a commodity service, and the system does not require RAG |
| Evaluation | Own | I wrote the dev/test split, baselines, metrics, threshold selection, drift audit, and ROI analysis | Evaluation choices determine whether the business and technical claims are valid, so I keep them under direct control |
| Observability | Own | I preserve frozen caches, prediction files, model and prompt versions, and drift results | These artefacts let me reproduce the reported results and investigate changes in model behaviour or SERP labels |

## 4 Business targets and actual outcomes

### Workflow and labor outcome

In my evaluated v2 policy, I send every Uncertain prediction to a writer and accept every non-Uncertain prediction automatically. The system abstained on 45 of 120 test pairs, giving me a 37.5% review rate.

| Policy | Cases sent to a writer | Review hours per 100 pairs | Hours saved | Labor and LLM cost per month | Saving before error costs |
|---|---:|---:|---:|---:|---:|
| All-manual process | 100% | 8.33 | 0.00 | $208.33 | $0.00 |
| Evaluated policy: review Uncertain only | 37.5% | 3.13 | 5.21 | $78.17 | $130.17 |
| Safer pilot: review Uncertain and confirm every Same | 40.8% | 3.40 | 4.93 | $85.11 | $123.22 |

My safer policy reflects the Problem Statement's responsible-use commitment to writer confirmation for every merge. I conservatively assign the full five-minute review time to Same confirmations. In a production pilot, I would measure the actual confirmation time rather than assume it.

### Error-aware return on investment

I treat the time-only saving as an upper bound because automated errors have business costs. On my evaluated test set, the system produced two definite errors among 57 answered pairs, both wrong splits. When I apply the observed coverage and class mix to 100 monthly pairs, this corresponds to approximately 1.67 wrong splits per month.

| Assumed cost of one wrong split | Monthly error cost | Net monthly saving under the evaluated policy |
|---:|---:|---:|
| $25 | $41.67 | +$88.50 |
| $50 | $83.33 | +$46.83 |
| $75 | $125.00 | +$5.17 |
| $78.10 | $130.17 | Break-even |
| $100 | $166.67 | −$36.50 |
| $150 | $250.00 | −$119.83 |

At a $75 article cost, I estimate that the evaluated policy saves only about $5 per month. I cannot solve this problem by increasing volume because labor savings and expected error costs both scale linearly with the number of pairs. I therefore treat the error rate and the cost of an incorrect content decision as the main business levers.

My error-cost estimate is itself uncertain. I extrapolate from two observed errors in a small test set and exclude lost traffic, management time, and the opportunity cost of delayed content. I therefore present it as a sensitivity analysis, not as a forecast.

### Business outcome

I demonstrated a reduction in expected review effort, but I did not achieve the promised maximum of 25 manual checks per 100 pairs. I also did not establish that the LLM creates more value than the lexical rule. I therefore make a limited pilot decision, not a deployment decision.

## 5 Technical design and evaluation

### System design

My implementation contains the following components:

- 82 B2B SaaS seed keywords.
- 25 adversarial variants generated by a model different from the classifier.
- A seeded Python pair generator producing 140 pairs: 20 development and 120 held-out test pairs.
- A frozen Serper.dev snapshot of Google top-10 organic results at `gl=sg` and `hl=en`.
- A lexical baseline using synonym normalisation and token Jaccard.
- An LLM classifier using `openai/gpt-4o-mini` through OpenRouter.
- Frozen caches, structured outputs, per-class metrics, confusion matrices, and ROI calculations.

As summarised in the layer-by-layer analysis above, I own the project-specific workflow, data design, evaluation, and observability logic. I rent LLM inference from OpenRouter, SERP collection from Serper.dev, and the Colab runtime. This keeps my implementation small and inexpensive, but transfers availability, pricing, and version-control risks to third parties.

I implemented the following final label rule:

- overlap = 0 → Different
- 0 < overlap < 0.3 → Uncertain
- overlap ≥ 0.3 → Same

This differs from my Week 3 Problem Statement, which described 0–1 shared URLs as Different and exactly two as Uncertain. My final rule treats any positive overlap below 0.3 as Uncertain. I froze it before LLM evaluation and accepted more manual review in exchange for a more cautious boundary around low-overlap cases.

### Leakage control and evaluation protocol

I never place SERP results in the LLM prompt. I froze labels and thresholds before LLM evaluation. I used the 20 development pairs for few-shot examples and threshold selection, and I scored the 120 test pairs once. I evaluated the majority and lexical baselines on the same held-out test set, with baseline thresholds selected on development data.

My v1 prompt allowed Same, Different, or Uncertain, but the model predicted Uncertain zero times on both development and test data. I therefore redesigned v2 to force a Same or Different choice and return a separate confidence value. I converted predictions below the development-selected threshold of 0.85 to Uncertain.

### Evaluation results

| System | Three-class macro-F1 on test n=120 |
|---|---:|
| Majority-class baseline | 0.234 |
| Tuned lexical rule | **0.591** |
| LLM v1 | 0.398 |
| LLM v2 | 0.438 |
| Pre-specified target | 0.700 |

I obtained the following v2 per-class results:

| Class | Precision | Recall | F1 | Support |
|---|---:|---:|---:|---:|
| Different | 0.718 | 0.785 | 0.750 | 65 |
| Uncertain | 0.244 | 0.379 | 0.297 | 29 |
| Same | 1.000 | 0.154 | 0.267 | 26 |

My Same precision of 1.000 is based on only four Same predictions. I do not treat this as sufficient evidence of zero wrong-merge risk, particularly because Same recall was only 0.154.

On the 91 pairs with definite Same or Different ground truth, I measured 0.744 forced-binary macro-F1 for the LLM and 0.741 for the re-tuned lexical rule. I treat this 0.002 difference as a practical tie, not an LLM win.

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

I consider the definition of Uncertain to be the most important construct limitation. In my ground truth, Uncertain means that observed SERP overlap fell inside a numeric band, while model confidence represents subjective certainty about semantic intent. These are different quantities. My v2 confidence threshold improved Uncertain F1 from 0 to 0.297 but reduced Same recall to 0.154. When I tested other thresholds, none closed the gap with the lexical baseline within the evaluated grid.

I identified a second limitation through the submission-week drift audit. On 2026-09-29, I re-collected the 30 unique keywords contained in a fixed 20-pair audit sample. Four of 20 labels changed and overlap MAE was 0.030. Three moved from Uncertain to Different and one from Different to Uncertain; none moved to or from Same. I kept the original labels frozen, so I interpret all evaluation and ROI results as specific to the original snapshot.

## 7 Operational risks and governance

| Risk | Business consequence | Control for a pilot |
|---|---|---|
| Wrong merge | Missed ranking opportunity, rewrite cost, and difficult-to-detect content failure | I would require writer confirmation for every Same decision and monitor Same precision and recall |
| Wrong split | Redundant article cost and cannibalisation | I would track downstream article creation and rework and include error cost in ROI |
| SERP and label drift | Decisions and measured performance may change over time | I would repeat a fixed drift sample monthly and investigate movement involving Same |
| Model or provider change | Fresh predictions may differ from the evaluated system | I version prompts, log the provider and model alias, preserve frozen responses, and would revalidate before upgrades |
| API availability and pricing | Workflow interruption or changing operating cost | I retain the lexical rule and manual process as fallbacks and would monitor cost per pair |
| Misleading explanation | A fluent reason may persuade a writer to accept a wrong decision | I would present reasons as supporting evidence rather than proof and review a sample before deployment |
| Automation bias | Writers may stop challenging confident outputs | I would display confidence and source limitations and retain accountable human ownership of merge decisions |

I treat a high-confidence Same prediction that is actually Different as the most serious silent failure. It can trigger a wrong merge without producing an immediate visible error; the resulting page may simply underperform until someone performs a later SERP or content audit. To detect it, I would require writer confirmation for every Same decision, periodically recheck a sample of high-confidence Same pairs against live SERPs, log writer overrides, and track pages that require later splitting or rewriting. The confidence threshold, frozen caches, and prediction logs are implemented controls in the current prototype. Writer confirmation and ongoing outcome monitoring are proposed pilot controls and were not validated as part of this experiment.

For a pilot, I keep the SEO writer as the decision owner. I may use the system to prioritise work and automate low-risk routing, but I would not allow it to publish content or merge keyword groups silently. I would retain the input pair, model and prompt version, confidence, prediction, writer override, and final page decision in the logs. I need these records to measure actual time savings, override rates, wrong merges, wrong splits, and model drift.

## 8 Final recommendation

I recommend that IntentPair proceed only as a small, monitored decision-support pilot. I would not use it to replace manual SEO judgement or present it as superior to the lexical rule.

I would use the following pilot workflow:

1. Run the lexical rule and LLM on each candidate pair.
2. Automatically accept only high-confidence Different decisions under a documented threshold.
3. Require writer confirmation for every Same decision.
4. Send every Uncertain decision to full SERP review.
5. Record writer overrides, review time, article outcomes, and error costs.

Before wider deployment, I would require the project to meet four gates on a new holdout and real workflow sample:

- Demonstrate that selective automation reduces measured review time, not only estimated time.
- Evaluate a calibrated abstention policy for the lexical rule and compare it fairly with LLM confidence.
- Establish an acceptable upper bound for wrong-merge risk with substantially more Same predictions.
- Show positive error-aware ROI under the organisation's actual content-production cost.

If these gates are not met, I recommend using either the lexical rule as a transparent prioritisation aid or continuing manual review. I interpret the current evidence as support for learning and a controlled pilot, but not production automation.
