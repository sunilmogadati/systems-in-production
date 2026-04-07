# PySpark - Hello World

**Series:** PySpark Concept Chapters (3 of 10)
**Notebook:** [Python for DE on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/M03_Python_for_Data_Engineering.ipynb) (PySpark sections 18-22)

---

## Goal

Write a PySpark job that reads a CSV (Comma-Separated Values) file, filters it, groups it, counts rows, and prints the result. The same code runs on your laptop and on a 100-node cluster. You change nothing.

---

## Step 1: Create a SparkSession (3 Lines)

Every PySpark program starts by creating a SparkSession. This is your entry point to all Spark functionality.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("HelloPySpark") \
    .getOrCreate()
```

That is it. Three lines. Spark is running.

- `appName("HelloPySpark")` -- names your application (visible in the Spark User Interface (UI)).
- `getOrCreate()` -- creates a new session or reuses an existing one.

**You Should See:** No errors. If running locally, a line like `Spark session available as 'spark'` in the console. If running in a Jupyter notebook, the cell completes without output (silence means success).

---

## Step 2: Create Sample Data

For this Hello World, we will create a small DataFrame inline so you do not need an external file:

```python
data = [
    ("Alice", "Engineering", 95000),
    ("Bob", "Engineering", 102000),
    ("Carol", "Marketing", 78000),
    ("Dave", "Marketing", 81000),
    ("Eve", "Engineering", 110000),
    ("Frank", "Sales", 72000),
    ("Grace", "Sales", 68000),
    ("Hank", "Engineering", 98000),
]

df = spark.createDataFrame(data, ["name", "department", "salary"])
```

**You Should See:** Cell completes with no output. The DataFrame exists but nothing has been printed yet.

---

## Step 3: Look at the Data

```python
df.show()
```

**You Should See:**

```
+-----+-----------+------+
| name| department|salary|
+-----+-----------+------+
|Alice|Engineering| 95000|
|  Bob|Engineering|102000|
|Carol|  Marketing| 78000|
| Dave|  Marketing| 81000|
|  Eve|Engineering|110000|
|Frank|      Sales| 72000|
|Grace|      Sales| 68000|
| Hank|Engineering| 98000|
+-----+-----------+------+
```

```python
df.printSchema()
```

**You Should See:**

```
root
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- salary: long (nullable = true)
```

---

## Step 4: Filter, Group, Count

This is the core pattern. It looks almost identical to pandas:

```python
from pyspark.sql.functions import col, avg, count

# Filter to employees earning more than 75,000
high_earners = df.filter(col("salary") > 75000)

# Group by department, compute average salary and headcount
result = high_earners \
    .groupBy("department") \
    .agg(
        count("*").alias("headcount"),
        avg("salary").alias("avg_salary")
    )

result.show()
```

**You Should See:**

```
+-----------+---------+----------+
| department|headcount|avg_salary|
+-----------+---------+----------+
|Engineering|        4|  101250.0|
|  Marketing|        2|   79500.0|
|      Sales|        0|      null|
+-----------+---------+----------+
```

Wait -- Sales shows 0 and null because both Frank (72,000) and Grace (68,000) were filtered out. That is correct behavior. The filter removed them before the groupBy.

---

## Step 5: Read from a CSV File

In a real job, you read from files, not inline data. Here is how:

```python
# Local file
df = spark.read.csv("employees.csv", header=True, inferSchema=True)

# File on Google Cloud Storage (GCS)
df = spark.read.csv("gs://my-bucket/bronze/employees.csv", header=True, inferSchema=True)
```

- `header=True` -- first row contains column names.
- `inferSchema=True` -- Spark scans the data to detect types (string, integer, etc.). In production, you define the schema explicitly (see Chapter 05).

**You Should See:** A DataFrame with the correct columns and types. Run `df.printSchema()` to verify.

---

## The Proof: Same Code, Any Scale

Here is the key insight. This code:

```python
df = spark.read.csv("gs://my-bucket/bronze/logs/", header=True, inferSchema=True)
result = df.filter(col("status") == "ERROR").groupBy("service").count()
result.write.parquet("gs://my-bucket/silver/error_counts/")
```

Runs identically whether:

| Environment | What Happens |
|---|---|
| Your laptop (`pip install pyspark`) | Spark runs with 1 executor using local CPU cores. Processes megabytes. |
| GCP Dataproc cluster (10 nodes) | Spark distributes work across 10 executors. Processes terabytes. |
| GCP Dataproc cluster (100 nodes) | Spark distributes work across 100 executors. Processes petabytes. |

**You change NOTHING in the code.** The only difference is where you submit it.

---

## Running Locally vs Running on Dataproc

### Local (development and testing)

```bash
# Install PySpark
pip install pyspark

# Run your script
python my_spark_job.py
```

### GCP Dataproc (production)

```bash
# Submit to an existing Dataproc cluster
gcloud dataproc jobs submit pyspark \
    my_spark_job.py \
    --cluster=my-cluster \
    --region=us-central1

# Or submit to a serverless Dataproc batch (no cluster to manage)
gcloud dataproc batches submit pyspark \
    my_spark_job.py \
    --region=us-central1
```

See [Cloud Pipeline Concepts](../cloud-pipeline/02_Concepts.md) for Dataproc setup details.

---

## Common First-Time Issues

| Issue | Symptom | Fix |
|---|---|---|
| **Java not installed** | `JAVA_HOME is not set` error | Install Java Development Kit (JDK) 11 or 17. PySpark requires Java under the hood. |
| **Wrong Python version** | `ModuleNotFoundError: No module named 'pyspark'` | Ensure you installed PySpark in the same Python environment you are running. Use `which python` and `pip show pyspark` to verify. |
| **Port conflict** | `BindException: Address already in use` | Another Spark session or process is using port 4040. Stop it, or set `.config("spark.ui.port", "4041")` in your SparkSession builder. |
| **inferSchema slow on large files** | Job hangs during read | `inferSchema=True` scans the entire file to detect types. For large files, define the schema explicitly (Chapter 05). |
| **collect() crashes** | `OutOfMemoryError` on the Driver | `collect()` pulls ALL data to one machine. Use `show()` to preview or `write()` to save results. Never `collect()` a large DataFrame. |
| **Parquet write fails on local** | `FileAlreadyExistsException` | PySpark will not overwrite by default. Add `.mode("overwrite")` before `.parquet()`. |
| **Column not found** | `AnalysisException: cannot resolve 'colname'` | Column names are case-sensitive by default. Check spelling with `df.columns`. |

---

## What You Just Did

1. Created a SparkSession in 3 lines.
2. Built a DataFrame (inline and from CSV).
3. Filtered, grouped, and aggregated -- same patterns as pandas.
4. Saw that the code is identical whether running locally or on a cluster.

You now have a working PySpark program. Chapter 04 explains what Spark is doing behind the scenes when you run this code.

---

## Quick Links: PySpark Chapter Series

| Chapter | Title |
|---|---|
| 01 | [Why It Matters](01_Why.md) |
| 02 | [Concepts](02_Concepts.md) |
| **03** | [Hello World](03_Hello_World.md) |
| 04 | [How It Works](04_How_It_Works.md) |
| 05 | [Building It](05_Building_It.md) |
| 06 | [Production Patterns](06_Production_Patterns.md) |
| 07 | [System Design](07_System_Design.md) |
| 08 | [Quality, Security, and Governance](08_Quality_Security_Governance.md) |
| 09 | [Observability and Troubleshooting](09_Observability_Troubleshooting.md) |
| 10 | [Decision Guide](10_Decision_Guide.md) |
