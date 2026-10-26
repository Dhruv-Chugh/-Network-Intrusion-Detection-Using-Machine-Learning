# PRD: ML-Based Network Intrusion Detection System (ML-NIDS) with Streamlit GUI

**Version:** 1.0
**Target environment:** Antigravity (agentic IDE), Python 3.10+
**Status:** Draft, see "Open Questions" at the end

---

## 1. Overview

Build an end-to-end machine learning pipeline that classifies network flow records as **Normal** or **Attack**, compares four classifiers (Logistic Regression, Decision Tree, Random Forest, SVM), and exposes everything through a simple Streamlit web interface. The interface must let a user upload traffic data, get per-flow classifications, view model metrics, and see which features drive detection.

## 2. Goals and Non-Goals

### Goals
1. Train and compare **LR, DT, RF, SVM** on the same data split with identical preprocessing.
2. Evaluate with **Accuracy, Precision, Recall, F1, False Alarm Rate (FAR)** plus confusion matrices (accuracy alone is misleading on imbalanced data).
3. Provide a **Streamlit GUI** to upload a CSV of flows and label each as Normal or Suspicious/Attack.
4. Show **feature importance** (which network features contribute most to detecting malicious traffic).
5. Be reproducible: one command to train, one command to run the app.

### Non-Goals
- Real-time packet capture or line-rate inline inspection.
- Deep learning models (out of scope for v1).
- Raw PCAP to flow extraction (v1 consumes already-extracted flow CSVs; optional stretch goal).

## 3. Users

| User | Need |
|---|---|
| Student / researcher | Reproducible comparison of 4 classifiers with clear metrics |
| SOC analyst (demo persona) | Upload flow CSV, see flagged flows and why |

## 4. Dataset

### Primary: UNSW-NB15
- Hybrid of real normal traffic and synthetic modern attacks.
- 9 attack categories: Fuzzers, Analysis, Backdoors, DoS, Exploits, Generic, Reconnaissance, Shellcode, Worms.
- **Use the pre-split files** `UNSW_NB15_training-set.csv` (~175k rows) and `UNSW_NB15_testing-set.csv` (~82k rows). They are clean, small, and train fast on a laptop.
- Key columns: `label` (0 = normal, 1 = attack), `attack_cat` (attack type), plus ~42 flow features (`dur`, `proto`, `service`, `state`, `sbytes`, `dbytes`, `sttl`, etc.).
- Source: UNSW Canberra project page (search "UNSW-NB15 dataset UNSW Canberra") or Kaggle mirrors.

### Optional secondary: CICIDS-2017
- ~78 CICFlowMeter features, 15 classes (benign plus 14 attacks), 2.8M+ rows, heavily imbalanced.
- Use the `MachineLearningCSV` version from the Canadian Institute for Cybersecurity (UNB CIC datasets page).
- Needs sampling or chunked loading; SVM will be impractical on the full set.

### Dataset decision for v1
UNSW-NB15 (pre-split files), **binary classification** using the `label` column. CICIDS-2017 is a stretch goal for cross-dataset validation.

### Data rules
- Put raw files in `data/raw/` (never committed to git; add to `.gitignore`).
- **Drop `id` and `attack_cat` from features.** `attack_cat` leaks the label. Keep `attack_cat` only for per-attack-type analysis.
- Use the provided test file as the final holdout. Do not tune on it.

## 5. Functional Requirements

### FR1: Data preprocessing (`src/preprocess.py`)
1. Load CSVs; report shape, dtypes, class balance.
2. Replace `inf`/`-inf` with NaN; drop columns with >50% nulls; impute the rest (median for numeric, mode for categorical).
3. Drop duplicate rows and ID columns.
4. Drop one of each highly correlated pair (|r| > 0.95); save correlation heatmap to `reports/figures/`.
5. Encode categoricals (`proto`, `service`, `state`) with One-Hot Encoding.
6. Scale numeric features with `StandardScaler`.
7. Wrap everything in a scikit-learn `Pipeline`/`ColumnTransformer`, **fit on train only**, and save with `joblib` so the GUI applies identical transforms.

### FR2: Class imbalance handling
- Check class balance. UNSW-NB15 binary is moderately balanced, so default to `class_weight="balanced"`.
- Provide an optional **SMOTE** flag (`imblearn`) applied **only to the training split, after splitting, inside the training pipeline**.

### FR3: Model training (`src/train.py`)
Train all four with a consistent config:

| Model | Notes |
|---|---|
| Logistic Regression | `max_iter=1000`, baseline |
| Decision Tree | cap `max_depth` / `min_samples_leaf` to limit overfitting |
| Random Forest | ~200 trees, `n_jobs=-1` |
| SVM | `LinearSVC` (or `SVC` RBF on a stratified subsample of ~30-50k rows, since training is O(n^2)-O(n^3)) |

- Use 5-fold stratified CV on the train set for hyperparameter tuning (`GridSearchCV` or `RandomizedSearchCV`, scoring = F1).
- Save each model to `models/<name>.joblib` and metrics to `reports/metrics.json`.
- Fixed `random_state=42` everywhere.

### FR4: Evaluation (`src/evaluate.py`)
For each model on the held-out test set, compute and save:
- Accuracy, Precision, Recall, F1, **FAR = FP / (FP + TN)**
- Confusion matrix plot
- ROC curve and AUC (use `decision_function` for SVM)
- Training time and inference time per 1,000 flows
- Comparison table (CSV and Markdown) across all four models

### FR5: Feature importance
- Random Forest and Decision Tree: `feature_importances_` (top 15 bar chart).
- Logistic Regression: coefficient magnitude and sign.
- Map one-hot columns back to readable names.
- Optional: permutation importance on the test set for a model-agnostic view.

### FR6: Streamlit app (`app/app.py`)
Pages/tabs:
1. **Analyze Traffic**
   - Upload CSV of flows (with a downloadable sample file).
   - Select a model from a dropdown (default: Random Forest).
   - Validate columns; show clear errors for missing or extra columns.
   - Output table with a `Prediction` column (Normal / Attack) and attack probability; color-code attacks.
   - Summary: total flows, % flagged, count by label; download results as CSV.
   - If the upload contains a `label` column, also show metrics against it.
2. **Model Comparison**: metrics table and bar chart for all four models, confusion matrices, ROC curves.
3. **Feature Insights**: top-feature charts per model plus a short plain-English explanation of why those features matter (volume, timing, protocol/state).
4. **Dataset Explorer (EDA)**: class distribution, feature histograms, correlation heatmap.

## 6. Non-Functional Requirements
- Training the full pipeline (excluding RBF SVM) completes in under ~15 minutes on a standard laptop CPU.
- App loads a prediction for 10k flows in under ~10 seconds (use `st.cache_resource` for models).
- No hard-coded absolute paths; config in `config.yaml`.
- Code typed where reasonable, with docstrings, and a `README.md` with setup steps.

## 7. Project Structure

```
ml-nids/
├── PRD.md
├── README.md
├── requirements.txt
├── config.yaml
├── data/
│   ├── raw/            # downloaded datasets (gitignored)
│   └── sample/         # small sample CSV for the demo
├── models/             # saved .joblib pipelines
├── reports/
│   ├── metrics.json
│   └── figures/
├── notebooks/
│   └── 01_eda.ipynb
├── src/
│   ├── preprocess.py
│   ├── train.py
│   ├── evaluate.py
│   └── explain.py
├── app/
│   └── app.py
└── tests/
    └── test_pipeline.py
```

**Dependencies:** pandas, numpy, scikit-learn, imbalanced-learn, matplotlib, seaborn, plotly, streamlit, joblib, pyyaml, pytest.

## 8. Milestones (suggested order for the agent)

1. **Setup:** scaffold repo, `requirements.txt`, `config.yaml`, README stub.
2. **EDA:** notebook with shapes, class balance, nulls, correlations. Report findings before moving on.
3. **Preprocessing:** build and save the transform pipeline; unit test that it handles unseen categories and inf values.
4. **Training:** train the four models, tune, save artifacts.
5. **Evaluation:** metrics, plots, comparison table.
6. **Explainability:** feature importance outputs.
7. **GUI:** Streamlit app wired to saved artifacts.
8. **Polish:** README, sample data, tests, final report summary.

## 9. Acceptance Criteria
- [ ] All four models trained and saved; reproducible with a fixed seed.
- [ ] Metrics table includes Accuracy, Precision, Recall, F1, FAR for every model.
- [ ] No data leakage: scaler/encoder/SMOTE fit on train only; `attack_cat` and `id` excluded from features.
- [ ] Streamlit app accepts a CSV, returns Normal/Attack per row, and shows feature importance.
- [ ] App rejects malformed CSVs with a clear message instead of crashing.
- [ ] `README.md` lets a new user go from clone to running app in under 10 minutes.
- [ ] Random Forest expected to be best or near-best on F1 with FAR in the low single digits (reference: Random Forest on UNSW-NB15 in the 2024 MDPI *Algorithms* paper, about 98.6% accuracy, 97.8% F1, 1.36% FAR). Treat these as a sanity benchmark, not a target to force.

## 10. Risks and Notes
- **Suspiciously perfect scores.** The reference paper reports ~100% for Decision Tree and ~99.99% for SVM under some configurations. Scores that high usually signal leakage, duplicates across train/test, or overfitting. Verify with duplicate checks and the official train/test split before trusting them.
- **Benchmark vs reality.** High accuracy on UNSW-NB15 overstates real-world performance; document concept drift and the need for retraining.
- **SVM scaling.** Full RBF SVM on large data is impractical; use LinearSVC or subsample, and document the choice.
- **Latency.** Random Forest is heavier than LR at inference; the GUI targets batch/forensic analysis, not inline blocking.
- **Reference implementations** (for inspiration only, do not copy blindly): `ShreyaJ-27/Intrusion-Detection-System` and `HanazonoArchive/ML-IDS-Streamlit` on GitHub.

## 11. Open Questions (to confirm before/while building)
1. UNSW-NB15 only, or also CICIDS-2017?
2. Binary (Normal vs Attack) only, or also multi-class attack type?
3. Hardware: CPU only? RAM size? (affects SVM and CICIDS choices)
4. Is this for a course/assignment with required deliverables (report, slides, specific metrics)?
5. Should the GUI accept raw PCAP files, or flow CSVs only?
6. Any preferred theme/branding for the Streamlit app?
