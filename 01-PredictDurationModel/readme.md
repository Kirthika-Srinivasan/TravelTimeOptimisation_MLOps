# 🚕 01 — Predict Trip Duration Model

Predicts NYC Green Taxi trip duration (in minutes) using January–February 2025 trip data. Covers the full ML workflow: data ingestion, feature engineering, model comparison, and experiment tracking with MLflow.

---

## Problem Statement

Given a taxi trip's pickup zone, dropoff zone, trip distance, and trip type, predict how long the trip will take in minutes. This is a supervised regression problem.

---

## Dataset

**Source:** [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) — Green Taxi Parquet files

| Split | File | Period |
|---|---|---|
| Train | `green_tripdata_2025-01.parquet` | January 2025 |
| Validation | `green_tripdata_2025-02.parquet` | February 2025 |

**Target variable:** `duration` — trip duration in minutes (dropoff time − pickup time)

**Outlier filter:** Trips outside 1–60 minutes are removed (~5% of data).

---

## Features

| Feature | Type | Description |
|---|---|---|
| `PU_DO` | Categorical | Concatenated pickup + dropoff location ID (e.g. `"75_82"`) — captures the full route |
| `trip_type` | Categorical | Street-hail vs dispatch |
| `trip_distance` | Numerical | Distance of the trip in miles |

Features are encoded using **`DictVectorizer`**, which one-hot encodes categoricals and passes numerics through — producing a sparse matrix suitable for linear models and XGBoost.

---

## Models Compared

| Model | Validation RMSE | Notes |
|---|---|---|
| Linear Regression | 6.15 | Baseline |
| **Ridge (α=0.1)** | **6.14** | ✅ Best linear model — L2 regularisation preserves all route features |
| Lasso (α=0.1) | 9.03 | L1 aggressively zeroes out predictive route features — not suitable here |

> **Why Ridge beats Lasso here:** The `PU_DO` route combinations are genuinely predictive. Lasso's feature elimination discards useful signal, significantly hurting accuracy.

---

## Project Structure

```
01-PredictDurationModel/
├── 01-predict-duration.ipynb   # Full notebook: EDA, feature engineering, training, MLflow logging
├── models/
│   └── lin_reg.bin             # Pickled (DictVectorizer, Ridge) tuple (Local)
└── mlflow.db                   # Local SQLite MLflow tracking store
```

---

## How to Run

### 1. Install dependencies

```bash
pip install pandas pyarrow scikit-learn mlflow xgboost hyperopt seaborn matplotlib
```

### 2. Launch MLflow UI

```bash
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

Open [http://localhost:5000](http://localhost:5000) to compare runs.

### 3. Run the notebook

```bash
jupyter notebook 01-predict-duration.ipynb
```

---

## Key Concepts Demonstrated

- **Time-based train/val split** — January for training, February for validation; avoids data leakage
- **Feature engineering** — `PU_DO` combined route feature vs separate location IDs
- **DictVectorizer** — handles mixed categorical/numerical features without manual one-hot encoding
- **Regularisation comparison** — L1 (Lasso) vs L2 (Ridge) on sparse high-cardinality features

---

## Tech Stack

`Python` · `pandas` · `scikit-learn` · `XGBoost` · `Hyperopt` · `Jupyter`
