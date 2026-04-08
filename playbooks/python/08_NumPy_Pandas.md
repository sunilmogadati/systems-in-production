# NumPy and Pandas -- Data Manipulation for AI

**NumPy makes math fast. Pandas makes data manipulation expressive. Together, they are the foundation that every AI/ML library and every data pipeline is built on. If you learn two Python libraries, learn these two.**

---

## NumPy: Arrays, Vectorized Operations, Broadcasting

### Why NumPy exists

Python lists are slow for math because each element is a full Python object with overhead. NumPy stores data as contiguous blocks of typed memory (like C arrays), then runs operations in compiled C code:

```python
import numpy as np

# Python list: each element is a Python object (~28 bytes each)
python_list = [1.0, 2.0, 3.0, 4.0, 5.0]

# NumPy array: contiguous block of 8-byte floats
np_array = np.array([1.0, 2.0, 3.0, 4.0, 5.0])
```

**Performance difference on 1 million elements:**

| Operation | Python list | NumPy array | Speedup |
|:---|:---|:---|:---|
| Element-wise multiply by 2 | ~150ms | ~1ms | 150x |
| Sum all elements | ~50ms | ~0.5ms | 100x |
| Dot product | ~200ms | ~1ms | 200x |

### Vectorized operations -- no loops needed

```python
import numpy as np

# Feature scaling: subtract mean, divide by std (standardization)
features = np.array([120, 45, 300, 180, 95, 420, 60, 210, 150])

# Vectorized -- operates on the entire array at once
mean = features.mean()         # 175.6
std = features.std()           # 108.7
standardized = (features - mean) / std  # One operation on all 9 values

# The loop equivalent is 100x slower and harder to read:
# standardized = [(x - mean) / std for x in features]
```

### Broadcasting -- different shapes, automatic alignment

Broadcasting is NumPy's rule for combining arrays of different shapes. This is what makes matrix math concise:

```python
# Add a scalar to every element (scalar is "broadcast" to match array shape)
predictions = np.array([0.1, 0.5, 0.9])
adjusted = predictions + 0.05  # [0.15, 0.55, 0.95]

# Add a column vector to every row of a matrix
# Shape (3, 4) + shape (3, 1) -> shape (3, 4)
batch = np.array([[1, 2, 3, 4],
                   [5, 6, 7, 8],
                   [9, 10, 11, 12]])
bias = np.array([[10], [20], [30]])  # Column vector
result = batch + bias
# [[11, 12, 13, 14],
#  [25, 26, 27, 28],
#  [39, 40, 41, 42]]
```

### Common NumPy operations for AI

```python
# Create arrays
zeros = np.zeros((3, 4))          # 3x4 matrix of zeros
ones = np.ones((3, 4))            # 3x4 matrix of ones
random = np.random.randn(3, 4)    # 3x4 matrix of random normal values

# Reshape -- critical for feeding data into neural networks
flat = np.array([1, 2, 3, 4, 5, 6])
matrix = flat.reshape(2, 3)       # 2 rows, 3 columns
# [[1, 2, 3],
#  [4, 5, 6]]

# Linear algebra -- used in embeddings, PCA, recommendations
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])
dot_product = A @ B               # Matrix multiplication (Python 3.5+)
```

---

## Pandas: DataFrame, Series, and the 10 Operations That Cover 90% of Work

### DataFrame basics

```python
import pandas as pd

# Create from dict (most common)
df = pd.DataFrame({
    "call_id": ["C-1001", "C-1002", "C-1003"],
    "duration_sec": [120, 45, 300],
    "agent": ["Alice", "Bob", "Alice"],
    "outcome": ["resolved", "resolved", "escalated"]
})

# Read from file
df = pd.read_csv("calls.csv")
df = pd.read_parquet("calls.parquet")

# Basic inspection
df.shape          # (rows, columns)
df.dtypes         # Column types
df.head(5)        # First 5 rows
df.describe()     # Summary statistics for numeric columns
df.info()         # Column names, types, non-null counts
```

### The 10 pandas operations you use 90% of the time

| # | Operation | Code | What It Does |
|:---|:---|:---|:---|
| 1 | **Filter rows** | `df[df["duration"] > 60]` | Keep rows matching a condition |
| 2 | **Select columns** | `df[["call_id", "agent"]]` | Keep specific columns |
| 3 | **Create columns** | `df["wait_ratio"] = df["wait"] / df["duration"]` | Add derived features |
| 4 | **Group and aggregate** | `df.groupby("agent")["duration"].mean()` | SQL-style GROUP BY |
| 5 | **Sort** | `df.sort_values("duration", ascending=False)` | Order rows |
| 6 | **Handle nulls** | `df.dropna()` or `df.fillna(0)` | Remove or fill missing values |
| 7 | **Remove duplicates** | `df.drop_duplicates(subset=["call_id"])` | Deduplicate |
| 8 | **Merge/join** | `pd.merge(calls, agents, on="agent_id")` | SQL-style JOIN |
| 9 | **Apply function** | `df["col"].apply(lambda x: x.upper())` | Transform with custom logic |
| 10 | **Value counts** | `df["outcome"].value_counts()` | Frequency distribution |

### Each operation with a real example

```python
# 1. Filter: keep only escalated calls
escalated = df[df["outcome"] == "escalated"]

# 2. Select: only the columns the model needs
features = df[["duration_sec", "wait_sec", "satisfaction"]]

# 3. Create: derive a new feature
df["call_minutes"] = df["duration_sec"] / 60

# 4. Group: average duration by agent
avg_by_agent = df.groupby("agent")["duration_sec"].mean()

# 5. Sort: longest calls first
df_sorted = df.sort_values("duration_sec", ascending=False)

# 6. Handle nulls: fill missing satisfaction with median
df["satisfaction"] = df["satisfaction"].fillna(df["satisfaction"].median())

# 7. Deduplicate: keep first occurrence of each call_id
df_deduped = df.drop_duplicates(subset=["call_id"], keep="first")

# 8. Merge: join call data with agent metadata
agents = pd.DataFrame({"agent": ["Alice", "Bob"], "team": ["A", "B"]})
merged = pd.merge(df, agents, on="agent", how="left")

# 9. Apply: categorize call duration
df["length_cat"] = df["duration_sec"].apply(
    lambda d: "short" if d < 60 else "medium" if d < 300 else "long"
)

# 10. Value counts: distribution of outcomes
print(df["outcome"].value_counts())
# resolved     6
# escalated    3
# dropped      1
```

---

## Method Chaining -- The Clean Way to Write Pandas

Method chaining reads like a pipeline: each step transforms the DataFrame and passes it to the next. This is the pattern used in production code:

```python
# Without chaining -- multiple intermediate variables
df1 = df.dropna(subset=["call_id"])
df2 = df1[df1["duration_sec"] > 0]
df3 = df2.drop_duplicates(subset=["call_id"])
df4 = df3.assign(wait_ratio=df3["wait_sec"] / df3["duration_sec"])
result = df4.sort_values("wait_ratio", ascending=False)

# With chaining -- one continuous pipeline, no intermediate variables
result = (
    df
    .dropna(subset=["call_id"])
    .query("duration_sec > 0")
    .drop_duplicates(subset=["call_id"])
    .assign(wait_ratio=lambda x: x["wait_sec"] / x["duration_sec"])
    .sort_values("wait_ratio", ascending=False)
)
```

**Why chaining matters:** Each line is one transformation. You can comment, reorder, or remove any line independently. This is the same pattern that PySpark and dbt use.

---

## Handling Missing Data

Missing data (NaN -- Not a Number) is the most common data quality issue in both AI and DE:

```python
# Detect missing values
df.isna().sum()               # Count nulls per column
df[df["agent"].isna()]        # Show rows where agent is missing

# Drop rows with any null
df.dropna()

# Drop rows where specific columns are null
df.dropna(subset=["call_id", "duration_sec"])

# Fill with a constant
df["agent"] = df["agent"].fillna("UNKNOWN")

# Fill with a computed value
df["satisfaction"] = df["satisfaction"].fillna(df["satisfaction"].median())

# Forward fill (useful for time series -- carry last known value forward)
df["price"] = df["price"].ffill()

# Interpolate (useful for sensor data, numeric time series)
df["temperature"] = df["temperature"].interpolate(method="linear")
```

---

## Pandas vs PySpark Comparison

Same concepts, different syntax. If you learn pandas, PySpark is a translation exercise:

| Operation | pandas | PySpark |
|:---|:---|:---|
| Create from dict | `pd.DataFrame(data)` | `spark.createDataFrame(data)` |
| Read CSV | `pd.read_csv("f.csv")` | `spark.read.csv("f.csv", header=True)` |
| Read Parquet | `pd.read_parquet("f.parquet")` | `spark.read.parquet("f.parquet")` |
| Filter rows | `df[df["col"] > 5]` | `df.filter(col("col") > 5)` |
| Select columns | `df[["a", "b"]]` | `df.select("a", "b")` |
| Add column | `df["new"] = df["a"] + 1` | `df.withColumn("new", col("a") + 1)` |
| Group + agg | `df.groupby("a")["b"].mean()` | `df.groupBy("a").agg(avg("b"))` |
| Sort | `df.sort_values("a")` | `df.orderBy("a")` |
| Drop nulls | `df.dropna()` | `df.dropna()` |
| Drop dupes | `df.drop_duplicates(["a"])` | `df.dropDuplicates(["a"])` |
| Join | `pd.merge(a, b, on="key")` | `a.join(b, on="key")` |
| Write Parquet | `df.to_parquet("out.parquet")` | `df.write.parquet("out.parquet")` |

**Key difference:** pandas runs on one machine (data must fit in memory). PySpark distributes across a cluster (handles TB-scale data). Start with pandas for prototyping, move to PySpark when data exceeds memory.

---

## AI Example: Feature Engineering with Pandas

```python
import pandas as pd
import numpy as np

df = pd.read_parquet("calls.parquet")

# 1. Create derived features
features = (
    df
    .assign(
        # Ratio features -- often more predictive than raw values
        wait_ratio=lambda x: x["wait_sec"] / x["duration_sec"].clip(lower=1),
        # Binned feature -- reduces noise for tree-based models
        duration_bucket=lambda x: pd.cut(
            x["duration_sec"],
            bins=[0, 60, 180, 600, float("inf")],
            labels=["short", "medium", "long", "very_long"]
        ),
    )
)

# 2. One-hot encode categorical columns
features = pd.get_dummies(features, columns=["outcome", "duration_bucket"])

# 3. Normalize numeric columns (min-max scaling)
numeric_cols = ["duration_sec", "wait_sec", "wait_ratio"]
for col in numeric_cols:
    min_val = features[col].min()
    max_val = features[col].max()
    features[col] = (features[col] - min_val) / (max_val - min_val)

# 4. Drop non-feature columns
X = features.drop(["call_id", "agent", "created_at"], axis=1, errors="ignore")
```

---

## DE Example: Data Quality Checks with Pandas

```python
import pandas as pd

df = pd.read_parquet("bronze/calls.parquet")

# 1. Null counts per column
null_report = df.isna().sum().to_frame("null_count")
null_report["null_pct"] = (null_report["null_count"] / len(df) * 100).round(2)
print(null_report[null_report["null_count"] > 0])

# 2. Duplicate detection
dupes = df.duplicated(subset=["call_id"], keep=False)
print(f"Duplicate call_ids: {dupes.sum()}")

# 3. Distribution stats for numeric columns
print(df.describe())

# 4. Value frequency checks (detect category drift)
print(df["outcome"].value_counts(normalize=True))

# 5. Range validation
out_of_range = df[
    (df["duration_sec"] < 0) |
    (df["duration_sec"] > 86400) |  # More than 24 hours is suspect
    (df["satisfaction"] < 1) |
    (df["satisfaction"] > 5)
]
if len(out_of_range) > 0:
    print(f"WARNING: {len(out_of_range)} records out of expected range")

# 6. Compile into a quality report
quality_summary = {
    "total_records": len(df),
    "duplicate_records": dupes.sum(),
    "null_percentage": df.isna().mean().mean() * 100,
    "columns_with_nulls": (df.isna().sum() > 0).sum(),
    "status": "PASS" if dupes.sum() == 0 and df.isna().mean().mean() < 0.05 else "REVIEW"
}
```

---

## Quick Links

| Resource | Link |
|:---|:---|
| Python for AI (notebook) | [Python for AI on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_for_AI.ipynb) |
| Python for DE (notebook) | [Python for DE on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_for_DE.ipynb) |
| Previous chapter | [07 -- File I/O and Data Formats](07_File_IO_Data_Formats.md) |
| Next chapter | [09 -- Advanced Patterns](09_Advanced_Patterns.md) |

---

*Foundations -- Python (Chapter 8 of 10)*
