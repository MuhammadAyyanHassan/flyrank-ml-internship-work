# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Muhammad Ayyan Hassan
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/MuhammadAyyanHassan/flyrank-ml-internship-work
- **Date:** August 2026

> This report mirrors the deployed research paper (`docs/index.html`) section-for-section.
> Sections 1–8 map to the Pass / Needs-Work rubric axes; sections 0 and 9 are the paper's
> abstract and data credit. Every number here traces to `work/outputs/capstone_metrics.json`,
> the run receipt committed beside the executed notebook.

## 0. Abstract

Can a content team, using only search behavior observable at decision time, rank a page library so the pages most likely to lose search visibility next month rise to the top of a short review queue? Working from a pseudonymized FlyRank search-warehouse release of 158,549 page-months across 46 clients, I defined a next-month decline proxy and engineered nine decision-window features covering both traffic level and within-window dynamics. A transparent rule baseline, logistic regression, and gradient boosting were compared under client-grouped cross-validation, scored by Precision@K against the base rate on the identical split, and re-tested on a later, unseen month. At the operational depth of the top 100 pages the logistic model reached 93.0% precision against a 47.82% base rate — while the rule baseline did no better than chance — but an ablation shows most of that lift comes from within-window momentum features, so the system is better described as triaging pages already sliding than as forecasting a decline before it begins. The deliverable is therefore a ranked, reason-coded review queue for prioritizing human attention — a decision-support tool, not a claim about Google's algorithm or the cause of any page's decline.

## 1. Problem framing

A team responsible for a large body of published content — here, an agency managing many client sites — has far more pages than it can ever hand-review. Each month only a small slice can be looked at closely. The practical question is not "what is wrong with the whole library," but a triage question: which pages should a human open first? Review a page that was going to stay healthy and you have spent scarce attention for nothing; miss a page that is about to lose search visibility and traffic erodes before anyone notices.

The unit of analysis is a single page (for a single client) in a single month. The output is a rank, a reason code, a suggested review action, and a confidence band. The action a person takes is to review the top of the queue downward until their time runs out; the cost of a wrong call is either wasted review effort (a false positive) or a silent slide caught too late (a false negative).

Why involve a model at all? The risk signal is spread thinly across the daily behavior of 158,549 page-months — no one eyeballs that at scale, and eyeballing invites inconsistent, gut-feel prioritization. A model can rank the whole library consistently and cheaply, and — crucially — can be checked against a transparent rule so we know whether it actually earns its complexity. To be clear from the outset: it prioritizes pages *already* at risk for human review. It is triage, not a forecast of decline before any sign of it appears.

## 2. Data safety

The source is the pseudonymized FlyRank ML Internship warehouse release `flyrank_pseudonymized_warehouse_release_v20260703`, specifically the `fact_content_daily_performance` table. It was queried in place with DuckDB over the gated Hugging Face release (`hf://`); the dataset itself is never downloaded into the repository or committed. Only the two calendar months needed for each analysis frame were pulled — not the full 78.8M-row fact table — and the release's held-out `_sample` table was never touched.

**Columns deliberately excluded to prevent leakage:** any outcome-window field or the label itself; the product's own `trend_direction` and `trend_pct` (both label-derived); the product flags `health_score`, `priority_score`, and `action_type`; and both hash identifiers, which are used only to group cross-validation folds and are never features. As a guardrail, the notebook asserts that every feature in the final list carries the `w_` decision-window prefix.

**Client safety:** the source carries only pseudonymous hash identifiers, and none are printed in this report, the paper, or the repository. No client names, domains, URLs, or search queries appear anywhere. The exported review queue carries no client name, URL, or query field. One data-quality caveat about the `w_avg_position` column is documented in section 6 rather than buried here.

## 3. Baseline

The model is measured against a frozen transparent rule from the earlier baseline week, chosen because it is exactly the kind of common-sense heuristic a team would otherwise use by hand: **score 5** if a page had at least 500 impressions *and* an average position of 4–20 *and* a CTR below the median CTR of that visible band; **score 3** for volume alone; **0** otherwise. The band-median CTR is refit inside each training fold only, never on the rows being scored — the per-fold CTR cutoffs were 0.1911%, 0.2155%, 0.1907%, 0.1886%, and 0.1734%. This is a fair comparison because it runs on the identical client-grouped, out-of-fold split as the models and is scored on the same Precision@K metric against the same base rate.

## 4. Model / analysis

**The label (a proxy).** A page is labeled a next-month decline (`future_decline_label = 1`) when it had impressions in the decision window *and* its outcome-window impressions fell below 80% of the decision-window level. It is a proxy for "lost meaningful visibility," not a ground-truth verdict; the 80% cutoff is a modeling choice.

**Features — nine, all measured strictly inside the decision window** (each carries a `w_` prefix to mark it decision-time-only):

- *Level (5):* `w_impressions`, `w_clicks`, `w_ctr_pct`, `w_avg_position` (impression-weighted), `w_impression_days` — how much traffic the page had and how visible it was.
- *Dynamics (4):* `w_late_share`, `w_max_day_share`, `w_impr_cv`, `w_active_day_share` — how that traffic was distributed across the days of the window: late-weighting, single-day spikes, day-to-day variability, and how many days were active at all.

**Models.** Two models sit on the same features and the same split: **logistic regression** (median-impute → standardize → `max_iter=1000`) as the primary model, and a **HistGradientBoostingClassifier** as a complexity test — there to answer "does a more flexible model actually do better here?" The primary model was chosen to be simple and explainable unless the heavier model clearly earned its place (section 5 shows it did not, at the operating depth).

## 5. Evaluation

**Split.** Scoring uses `StratifiedGroupKFold` (5 folds, shuffled, `random_state=42`) grouped on client, with out-of-fold predictions and a per-fold assertion that no client appears in both train and test. Every score is therefore earned on clients the model did not train on. Both frames are sorted on their join keys before any split, which makes the run deterministic — an earlier non-determinism from DuckDB's `GROUP BY` row order is what made two prior weeks' AUC numbers disagree, and sorting fixed it.

**Two time frames.** Frame A (development): March 2026 features → April 2026 outcome; 158,549 rows, 46 clients, decline rate 47.82%. Frame B (forward-time test): April 2026 features → May 2026 outcome; 183,345 rows, 51 clients, decline rate 47.21%; 46 clients appear in both.

**Primary metric.** Precision@K at K = 10, 50, 100, 500, always reported next to the base rate; headline operating depth is K = 100 (a realistic monthly review queue). Bands are a 2000-replicate bootstrap (seed 42) and describe how the outcome varies *within the selected top-K with the ranking held fixed* — not a confidence interval over redrawing the population.

*Precision@K (%), Frame A. Bootstrap 95% band in brackets. Base rate = 47.82%.*

| Scorer | Top 10 | Top 50 | Top 100 | Top 500 |
|---|---|---|---|---|
| Rule baseline | 50.0 [20.0, 80.0] | 46.0 [32.0, 60.0] | 43.0 [33.0, 53.0] | 49.6 [45.2, 54.0] |
| Logistic regression | 100.0 [100, 100] | 96.0 [90.0, 100] | **93.0 [87.0, 98.0]** | 86.8 [83.8, 89.6] |
| Gradient boosting | 90.0 [70.0, 100] | 90.0 [80.0, 98.0] | 88.0 [81.0, 94.0] | 93.4 [91.2, 95.6] |

At the top 100, logistic regression flagged pages that declined **93.0%** of the time against a 47.8% base rate — roughly 93 of its top 100 picks were genuine decliners. The rule baseline landed at **43.0%**, with a band of [33.0, 53.0] that straddles the base rate: at this depth the hand rule is no better than picking at random. Logistic regression's advantage over the rule clears its bootstrap band, so it is a real separation, not noise.

**Did the more complex model earn its keep?** Not at the operating depth. At the top 100, gradient boosting scored 88.0% [81.0, 94.0] versus logistic's 93.0% [87.0, 98.0] — bands overlap, so there is no honest case the heavier model is better where it matters, and the simpler, more transparent logistic model wins by default. Reported plainly: gradient boosting *does* pull ahead deeper (top 500: 93.4% vs 86.8%) and on whole-population ranking (ROC-AUC 0.7155 vs 0.6644). If the job were to rank the entire library rather than skim the top 100, gradient boosting would be the pick; for a short triage queue, it is not.

*Whole-population ranking quality (secondary). Random ranker ≈ ROC-AUC 0.50, AP near the 0.478 base rate.*

| Scorer | ROC-AUC | Average precision |
|---|---|---|
| Rule baseline | 0.5249 | 0.4994 |
| Logistic regression | 0.6644 | 0.6378 |
| Gradient boosting | 0.7155 | 0.6874 |

**Forward-time test (Frame B, April→May 2026, base rate 47.21%).** Everything was fit on Frame A, frozen, and applied untouched to Frame B; the rule's CTR cutoff (0.1908%) was carried over from Frame A.

*Precision@K (%) with bootstrap band; whole-population metrics at right. Base rate = 47.21%.*

| Scorer | Top 10 | Top 50 | Top 100 | Top 500 | ROC-AUC | AP |
|---|---|---|---|---|---|---|
| Rule baseline | 40.0 [10, 70] | 26.0 [14, 38] | 32.0 [23, 41] | 38.8 [34.6, 43.2] | 0.5582 | 0.5090 |
| Logistic regression | 90.0 [70, 100] | 96.0 [90, 100] | **98.0 [95, 100]** | 96.4 [94.8, 98.0] | 0.7579 | 0.7342 |
| Gradient boosting | 100.0 [100, 100] | 98.0 [94, 100] | 99.0 [97, 100] | 98.0 [96.8, 99.2] | 0.7746 | 0.7633 |

This is a **time-forward** test, not a client-holdout test. The same 46 clients appear in both months, so it measures whether the model *stays stable over time* — not generalization to brand-new clients (that is what the client-grouped CV on Frame A answers). Because Frame B has its own base rate (47.21%), its Precision@K is only meaningful against 47.21%. The takeaway is narrow and honest: on an unseen later month, the model did not degrade.

## 6. Interpretation

**Where the lift comes from — an ablation.** Holding the split and model fixed and changing *only* the feature set:

*Ablation detail, Frame A. Precision@K as point estimates; ranking metrics on the whole population.*

| Feature set | Top 10 | Top 50 | Top 100 | Top 500 | ROC-AUC | AP |
|---|---|---|---|---|---|---|
| Level only (5) | 60.0 | 64.0 | 60.0 | 56.6 | 0.5900 | 0.5382 |
| Level + dynamics (9) | 100.0 | 96.0 | **93.0** | 86.8 | 0.6644 | 0.6378 |

At the top 100, adding the four within-window dynamics features lifts precision from 60.0% to 93.0% — a **+33.0 pp** jump. Level features alone already beat the base rate, but the dynamics features carry most of the usable signal.

**Which features carry the signal.** Permutation importance (one held-out client fold, scored by average precision, five repeats; descriptive only, nothing causal):

| Feature | Importance |
|---|---|
| `w_late_share` (dynamics) | 0.11601 |
| `w_impr_cv` (dynamics) | 0.04854 |
| `w_max_day_share` (dynamics) | 0.03426 |
| `w_clicks` (level) | 0.00601 |
| `w_impressions` (level) | 0.00005 |
| `w_ctr_pct` (level) | 0.00000 |
| `w_impression_days` (level) | -0.00029 |
| `w_active_day_share` (dynamics) | -0.00029 |
| `w_avg_position` (level) | -0.00863 |

Almost all the usable signal sits in three dynamics features — `w_late_share`, `w_impr_cv`, `w_max_day_share`. Once those are present, the level features add essentially nothing.

**The most important interpretation — triage, not forecast.** That lift is real, but it is *partly mechanical with the label*. The label asks whether next month's impressions fall below 80% of this month's; those three features measure whether a page's impressions were *already* trailing off, spiking, or unstable *inside* the decision window. A page visibly sliding by the end of the month tends, by simple momentum, to keep sliding. This is **not leakage** — every feature uses decision-window data only, no outcome-window value ever touches the model (the ten-check audit below confirms it). But the honest description is: the model *detects pages already in decline that are likely to continue*, not one that foresees a drop before any sign appears.

**Leakage audit — all ten checks passed:** (1) no outcome-window or label-derived column among features; (2) no hash identifier used as a feature; (3) every feature carries the `w_` prefix; (4) the outcome window is strictly later than the decision window; (5) exactly one row per client and content key in each frame; (6) zero client overlap inside every grouped fold; (7) the rule's CTR cutoff is fitted on training rows only; (8) the forward test uses parameters fitted on Frame A only; (9) frames are sorted on join keys before any split; (10) the exported queue carries no client name, URL, or query field.

**Boundaries, stated plainly.** *Survivorship:* a page must appear in both windows to have a defined outcome, so "decline" means "still present but down," never "disappeared." *Proxy label:* the 80% threshold is arbitrary — a page at 79% is a decline, one at 81% is not. *Data-quality caveat:* `w_avg_position` contains values below 1.0 (minimum 0.0196; examples 0.116, 0.069, 0.541), which is impossible for a real Google average position and is an artifact of pseudonymization — so it is usable only as a *relative* signal, and it carries almost no weight anyway (permutation −0.00863); coverage is otherwise near-complete (157,790 of 158,549 rows non-null, 759 median-imputed). *No causal claim:* the data is observational — nothing here says refreshing a page prevents decline. *Scope:* 46–51 clients of a single agency across three consecutive months of one warehouse release — not the web at large, and not a statement about how Google's algorithm works.

## 7. Recommendation

The deliverable is a **ranked, reason-coded review queue**. Every eligible page is scored by the logistic model's probability (`score_logreg`) and sorted high to low; pages with fewer than `MIN_ACTIONABLE_IMPRESSIONS = 100` decision-window impressions are set aside. Each surviving row carries a rank, reason code, suggested review action, and confidence band. An editor works the list top-down until the month's review budget runs out.

What a FlyRank editor should do, in priority order: (1) **Work the top of the queue, not the whole library** — evidence is strongest at shallow depth (~93% precision at the top 100 on development, holding on the unseen month), and precision decays deeper. (2) **Read it as "review first," not "these will decline"** — a high rank means the page shows within-window instability associated with continued decline, a reason to open it, not a verdict. (3) **Let the reason code choose the job** — a single-day-spike page needs a stability check; a visible-but-low-CTR page needs a title/snippet look. (4) **Respect the confidence band and the Hold override** — lower-confidence rows and Hold-flagged pages get human judgment before any automated workflow.

*Composition of the top-100 queue by reason code (development frame). "Declined" = share of that group that actually declined; median impressions = decision-window median.*

| Reason code | Pages | Declined | Median impressions | Review action |
|---|---|---|---|---|
| single_day_spike | 49 | 95.92% | 752.0 | Stability review |
| visible_but_low_ctr | 34 | 88.24% | 3,725.5 | Title/snippet review |
| front_loaded_window | 8 | 100.00% | 1,170.5 | Freshness review |
| thin_daily_coverage | 6 | 100.00% | 206.5 | Coverage/indexing review |
| model_flag_no_single_pattern | 3 | 66.67% | 143,907.0 | General review |

Those map to a concrete action mix across the top 100: **Stability review 49, Title/snippet review 34, Freshness review 8, Coverage/indexing review 5, General review 3, Hold 1.** The one Hold is the no-go rule working as designed — it pulled a single `thin_daily_coverage` page out of Coverage/indexing into Hold, which is why that action shows 5 rather than 6. Confidence bands from the model's probability: **Higher** at ≥0.60, **Medium** at ≥0.45, **Lower** below that.

## 8. Reproducibility

The analysis is deterministic and re-runnable from a fresh clone. Randomness is pinned in two places: `random_state=42` for the `StratifiedGroupKFold` split and `np.random.default_rng(42)` for the bootstrap. Both frames are sorted on their join keys before any split, which removes the DuckDB row-order non-determinism that made two earlier weeks' AUC numbers disagree.

To re-run: (1) open `work/notebooks/capstone.ipynb` from the repository in Colab; (2) put your Hugging Face token in **Colab Secrets** as `HF_TOKEN` — never in the notebook source, never committed — since the notebook queries the gated release in place with DuckDB over `hf://`; (3) run top to bottom. The run writes its receipt to `work/outputs/capstone_metrics.json`, committed alongside the executed notebook so the reported numbers are checkable from the repo rather than taken on faith.

*Consistency checks against the upstream ML-04 / ML-07 / ML-09 notebooks (development frame):*

| Check | Expected | Status |
|---|---|---|
| Row count, Frame A | 158,549 | PASS |
| Decline rate, Frame A | 0.4782 | PASS |

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset ([flyrank.ai](https://flyrank.ai)). Crediting the data source is standard research practice; the pseudonymized warehouse release made this work possible while keeping every client anonymous.

---

> **Claims checklist (all satisfied):** language throughout is observed / measured / directional / decision-support — no causal claims, no "predicted Google's algorithm." Every Precision@K and accuracy figure is reported next to its base rate (47.82% Frame A, 47.21% Frame B); ROC-AUC and average precision are given as the honest whole-population discrimination numbers. No client-identifying details anywhere. Numbers match `work/outputs/capstone_metrics.json` and the deployed paper at `docs/index.html`.
