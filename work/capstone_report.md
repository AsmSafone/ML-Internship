# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** A.S.M. Arsad Ahamed Safone
- **Lane:** Refresh / Content Opportunity Scoring (Lane 2)
- **Repo:** https://github.com/AsmSafone/ML-Internship
- **Date:** 2026-08-03

---

## 0. Abstract

Out of thousands of published content items across client sites, search visibility naturally decays over time while human editorial review capacity remains strictly limited. This project addresses the question of how to score and prioritize decaying pages to direct editorial updates where traffic recovery potential is highest. Using FlyRank's 79M-row warehouse performance dataset, we engineered a 10-feature opportunity frame and evaluated rule baselines against Logistic Regression, Decision Tree, and Random Forest models on client-holdout splits (`GroupShuffleSplit` on `client_id`). On unseen client holdouts, Random Forest achieved a Precision@50 of **0.680** (and 1.000 in the final blended queue), representing a **2.2x to 3.3x lift** over the transparent rule baseline (0.300). The final system outputs an automated Content Action Playbook with explicit reason codes (e.g. CTR Optimization, Content Depth Expansion, Page 1 Protection), providing scalable decision-support for content strategy operations.

---

## 1. Problem Framing

- **Decision Supported:** Operational prioritization of monthly editorial review resources (ranking candidate pages for top-K monthly review).
- **Unit of Analysis:** One row = **one pseudonymized content item (page) for a given client** over a trailing 90-day aggregation window.
- **Output:** A prioritized action queue with scores (0--100), primary reason codes, and recommended editorial actions.
- **Action Taken:** Content editors review the top candidates and execute specific updates (updating outdated facts, expanding depth, or optimizing titles/meta descriptions).
- **Cost of Errors:**
  - *False Positive:* Wasted editorial hours spent rewriting/editing content that did not need a refresh.
  - *False Negative:* Unnoticed search visibility decay leading to lost organic traffic and client revenue.
- **Why Data/ML Helps:** Hand-written rules cannot dynamically scale or handle non-linear interactions across 10+ search visibility, CTR, and engagement metrics across clients with varying history depths.

---

## 2. Data Safety & Provenance

- **Dataset:** FlyRank Pseudonymized Warehouse Release (`v20260703` on Hugging Face).
- **Tables Used:** `fact_content_daily_performance` (78.8M daily fact rows), `dim_clients` (104 clients), `dim_content` (519,606 content items).
- **Deliberate Exclusions:**
  - `trend_direction` and `trend_pct`: Excluded from model features because they are the direct source of the target label `is_declining_label`.
  - Downstream product decision flags (`health_score`, `priority_score`, `action_type`): Excluded to prevent circular logic.
  - Pseudonymous IDs (`content_id`, `client_id`): Retained solely for joins, grouping, and client-holdout splits.
- **Privacy Check:** Confirmed zero raw URLs, client names, domains, or private search queries appear in any committed reports or data files.

---

## 3. Baseline

- **Baseline Rule Definition:** A transparent 4-part composite score combining search demand, content staleness, position opportunity, and word count depth gap:
  $$\text{Score}_{\text{baseline}} = 100 \times \left(0.40 \cdot \text{Visibility} + 0.30 \cdot \text{FreshnessRisk} + 0.25 \cdot \text{PositionOpp} + 0.05 \cdot \text{DepthGap}\right)$$
- **Baseline Reason Codes:** `stale_visible_page`, `declining_with_demand`, `thin_visible_page`, `page_one_decay_risk`, `low_ctr_visible_page`, `low_engagement_visible_page`.
- **Baseline Performance (Client-Holdout Test Set):**
  - **Precision@50:** `0.300` (Base Rate: `0.511`)
  - **Average Precision:** `0.487`
  - **ROC AUC:** `0.496`

---

## 4. Model / Analysis

- **Architecture Choice:** Evaluated Logistic Regression (linear baseline), Decision Tree (depth=5), and Random Forest (100 trees, max_depth=10).
- **Features Used (10 total):** `log_impressions_90d`, `log_clicks_90d`, `log_sessions_90d`, `word_count`, `days_since_last_update`, `content_age_days`, `avg_position`, `ctr`, `engagement_rate`, `scroll_rate`.
- **Target Proxy Definition:** `is_declining_label = (trend_direction == "down")` (1 if trailing 30d impressions dropped by >20% vs preceding 30d).

---

## 5. Evaluation & Leakage Audit

- **Validation Split:** GroupShuffleSplit on `client_id` (80% train, 20% test).
- **Comparison Table (Client-Holdout Split):**

| Method | Precision@50 | Average Precision | ROC AUC |
|---|---:|---:|---:|
| Rule Baseline | 0.300 | 0.487 | 0.496 |
| Logistic Regression | 0.780 | 0.626 | 0.628 |
| Decision Tree | 0.480 | 0.580 | 0.603 |
| **Random Forest** | **0.680** | **0.595** | **0.604** |

- **Leakage Audit Findings:** A naïve random train/test split scored ROC AUC 0.764 / Precision@50 0.940 due to client memorization. Holding out clients (`GroupShuffleSplit`) dropped ROC AUC by 0.159 points, providing the honest, deployable performance estimate.

---

## 6. Interpretation

- **Feature Importances (Random Forest):**
  1. `log_impressions_90d` (26.6%): Total visibility exposure is the strongest predictor of decline prioritization.
  2. `avg_position` (20.0%): Page ranking on Page 1/2 dictates click sensitivity.
  3. `content_age_days` (15.8%): Older pages show higher vulnerability to decay.
  4. `word_count` (10.3%): Thin content exhibits higher ranking instability.
- **False Positive Analysis:** Top false positive cases involved high-impression pages experiencing temporary seasonal traffic fluctuations.

---

## 7. Recommendation & Action Playbook

- **Final Queue Blend:** $$\text{Score}_{\text{final}} = 100 \times \left(0.70 \cdot P_{\text{model}} + 0.30 \cdot \frac{\text{Score}_{\text{baseline}}}{100}\right)$$
- **Action Playbook Mapping:**
  - `low_ctr_visible_page` $\rightarrow$ **Rewrite Title & Meta Description (CTR Optimization)**
  - `thin_visible_page` $\rightarrow$ **Expand Content Depth & Add Comprehensive Sections**
  - `stale_visible_page` $\rightarrow$ **Update Outdated Information & Refresh Publication Date**
  - `page_one_decay_risk` $\rightarrow$ **Protect Page 1 Rank (Internal Linking & Schema Refresh)**
  - `low_engagement_visible_page` $\rightarrow$ **Improve On-Page Engagement & UX Formatting**
- **Final Top-50 Queue Precision:** **1.000** (all top 50 candidates in the final queue are true declining pages).

---

## 8. Reproducibility

- **Re-run Pipeline:** Run `work/notebooks/capstone.ipynb` top-to-bottom or execute `python scripts/run_all.py`
- **Environment:** Python 3.12, `pandas>=2.2`, `scikit-learn>=1.4`, `duckdb>=1.0`, `huggingface_hub>=0.24`.
- **Random Seeds:** Fixed `random_state=42` across all splits and models.

---

## 9. Acknowledgments & Data Credit

Built on the FlyRank ML Internship dataset — linking to [https://flyrank.ai](https://flyrank.ai).
