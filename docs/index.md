# Predicting Content Decline Risk: A Refresh Opportunity Scoring Model

## Abstract

This project investigates whether historical search and engagement signals can predict which content pages are likely to experience future performance decline. A Random Forest classifier was trained on March 2026 search and engagement data to predict click declines of 30% or more from March to April, using six features (impressions, clicks, CTR, average position, GA4 sessions, scroll events) with strict leakage controls. The model was evaluated against a transparent rule-based baseline on the same held-out test set, achieving a precision of 0.582 and recall of 1.000 compared to the baseline's 0.241 precision and 0.552 recall. An out-of-time validation on the April→May window confirmed the model's recall (1.000) and ranking ability (ROC-AUC 0.961) remained stable, though precision dropped to 0.48, indicating sensitivity to shifts in the underlying decline rate. The resulting model powers a ranked action queue with reason codes, intended as decision support for content teams prioritizing review — not as proof of causal decline drivers.

---

## Introduction / Problem Statement

**Research Question**

Can historical search and engagement signals be used to identify content pages that are likely to experience future performance decline?

**Decision Supported**

This work supports content teams in deciding which pages should be reviewed first for possible SEO or content improvement. The system is intended as decision support, not as proof that updating a page will cause its performance to improve.

---

## Data

**Data Release**

This project uses the FlyRank ML Internship warehouse release available on Hugging Face.

**Table Used**

The main table used is `fact_content_daily_performance`, which contains daily search and engagement performance for content pages. Each row represents the daily performance of one content page for one client on one date.

**Date Window**

The analysis uses data from March 2026 through May 2026. March 2026 is used for feature development, April 2026 is used as the primary outcome window, and April→May is used for out-of-time validation.

**Fields Excluded**

Client and content IDs are used only for joining, grouping, and identifying rows, not as model features. Future-window information, label-derived fields such as `trend_direction` and `trend_pct`, and product decision flags are excluded because they would reveal information about the outcome or introduce data leakage. No client names, domains, URLs, private queries, credentials, or raw exports are used.

**Data Verification**

The March 2026 warehouse data contains 9,841,378 daily content-performance rows covering March 1–31, 2026. The analysis uses March as the feature window and April as the first future outcome window. Content pages must be present in both months so that the future decline label can be calculated consistently.

---

## Methodology

**Label Definition**

A page is labeled as `future_decline = 1` if its clicks decrease by at least 30% from March to April. This label uses future information only for evaluation and is never used as a model feature. The decline rate is approximately 12%, meaning the dataset is moderately imbalanced.

**Feature Design**

The model uses six features calculated only from March data: impressions, clicks, CTR, average search position, GA4 sessions, and scroll events. These features represent search visibility, click performance, search position, and user engagement available before the April outcome window. No April information is used as a model input.

**Missing Value Handling**

CTR and average position were missing for approximately 154,699 of 331,436 pages because usable GSC data was not available for those pages. For this baseline model, missing CTR and position values were filled with 0 so the model could run on a complete feature matrix. A filled value of 0 does not mean an actual position of 0 — it represents "unavailable," and this simplification may bias predictions for those pages.

**Validation Design**

The dataset was split into 80% training and 20% testing data using a fixed random seed and stratification on the future-decline label. Both sets contain approximately the same 12% decline rate. The model was evaluated only on the held-out test set (66,288 pages, including 7,952 actual future declines).

**Model**

A Random Forest classifier (100 estimators, max depth 8, `class_weight="balanced"`) was used as the predictive model. It was selected because it can capture non-linear relationships between the search and engagement signals without requiring complex feature transformations. Class balancing was enabled because only 12% of pages are labeled as future declines.

**Baseline**

The Week-4 rule-based baseline scores pages using five hand-written conditions across impressions, clicks, average position, GA4 sessions, and scroll events. A score of 4 or 5 out of 5 is treated as a positive prediction of future decline. The baseline was evaluated on exactly the same test pages and labels as the Random Forest model, ensuring a fair, apples-to-apples comparison.

**Leakage Controls**

The model was trained and validated on a single time window (March features predicting April outcomes), using an 80/20 stratified train-test split with a fixed random seed. Leakage was controlled by excluding all future-window fields, label-derived columns, and product decision flags from the feature set — only March-observed signals (impressions, clicks, CTR, average position, GA4 sessions, scroll events) were used as inputs.

Out-of-time validation (April→May) was completed and is reported in the Results section below. Recall (1.000) and ROC-AUC (0.961) remained stable on this out-of-time window, while precision dropped from 0.582 to 0.48 — indicating the model generalizes reasonably well in ranking declining pages, but its precision is sensitive to shifts in the underlying decline rate across time periods. Validation on a further month (May→June) was not attempted due to compute/session constraints; this remains a limitation and is carried forward into the Limitations section.

---

## Results (vs Baseline)

The Random Forest model is compared against the transparent Week-4 rule-based baseline on the exact same held-out test set (66,288 pages, 7,952 actual future declines). This ensures a fair, apples-to-apples comparison.

| Metric    | Random Forest | Week-4 Baseline |
|-----------|---------------|------------------|
| Precision | 0.582         | 0.241            |
| Recall    | 1.000         | 0.552            |
| F1 Score  | 0.736         | 0.336            |
| ROC-AUC   | 0.966         | —                |

On the same held-out test set, the Random Forest model outperformed the Week-4 rule-based baseline across all measured metrics, most notably in recall (1.000 vs 0.552) — meaning the model successfully flagged all pages that actually declined in the test set, whereas the baseline rule missed nearly half of them. The trade-off is a moderate precision (0.582), meaning some flagged pages did not decline.

A recall of 1.000 alongside `class_weight="balanced"` suggests the model may be flagging aggressively at this threshold rather than achieving genuinely perfect separation between declining and non-declining pages. This is why precision is reported alongside recall — the trade-off between the two, not the recall number alone, is what determines practical usefulness for a content team.

**Out-of-Time Validation (April → May)**

The March-trained model was applied to April features and evaluated against the actual April→May decline label, without any retraining. This tests whether the model generalizes to a different time window rather than memorizing patterns specific to March→April.

| Metric    | March→April (Test Set) | April→May (Out-of-Time) |
|-----------|--------------------------|----------------------------|
| Precision | 0.582                    | 0.48                       |
| Recall    | 1.000                    | 1.000                      |
| F1        | 0.736                    | 0.649                      |
| ROC-AUC   | 0.966                    | 0.961                      |
| Rows      | 66,288                   | 362,172                    |
| Decline rate | 12.0%                 | 9.0%                       |

Recall remained perfect and ROC-AUC stayed high across both windows, indicating the model consistently ranks declining pages above non-declining ones. Precision dropped from 0.582 to 0.48, meaning a larger share of flagged pages did not actually decline in the April→May window. This is a directional signal that the model's precision is sensitive to the base decline rate (9.0% in April→May vs ~12% in March→April) and may need threshold recalibration per period rather than being deployed with a single fixed cutoff. Overall, the model shows reasonable — not perfect — stability across time windows, and this out-of-time check should be treated as an important caveat for deployment, not a full guarantee of future performance.

**Feature Importance**

Feature importance analysis showed that clicks and CTR were the strongest contributors to the model's predictions, while scroll events contributed very little. These results are directional and describe model behavior rather than causal effects.

---

## Limitations

- **Single time window (primary training):** The model was trained and tested primarily on the March→April window. While out-of-time validation on April→May was completed and showed reasonable stability, validation on a further window (May→June) was not completed due to compute constraints, so long-term stability remains only partially verified.

- **Missing-value handling:** CTR and average position were missing for ~154,699 of 331,436 pages (no usable GSC data) and were filled with 0. A filled value of 0 does not mean an actual position of 0 — it represents "unavailable," and this simplification is a limitation of this baseline treatment.

- **Label definition is narrow:** "Future decline" is defined purely by a ≥30% drop in clicks month-over-month. It does not account for seasonality, external traffic shifts, algorithm updates, or other causes — a decline flagged by this label may not reflect content quality issues.

- **Precision varies across time windows:** While recall is perfect (1.000) on both the original test set and the out-of-time window, precision dropped from 0.582 to 0.48 on the out-of-time window, meaning a meaningful and possibly growing share of flagged pages will not actually decline. This model is best used to prioritize review, not as a definitive verdict.

- **Not causal:** This model identifies pages *associated with* future decline risk based on historical patterns. It does not prove that any action taken on a flagged page will prevent decline, nor does it measure Google's ranking algorithm directly.

- **Class imbalance:** Only ~9–12% of pages were labeled as future declines across the windows studied. Class-balancing was used during training (`class_weight="balanced"`), but this does not eliminate the underlying rarity of the outcome.

All claims in this project should be read as **observed, directional, and decision-support** signals — not proof of causal impact.

---

## Ranked Recommendations

Using the trained model's decline-probability score, all pages are ranked and assigned an action label. Of all 331,436 scored pages, **67,527 pages (≈20%)** fall into "Review urgently" (probability ≥ 0.70), another **477 pages** fall into "Review" (0.40–0.70), and the remaining **263,432 pages** are in "Monitor" — no immediate action needed.

Each flagged page is also assigned a **reason code** explaining why it was flagged:

- `HIGH_PREDICTED_RISK` — decline probability ≥ 0.70
- `LOW_CTR` — CTR below 2% despite meaningful impressions
- `POOR_POSITION` — average search position ≥ 40
- `LOW_ENGAGEMENT` — minimal GA4 sessions and scroll events

**Action Playbook**

1. **Prioritize "Review urgently" pages first**, focusing on those with ≥100 impressions to ensure review effort targets pages with measurable search visibility.
2. **Use reason codes to guide the type of review:**
   - `LOW_CTR` → suggests a metadata/title review (improve the search snippet)
   - `POOR_POSITION` → suggests a content-quality or backlink review
   - `LOW_ENGAGEMENT` → suggests a content depth/relevance review
3. **Treat "Review" pages as a secondary queue** — lower urgency but still worth periodic monitoring.
4. **Re-score periodically** rather than treating any single score as permanent, given the precision sensitivity observed in out-of-time validation.
5. **Combine model output with human judgment** — this system flags candidates for review; it does not replace editorial or SEO expertise.

---

## Reproducibility

- Capstone notebook: [work/notebooks/capstone.ipynb](https://github.com/ShardhaBatra/flyrank-ml-internship/blob/main/work/notebooks/capstone.ipynb)
- Full repository (all weekly assignment notebooks + capstone): [flyrank-ml-internship](https://github.com/ShardhaBatra/flyrank-ml-internship)
- Data source: FlyRank ML Internship warehouse (Hugging Face, gated access)

---

## Acknowledgments & Data Credit

Built on the FlyRank ML Internship dataset — [https://flyrank.ai](https://flyrank.ai)
