# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Eiman Zahra
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** [https://github.com/EimanZahra1472/flyrank-ml-internship-starter](https://github.com/EimanZahra1472/flyrank-ml-internship-starter)
- **Date:** August 2026

---

## 0. Abstract

Prioritizing content inventory for manual editorial refresh under finite operational capacity requires identifying pages experiencing real performance decline without wasting review bandwidth on healthy URLs. Using an anonymized 30,000-page dataset across 32 enterprise clients alongside a 78.8M-row daily performance warehouse from FlyRank, we evaluated rule-based heuristics, current-window classifiers, and forward-looking time-series predictive models. In client-holdout validation on the 30K dataset, Logistic Regression achieved a Precision@20 of 0.700 and Precision@50 of 0.600 against a base decline rate of 0.511, outperforming a volume-weighted baseline rule (Precision@20 = 0.400, Precision@50 = 0.420). However, when evaluating a forward-looking predictive model on the full warehouse, honest time-aware validation caused accuracy to collapse from 0.669 (naive random split) to 0.274, exacerbated by severe autocorrelation (r = 0.819) between historical position and the target outcome. Consequently, we rejected the predictive model for deployment, implementing instead an evidence-backed, transparent action playbook with structured reason codes (`STALE_LOW_CTR` and `PAGE1_LOW_CTR`) to govern human-in-the-loop content refresh queues safely.

---

## 1. Problem Framing

- **Decision Supported:** Identifying which pages in an enterprise content inventory should be audited and refreshed first given strict editorial bandwidth constraints (typically 20–50 pages per weekly cycle).
- **Unit of Analysis:** One row = one content item (page) over 90-day search performance windows.
- **Output:** A ranked priority queue with transparent reason codes and specific editorial actions.
- **Action Taken:** Human editors inspect flagged pages, identify causes for underperformance (e.g., outdated content vs. sub-optimal SERP snippets), and execute updates.
- **Cost of a Wrong Call:** 
  - *False Positive:* Wastes high-cost editorial hours on pages that are already holding steady in SERPs.
  - *False Negative:* Leaves high-volume declining assets undetected, resulting in compounding traffic loss.
- **Metric Selection:** Precision@K (Precision@20, Precision@50) evaluated directly against the task base rate.

---

## 2. Data Safety

- **Data Sources:** 
  1. `data/raw/content_refresh_anonymized.csv`: 30,000 rows & 44 columns across 32 clients.
  2. `FlyRank/internship-warehouse`: 78.8M daily fact performance records and 519K dim_content entities.
- **Deliberate Exclusions & Hygiene:**
  - `trend_direction` and `trend_pct` were barred from model feature sets as they directly derive the decline label.
  - Identifiers (`client_id`, `content_id`, `keyword_hash_id`) were strictly isolated for grouping and joins.
  - Excluded AI sessions (`sessions_ai`) from classifier inputs due to high sparsity (0.04% presence).
- **Leakage Audits:**
  - ML-04 demonstrated scale-masking risks: regularized linear models masked leaky features until standardized (AUC jump from 0.802 to 0.999).
  - Anonymization verified: zero client names, domains, or raw search queries present anywhere in `work/` or `docs/`.

---

## 3. Baseline Rule

We established a transparent heuristic score combining volume gating and position-tier relative performance:
$$\text{Baseline Score} = \mathbb{I}(\text{CTR} < \text{Tier Avg CTR}) \times \mathbb{I}(\text{impressions\_90d} \ge 500) \times \text{impressions\_90d}$$

- **Signal Audit Findings:**
  - *Staleness Signal (`days_since_last_update >= 180`):* Stale pages exhibited a 47.1% decline rate vs. 54.2% for fresh pages (**Verdict: MIXED/OPPOSITE**).
  - *CTR Under-Performance vs. Tier:* Pages below tier average CTR had a 57.7% decline rate vs. 40.9% for above-tier (**Verdict: CONFIRMED**).
- **Baseline Performance:** On the test set, Precision@20 = 0.400, Precision@50 = 0.420 (below the 0.511 base rate due to volume-weight bias).

---

## 4. Model / Analysis

- **Task Formulation:** Scoring pages for decline probability on the client-holdout split (80/20 GroupShuffleSplit, 25 train clients, 7 test clients, 0 overlap).
- **Feature Set (8 features):** `impressions_90d`, `avg_position`, `ctr`, `content_age_days`, `days_since_last_update`, `word_count`, `sessions_90d`, `engagement_rate`.
- **Target Definition:** `is_declining_label = (trend_direction == 'down')`.

---

## 5. Evaluation

### Test Split Results (n=6,163, Base Rate = 0.511)

| Model / Baseline | Precision@20 | Precision@50 | Lift (P@20) |
| :--- | :---: | :---: | :---: |
| Baseline Rule | 0.400 | 0.420 | 0.78x |
| Random Forest Classifier | 0.500 | 0.580 | 0.98x |
| **Logistic Regression (Winner)** | **0.700** | **0.600** | **1.37x (+30.0 pp)** |

- **Warehouse Forward Model (ML-09):**
  - Naive Random Split: 66.9% accuracy.
  - Honest Time-Aware Split (Feb->Mar train, Apr->May test): **27.4% accuracy** (collapsed).
  - Autocorrelation finding: $r = 0.819$ between `avg_position` and future position created a fragile shortcut that failed out-of-period. **Model killed.**

---

## 6. Interpretation

- Logistic Regression outperformed Random Forest by maintaining strong linear penalty on low `ctr` (coef = -0.0528) and moderate penalty on staleness (+0.0058), avoiding overfitting to client-specific impression magnitudes.
- The forward modeling failure demonstrated that past position stability does not forecast quarterly inflection points.

---

## 7. Recommendation & Action Playbook

Deployed an interpretable, re-validated action scoring formula over the March 2026 warehouse dataset, outputting a 59-page actionable triage queue:
- **`STALE_LOW_CTR` (47 pages / 79.7%):** Aged content receiving impressions but suffering low click-through rates.
- **`PAGE1_LOW_CTR` (12 pages / 20.3%):** Top-10 ranking pages on Google with CTR < 0.5%, indicating sub-optimal SERP snippets.

### Human No-Go Boundary
- Never auto-publish unreviewed AI rewrites.
- Never auto-delete or de-index candidate URLs based solely on algorithmic priority scores.
- Never quote predictive revenue guarantees to stakeholders based on decision-support rankings.

---

## 8. Reproducibility

- **Repository:** `https://github.com/EimanZahra1472/flyrank-ml-internship-starter`
- **Random Seed:** `42` across all splits and models.
- **Environment:** `python>=3.10`, `duckdb>=1.0.0`, `scikit-learn>=1.3.0`, `pandas>=2.0.0`, `matplotlib>=3.7.0`.

---

## 9. Acknowledgments & Data Credit

Built on the FlyRank ML Internship dataset — [https://flyrank.ai](https://flyrank.ai)
