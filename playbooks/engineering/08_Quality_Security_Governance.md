# Quality, Security, and Governance

**Code that works is not the same as code you can trust. Quality means every change is tested before it reaches production. Security means attackers cannot exploit your system. Governance means you can prove who changed what, when, and why. This chapter covers all three.**

---

## Testing Strategy

The testing pyramid: many fast unit tests at the base, fewer integration tests in the middle, a small number of slow end-to-end tests at the top. Invert this pyramid and your test suite takes 45 minutes to run and nobody runs it.

```
        /  E2E  \          Few, slow, expensive
       /----------\
      / Integration \      Some, moderate speed
     /----------------\
    /    Unit Tests     \  Many, fast, cheap
   /--------------------\
```

### Unit Tests

Test individual functions in isolation. Mock external dependencies (database, API calls, file system). These run in milliseconds.

```python
# src/scoring.py
def calculate_health_score(metrics: dict) -> float:
    """Score delivery health from 0.0 (critical) to 1.0 (healthy)."""
    if not metrics:
        raise ValueError("Metrics cannot be empty")
    weights = {"velocity": 0.3, "defect_rate": 0.3, "cycle_time": 0.4}
    score = sum(metrics.get(k, 0) * w for k, w in weights.items())
    return max(0.0, min(1.0, score))

# tests/test_scoring.py
import pytest
from src.scoring import calculate_health_score

def test_perfect_score():
    metrics = {"velocity": 1.0, "defect_rate": 1.0, "cycle_time": 1.0}
    assert calculate_health_score(metrics) == 1.0

def test_empty_metrics_raises():
    with pytest.raises(ValueError, match="cannot be empty"):
        calculate_health_score({})

def test_score_clamped_to_range():
    metrics = {"velocity": 2.0, "defect_rate": 2.0, "cycle_time": 2.0}
    assert calculate_health_score(metrics) == 1.0  # Clamped, not 2.0

def test_missing_keys_default_to_zero():
    metrics = {"velocity": 1.0}
    assert calculate_health_score(metrics) == pytest.approx(0.3)
```

**What to unit test:** Pure functions, business logic, data transformations, validation rules, error handling paths.

**What NOT to unit test:** Database queries (that is an integration test), third-party libraries (they test themselves), trivial getters/setters.

### Integration Tests

Test components working together. Use a real (or containerized) database, real HTTP calls to your own API, real message queue.

```python
# tests/integration/test_api.py
import httpx
import pytest

@pytest.fixture
def api_client():
    """Start the app and return a test client."""
    from src.main import app
    from fastapi.testclient import TestClient
    return TestClient(app)

def test_submit_diagnostic_returns_job_id(api_client):
    response = api_client.post(
        "/api/v1/diagnostics",
        json={"client_id": "test-123", "data_source": "upload"},
        files={"file": ("data.csv", b"col1,col2\n1,2\n3,4")},
    )
    assert response.status_code == 202
    assert "job_id" in response.json()

def test_health_check_reports_db_status(api_client):
    response = api_client.get("/health/ready")
    assert response.status_code == 200
    assert response.json()["db"] == "connected"
```

**Use testcontainers** to spin up real PostgreSQL, Redis, and Kafka in Docker for integration tests. No mocking -- real dependencies, real behavior.

### End-to-End Tests

Test the full workflow as a user would experience it. Submit data, wait for processing, retrieve the report.

```python
def test_full_diagnostic_workflow():
    # Submit a diagnostic
    submit = httpx.post(f"{BASE_URL}/api/v1/diagnostics", json={...})
    job_id = submit.json()["job_id"]

    # Poll until complete (with timeout)
    for _ in range(30):
        status = httpx.get(f"{BASE_URL}/api/v1/diagnostics/{job_id}")
        if status.json()["status"] == "complete":
            break
        time.sleep(2)

    # Verify the report
    report = status.json()["report"]
    assert "health_score" in report
    assert 0.0 <= report["health_score"] <= 1.0
    assert len(report["sections"]) == 6
```

**Run E2E tests against a staging environment, not production.** They are slow and can create real data.

### Load Tests

Can the system handle expected traffic? Use locust (Python) or k6 (JavaScript) to simulate concurrent users.

```python
# locustfile.py
from locust import HttpUser, task, between

class DiagnosticUser(HttpUser):
    wait_time = between(1, 3)  # Wait 1-3 seconds between requests

    @task(3)
    def check_report_status(self):
        self.client.get("/api/v1/diagnostics/test-job-123")

    @task(1)
    def submit_diagnostic(self):
        self.client.post("/api/v1/diagnostics", json={
            "client_id": "load-test",
            "data_source": "upload",
        })
```

```bash
# Run with 100 concurrent users, ramp up 10 per second
locust -f locustfile.py --host=https://staging.example.com --users=100 --spawn-rate=10
```

**What to look for:** p50, p95, p99 latency. Error rate under load. At what concurrency does the system start degrading?

### ML-Specific Tests

ML systems need tests that traditional software does not.

| Test | What It Checks | Example |
|:---|:---|:---|
| **Model accuracy** | Does the model meet minimum performance? | `assert accuracy >= 0.85` on a held-out test set |
| **Data distribution** | Has the input distribution shifted? | Compare training data stats to production data stats |
| **Prediction latency** | Is inference fast enough? | `assert p95_latency < 200  # milliseconds` |
| **Input validation** | Does the model handle bad input? | Pass null values, wrong types, extreme outliers |
| **Reproducibility** | Same input = same output? | Fixed seed, deterministic inference |
| **Bias / fairness** | Does the model discriminate? | Accuracy across demographic groups |

---

## Code Quality

Three tools that catch bugs before they reach code review:

| Tool | What It Does | Command |
|:---|:---|:---|
| **ruff** | Linting + formatting (replaces flake8, isort, black) | `ruff check . && ruff format .` |
| **mypy** | Static type checking | `mypy src/` |
| **pre-commit** | Run checks before every commit | `pre-commit run --all-files` |

```toml
# pyproject.toml
[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "SIM"]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
disallow_untyped_defs = true
```

**The pre-commit hook setup** ensures every developer on the team runs the same checks:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, fastapi]
```

---

## Code Review

Code review is not about catching syntax errors (tools do that). It is about catching design mistakes, missed edge cases, and maintainability problems that tools cannot detect.

**What to look for in a code review:**

| Category | Questions to Ask |
|:---|:---|
| **Correctness** | Does this handle the error case? What if the input is null/empty/huge? |
| **Design** | Is this the right abstraction? Will this be easy to change in 6 months? |
| **Security** | Is user input validated? Are secrets hardcoded? Is auth checked? |
| **Performance** | Will this scale? Is there an N+1 query? Is this doing work in a loop that could be batched? |
| **Readability** | Can I understand this without the author explaining it? Are the names descriptive? |
| **Tests** | Are the important paths tested? Are edge cases covered? |

**How to give feedback:**
- Ask questions instead of making demands: "What happens if `client_id` is None?" instead of "You need to add a null check."
- Distinguish between must-fix (blocks merge) and nit (style preference, can merge without).
- Never approve code you do not understand. If you cannot explain what it does, ask.

---

## Security

### OWASP Top 10 for AI Systems

| Risk | What It Means | Mitigation |
|:---|:---|:---|
| **Prompt injection** | Attacker manipulates LLM input to bypass instructions | Input sanitization, output validation, system/user prompt separation |
| **Broken authentication** | Weak or missing API auth | OAuth 2.0, API keys with rotation, no default credentials |
| **Sensitive data exposure** | PII in logs, model outputs, or training data | Data classification, PII redaction, encrypted storage |
| **Insecure model deployment** | Model endpoint exposed without auth | VPC, network policies, authentication on every endpoint |
| **Insufficient input validation** | Malformed data crashes the service or triggers unexpected behavior | Pydantic models, type checking, size limits |
| **Supply chain attacks** | Compromised dependencies or pre-trained models | Dependency scanning, model provenance verification |
| **Excessive permissions** | Service account with admin access | Least privilege, scoped IAM roles |

### Secrets Management

**Never hardcode secrets.** Not in code, not in config files, not in environment variable defaults.

| Method | Security Level | When to Use |
|:---|:---|:---|
| **Environment variables** | Basic | Local development, simple deployments |
| **AWS Secrets Manager / GCP Secret Manager** | High | Production secrets (DB passwords, API keys) |
| **HashiCorp Vault** | Highest | Dynamic secrets, automatic rotation, audit trail |
| **.env files** (gitignored) | Development only | Local dev, never in CI/CD or production |

```python
# WRONG -- hardcoded secret
db_url = "postgresql://admin:s3cret@prod-db:5432/mydb"

# RIGHT -- from environment
import os
db_url = os.environ["DATABASE_URL"]

# BETTER -- from Secret Manager with caching
from google.cloud import secretmanager

def get_secret(secret_id: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/my-project/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(name=name)
    return response.payload.data.decode("utf-8")
```

### Input Validation

Every API endpoint must validate input. Use Pydantic -- it rejects bad data before your business logic ever sees it.

```python
from pydantic import BaseModel, Field, validator

class DiagnosticRequest(BaseModel):
    client_id: str = Field(..., min_length=1, max_length=100, pattern=r"^[a-zA-Z0-9-]+$")
    data_source: Literal["upload", "oauth"]
    config: dict | None = None

    @validator("config")
    def validate_config(cls, v):
        if v and "depth" in v:
            if v["depth"] not in ("quick", "full"):
                raise ValueError("depth must be 'quick' or 'full'")
        return v

@app.post("/api/v1/diagnostics")
async def submit_diagnostic(request: DiagnosticRequest):
    # request is already validated -- safe to use
    ...
```

### Dependency Scanning

Your code is only as secure as your dependencies. Automate scanning:

| Tool | What It Does | Integration |
|:---|:---|:---|
| **Dependabot** (GitHub) | Auto-creates PRs for vulnerable dependencies | GitHub Settings > Security |
| **safety** | Scans Python dependencies against known vulnerabilities | `safety check --full-report` |
| **Snyk** | Deep vulnerability scanning + fix suggestions | CI/CD integration |
| **pip-audit** | Audits installed packages against PyPI advisories | `pip-audit` in CI |

### API Authentication and Authorization

```python
from fastapi import Depends, HTTPException, Security
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Security(api_key_header)):
    # In production, check against database or secrets manager
    valid_keys = get_valid_api_keys()  # From secure storage
    if api_key not in valid_keys:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return api_key

@app.get("/api/v1/diagnostics/{job_id}")
async def get_diagnostic(job_id: str, api_key: str = Depends(verify_api_key)):
    # Only authenticated clients reach this code
    ...
```

### Network Security

| Layer | What to Do | Why |
|:---|:---|:---|
| **HTTPS** | TLS on every endpoint, including internal | Prevent eavesdropping and man-in-the-middle |
| **VPC** | Private subnets for databases and workers | Databases should not be reachable from the internet |
| **Security groups / firewall rules** | Allow only necessary ports between services | Limit blast radius of a compromised service |
| **WAF** (Web Application Firewall) | Block common attack patterns (SQL injection, XSS) | Defense in depth at the edge |

---

## Governance

### Change Management

Every change to production follows a predictable process:

```
Developer writes code
    --> Creates Pull Request
        --> Automated tests run (CI)
            --> Code review by peer
                --> Approval required (1-2 reviewers)
                    --> Merge to main
                        --> Automated deployment (CD)
                            --> Canary / staged rollout
                                --> Monitor for errors
```

**Rules that prevent disasters:**
- No direct commits to `main`. All changes go through PRs.
- CI must pass before merge is allowed (branch protection rules).
- At least one approval required. For infrastructure changes, two.
- Deployment is automated. No one SSH-es into production to deploy manually.

### Audit Trails

You must be able to answer: "Who deployed this change, when, and what did it contain?"

| What to Track | Where to Track It | How Long to Keep |
|:---|:---|:---|
| Code changes | Git history (commit log, PR descriptions) | Forever |
| Deployments | CI/CD logs (GitHub Actions, ArgoCD) | 1-2 years |
| Configuration changes | Infrastructure as Code (Terraform state) | Forever |
| Data access | Database audit logs, application logs | Per compliance requirement |
| Incident responses | Post-mortem documents, ticketing system | Forever |

### Compliance Considerations

| Framework | Who Needs It | Key Requirements for AI Systems |
|:---|:---|:---|
| **SOC 2** | Any SaaS handling customer data | Access controls, encryption, monitoring, incident response, change management |
| **HIPAA** | Health data | PHI (Protected Health Information) encryption, access logging, BAA (Business Associate Agreement) with cloud provider |
| **GDPR** | EU user data | Right to deletion, data portability, consent management, data processing agreements |
| **AI-specific** (EU AI Act, NIST AI RMF) | AI systems affecting decisions | Model documentation, bias testing, explainability, human oversight |

**Practical starting point for any AI system:**
1. Encrypt data at rest and in transit
2. Log all access to sensitive data
3. Implement role-based access control
4. Document what data the model was trained on
5. Test for bias across protected classes
6. Maintain an incident response plan
7. Run regular dependency and vulnerability scans

---

## Quick Links -- All Chapters

| Chapter | Title |
|:---|:---|
| [01](01_Why.md) | Why |
| [02](02_Concepts.md) | Concepts |
| [03](03_Hello_World.md) | Hello World |
| [04](04_How_It_Works.md) | How It Works |
| [05](05_Building_It.md) | Building It |
| [06](06_Production_Patterns.md) | Production Software Patterns |
| [07](07_System_Design.md) | System Design for AI/Data Services |
| [08](08_Quality_Security_Governance.md) | **Quality, Security, and Governance** |
| [09](09_Observability_Troubleshooting.md) | Observability and Troubleshooting |
| [10](10_Decision_Guide.md) | Software Engineering Decision Guide |
