# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Timothy Gona Muthoni
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Gonablitz/flyrank-capstone
- **Date:** October 2026

---

## 0. Abstract

This paper develops an automated machine learning framework to identify high-value web content experiencing traffic decay and accurately predict future 30-day click trajectories. Using the public FlyRank ML Internship dataset release across a 120-day observation window, we analyzed 50,000 anonymized page-level performance records after filtering low-impression noise. We engineered a rolling 60-day feature window measuring position velocity, impression momentum, and rank volatility to train a Random Forest classifier designed to detect $\ge 25\%$ traffic declines in a subsequent 30-day evaluation window. Evaluated on a strict out-of-time validation split, the model achieved an $F_1$-score of 0.81 (Precision: 0.84, Recall: 0.78) and ROC-AUC of 0.88, significantly outperforming the momentum rule-based baseline $F_1$-score of 0.58. Content strategists and editorial teams can deploy this scoring engine to systematically prioritize declining high-yield pages for targeted content updates prior to structural traffic loss.

---

## 1. Problem Framing

- **Decision Supported:** Resource allocation for content maintenance and editorial refreshes. SEO teams must decide which URL assets to rewrite or re-optimize before organic decay becomes unrecoverable.
- **Unit of Analysis:** Anonymized Content Asset (`content_id`) grouped within a Client Workspace (`client_id`) over a fixed temporal window.
- **System Output:** A continuous **Decay Risk Score** $[0.0, 1.0]$ and a categorical risk classification (*Critical Decay*, *Moderate Drift*, *Stable Core*).
- **Actionable Decision:** 
  - *High Decay Score ($\ge 0.80$):* Immediate content rewrite, intent alignment update, and metadata overhaul.
  - *Moderate Score ($0.50 - 0.79$):* Title tag / snippet optimization and internal linking boost.
  - *Low Score ($< 0.50$):* Protect current state; no manual intervention required.
- **Cost of Errors:**
  - *False Positive (High Cost):* Wastes editorial budget and writer hours updating stable pages that did not require intervention.
  - *False Negative (Moderate-High Cost):* Misses decaying revenue-generating assets, resulting in compounding organic traffic loss to competitors.
- **Why ML Helps:** Rule-based heuristics (e.g., checking if last week's clicks dropped) suffer from high false-alarm rates due to short-term search volatility. Supervised ML integrates non-linear interactions across position drift, velocity deltas, and multi-week rank volatility to isolate structural decay from transient noise.

---

## 2. Data Safety

- **Data Used:** Parquet tables from `hf://datasets/FlyRank/internship-warehouse`, specifically `fact_content_daily_performance`.
- **Deliberately Excluded Fields:** 
  - Raw query strings, client names, domain names, target URLs, and location identifiers (all omitted from input sets).
  - Explicit target-derived trend indicators like pre-aggregated `trend_direction` or `trend_pct` to prevent label leakage.
- **Identifiers Policy:** Pseudonymous identifiers (`client_id`, `content_id`) were retained strictly as entity grouping keys for time-aware joins and cross-validation splitting. They were excluded as numerical/categorical predictor features in model training.
- **Public Safety Confirmation:** No unmasked domain names, private queries, API tokens, or client-identifying details exist anywhere within the repository or the `/work` directory.

---

## 3. Baseline

- **Baseline Rule Definition:** Historical Momentum Heuristic. A page is classified as "Decaying" ($1$) if its clicks in the most recent 30-day feature window declined by $\ge 15\%$ compared to the preceding 30-day baseline window ($\frac{\text{clicks}_{\text{recent}} - \text{clicks}_{\text{base}}}{\text{clicks}_{\text{base}}} \le -0.15$).
- **Fair Comparison Justification:** This mirrors the standard industry approach used by SEO practitioners in manual spreadsheet audits.
- **Baseline Performance Metrics (Identical Holdout Split):**
  - **Dataset Base Rate (Majority Class - Non-Decaying):** $74.2\%$ ($25.8\%$ Positive Class Base Rate)
  - **Precision:** 0.61
  - **Recall:** 0.55
  - **$F_1$-Score:** 0.58
  - **ROC-AUC:** 0.62

---

## 4. Model / Analysis

- **Model Selection:** Random Forest Classifier (`n_estimators=200`, `max_depth=8`, `random_state=42`) trained via `scikit-learn`. Chosen for non-linear feature interaction capabilities, stability against outliers, and direct probability calibration outputs.
- **Exact Feature List:**
  1. `total_clicks_60d` (Integer aggregate over pre-cutoff feature window)
  2. `total_impressions_60d` (Integer aggregate over pre-cutoff feature window)
  3. `avg_position_60d` (Float average ranking position)
  4. `avg_ctr_60d` (Float average click-through rate)
  5. `pos_volatility_60d` ($\sigma$ standard deviation of daily ranking position)
  6. `click_velocity_pct` ($\frac{\text{clicks}_{31-60d} - \text{clicks}_{1-30d}}{\text{clicks}_{1-30d}}$)
  7. `impression_velocity_pct` ($\frac{\text{impressions}_{31-60d} - \text{impressions}_{1-30d}}{\text{impressions}_{1-30d}}$)
  8. `position_drift_delta` ($\text{pos}_{31-60d} - \text{pos}_{1-30d}$)
- **Target Definition:** Binary label `is_decaying_target` = $1$ if post-cutoff 30-day clicks experienced a decline $\ge 25\%$ relative to the pre-cutoff recent 30-day window ($\frac{\text{clicks}_{\text{future}} - \text{clicks}_{\text{recent}}}{\text{clicks}_{\text{recent}}} \le -0.25$), else $0$.

---

## 5. Evaluation

- **Validation Split Scheme:** **Strict Out-of-Time Temporal Split**.
  - *Feature Window:* March 1, 2026 – April 30, 2026 (60 days)
  - *Temporal Cutoff:* May 1, 2026
  - *Target Evaluation Window:* May 1, 2026 – May 31, 2026 (30 days)
  - *Rationale:* Temporal splits prevent future data leakage inherent to standard random k-fold cross-validation in time-series search data.

### Comparative Metric Performance (Holdout Set)

| Model / Baseline | Base Rate | Precision | Recall | $F_1$-Score | ROC-AUC | Lift over Baseline |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Historical Momentum Baseline** | 25.8% | 0.61 | 0.55 | 0.58 | 0.62 | Ref Baseline |
| **Random Forest Candidate** | 25.8% | **0.84** | **0.78** | **0.81** | **0.88** | **+39.6% ($F_1$)** |

- **Error Analysis:**
  - *False Positives (16% of predictions):* Primarily caused by highly seasonal keywords experiencing expected post-holiday demand drops that mimicked structural content decay.
  - *False Negatives (22% of actual decay):* Concentrated in long-tail pages with low initial impressions ($100 - 300$ impressions), where statistical variance masked underlying rank shifts.

---

## 6. Interpretation

- **Feature Importance Ranking:**
  1. `position_drift_delta` (Weight: 0.34): The strongest single predictor. Rank drops in the 30 days prior to the cutoff strongly heralded future click collapse.
  2. `click_velocity_pct` (Weight: 0.26): Decelerating click momentum provided early warning before position shifts registered.
  3. `pos_volatility_60d` (Weight: 0.18): High position variance indicated unstable search engine intent matching.
  4. `impression_velocity_pct` (Weight: 0.12): Drops in impressions signaled broader loss of search query relevance.
  5. Base aggregates (`avg_ctr`, `total_clicks`) (Combined Weight: 0.10).
- **Key Insight:** Raw traffic volume alone is a poor indicator of decay risk. A high-traffic page with subtle positive rank drift is far safer than a medium-traffic page experiencing rank volatility and impression contraction.

---

## 7. Recommendation

### Action Playbook for FlyRank Editorial Teams

1. **Critical Decay Tier (Score $\ge 0.80$):**
   - *Action:* Full Content Refresh & Intent Realignment.
   - *Execution:* Update outdated statistics, rewrite metadata, add missing secondary search intent sections, and submit for instant re-indexing.
2. **Moderate Drift Tier (Score $0.50 - 0.79$):**
   - *Action:* Snippet & On-Page Optimization.
   - *Execution:* Optimize Title Tag / Meta Description for higher CTR, internal link insertion from rising core pages.
3. **Stable Core Tier (Score $< 0.50$):**
   - *Action:* Protect & Monitor.
   - *Execution:* No manual intervention; preserve ranking stability.

- **Explicit Limits & Framing:** This scoring framework functions purely as a **predictive decision-support engine**. It highlights probabilistic risk based on observational search features and does not claim causal certainty regarding Google's ranking algorithms.

---

## 8. Reproducibility

### Execution Commands (Fresh Environment)

```bash
# 1. Clone repo and create environment
git clone [https://github.com/Gonablitz/flyrank-capstone.git](https://github.com/Gonablitz/flyrank-capstone.git)
cd flyrank-capstone

# 2. Install required dependencies
pip install duckdb pandas scikit-learn numpy lightgbm matplotlib

# 3. Set Hugging Face Access Token
export HF_TOKEN="your_hf_read_token"

# 4. Execute Feature Engineering & Model Training Pipeline
python work/01_feature_extraction.py
python work/02_model_training_evaluation.py
