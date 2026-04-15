# Data Quality Tools - Building It

**Implement automated quality checks three ways: Great Expectations, dbt tests, and custom Python. Each approach runs against the call center dataset.**

---

## Approach 1: Great Expectations

### Setup

```python
pip install great_expectations
```

### Define an Expectation Suite

```python
import great_expectations as gx

# Create a data context (configuration hub)
context = gx.get_context()

# Connect to the data source
datasource = context.sources.add_pandas("calls_source")
data_asset = datasource.add_csv_asset("calls", filepath_or_buffer="silver_calls.csv")
batch_request = data_asset.build_batch_request()

# Create a validator
validator = context.get_validator(
    batch_request=batch_request,
    expectation_suite_name="calls_quality_suite",
    create_expectation_suite_with_name="calls_quality_suite",
)

# --- Define expectations ---

# Table-level
validator.expect_table_row_count_to_be_between(min_value=100, max_value=500000)

# Primary key: never null, always unique
validator.expect_column_values_to_not_be_null("call_id")
validator.expect_column_values_to_be_unique("call_id")

# Status: must be a known value
validator.expect_column_values_to_be_in_set(
    "status",
    ["in-progress", "resolved", "missed", "voicemail", "transferred"]
)

# Duration: non-negative, max 8 hours (28800 seconds)
validator.expect_column_values_to_be_between(
    "duration", min_value=0, max_value=28800
)

# No nulls in required fields
validator.expect_column_values_to_not_be_null("customer_id")
validator.expect_column_values_to_not_be_null("created_at")

# Timestamp sanity: created_at should not be in the future
validator.expect_column_values_to_be_between(
    "created_at", max_value=datetime.now().isoformat()
)

# Save the suite
validator.save_expectation_suite(discard_failed_expectations=False)
print("Expectation suite saved: calls_quality_suite")
```

### Run as a Checkpoint (Pipeline Integration)

```python
# A checkpoint is the runnable unit — call this from Airflow or your pipeline
checkpoint = context.add_or_update_checkpoint(
    name="calls_checkpoint",
    validator=validator,
    action_list=[
        # Action 1: Store results
        {
            "name": "store_validation_result",
            "action": {"class_name": "StoreValidationResultAction"},
        },
        # Action 2: Update Data Docs (HTML report)
        {
            "name": "update_data_docs",
            "action": {"class_name": "UpdateDataDocsAction"},
        },
    ],
)

# Run the checkpoint
result = checkpoint.run()

if not result.success:
    # Pipeline should halt — quality gate failed
    failed = [r for r in result.run_results.values() if not r.success]
    raise Exception(f"Quality gate FAILED: {len(failed)} checks failed")
else:
    print("All quality checks passed")
```

### Airflow Integration

```python
# In your Airflow DAG
from airflow.operators.python import PythonOperator

def run_quality_checks():
    context = gx.get_context()
    checkpoint = context.get_checkpoint("calls_checkpoint")
    result = checkpoint.run()
    if not result.success:
        raise Exception("Data quality gate failed")

quality_gate = PythonOperator(
    task_id="quality_gate",
    python_callable=run_quality_checks,
    dag=dag,
)

# Pipeline flow: transform → quality_gate → build_gold
transform_silver >> quality_gate >> build_gold_marts
```

---

## Approach 2: dbt Tests

### Generic Tests (YAML)

```yaml
# models/schema.yml
version: 2

models:
  - name: silver_calls
    description: "Cleaned call records — deduplicated, timezone-fixed, validated"
    columns:
      - name: call_id
        description: "Unique call identifier"
        tests:
          - not_null
          - unique

      - name: status
        description: "Call outcome"
        tests:
          - not_null
          - accepted_values:
              values: ['in-progress', 'resolved', 'missed', 'voicemail', 'transferred']

      - name: duration
        description: "Call duration in seconds"
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 28800

      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_customer')
              field: customer_id

      - name: created_at
        tests:
          - not_null
```

### Singular Tests (Custom SQL)

```sql
-- tests/assert_no_duplicate_calls_per_day.sql
-- Fails if any call_id appears more than once on the same day
SELECT
    call_id,
    call_date,
    COUNT(*) AS occurrences
FROM {{ ref('silver_calls') }}
GROUP BY call_id, call_date
HAVING COUNT(*) > 1
```

```sql
-- tests/assert_row_count_reasonable.sql
-- Fails if today's load has < 50% of the 7-day average
WITH daily_counts AS (
    SELECT
        call_date,
        COUNT(*) AS daily_count
    FROM {{ ref('silver_calls') }}
    WHERE call_date >= CURRENT_DATE - 8
    GROUP BY call_date
),
avg_count AS (
    SELECT AVG(daily_count) AS avg_7d
    FROM daily_counts
    WHERE call_date < CURRENT_DATE
)
SELECT
    dc.call_date,
    dc.daily_count,
    ac.avg_7d,
    dc.daily_count / ac.avg_7d AS ratio
FROM daily_counts dc
CROSS JOIN avg_count ac
WHERE dc.call_date = CURRENT_DATE
    AND dc.daily_count < ac.avg_7d * 0.5
```

### Run

```bash
# Run all tests
dbt test

# Run tests for one model only
dbt test --select silver_calls

# Run tests as part of the full build
dbt build  # runs models + tests together
```

---

## Approach 3: Custom Python (No Framework)

When you don't want a framework — just Python functions that return pass/fail.

```python
import pandas as pd
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class CheckResult:
    check_name: str
    passed: bool
    details: Optional[str] = None
    failing_rows: int = 0

def run_quality_checks(df: pd.DataFrame) -> List[CheckResult]:
    """Run all quality checks on a DataFrame. Return results."""
    results = []
    
    # Check 1: Row count
    count = len(df)
    results.append(CheckResult(
        check_name="row_count_minimum",
        passed=count >= 100,
        details=f"Row count: {count}" if count < 100 else None,
        failing_rows=0 if count >= 100 else 1,
    ))
    
    # Check 2: No null primary keys
    null_pks = df["call_id"].isna().sum()
    results.append(CheckResult(
        check_name="call_id_not_null",
        passed=null_pks == 0,
        details=f"{null_pks} null call_ids" if null_pks > 0 else None,
        failing_rows=null_pks,
    ))
    
    # Check 3: No duplicates
    dupes = df.duplicated(subset=["call_id"]).sum()
    results.append(CheckResult(
        check_name="call_id_unique",
        passed=dupes == 0,
        details=f"{dupes} duplicate call_ids" if dupes > 0 else None,
        failing_rows=dupes,
    ))
    
    # Check 4: Valid status values
    valid_statuses = {"in-progress", "resolved", "missed", "voicemail", "transferred"}
    invalid = df[~df["status"].isin(valid_statuses)]
    results.append(CheckResult(
        check_name="status_valid",
        passed=len(invalid) == 0,
        details=f"Invalid statuses: {invalid['status'].unique().tolist()}" if len(invalid) > 0 else None,
        failing_rows=len(invalid),
    ))
    
    # Check 5: Duration range
    bad_duration = df[(df["duration"] < 0) | (df["duration"] > 28800)]
    results.append(CheckResult(
        check_name="duration_in_range",
        passed=len(bad_duration) == 0,
        details=f"Out of range: min={df['duration'].min()}, max={df['duration'].max()}" if len(bad_duration) > 0 else None,
        failing_rows=len(bad_duration),
    ))
    
    return results

def report_results(results: List[CheckResult]):
    """Print results and raise if any failed."""
    passed = sum(1 for r in results if r.passed)
    failed = sum(1 for r in results if not r.passed)
    
    print(f"\nQuality Gate: {passed} passed, {failed} failed")
    print("-" * 50)
    for r in results:
        status = "PASS" if r.passed else "FAIL"
        print(f"  [{status}] {r.check_name}", end="")
        if r.details:
            print(f" — {r.details}", end="")
        print()
    
    if failed > 0:
        raise Exception(f"Quality gate FAILED: {failed} checks failed")

# Usage
# df = pd.read_parquet("silver_calls.parquet")
# results = run_quality_checks(df)
# report_results(results)
```

**When to use custom instead of a framework:**
- You have < 10 tables to check
- You don't want to learn Great Expectations' configuration system
- You need full control over check logic and reporting
- You plan to migrate to a framework later (custom checks translate easily)

---

## Which Approach for Which Situation

| Situation | Recommended Approach |
|---|---|
| Already using dbt | dbt tests — zero new tooling |
| PySpark pipeline, many tables | Great Expectations — richest library |
| Small team, few tables | Custom Python — simplest, no dependencies |
| Want YAML config, low code | Soda — SodaCL is concise |
| All on GCP BigQuery | Dataplex Auto DQ + dbt tests or custom |
| Enterprise, need observability | Monte Carlo or Great Expectations + dashboarding |

---

## Quick Links

| Chapter | Topic |
|---|---|
| [02 - Tools Compared](02_Tools_Compared.md) | Feature comparison matrix |
| [03 - Building It](03_Building_It.md) | This page |
| [04 - Cloud Walkthroughs](04_Cloud_Walkthroughs.md) | Dataplex, Glue DQ, Synapse DQ |
