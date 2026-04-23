# 🔬 02 — Experiment Tracking with MLflow

Extends the duration prediction model with systematic hyperparameter search using **Hyperopt + XGBoost**, full experiment tracking via **MLflow**, and model artifact logging in both pickle and native XGBoost formats.

---

## What This Module Covers

| Topic | Detail |
|---|---|
| Experiment tracking | MLflow runs with params, metrics, tags, and artifacts |
| Hyperparameter tuning | Bayesian optimisation via Hyperopt (50 trials) |
| Model comparison | Ridge vs XGBoost (tuned) on the same validation set |
| Autologging | `mlflow.xgboost.autolog()` for zero-boilerplate tracking |
| Model saving | Pickle artifacts vs `mlflow.xgboost.log_model` |
| Preprocessor versioning | DictVectorizer saved and logged alongside the model |
| GBRegressor Experiment with MLflow | Improving rmse using GradientBoostingRegressor and RandomForest with Hyperopt|
| MLflow's Model Registry | Using MLflowClient to interact with MLflow server, Model Registry, and selecting the production model |

---

## Experiment Setup

**Tracking URI:** `sqlite:///mlflow.db` (local SQLite)

**Experiments:**

| Experiment Name | Description |
|---|---|
| `nyc-taxi-experiment` | Linear model runs (Linear Regression, Lasso, Ridge) |
| `xgboost-hyperopt-tuning` | 50 Hyperopt XGBoost trials |

Each run is tagged with `developer` and the model type for easy filtering in the MLflow UI.

---

## Hyperparameter Search

**Algorithm:** TPE (Tree-structured Parzen Estimator) via `hyperopt.tpe.suggest` — Bayesian optimisation that learns from prior trials to sample promising regions of the search space.

**Search space:**

| Hyperparameter | Distribution | Range |
|---|---|---|
| `max_depth` | Uniform int | 4 – 100 |
| `learning_rate` | Log-uniform | ~0.05 – 1.0 |
| `reg_alpha` (L1) | Log-uniform | ~0.007 – 0.37 |
| `reg_lambda` (L2) | Log-uniform | ~0.002 – 0.37 |
| `min_child_weight` | Log-uniform | ~0.37 – 20 |

**Trials:** 50 — each trial is a full MLflow run with its own params and RMSE metric.

**Early stopping:** `early_stopping_rounds=50` — each XGBoost run stops when validation RMSE stops improving, preventing overfitting and reducing wasted compute.

---

## Best Model

After 50 Hyperopt trials, the best XGBoost configuration was selected based on **lowest validation RMSE**. The winning parameters were retrained with `mlflow.xgboost.autolog()` enabled:

```python
params = {
    'learning_rate': 0.2528,
    'max_depth': 5,
    'min_child_weight': 5.181,
    'reg_alpha': 0.0326,
    'reg_lambda': 0.0437,
    'objective': 'reg:linear',
    'seed': 42
}
```

> To identify your best run: open the MLflow UI → `xgboost-hyperopt-tuning` experiment → sort by `rmse` ascending → top row is your best model.

---

## Model Artifact Logging

Two patterns are demonstrated — both valid, with different trade-offs:

### Way 1 — Pickle artifact
```python
mlflow.log_artifact(local_path="models/lin_reg.bin", artifact_path="models_pickle")
```
- Simple, framework-agnostic
- Requires manual loading with `pickle`
- Not compatible with MLflow Model Registry's serving infrastructure

### Way 2 — Native MLflow model (recommended)
```python
mlflow.xgboost.log_model(booster, name="models_mlflow")
```
- Enables MLflow Model Registry, model versioning, and `mlflow models serve`
- Includes model schema and environment dependencies
- Preferred for production pipelines

### Preprocessor versioning
```python
mlflow.log_artifact("models/preprocessor.b", artifact_path="preprocessor")
```
The `DictVectorizer` is versioned alongside the model — critical for reproducible inference, since the same vectorizer used at training time must be used at prediction time.

---

## Project Structure

```
02-ExperimentTracking/
├── running-mlflow-examples/
│   ├── scenario-1.ipynb
|   ├── scenario-2.ipynb                
│   └── scenario-3.ipynb         
├── 01-predict-duration.ipynb   # Notebook with full experiment tracking workflow
├── 02-mlflow-improve-rmse.ipynb
├── 03-model-registry.ipynb
├── models/
│   ├── lin_reg.bin             # Pickled (DictVectorizer, Ridge) — best linear model
│   └── preprocessor.b         # Pickled DictVectorizer
└── mlflow.db                   # SQLite tracking store (all runs)
```

---

## How to Run

### 1. Install dependencies

```bash
pip install pandas pyarrow scikit-learn mlflow xgboost hyperopt
```

### 2. Launch MLflow UI

```bash
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

Open [http://localhost:5000](http://localhost:5000)

### 3. Run the notebook

```bash
jupyter notebook 01-predict-duration.ipynb
```

Execute from the **"Tracking this experiment with XGBoost, HyperOp"** cell onwards for the experiment tracking section.

---

## How to Compare Runs in MLflow UI

1. Open the `xgboost-hyperopt-tuning` experiment
2. Select all 50 runs → click **Compare**
3. Plot `rmse` vs `learning_rate`, `max_depth` etc. to visualise the search landscape
4. Sort by `rmse` ascending to identify the best configuration
5. Navigate to the best run → **Artifacts** tab → register to Model Registry if promoting to staging/production

---

## Key Concepts Demonstrated

- **Bayesian hyperparameter optimisation** — more efficient than grid search or random search; uses prior trial results to guide sampling
- **MLflow autologging** — `mlflow.xgboost.autolog()` captures all params, metrics, and model without manual `log_param` calls
- **Experiment isolation** — separate MLflow experiments for linear models vs XGBoost keeps the UI clean and comparable
- **Preprocessor co-versioning** — logging the `DictVectorizer` alongside the model ensures inference reproducibility
- **Early stopping as regularisation** — prevents XGBoost from overfitting without manual round selection

---

### Bonus - Three MLflow setups according to the scenario
| Scenario | MLflow Setup |
|---|---|
| A single data scientist participating in an ML competition | Tracking server: no, Backend store: local filesystem, Artifacts store: local filesystem |
| A cross-functional team with one data scientist working on an ML model | tracking server: yes, local server, backend store: sqlite database, artifacts store: local filesystem |
| Multiple data scientists working on multiple ML models | Tracking server: yes, remote server (EC2), Backend store: PostgreSQL database, Artifacts store: S3 bucket. |

## Tech Stack

`Python` · `XGBoost` · `Hyperopt` · `MLflow` · `scikit-learn` · `pandas` · `SQLite` . `PostgreSQL`. `EC2` . `S3`
