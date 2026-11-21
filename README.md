# CERT r4.2 Insider Threat Detection: Phase 1 Report & Phase 2 ML Roadmap

Behavioral anomaly detection and Security Operations Center (SOC) review budget optimization across daily employee telemetry (`user-day` granularity) on the **CERT Insider Threat Dataset v4.2 (`r4.2`)**[cite: 3].

**Project Team:** Alieldin Tamer | Simon Ehab | Essameldin Amr[cite: 3]

---

## Key Phase 1 Highlights

* **Dataset Scope:** `329,987` clean `user-day` records across `1,000` employees and `501` days (`521` positive insider days; `0.158%` base prevalence)[cite: 3].
* **Leakage Prevention:** Strict chronological split (`CUT = 2010-12-31`), zero data leakage verified across `8` structural code assertions, and an `8`-run label-shuffle permutation test (`0.0493` real test PR-AUC vs. `0.0020` median shuffled)[cite: 3].
* **SOC Efficiency:** Under a top-`1%` daily review budget (`840` reviews out of `83,913` holdout test days), the baseline model catches **13 of 18 test insiders** (`36.2%` recall@budget; `47` of `130` positive days) and saves **20,768 analyst hours** (`210h` required vs. `20,978h` for full review at `15` minutes/review)[cite: 3].

---

## Pipeline Architecture (Phase 1 Implemented → Phase 2 Planned)

1. **1. CERT Dataset** — `Logon · Device · File · Email` (`4,335,799` raw events across `1,000` users)[cite: 3].
2. **2. Preprocessing & EDA** — Timestamp parsing, chunked email processing (`500k` chunks), and quality validation (`0` invalid dates or duplicate IDs)[cite: 3].
3. **3. User-Day Aggregation** — Outer-join daily activity summaries (`330,452` user-days before filtering)[cite: 3].
4. **4. Feature Engineering** — `17` base behavioral and weekend context features (`user_day.parquet`)[cite: 3].
5. **5. Evidence-Based Labeling** — `521` positive non-web insider days (`y = 1`); excludes `445` web-only malicious days and `20` victim-account days (`329,987` clean rows)[cite: 3].
6. **6. Temporal Data Splitting** — Fit (`Jan–Sep 2010`: `189,007` rows), Validation (`Oct–Dec 2010`: `57,067` rows), Full Train (`Jan–Dec 2010`: `246,074` rows), and Holdout Test (`Jan–May 2011`: `83,913` rows; `CUT = 2010-12-31`)[cite: 3].
7. **7. UserBaseline** — Per-user z-score deviations (`[-10, 10]`, fallback to global stats when `min_days < 5`) fitted strictly on benign (`y == 0`) training days across `9` features (`26` total features)[cite: 3].
8. **8. StandardScaler** — Feature standardization fitted strictly on training data[cite: 3].
9. **9. Logistic Regression** — Phase 1 baseline classifier (`class_weight="balanced"`, tuned `C = 0.01`)[cite: 3].
10. **10. Risk Scores & Evaluation** — Holdout PR-AUC (`0.0493`), top-`1%` budget (`840` reviews, `13/18` insiders caught) — **Phase 1 Implemented**[cite: 3].

**Phase 2 (Planned):** `Isolation Forest · Random Forest / LightGBM · LOF · SHAP · Analyst Watchlist`[cite: 3].

---

## Holdout Test Results (Jan–May 2011)

| Strategy | Flagged Days | TP | FP | FN | TN | Precision | Recall | F1 | Insiders Caught |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Standard Threshold (`p >= 0.5`) | 4,151 | 105 | 4,046 | 25 | 79,737 | 0.025 | 0.808 | 0.049 | 18 / 18 |
| **Top 1% SOC Budget (`840` Reviews)** | **840** | **47** | **793** | **83** | **82,990** | **0.056** | **0.362** | **0.097** | **13 / 18** |

* **Top Positive Risk Drivers:** `usb_connects` (`+1.10`), `emails` (`+0.98`), `emails_dev` (`+0.74`), `is_weekend` (`+0.70`), `ext_emails_dev` (`+0.69`)[cite: 3].
* **Top Negative Risk Drivers:** `email_size_log` (`-1.01`), `file_copies` (`-0.98`), `wk_logons` (`-0.79`)[cite: 3].

---

## Phase 2 Roadmap

1. **Data Expansion:** Ingest `http.csv` (recovering `445` web-only days), `LDAP/` organizational structure, and `psychometric.csv`[cite: 3].
2. **Temporal & Peer Features:** Add `7d` / `14d` / `30d` rolling baselines, peer-group deviations, and cross-channel event sequences[cite: 3].
3. **Model Benchmarking:** Compare `LightGBM`, `XGBoost`, `CatBoost`, and `Isolation Forest` / anomaly detectors against the Phase 1 baseline[cite: 3].
4. **SOC Operations:** Calibrate probabilities, aggregate entity-level risk scores, sweep review budgets (`0.5%`–`5.0%`), and add SHAP alert explanations[cite: 3].
5. **Production Packaging:** Refactor into modular Python code (`src/`), persist `.joblib` artifacts, and finalize reporting outputs[cite: 3].
