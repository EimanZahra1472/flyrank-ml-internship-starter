# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Eiman Zahra
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** [https://github.com/EimanZahra1472/flyrank-ml-internship-starter](https://github.com/EimanZahra1472/flyrank-ml-internship-starter)
- **Date:** August 2026

## 0. Abstract

Prioritizing content inventory for manual editorial refresh under finite operational capacity requires identifying pages experiencing real performance decline without wasting review bandwidth on healthy URLs. Using an anonymized 30,000-page dataset across 32 enterprise clients alongside a 78.8M-row daily performance warehouse from FlyRank, we evaluated rule-based heuristics, current-window classifiers, and forward-looking time-series predictive models. In client-holdout validation on the 30K dataset, Logistic Regression achieved a Precision@20 of 0.700 and Precision@50 of 0.600 against a base decline rate of 0.511, outperforming a volume-weighted baseline rule (Precision@20 = 0.400, Precision@50 = 0.420). However, when evaluating a forward-looking predictive model on the full warehouse, honest time-aware validation caused accuracy to collapse from 0.669 (naive random split) to 0.274, exacerbated by severe autocorrelation (r = 0.819) between historical position and the target outcome. Consequently, we rejected the predictive model for deployment, implementing instead an evidence-backed, transparent action playbook with structured reason codes (`STALE_LOW_CTR` and `PAGE1_LOW_CTR`) to govern human-in-the-loop content refresh queues safely.

## 1. Problem framing

- **Decision Supported:** Identifying which pages in an enterprise content inventory should be audited and refreshed first given strict editorial bandwidth constraints (typically 20–50 pages per weekly cycle).
- **Unit of Analysis:** One row = one content item (page) over 90-day search performance windows.
- **Output:** A ranked priority queue with transparent reason codes and specific editorial actions.
- **Action Taken:** Human editors inspect flagged pages, identify causes for underperformance (e.g., outdated content vs. sub-optimal SERP snippets), and execute updates.
- **Cost of a Wrong Call:** 
  - *False Positive:* Wastes high-cost editorial hours on pages that are already holding steady in SERPs.
  - *False Negative:* Leaves high-potential declining pages decaying unnoticed until ranking recovery requires extensive structural overhauls.
- **Why ML / Data Helps:** Hand-written rules fall into scale traps (e.g., ranking purely by volume) or single-variable blind spots. Combining multi-dimensional observable signals improves triage precision at the top of the queue.

## 2. Data safety

- **Data Sources:** 
  1. `data/raw/content_refresh_anonymized.csv` (30,000 pages, 32 clients, 44 columns).
  2. `FlyRank/internship-warehouse` (`fact_content_daily_performance`, 78.8M rows, 28.9M GSC-active rows; `dim_content`).
- **Deliberate Exclusions & Safety:**
  - Zero client names, domains, URLs, titles, keywords, or raw query text. All entities are pseudonymous hashes.
  - Hashed IDs (`client_id`, `content_id`) were used strictly for joins and grouped holdout splitting—never as model features.
  - Target-derived columns (`trend_direction`, `trend_pct`) were strictly excluded from model feature sets to prevent circular data leakage.
- **Sanitization Confirmation:** No private client identifiers or unapproved data dumps exist anywhere in the repository or report.

## 3. Baseline

- **Rule Logic:** `Baseline Score = (CTR < Position_Tier_Mean_CTR) × (Impressions_90d >= 500) × Impressions_90d`
- **Why It's a Fair Comparison:** Represents the standard practitioner heuristic of prioritizing high-volume pages with below-average CTR for their ranking position tier.
- **Baseline Benchmark Numbers (on client-holdout test split, 6,163 rows, 7 clients, base rate = 0.511):**
  - **Precision@20:** 0.400 (8 / 20)
  - **Precision@50:** 0.420 (21 / 50)
  - *Note:* The baseline underperforms the random base rate because multiplying by raw impressions heavily biases the queue toward page size rather than true decline probability.

## 4. Model / analysis

- **Methods Benchmarked:** Logistic Regression and Random Forest (200 estimators, max_depth=6, class_weight='balanced').
- **Feature Set:** `impressions_90d`, `avg_position`, `ctr`, `content_age_days`, `days_since_last_update`, `word_count`, `sessions_90d`, `engagement_rate`.
- **Target Proxy (Current-Window):** `is_declining_label = (trend_direction == "down")` (1 = declining, 0 = stable/up).

## 5. Evaluation

- **Split Design:** Grouped client holdout via `GroupShuffleSplit(n_splits=1, test_size=0.2, random_state=42)` ensuring 25 training clients (23,837 rows) and 7 test clients (6,163 rows) with zero client overlap.
- **Benchmark Results on Same Test Split:**
  - **Baseline Rule:** Precision@20 = 0.400, Precision@50 = 0.420
  - **Logistic Regression (Winner):** Precision@20 = 0.700, Precision@50 = 0.600 (+42.9% lift over baseline at P@50)
  - **Random Forest:** Precision@20 = 0.500, Precision@50 = 0.580
  - **Test Cohort Base Rate:** 0.511
- **Error Analysis:**
  - False positives in Logistic Regression clustered in two categories:
    1. Ultra-low-volume pages where near-zero CTR reflects sparse sampling noise rather than true decline.
    2. Low-tier ranking positions (positions 30+) where low CTR is structurally expected rather than a symptom of decay.

## 6. Interpretation & Longitudinal Failure Audit

- **Feature Weights in Current-Window Classifier:**
  - Logistic Regression: CTR had the largest negative coefficient ($-0.0528$), confirming that relative CTR underperformance is the strongest linear decline indicator.
  - Random Forest: Leaned heavily on `impressions_90d` (0.303) and `avg_position` (0.228).
- **Longitudinal Warehouse Audit (ML-09):**
  - Evaluating a forward-looking predictive model on 78.8M warehouse facts across consecutive quarters revealed severe out-of-time breakdown:
    - *Naive Random Split:* Accuracy = 0.669
    - *Honest Time-Aware Split (Feb→Mar train, Apr→May test):* Accuracy = **0.274** (worse than majority-class guessing).
  - *Near-Leakage Autocorrelation Finding:* `avg_position` exhibited an $r = 0.819$ correlation with `future_avg_position`. The model learned an autocorrelative shortcut that failed when quarterly SERP dynamics shifted.
  - *Decision:* The predictive forward-looking model was **killed** and disqualified from deployment.

## 7. Recommendation & Action Playbook

- **Deployed System:** Because the predictive model failed longitudinal validation, the production action playbook uses a re-validated, transparent prioritization score on warehouse data:
  $$\text{Priority Score} = \frac{\text{Avg Impressions} \times (\text{Days Since Update} / 365)}{\text{Avg CTR} + 0.005}$$
- **Action Queue Breakdown (March 2026 snapshot, 59 pages):**
  - **`STALE_LOW_CTR` (47 pages / 79.7%):** Aged content with depressed CTR requiring thorough editorial update.
  - **`PAGE1_LOW_CTR` (12 pages / 20.3%):** Page 1 ranking pages (positions 1–10) capturing below 0.50% CTR requiring SERP snippet and title optimization.
- **Human-in-the-Loop Protocol:** Mandatory human verification of page relevance, ongoing staging edits, technical anomaly checks, and tracking tag health before executing refreshes.
- **Operational No-Go List:** No auto-publishing, no automated page pruning, and no client-facing traffic guarantees.

## 8. Reproducibility

- **Repository:** `https://github.com/EimanZahra1472/flyrank-ml-internship-starter`
- **Notebooks:** `work/notebooks/capstone.ipynb` runs top-to-bottom error-free and reproduces all numbers.
- **Receipts:** `work/outputs/w07_metrics.json` records metrics and queue counts.
- **Random Seed:** Fixed to `42` throughout all splits and model initializations.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — [https://flyrank.ai](https://flyrank.ai)
