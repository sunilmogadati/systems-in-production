# Advanced Python Patterns for Production

**Decorators, context managers, type hints, async/await -- these are the patterns that separate prototype code from production code. You will encounter every one of them in real AI and data engineering systems.**

---

## Decorators: What They Are and How to Read Them

A decorator is a function that wraps another function with additional behavior. The `@` syntax is shorthand for wrapping:

```python
# These two are identical:
@my_decorator
def my_function():
    pass

# is the same as:
def my_function():
    pass
my_function = my_decorator(my_function)
```

### Decorators you will see everywhere

| Decorator | Library | What It Does |
|:---|:---|:---|
| `@app.get("/path")` | FastAPI | Registers a function as an HTTP GET endpoint |
| `@app.post("/path")` | FastAPI | Registers a function as an HTTP POST endpoint |
| `@task` | Airflow | Turns a function into a DAG (Directed Acyclic Graph) task |
| `@dag` | Airflow | Turns a function into a DAG definition |
| `@torch.no_grad()` | PyTorch | Disables gradient tracking (saves memory during inference) |
| `@staticmethod` | Python | Method that doesn't need `self` |
| `@classmethod` | Python | Method that takes `cls` instead of `self` |
| `@property` | Python | Makes a method behave like a read-only attribute |
| `@dataclass` | Python | Auto-generates `__init__`, `__repr__`, `__eq__` |
| `@pytest.fixture` | pytest | Defines reusable test setup |
| `@retry(tries=3)` | tenacity | Retries the function on failure |

### Writing a decorator (for when you need one)

**AI example -- timing model training:**

```python
import time
from functools import wraps

def timer(func):
    """Decorator that logs how long a function takes to execute."""
    @wraps(func)  # Preserves the original function's name and docstring
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"[TIMER] {func.__name__} took {elapsed:.2f}s")
        return result
    return wrapper

@timer
def train_model(X, y):
    from sklearn.ensemble import RandomForestClassifier
    return RandomForestClassifier(n_estimators=100).fit(X, y)

# Output: [TIMER] train_model took 1.23s
model = train_model(X_train, y_train)
```

**DE example -- retry with backoff for flaky API calls:**

```python
import time
from functools import wraps

def retry(max_attempts: int = 3, backoff_sec: float = 1.0):
    """Decorator factory: retry a function on exception with exponential backoff."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise  # Re-raise on final attempt
                    wait = backoff_sec * (2 ** (attempt - 1))
                    print(f"[RETRY] {func.__name__} failed (attempt {attempt}), "
                          f"retrying in {wait}s: {e}")
                    time.sleep(wait)
        return wrapper
    return decorator

@retry(max_attempts=3, backoff_sec=2.0)
def fetch_from_api(url: str) -> dict:
    import requests
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

---

## Context Managers: The `with` Statement

A context manager guarantees cleanup happens even if an error occurs. It replaces try/finally patterns:

```python
# Without context manager -- if process() crashes, file stays open
f = open("data.csv", "r")
data = f.read()
process(data)
f.close()  # Might never execute

# With context manager -- file is ALWAYS closed, even on error
with open("data.csv", "r") as f:
    data = f.read()
    process(data)
# f.close() happens automatically here
```

### Context managers you will see

| Pattern | What It Manages |
|:---|:---|
| `with open(path) as f:` | File handle -- always closed |
| `with engine.connect() as conn:` | Database connection -- always released |
| `with torch.no_grad():` | GPU gradient memory -- always freed |
| `with tempfile.NamedTemporaryFile() as tmp:` | Temp file -- always deleted |
| `with ThreadPoolExecutor() as pool:` | Thread pool -- always shut down |
| `with mlflow.start_run():` | MLflow experiment run -- always ended |

**AI example -- context manager for GPU memory:**

```python
import torch

# During training: gradients are tracked (uses GPU memory)
model.train()
output = model(input_tensor)
loss = criterion(output, target)
loss.backward()

# During inference: no gradients needed (saves ~50% GPU memory)
model.eval()
with torch.no_grad():
    predictions = model(test_input)
# Gradient tracking is automatically restored after the block
```

**DE example -- database connection context manager:**

```python
from contextlib import contextmanager
import psycopg2

@contextmanager
def db_connection(connection_string: str):
    """Yield a database connection that is always properly closed."""
    conn = psycopg2.connect(connection_string)
    try:
        yield conn
        conn.commit()  # Commit if no errors
    except Exception:
        conn.rollback()  # Rollback on error
        raise
    finally:
        conn.close()  # Always close the connection

with db_connection("postgresql://user:pass@host/db") as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT count(*) FROM calls")
    count = cursor.fetchone()[0]
```

---

## Error Handling: Production-Grade try/except/finally

### The basics

```python
try:
    result = risky_operation()
except SpecificError as e:
    # Handle the specific error -- never bare 'except:'
    handle_error(e)
except (TypeError, ValueError) as e:
    # Handle multiple error types
    handle_other(e)
else:
    # Runs ONLY if no exception was raised
    process(result)
finally:
    # Runs ALWAYS -- cleanup goes here
    cleanup()
```

### Production patterns

**DE example -- error handling in pipeline steps:**

```python
import logging

logger = logging.getLogger(__name__)

def process_batch(records: list[dict]) -> tuple[list[dict], list[dict]]:
    """Process a batch of records. Return (successes, failures).

    Production pipelines never crash on bad records. They separate
    good records from bad ones and log the failures for investigation.
    """
    successes = []
    failures = []

    for record in records:
        try:
            validated = validate_and_transform(record)
            successes.append(validated)
        except Exception as e:
            # Attach error context to the failed record
            record["_error"] = str(e)
            record["_error_type"] = type(e).__name__
            failures.append(record)
            logger.warning(f"Record {record.get('id', 'unknown')} failed: {e}")

    logger.info(f"Batch complete: {len(successes)} success, {len(failures)} failed")
    return successes, failures
```

**AI example -- graceful model fallback:**

```python
def predict_with_fallback(model, features: dict) -> dict:
    """Try the primary model. Fall back to a default on failure."""
    try:
        prediction = model.predict([list(features.values())])[0]
        confidence = model.predict_proba([list(features.values())])[0].max()
        return {"prediction": prediction, "confidence": confidence, "model": "primary"}
    except Exception as e:
        logger.error(f"Primary model failed: {e}")
        # Return a safe default instead of crashing the API
        return {"prediction": "unknown", "confidence": 0.0, "model": "fallback"}
```

---

## Type Hints: Why They Matter in Production

Type hints do not enforce types at runtime. They are documentation that IDEs (Integrated Development Environments), linters, and teammates can read:

```python
# Without type hints -- what does this return? What types are valid?
def process(data, config):
    pass

# With type hints -- intent is clear without reading the implementation
def process(data: pd.DataFrame, config: dict[str, any]) -> pd.DataFrame:
    pass
```

### Common type hint patterns

```python
from typing import Optional

# Optional -- value can be the type or None
def find_agent(call_id: str) -> Optional[str]:
    """Return agent name or None if call not found."""
    pass

# Union (Python 3.10+ uses | syntax)
def parse_input(raw: str | bytes) -> dict:
    pass

# List with element type
def get_predictions(model, inputs: list[dict]) -> list[float]:
    pass

# Dict with key/value types
def load_config(path: str) -> dict[str, str | int | float]:
    pass

# Callable (for functions as arguments)
from typing import Callable
def apply_transform(df: pd.DataFrame,
                    func: Callable[[pd.DataFrame], pd.DataFrame]) -> pd.DataFrame:
    return func(df)
```

### Pydantic -- runtime type validation

Pydantic enforces types at runtime. FastAPI and LangChain use it for request/response validation:

```python
from pydantic import BaseModel, Field

class PredictionRequest(BaseModel):
    """Request body for the prediction API endpoint.
    Pydantic validates types on instantiation -- invalid data raises an error."""
    call_id: str
    duration_sec: int = Field(gt=0, description="Must be positive")
    wait_sec: int = Field(ge=0, description="Must be non-negative")
    agent: str | None = None

# Valid -- passes validation
req = PredictionRequest(call_id="C-1001", duration_sec=120, wait_sec=15)

# Invalid -- raises ValidationError immediately
# req = PredictionRequest(call_id="C-1001", duration_sec=-5, wait_sec=15)
```

---

## f-strings and String Formatting

f-strings (formatted string literals, Python 3.6+) are the standard for string formatting:

```python
model_name = "gradient_boost"
accuracy = 0.9234
latency_ms = 12.7

# f-string -- the default choice
print(f"Model: {model_name}, Accuracy: {accuracy:.2%}, Latency: {latency_ms:.1f}ms")
# "Model: gradient_boost, Accuracy: 92.34%, Latency: 12.7ms"

# Format specifiers
print(f"{accuracy:.4f}")    # 0.9234 -- 4 decimal places
print(f"{accuracy:.2%}")    # 92.34% -- percentage
print(f"{1000000:,}")       # 1,000,000 -- comma separator
print(f"{42:05d}")          # 00042 -- zero-padded
print(f"{'left':<20}")      # "left                " -- left-aligned
print(f"{'right':>20}")     # "               right" -- right-aligned
```

---

## Regular Expressions -- Enough to Parse Logs and Extract Patterns

```python
import re

# DE example: extract timestamp and error level from log lines
log_line = "2026-04-01T14:23:45 ERROR [pipeline.extract] Connection timeout to s3://bucket"

# Match pattern: ISO timestamp, log level, module, message
pattern = r"(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})\s+(\w+)\s+\[(.+?)\]\s+(.+)"
match = re.match(pattern, log_line)
if match:
    timestamp, level, module, message = match.groups()
    # timestamp="2026-04-01T14:23:45", level="ERROR",
    # module="pipeline.extract", message="Connection timeout to s3://bucket"

# Extract all S3 paths from a config file
text = "source: s3://raw-data/calls/ destination: s3://clean-data/calls/"
s3_paths = re.findall(r"s3://[\w\-/]+", text)
# ["s3://raw-data/calls/", "s3://clean-data/calls/"]

# Validate email format
def is_valid_email(email: str) -> bool:
    return bool(re.match(r"^[\w.+-]+@[\w-]+\.[\w.]+$", email))
```

### Common regex patterns for AI/DE

| Pattern | Regex | Matches |
|:---|:---|:---|
| ISO timestamp | `\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}` | `2026-04-01T14:23:45` |
| S3 path | `s3://[\w\-/]+` | `s3://bucket/path/` |
| GCS path | `gs://[\w\-/]+` | `gs://bucket/path/` |
| UUID (Universally Unique Identifier) | `[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}` | `a1b2c3d4-...` |
| IP address | `\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}` | `192.168.1.1` |
| Email | `[\w.+-]+@[\w-]+\.[\w.]+` | `user@domain.com` |

---

## Async/Await -- When You Need It

Async (asynchronous) code lets your program do other work while waiting for I/O operations (API calls, database queries, file reads). It does not make CPU-bound work faster.

```mermaid
graph LR
    subgraph "Synchronous (blocking)"
        S1["Call API 1<br/>wait 500ms"] --> S2["Call API 2<br/>wait 500ms"] --> S3["Call API 3<br/>wait 500ms"]
    end

    subgraph "Asynchronous (concurrent)"
        A1["Call API 1"]
        A2["Call API 2"]
        A3["Call API 3"]
    end

    style S1 fill:#d94a4a,color:#fff
    style S2 fill:#d94a4a,color:#fff
    style S3 fill:#d94a4a,color:#fff
    style A1 fill:#50a050,color:#fff
    style A2 fill:#50a050,color:#fff
    style A3 fill:#50a050,color:#fff
```

**Synchronous: 1500ms total. Asynchronous: 500ms total (all three run concurrently).**

```python
import asyncio
import aiohttp  # pip install aiohttp

async def fetch_prediction(session, call_id: str) -> dict:
    """Fetch a single prediction from the model API."""
    async with session.get(f"https://api.example.com/predict/{call_id}") as response:
        return await response.json()

async def fetch_all_predictions(call_ids: list[str]) -> list[dict]:
    """Fetch predictions for all calls concurrently."""
    async with aiohttp.ClientSession() as session:
        # Create all tasks, then run them concurrently
        tasks = [fetch_prediction(session, cid) for cid in call_ids]
        return await asyncio.gather(*tasks)

# Run the async function
results = asyncio.run(fetch_all_predictions(["C-1001", "C-1002", "C-1003"]))
```

### When to use async

| Scenario | Use Async? | Why |
|:---|:---|:---|
| Multiple API calls | Yes | Waiting for network I/O -- run them concurrently |
| Multiple database queries | Yes | Waiting for database I/O |
| FastAPI endpoints | Yes (built-in) | FastAPI is async-native |
| Model training | No | CPU/GPU-bound -- async won't help |
| pandas transforms | No | CPU-bound, single-threaded |
| File reads from local disk | Rarely | Disk I/O is fast; async adds complexity |

---

## AI Example: Decorator for Timing + Context Manager for GPU

```python
import time
import torch
from functools import wraps

def log_inference(func):
    """Decorator: log inference time and batch size."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        batch_size = len(args[1]) if len(args) > 1 else "unknown"
        print(f"[INFERENCE] {func.__name__}: {batch_size} samples in {elapsed:.3f}s")
        return result
    return wrapper

@log_inference
def predict_batch(model, inputs: torch.Tensor) -> torch.Tensor:
    model.eval()
    with torch.no_grad():  # Context manager: disable gradient tracking
        return model(inputs)

# [INFERENCE] predict_batch: 256 samples in 0.042s
```

---

## DE Example: Error Handling + Regex for Log Parsing

```python
import re
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)

@dataclass
class ParsedLogEntry:
    timestamp: str
    level: str
    module: str
    message: str

LOG_PATTERN = re.compile(
    r"(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})\s+(\w+)\s+\[(.+?)\]\s+(.+)"
)

def parse_log_file(filepath: str) -> tuple[list[ParsedLogEntry], int]:
    """Parse a structured log file. Return (entries, error_count)."""
    entries = []
    parse_errors = 0

    with open(filepath, "r") as f:
        for line_num, line in enumerate(f, start=1):
            line = line.strip()
            if not line:
                continue
            try:
                match = LOG_PATTERN.match(line)
                if not match:
                    raise ValueError(f"Line does not match expected format")
                entry = ParsedLogEntry(*match.groups())
                entries.append(entry)
            except Exception as e:
                parse_errors += 1
                logger.warning(f"Failed to parse line {line_num}: {e}")

    logger.info(f"Parsed {len(entries)} entries, {parse_errors} errors")
    return entries, parse_errors
```

---

## Quick Links

| Resource | Link |
|:---|:---|
| Python for AI (notebook) | [Python for AI on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_Basics.ipynb) |
| Python for DE (notebook) | [Python for DE on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_NumPy_Pandas.ipynb) |
| Previous chapter | [08 -- NumPy and Pandas](08_NumPy_Pandas.md) |
| Next chapter | [10 -- Python in Production](10_Python_for_Production.md) |

---

*Foundations -- Python (Chapter 9 of 10)*
