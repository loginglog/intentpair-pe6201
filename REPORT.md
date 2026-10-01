# IntentPair Business and Technical Trade-off Analysis

**Jiayao Ge · Section B · End-of-Course Project**

## 1 Executive decision and business value

I designed IntentPair to help a B2B SaaS SEO writer decide whether two similar keywords should be served by one page or two. Today, comparing 100 keyword pairs manually takes about 8.33 hours. IntentPair uses keyword text to return Same, Different, or Uncertain with a short reason, so the writer can focus on difficult cases.

The prototype works end to end, but I do not recommend unconditional deployment. The final LLM scored 0.438 three-class macro-F1, below my 0.700 target and the tuned lexical rule at 0.591. It also sent 37.5% of cases to review, above my 25% target. Its useful result was selective: on 91 test pairs with definite ground truth, answered cases had a 3.5% forced-choice error rate, while abstained cases would have been wrong 41.2% of the time. Confidence therefore identified harder cases, but did not make the LLM the best classifier.

The project changes the workflow from exhaustive comparison to selective review, but only under human control. I would pilot it as a decision-support tool: automatically accept only high-confidence Different decisions, confirm every Same decision, and manually review every Uncertain decision.

## 2 Business and build-versus-buy trade-off

Manual review uses current SERPs and human context, but costs about $208.33 per 100 pairs at five minutes per pair and $25 per hour. A commercial SERP-clustering product would provide a more mature live-data workflow, but introduces subscription cost and less control. My transparent lexical rule is the strongest low-cost baseline. The LLM adds a reason and a provisional confidence signal, but its lower primary score and third-party dependency mean that AI is not justified by accuracy alone.

My own-versus-rent decisions were:

| Layer | Decision | Implementation and rationale |
|---|---|---|
| Interface | Hybrid | I own the notebook interaction and rent the Colab runtime; a separate application was unnecessary for the prototype. |
| Orchestration | Own | Python controls pair generation, API calls, caching, routing, and exports because these are project-specific. |
| Model | Rent | `openai/gpt-4o-mini` through OpenRouter kept cost and build time low. |
| Data | Hybrid | I own the frozen pairs and labels and rent live SERP collection from Serper.dev. |
| Evaluation | Own | I wrote the splits, baselines, metrics, threshold selection, drift audit, and ROI analysis. |
| Observability | Own | Frozen responses, versions, predictions, and drift results support reproduction and diagnosis. |

These choices keep the parts that define the project's evidence under my control while renting components that would be uneconomic to reproduce. Training my own language model or building a search engine would add cost without answering the central question: whether a text-only AI decision can reduce review work safely. Renting the model and live retrieval services let me spend the limited project time on leakage control, evaluation, and failure analysis. The trade-off is exposure to provider pricing, availability, and model changes. I reduced that exposure with pinned model settings, cached responses, frozen labels, and recorded metadata, but I cannot eliminate it. This is acceptable for a monitored prototype and unacceptable for autonomous publishing.

## 3 Technical design and evaluation

The dataset contains 82 seed keywords and 25 adversarial variants. A seeded generator produced 140 pairs: 20 development pairs and 120 held-out test pairs. Google top-10 results at `gl=sg` and `hl=en` provided the label proxy. I froze the labels before LLM evaluation and never placed SERP results in the prompt, preventing the model from seeing the evidence used to score it.

The final rule was overlap = 0 for Different, 0 < overlap < 0.3 for Uncertain, and overlap ≥ 0.3 for Same. I compared a majority baseline, a synonym-normalised token-Jaccard rule, and two LLM designs on the same test set. Baseline thresholds and the LLM confidence threshold were selected only on development data.

| System | Test macro-F1 |
|---|---:|
| Majority baseline | 0.234 |
| Tuned lexical rule | 0.591 |
| LLM v1 | 0.398 |
| LLM v2 | 0.438 |
| Pre-specified target | 0.700 |

The main implementation difficulty was abstention. V1 asked the model to output three classes directly, but it predicted Uncertain zero times. I redesigned v2 to force a Same or Different choice and return confidence separately. A development-selected threshold of 0.85 converted low-confidence answers to Uncertain. This tuning raised macro-F1 from 0.398 to 0.438 and Uncertain F1 from 0 to 0.297, but Same recall fell to 0.154. The redesign repaired a failed mechanism without changing the metric after seeing test results, although it did not close the gap with the lexical rule.

## 4 Outcome and evaluation critique

The negative headline result is meaningful because the simpler rule performed better. On the secondary binary view, excluding ground-truth Uncertain pairs, the LLM scored 0.744 and the rule 0.741. I treat the 0.002 difference as a practical tie. The LLM's remaining case depends on whether its confidence and explanations improve human review beyond a calibrated rule; I did not test an equivalent rule-based abstention policy or conduct a user study of the reasons.

The largest construct problem is Uncertain. The ground-truth class means that observed SERP overlap fell inside a numeric band, while LLM confidence represents subjective semantic certainty. These are not the same quantity. Other limitations also weaken generalisation: the development set has only 20 pairs, the Jaccard-stratified sampling favours the lexical feature, the full dataset has only 140 pairs, and Same precision of 1.000 came from only four predictions.

Macro-F1 was the right headline metric because it gives equal importance to all three decisions and prevents the majority Different class from dominating the result. However, it is unstable when a class has few examples. Uncertain has 29 test pairs and Same has 26, so a small number of changed predictions can move the headline noticeably. I therefore report class counts, per-class precision and recall, review rate, forced-choice error, and drift alongside macro-F1. These diagnostics changed my interpretation: the model's low answered-case error is operationally promising, but its 0.154 Same recall and high review rate show that this safety came from avoiding difficult commitments. A larger confidence interval or repeated holdout study would be needed before treating the observed differences as reliable performance gains.

SERP labels also drift. During submission week I re-collected the 30 keywords in a fixed 20-pair sample. Four labels moved and overlap MAE was 0.030; all movements were between Different and Uncertain. This inexpensive audit shows that the reported metrics describe a frozen snapshot rather than permanent truth. A fresh holdout and repeated snapshots are needed before deployment.

## 5 Business outcome and operational risk

Reviewing only Uncertain predictions reduces estimated work from 8.33 to 3.13 hours per 100 pairs, saving 5.21 hours or $130.17 before error costs. However, the test set produced two wrong splits among 57 answered definite cases. Extrapolating the observed coverage gives about 1.67 wrong splits per 100 pairs. At $75 per unnecessary article, net monthly saving falls to about $5.17; $78.10 is the break-even error cost. This estimate is uncertain because it is based on two errors and excludes traffic loss, delayed content, and management time.

The most serious silent failure is a high-confidence Same that is actually Different. A wrong merge may simply underperform, so it can remain unnoticed longer than a redundant article. The current prototype preserves predictions, confidence, prompts, model identity, and frozen caches. A pilot must add writer confirmation for every Same decision, periodic live-SERP audits, override logging, and tracking of pages later split or rewritten. The writer remains accountable for the final content decision.

## 6 Recommendation and future path

I recommend a small monitored pilot, not production automation. Before wider use, I would: compare LLM confidence with a calibrated lexical-rule abstention policy; test on a larger repeated-snapshot holdout; measure actual review time rather than assumed five-minute checks; and calculate ROI using real rewrite and content-production costs. Deployment should proceed only if selective automation lowers measured effort, the upper bound on wrong-merge risk is acceptable, and error-aware ROI remains positive. Otherwise, the lexical rule should remain a transparent prioritisation aid and the final decision should stay manual.

Repository URL: `https://github.com/loginglog/intentpair-pe6201`.
