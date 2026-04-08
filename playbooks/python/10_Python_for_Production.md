# Python in Production -- Packaging, Testing, Deployment

**Writing Python that works on your laptop is chapter 1. Writing Python that runs reliably in production -- tested, packaged, containerized, and deployed behind an API -- is the rest of the book. This chapter covers the gap.**

---

## Virtual Environments: venv and uv

Every production project needs an isolated environment. Without one, package version conflicts will break your system unpredictably.

| Tool | Speed | Lockfile | Resolves deps? | When to Use |
|:---|:---|:---|:---|:---|
| `venv` + `pip` | Slow | `requirements.txt` (manual) | No (installs what you ask) | Legacy projects, restricted environments |
| `uv` | 10-100x faster | `uv.lock` (auto) | Yes (full resolver) | New projects, modern tooling |
| `conda` | Slow | `environment.yml` | Yes | Data science with non-Python deps (CUDA, GDAL) |

```bash
# uv -- the modern choice (recommended)
uv init my-project && cd my-project
uv add pandas scikit-learn fastapi uvicorn
uv run python main.py

# venv + pip -- always available
python -m venv .venv
source .venv/bin/activate
pip install pandas scikit-learn fastapi uvicorn
pip freeze > requirements.txt
python main.py
```

---

## Project Structure: The src Layout

A production Python project has a standard layout. This is not optional -- it is how tools (pytest, Docker, CI/CD pipelines) expect your code to be organized:

```
my-project/
├── src/
│   └── my_project/
│       ├── __init__.py        # Makes this a Python package
│       ├── main.py            # Application entry point
│       ├── models/
│       │   ├── __init__.py
│       │   └── churn.py       # Model training and inference
│       ├── pipelines/
│       │   ├── __init__.py
│       │   └── ingest.py      # Data ingestion pipeline
│       └── utils/
│           ├── __init__.py
│           └── config.py      # Configuration management
├── tests/
│   ├── __init__.py
│   ├── test_churn.py
│   └── test_ingest.py
├── pyproject.toml             # Project metadata + dependencies
├── Dockerfile
├── .env.example               # Template for environment variables
├── .gitignore
└── README.md
```

### pyproject.toml -- the modern standard

```toml
[project]
name = "call-center-ml"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "pandas>=2.0",
    "scikit-learn>=1.3",
    "fastapi>=0.100",
    "uvicorn>=0.20",
]

[project.optional-dependencies]
dev = ["pytest>=7.0", "ruff>=0.1", "mypy>=1.0"]

[project.scripts]
# Defines a CLI command that runs your main function
serve = "my_project.main:start_server"

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.ruff]
line-length = 100
```

---

## Testing: pytest Basics

You do not need to write exhaustive tests. You need to write tests for the parts that break: data transforms, business logic, and edge cases.

```python
# tests/test_churn.py
import pytest
import pandas as pd
from my_project.models.churn import calculate_churn_rate, validate_features

def test_churn_rate_basic():
    """Normal case: 10 customers, 2 churned = 20%."""
    assert calculate_churn_rate(total=10, churned=2) == 0.2

def test_churn_rate_zero_customers():
    """Edge case: zero customers should return 0, not divide-by-zero."""
    assert calculate_churn_rate(total=0, churned=0) == 0.0

def test_validate_features_rejects_negative_duration():
    """Business rule: negative duration is invalid."""
    df = pd.DataFrame({"duration_sec": [-5, 120], "wait_sec": [10, 15]})
    with pytest.raises(ValueError, match="negative duration"):
        validate_features(df)

# Fixture -- reusable test data
@pytest.fixture
def sample_calls():
    """Create a small DataFrame for testing transforms."""
    return pd.DataFrame({
        "call_id": ["C-1001", "C-1002", "C-1003"],
        "duration_sec": [120, 0, 300],
        "agent": ["Alice", None, "Bob"],
    })

def test_clean_removes_zero_duration(sample_calls):
    """Pipeline should drop calls with zero duration."""
    from my_project.pipelines.ingest import clean_calls
    result = clean_calls(sample_calls)
    assert len(result) == 2
    assert 0 not in result["duration_sec"].values
```

```bash
# Run tests
uv run pytest                        # All tests
uv run pytest tests/test_churn.py    # One file
uv run pytest -v                     # Verbose output
uv run pytest -k "test_churn"        # Tests matching a pattern
uv run pytest --tb=short             # Short tracebacks on failure
```

---

## Logging: The logging Module (Not print)

`print()` disappears into stdout and has no levels, no timestamps, and no way to route to files or monitoring systems. The `logging` module solves all of this:

```python
import logging

# Configure once at application startup
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s [%(name)s] %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S"
)

# Create a logger for this module
logger = logging.getLogger(__name__)

def train_model(X, y):
    logger.info(f"Training started: {X.shape[0]} samples, {X.shape[1]} features")
    try:
        model = RandomForestClassifier().fit(X, y)
        score = model.score(X, y)
        logger.info(f"Training complete: accuracy={score:.4f}")
        return model
    except Exception as e:
        logger.error(f"Training failed: {e}", exc_info=True)
        raise

# Output: 2026-04-01T14:23:45 INFO [my_project.models.churn] Training started: 10000 samples, 12 features
```

| Level | When to Use |
|:---|:---|
| `DEBUG` | Detailed diagnostic info (off in production) |
| `INFO` | Confirmation that things are working |
| `WARNING` | Something unexpected but not a failure |
| `ERROR` | Something failed but the application continues |
| `CRITICAL` | The application cannot continue |

---

## Configuration: Environment Variables and Config Classes

Never hardcode credentials, database URLs, or model paths. Use environment variables with a config class:

```python
# src/my_project/utils/config.py
import os
from dataclasses import dataclass

@dataclass
class AppConfig:
    """Application configuration loaded from environment variables.

    In production: set via Docker env, Kubernetes secrets, or .env file.
    Locally: use a .env file (never commit it to git).
    """
    db_url: str = os.getenv("DATABASE_URL", "postgresql://localhost:5432/dev")
    model_path: str = os.getenv("MODEL_PATH", "models/churn_v1.pkl")
    api_key: str = os.getenv("API_KEY", "")
    log_level: str = os.getenv("LOG_LEVEL", "INFO")
    batch_size: int = int(os.getenv("BATCH_SIZE", "1000"))

    def validate(self):
        """Fail fast if required config is missing."""
        if not self.api_key:
            raise ValueError("API_KEY environment variable is required")

config = AppConfig()
```

**.env file (never commit this):**

```
DATABASE_URL=postgresql://user:pass@prod-db:5432/analytics
MODEL_PATH=s3://models/churn_v3.pkl
API_KEY=sk-abc123...
LOG_LEVEL=WARNING
BATCH_SIZE=5000
```

**.gitignore must include:**

```
.env
*.pkl
*.pth
__pycache__/
.venv/
```

---

## FastAPI: Build a Prediction API in 10 Lines

FastAPI is the standard for serving ML models and building data APIs. It is async-native, auto-generates documentation, and validates request/response types with Pydantic:

```python
# src/my_project/main.py
from fastapi import FastAPI
from pydantic import BaseModel, Field
import joblib

app = FastAPI(title="Churn Prediction API")
model = joblib.load("models/churn_v1.pkl")  # Load model once at startup

class PredictionRequest(BaseModel):
    duration_sec: int = Field(gt=0)
    wait_sec: int = Field(ge=0)
    satisfaction: float = Field(ge=1.0, le=5.0)

class PredictionResponse(BaseModel):
    prediction: str
    confidence: float

@app.get("/health")
def health():
    return {"status": "healthy"}

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    features = [[request.duration_sec, request.wait_sec, request.satisfaction]]
    proba = model.predict_proba(features)[0]
    label = "escalated" if proba[1] > 0.5 else "resolved"
    return PredictionResponse(prediction=label, confidence=float(max(proba)))
```

```bash
# Run the API server
uv run uvicorn my_project.main:app --reload --port 8000

# Auto-generated API docs at: http://localhost:8000/docs
# Test: curl -X POST http://localhost:8000/predict \
#   -H "Content-Type: application/json" \
#   -d '{"duration_sec": 250, "wait_sec": 40, "satisfaction": 2.0}'
```

```mermaid
graph LR
    A["Client<br/>curl / frontend"] -->|POST /predict| B["FastAPI<br/>Pydantic validates input"]
    B --> C["Model<br/>sklearn predict_proba()"]
    C --> D["Response<br/>JSON with prediction + confidence"]

    style B fill:#4a90d9,color:#fff
    style C fill:#50a050,color:#fff
```

---

## Docker: Containerize a Python Application

Docker packages your application with its exact dependencies so it runs the same everywhere:

```dockerfile
# Dockerfile
FROM python:3.12-slim

# Install uv for fast dependency resolution
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# Copy dependency files first (Docker caches this layer)
COPY pyproject.toml uv.lock ./

# Install dependencies (cached unless pyproject.toml changes)
RUN uv sync --frozen --no-dev

# Copy application code (changes more often than dependencies)
COPY src/ ./src/
COPY models/ ./models/

# Run the API server
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "my_project.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
# Build the image
docker build -t churn-api:v1 .

# Run the container
docker run -p 8000:8000 -e API_KEY=sk-abc123 churn-api:v1

# Test
curl http://localhost:8000/health
```

**Docker best practices for Python:**

| Practice | Why |
|:---|:---|
| Use `python:3.12-slim` not `python:3.12` | Slim image is ~150MB vs ~900MB |
| Copy dependency files before code | Docker caches dependency install across builds |
| Use `--no-dev` in production | Don't install test/dev tools in production image |
| Set `EXPOSE` | Documents which port the app uses |
| Never copy `.env` into the image | Pass secrets via `-e` flags or orchestrator secrets |

---

## CI/CD: GitHub Actions for Python

A minimal GitHub Actions workflow that lints, tests, and builds on every push:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Install dependencies
        run: uv sync

      - name: Lint with ruff
        run: uv run ruff check src/ tests/

      - name: Type check with mypy
        run: uv run mypy src/

      - name: Run tests
        run: uv run pytest --tb=short

  build:
    needs: test  # Only build if tests pass
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t my-app:${{ github.sha }} .
```

```mermaid
graph LR
    A["Push to GitHub"] --> B["Lint (ruff)"]
    B --> C["Type Check (mypy)"]
    C --> D["Test (pytest)"]
    D --> E["Build Docker Image"]
    E --> F["Deploy"]

    style A fill:#4a90d9,color:#fff
    style D fill:#50a050,color:#fff
    style F fill:#d94a4a,color:#fff
```

---

## The Production Python Checklist

Before your Python code runs in production, verify each item:

| Category | Check | Tool |
|:---|:---|:---|
| **Dependencies** | Pinned versions with lockfile | `uv.lock` or `requirements.txt` |
| **Dependencies** | No `pip install` in production -- use lockfile | `uv sync --frozen` |
| **Code Quality** | Linted (no unused imports, consistent style) | `ruff` |
| **Code Quality** | Type hints on all public functions | `mypy` |
| **Testing** | Tests for business logic and edge cases | `pytest` |
| **Testing** | No tests that depend on external services | Mocks, fixtures |
| **Configuration** | No hardcoded secrets or paths | Environment variables |
| **Configuration** | `.env` in `.gitignore` | Manual check |
| **Logging** | No `print()` statements | `logging` module |
| **Logging** | Structured log format with timestamps | `logging.basicConfig()` |
| **Error Handling** | No bare `except:` clauses | `ruff` rule E722 |
| **Error Handling** | Graceful degradation on failure | try/except with fallback |
| **Packaging** | `pyproject.toml` with metadata | Manual check |
| **Packaging** | src layout with `__init__.py` files | Manual check |
| **Container** | Slim base image, multi-stage if needed | `Dockerfile` |
| **Container** | No secrets baked into the image | Build args or runtime env |
| **CI/CD** | Automated lint + test + build on push | GitHub Actions |

---

## AI Example: Deploy a Model Behind FastAPI + Docker

The complete flow from trained model to production API:

```python
# 1. Train and save the model (one-time or scheduled)
import joblib
from sklearn.ensemble import RandomForestClassifier

model = RandomForestClassifier(n_estimators=100).fit(X_train, y_train)
joblib.dump(model, "models/churn_v1.pkl")
# In production: save to S3/GCS, version with MLflow

# 2. Serve the model (FastAPI -- see code above)
# 3. Containerize (Dockerfile -- see above)
# 4. Deploy (Kubernetes, ECS, Cloud Run, etc.)
```

```mermaid
graph LR
    A["Train Model<br/>sklearn + MLflow"] -->|joblib.dump| B["Model Artifact<br/>churn_v1.pkl"]
    B -->|COPY in Dockerfile| C["Docker Image<br/>FastAPI + model"]
    C -->|Deploy| D["Cloud Run / ECS<br/>Auto-scaling API"]
    D -->|POST /predict| E["Predictions<br/>JSON response"]

    style A fill:#4a90d9,color:#fff
    style C fill:#50a050,color:#fff
    style D fill:#d94a4a,color:#fff
```

---

## DE Example: Package a Pipeline as a Python Module with Tests

```python
# src/my_project/pipelines/ingest.py
import pandas as pd
import logging

logger = logging.getLogger(__name__)

def clean_calls(df: pd.DataFrame) -> pd.DataFrame:
    """Apply data quality rules to raw call records.

    Rules:
    1. Drop records with zero or negative duration
    2. Fill missing agent names with UNKNOWN
    3. Remove duplicate call_ids (keep first)
    """
    initial_count = len(df)
    result = (
        df
        .query("duration_sec > 0")
        .assign(agent=lambda x: x["agent"].fillna("UNKNOWN"))
        .drop_duplicates(subset=["call_id"], keep="first")
    )
    logger.info(f"Cleaned {initial_count} -> {len(result)} records "
                f"({initial_count - len(result)} dropped)")
    return result

def run_pipeline(source_path: str, dest_path: str) -> int:
    """Read from source, clean, write to destination. Return record count."""
    df = pd.read_parquet(source_path)
    clean = clean_calls(df)
    clean.to_parquet(dest_path, index=False)
    return len(clean)
```

```python
# tests/test_ingest.py
import pandas as pd
import pytest
from my_project.pipelines.ingest import clean_calls

@pytest.fixture
def raw_calls():
    return pd.DataFrame({
        "call_id": ["C-1", "C-2", "C-3", "C-1"],  # C-1 is duplicated
        "duration_sec": [120, 0, 300, 120],          # C-2 has zero duration
        "agent": ["Alice", None, None, "Alice"],
    })

def test_removes_zero_duration(raw_calls):
    result = clean_calls(raw_calls)
    assert (result["duration_sec"] > 0).all()

def test_fills_missing_agent(raw_calls):
    result = clean_calls(raw_calls)
    assert result["agent"].isna().sum() == 0
    assert "UNKNOWN" in result["agent"].values

def test_removes_duplicates(raw_calls):
    result = clean_calls(raw_calls)
    assert result["call_id"].is_unique

def test_preserves_valid_records(raw_calls):
    result = clean_calls(raw_calls)
    # C-2 dropped (zero duration), C-1 deduped -> 2 records remain
    assert len(result) == 2
```

---

## Quick Links

| Resource | Link |
|:---|:---|
| Python for AI (notebook) | [Python for AI on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_Basics.ipynb) |
| Python for DE (notebook) | [Python for DE on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_NumPy_Pandas.ipynb) |
| Previous chapter | [09 -- Advanced Patterns](09_Advanced_Patterns.md) |
| Chapter 01 | [01 -- Why Python](01_Why.md) |

---

*Foundations -- Python (Chapter 10 of 10)*
