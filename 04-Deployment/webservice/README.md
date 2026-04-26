# 🌐 04 — Deployment: Model as a Web Service

Deploys the trained XGBoost trip duration model as a **REST API** using Flask, served with Gunicorn in production, and packaged into a **Docker container** for portable deployment.

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Docker Container                  │
│                                                     │
│   ┌─────────────┐        ┌─────────────────────┐   │
│   │   Gunicorn  │ ──────▶│   Flask App         │   │
│   │  (WSGI)     │        │   predict.py        │   │
│   └─────────────┘        │                     │   │
│                           │  DictVectorizer     │   │
│   PORT 9696               │  XGBoost Booster    │   │
└───────────┬───────────────┴─────────────────────────┘
            │
            ▼
    POST /predict
    { pickup, dropoff, distance }
         │
         ▼
    { duration_minutes: 12.4 }
```

| Layer | Tool | Purpose |
|---|---|---|
| Prediction logic | `predict.py` | Loads model, transforms input, returns prediction |
| Dev server | Flask | Lightweight HTTP server for local testing |
| Production server | Gunicorn | Multi-worker WSGI server — handles concurrent requests |
| Container | Docker | Packages app + dependencies for reproducible deployment |

---

## Project Structure

```
04-Deployment/webservice/
├── predict.py          # Flask app — loads model and serves /predict endpoint
├── test.py             # Test client — sends a sample request to the running service
├── Pipfile             # Dependency spec (scikit-learn, flask, gunicorn)
├── Pipfile.lock        # Pinned dependency versions for reproducibility
└── Dockerfile          # Container definition
```

---

## Setup & How to Run

### 1. Check your scikit-learn version

The model was trained with `scikit-learn==1.6.0` — the serving environment must match to avoid deserialization errors when loading the pickled `DictVectorizer`.

```bash
pip freeze | grep scikit-learn
# scikit-learn==1.6.0
```

### 2. Create the virtual environment with Pipenv

```bash
pipenv install scikit-learn==1.6.0 flask --python=3.9.12
```

Install dev dependencies (test client):
```bash
pipenv install --dev requests
```

Install Gunicorn for production serving:
```bash
pipenv install gunicorn
```

### 3. Activate the environment

```bash
pipenv shell
PS1="> "   # optional — shortens the prompt inside the virtualenv
```

---

## Running the Service

### Development — Flask

Use Flask's built-in server for local testing only. Not suitable for production (single-threaded, no process management).

```bash
python predict.py
```

Service runs at `http://localhost:9696`

### Production — Gunicorn

Gunicorn is a production-grade WSGI server that spawns multiple worker processes to handle concurrent requests.

```bash
gunicorn --bind=0.0.0.0:9696 predict:app
```

---

## Testing the Service

With either Flask or Gunicorn running, send a test request:

```bash
python test.py
```

Or with `curl`:
```bash
curl -X POST http://localhost:9696/predict \
  -H "Content-Type: application/json" \
  -d '{"PULocationID": "75", "DOLocationID": "82", "trip_distance": 2.5}'
```

Expected response:
```json
{"duration": 12.4}
```

---

## Docker Deployment

### Build the image

```bash
docker build -t ride-duration-prediction-service:v1 .
```

### Run the container

```bash
docker run -it --rm -p 9696:9696 ride-duration-prediction-service:v1
```

| Flag | Meaning |
|---|---|
| `-it` | Interactive mode — see logs in terminal |
| `--rm` | Automatically remove container when stopped |
| `-p 9696:9696` | Map host port 9696 to container port 9696 |

The service is now reachable at `http://localhost:9696/predict` exactly as before — the Docker layer is transparent to the client.

### Dockerfile overview

```dockerfile
FROM python:3.9.12-slim

RUN pip install pipenv

WORKDIR /app

COPY Pipfile Pipfile.lock ./
RUN pipenv install --system --deploy   # installs into system Python, not a nested venv

COPY predict.py .

EXPOSE 9696

ENTRYPOINT ["gunicorn", "--bind=0.0.0.0:9696", "predict:app"]
```

`--system --deploy` installs dependencies directly into the container's Python — no virtualenv-inside-Docker nesting, which is the correct pattern for containers.

---

## Dev vs Production vs Docker

| Concern | Flask (dev) | Gunicorn (prod) | Docker |
|---|---|---|---|
| Concurrent requests | ❌ Single-threaded | ✅ Multi-worker | ✅ Via Gunicorn inside |
| Auto-reload on code change | ✅ `debug=True` | ❌ | ❌ |
| Dependency isolation | Pipenv venv | Pipenv venv | Container filesystem |
| Portability | Local only | Local only | ✅ Runs anywhere |
| Use case | Development & testing | Staging / VM deployment | Production / cloud |

---

## Why Pipenv for Dependency Management?

`pip freeze` captures every package in your entire conda base environment — hundreds of packages, most irrelevant to this service. Pipenv:
- Separates **direct dependencies** (`Pipfile`) from **resolved transitive dependencies** (`Pipfile.lock`)
- Pins exact versions for reproducibility — the Docker build installs exactly what was tested
- Separates dev-only packages (e.g. `requests` for testing) from production packages

This means if a colleague or CI runner builds the Docker image six months later, they get the exact same environment.

---

## Tech Stack

`Python 3.9` · `Flask` · `Gunicorn` · `scikit-learn 1.6.0` · `XGBoost` · `Docker` · `Pipenv`
