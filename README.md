# 🛡️ Machine Learning Network Intrusion Detection System (ML-NIDS)

An end-to-end Machine Learning pipeline that analyzes, classifies, and evaluates network traffic flows as **Normal** or **Attack** on the **UNSW-NB15** cybersecurity benchmark dataset.

---

## 📌 Project Overview

This repository provides a modular, leak-free ML pipeline comparing four core classification models (**Decision Tree**, **Random Forest**, **Logistic Regression**, and **Linear SVM**) across critical security metrics, including **Accuracy**, **Precision**, **Recall (Detection Rate)**, **F1-Score**, and **False Alarm Rate (FAR)**.

The project is structured into two self-contained, reproducible Jupyter Notebooks:
1. **[01_eda_and_data_cleaning.ipynb](notebooks/01_eda_and_data_cleaning.ipynb)**: Exploratory Data Analysis, anomaly detection, categorical sanitization, traffic behavioral profiling, and multicollinearity heatmaps.
2. **[02_model_training_and_evaluation.ipynb](notebooks/02_model_training_and_evaluation.ipynb)**: Train-test splitting, feature scaling (`StandardScaler`) & one-hot encoding (`OneHotEncoder`), multi-model benchmarking, confusion matrices, ROC/PR curves, feature importance extraction, and artifact serialization.

---

## 📊 Model Evaluation Benchmark

Evaluated on the held-out test split (**82,332 flows**: 45,332 attack flows, 37,000 normal flows):

| Model | Accuracy | Precision | Recall (Detection Rate) | F1-Score | FAR (False Alarm Rate) | ROC-AUC | Inference Latency |
|---|---|---|---|---|---|---|---|
| **Decision Tree** | **91.64%** | **90.12%** | 95.24% | **92.61%** | **12.79%** | 0.9711 | 0.56 ms / 1k flows |
| **Random Forest** | **91.08%** | 88.28% | 96.63% | **92.27%** | 15.72% | **0.9838** | 2.84 ms / 1k flows |
| **Logistic Regression** | 83.59% | 80.28% | 93.05% | 86.20% | 28.01% | 0.9561 | 0.15 ms / 1k flows |
| **Linear SVM** | 81.19% | 75.13% | **98.42%** | 85.21% | 39.92% | 0.9532 | 0.94 ms / 1k flows |

> **Key Cybersecurity Takeaways:**
> - **False Alarm Rate ($FAR = \frac{FP}{FP + TN}$)**: Measures normal traffic mistakenly flagged/blocked. **Decision Tree** and **Random Forest** achieve the lowest FAR while maintaining over 95% attack recall.
> - **Detection Drivers (Feature Importance)**: `sttl` (Source-to-Destination Time-to-Live) and `ct_state_ttl` are the most decisive features for separating attack traffic from legitimate communication.

---

## 📁 Repository Structure

```
├── PRD.md                                      # Project Requirements Document
├── README.md                                   # Project documentation & overview
├── .gitignore                                  # Git ignore rules
├── Training and Testing Sets/                  # Pre-split UNSW-NB15 dataset files
│   ├── UNSW_NB15_training-set.csv              # ~175k training flows
│   └── UNSW_NB15_testing-set.csv               # ~82k testing flows
├── notebooks/                                  # Jupyter Notebooks
│   ├── 01_eda_and_data_cleaning.ipynb          # Notebook 1: EDA & Data Cleaning
│   └── 02_model_training_and_evaluation.ipynb # Notebook 2: Training, Scaling & Evaluation
├── models/                                     # Serialized scikit-learn models & pipelines
│   ├── best_model.joblib                       # Production end-to-end inference pipeline
│   ├── preprocessor.joblib                     # Fitted ColumnTransformer (Scaler + Encoder)
│   ├── decision_tree.joblib                    # Fitted Decision Tree classifier
│   ├── random_forest.joblib                    # Fitted Random Forest classifier
│   ├── logistic_regression.joblib              # Fitted Logistic Regression classifier
│   └── linear_svm.joblib                       # Fitted Linear SVM classifier
└── reports/                                    # Evaluation reports and figures
    ├── metrics.json                            # JSON export of evaluation metrics
    ├── model_comparison.csv                   # CSV table comparing all models
    └── figures/
        ├── class_distribution.png              # Class & attack category distributions
        ├── correlation_heatmap.png             # Pearson correlation heatmap
        ├── confusion_matrices.png              # 2x2 grid of annotated confusion matrices
        ├── roc_curves.png                      # Multi-model ROC and Precision-Recall curves
        └── feature_importance.png              # Top feature importance rankings
```

---

## 🚀 Quickstart & Setup

### 1. Install Dependencies
```bash
pip install numpy pandas scikit-learn matplotlib seaborn joblib
```

### 2. Run the Notebooks
Open either notebook in VS Code or Jupyter Lab:
- Run `notebooks/01_eda_and_data_cleaning.ipynb` for dataset inspection and cleaning.
- Run `notebooks/02_model_training_and_evaluation.ipynb` for model training, evaluation, and pipeline export.

### 3. Load & Predict using the Saved Pipeline
```python
import joblib
import pandas as pd

# Load the production pipeline (handles preprocessing + classification)
pipeline = joblib.load('models/best_model.joblib')

# Predict on new flow data (DataFrame with the original flow feature columns)
predictions = pipeline.predict(new_flows_df)
probabilities = pipeline.predict_proba(new_flows_df)[:, 1]
```
