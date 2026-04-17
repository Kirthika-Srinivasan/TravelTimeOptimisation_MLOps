# Travel Time Optimisation — End-to-End MLOps Pipeline

> A production-grade MLOps pipeline that predicts travel duration between two points, with full experiment tracking, workflow orchestration, model deployment, real-time monitoring, and automated retraining — built on the MLOps Zoomcamp curriculum.

[![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)](https://www.python.org/)
[![MLflow](https://img.shields.io/badge/Tracking-MLflow-0194E2)](https://mlflow.org/)
[![Prefect](https://img.shields.io/badge/Orchestration-Prefect-2563EB)](https://www.prefect.io/)
[![Evidently](https://img.shields.io/badge/Monitoring-Evidently-brightgreen)](https://www.evidentlyai.com/)
[![Grafana](https://img.shields.io/badge/Dashboards-Grafana-F46800)](https://grafana.com/)
[![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900?logo=amazonaws)](https://aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC)](https://www.terraform.io/)

---

## Overview

This project implements a complete end-to-end MLOps workflow for a travel time prediction model. Starting from raw NYC taxi trip data, the pipeline covers every phase of the ML lifecycle: data preprocessing, experiment tracking, model selection, batch and real-time deployment, and production monitoring with automated alerting.

The goal isn't just a working model — it's a reproducible, observable, production-ready system where every model run is tracked, every deployment is automated, and every drift event triggers a retraining signal.

---

## Pipeline Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                   │
│   NYC Taxi Trip Data (Parquet) → Feature Engineering → Train/Val   │
└─────────────────────────┬──────────────────────────────────────────┘
                          │
┌─────────────────────────▼──────────────────────────────────────────┐
│                   EXPERIMENT TRACKING (MLflow)                      │
│   10+ model runs tracked: XGBoost, Random Forest, Linear Regression │
│   Metrics: RMSE, MAE, R²  |  Artifacts: model pkl, feature schema  │
└─────────────────────────┬──────────────────────────────────────────┘
                          │  Best model promoted to registry
┌─────────────────────────▼──────────────────────────────────────────┐
│              WORKFLOW ORCHESTRATION (Prefect)                       │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐    │
│  │   Ingest &  │  │   Train &    │  │   Score & Batch        │    │
│  │   Validate  │─▶│   Evaluate   │─▶│   Deployment           │    │
│  └─────────────┘  └──────────────┘  └────────────────────────┘    │
│  Scheduled flows · Retry logic · Slack notifications on failure    │
└─────────────────────────┬──────────────────────────────────────────┘
                          │
       ┌──────────────────┼────────────────────────┐
       │                  │                         │
┌──────▼──────┐   ┌───────▼──────┐        ┌───────▼──────────┐
│  Batch      │   │  Real-time   │        │  Infrastructure  │
│  Scoring    │   │  REST API    │        │  (Terraform)     │
│  (AWS S3 +  │   │  (Flask /    │        │  AWS Lambda      │
│  Lambda)    │   │  FastAPI)    │        │  S3  Kinesis     │
└──────┬──────┘   └───────┬──────┘        └──────────────────┘
       │                  │
┌──────▼──────────────────▼──────────────────────────────────────────┐
│                    MONITORING LAYER                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  Evidently   │  │  Prometheus  │  │        Grafana           │ │
│  │  Data Drift  │  │  Metrics     │  │   Drift + Perf Dashboard │ │
│  │  Detection   │  │  Collection  │  │   Alerting Rules         │ │
│  └──────┬───────┘  └──────────────┘  └──────────────────────────┘ │
│         │ drift detected → retraining trigger                      │
└─────────┼──────────────────────────────────────────────────────────┘
          │
┌─────────▼──────────────────────────────────────────────────────────┐
│              CI/CD (GitHub Actions)                                 │
│  lint → test → build → deploy  |  Triggered on main branch push    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Module Breakdown

### `01-PredictDurationModel` — Baseline Model
Exploratory data analysis on NYC Taxi trip data. Feature engineering: pickup/dropoff location encoding, datetime features, trip distance calculation. Trains a baseline Linear Regression model and establishes evaluation benchmarks (RMSE target).

### `02-ExperimentTracking` — MLflow Tracking
Registers all model runs in MLflow with full parameter and metric logging. Experiments with XGBoost, Random Forest, and Linear Regression. Best model is promoted to the MLflow Model Registry with staging → production lifecycle management.

**Key result:** XGBoost with tuned hyperparameters achieved best RMSE. All 10+ runs tracked with reproducible configs.

### `03-WorkflowOrchestration` — Prefect Pipelines
Transforms the ad-hoc training scripts into a scheduled, observable Prefect flow. Includes retry logic, parameterised runs, and failure notifications. The training pipeline runs on a weekly schedule; the scoring pipeline runs daily on new trip data.

### `04-ModelDeployment` — Batch & Real-Time Serving
**Batch:** AWS Lambda reads new trip data from S3, runs inference, and writes predictions back to S3. Triggered on a schedule via AWS EventBridge.

**Real-time:** Flask REST API wraps the MLflow model artifact for synchronous predictions. Deployed to AWS EC2 with a `/predict` endpoint.

### `05-ModelMonitoring` — Drift Detection & Alerting
Evidently generates data drift reports comparing the current week's feature distributions against the training baseline. Reports are ingested by Prometheus and visualised in Grafana. A Grafana alert fires when the dataset drift score exceeds a threshold, triggering a Prefect retraining flow.

**Monitored metrics:** Feature drift (PSI score), prediction distribution shift, input data schema validation.

### `06-BestPractices` — Production Hardening
- Unit and integration tests with pytest
- Pre-commit hooks (black, isort, flake8)
- Makefile for reproducible dev commands
- Docker containerisation for all services
- GitHub Actions CI/CD: lint → test → build → deploy on every push to `main`
- Terraform manages all AWS infrastructure (Lambda, S3, EC2, IAM roles)

---

## Tech Stack

| Category | Tools |
|---|---|
| Model Training | scikit-learn, XGBoost |
| Experiment Tracking | MLflow |
| Workflow Orchestration | Prefect |
| Batch Deployment | AWS Lambda, S3 |
| Real-time Deployment | Flask, FastAPI, AWS EC2 |
| Streaming | AWS Kinesis |
| Monitoring | Evidently, Prometheus, Grafana |
| CI/CD | GitHub Actions |
| Infrastructure as Code | Terraform |
| Containerisation | Docker |
| Testing | pytest |

---

## Dataset

**NYC TLC Trip Record Data** — publicly available from the [NYC Taxi & Limousine Commission](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page).

- Source: Yellow/Green taxi trip records (Parquet format)
- Target variable: `trip_duration` (seconds)
- Features: pickup/dropoff location IDs, pickup datetime, passenger count, trip distance
- Size: ~1M rows per month

---

## What I Learned

- **Orchestration pays off immediately.** Moving from ad-hoc scripts to Prefect flows meant I could catch data ingestion failures before they silently corrupted model training.
- **Monitoring is the hardest part.** Getting Evidently drift reports into Prometheus required custom metric exporters — the integration isn't plug-and-play.
- **Terraform for ML infra is worth the upfront cost.** Reproducing the full AWS stack from scratch takes under 5 minutes with `terraform apply` vs. hours of manual console work.
- **The RMSE number matters less than the pipeline.** The baseline linear model was already reasonable. The real engineering challenge was making the full loop — ingest → train → deploy → monitor → retrain — reliable and automated.

---

## References

- [MLOps Zoomcamp](https://github.com/DataTalksClub/mlops-zoomcamp) by DataTalks.Club
- [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
- [Evidently AI Documentation](https://docs.evidentlyai.com/)
- [MLflow Documentation](https://mlflow.org/docs/latest/index.html)

---

## Author

**Kirthika Srinivasan** — Applied AI Engineer | Melbourne, VIC  
[LinkedIn](https://www.linkedin.com/in/kirthikasrinivasan) · [GitHub](https://github.com/Kirthika-Srinivasan)
