# Cloud Data Pipelines - Why They Matter

**Why data doesn't just "appear" in a dashboard, and why the path it takes determines whether you can trust it.**

> These concepts apply to any cloud (GCP, AWS, Azure). Examples use GCP as the primary, with AWS equivalents noted. The hands-on notebooks are cloud-specific: [GCP Pipeline](../../../../implementation/notebooks/GCP_Full_Pipeline.ipynb) | AWS Pipeline (coming soon).

---

## The 3 AM Call

A VP of Operations looks at the morning dashboard. Total orders yesterday: 12,847. But Finance says their system shows 13,201. The warehouse shipped based on a third number. Three systems, three different answers, and a meeting in 2 hours where someone has to explain which one is right.

Nobody can. Because nobody knows where each number came from, what transformations it went through, or when it was last updated.

This is what happens when data moves through a system without a pipeline.

---

## What Is a Data Pipeline?

A data pipeline is the path data travels from where it's created to where it's used.

Think of it like a water treatment plant:

```mermaid
graph LR
    A["River Water<br/>(raw, untreated)"] --> B["Intake Station<br/>(collect from source)"]
    B --> C["Treatment Plant<br/>(filter, clean, test)"]
    C --> D["Storage Tank<br/>(clean, ready)"]
    D --> E["Your Tap<br/>(safe to drink)"]
```

Now replace water with data:

```mermaid
graph LR
    A["Source Systems<br/>(databases, APIs, files)"] --> B["Ingestion<br/>(collect from sources)"]
    B --> C["Transformation<br/>(clean, deduplicate, validate)"]
    C --> D["Storage<br/>(warehouse, organized)"]
    D --> E["Consumers<br/>(dashboards, ML models, reports)"]
```

Without the treatment plant, you're drinking river water. Without a pipeline, you're making decisions on raw, unvalidated data.

---

## Why Not Just Query the Source Directly?

This is the most common question, and the answer matters.

Source systems (the database behind your application, the CRM, the payment processor) are designed to **write fast**. They're optimized for transactions: "record this order," "update this customer," "log this event."

They are NOT designed to answer questions like: "What was the average order value by campaign by month for the last year, compared to the same period last year?"

| Source System | Optimized For | Bad At |
|---|---|---|
| Application database (PostgreSQL, MongoDB) | Writing one record fast | Aggregating millions of records |
| CRM (Salesforce, HubSpot) | Managing customer records | Joining with order and payment data |
| Payment processor (Stripe) | Processing transactions | Historical trend analysis |
| Log system (Datadog, CloudWatch) | Streaming events | Ad-hoc queries across months |

If you query the source directly:
- You slow down the production system (your app gets slow)
- You get inconsistent results (data is changing while you query)
- You can't join across systems (data lives in different places)
- You can't go back in time (source systems overwrite, they don't version)

A pipeline solves all of these by copying data out of source systems, transforming it, and putting it somewhere designed for analysis.

---

## The Three Layers: Bronze, Silver, Gold

This is called the **Medallion Architecture**. Every modern data pipeline uses some version of this.

```mermaid
graph TD
    subgraph Sources
        S1["Phone System<br/>(calls.json)"]
        S2["Order System<br/>(orders.csv)"]
        S3["Campaign Platform<br/>(campaigns.csv)"]
        S4["Payment Processor<br/>(payments.xml)"]
    end

    subgraph Bronze ["Bronze Layer (Raw)"]
        B1["Exact copy of source data<br/>No changes, no cleaning<br/>Timestamp of when it arrived"]
    end

    subgraph Silver ["Silver Layer (Cleaned)"]
        SV1["Deduplicated<br/>Timezone-fixed<br/>Type-corrected<br/>Orphans quarantined"]
    end

    subgraph Gold ["Gold Layer (Business-Ready)"]
        G1["Star schema<br/>Pre-aggregated marts<br/>Ready for dashboards and ML"]
    end

    S1 --> B1
    S2 --> B1
    S3 --> B1
    S4 --> B1
    B1 --> SV1
    SV1 --> G1

    G1 --> D1["Dashboards"]
    G1 --> D2["ML Models"]
    G1 --> D3["Reports"]
```

### Bronze: "Save everything, change nothing"

The raw data exactly as it arrived from the source. If the source sent a duplicate, Bronze has the duplicate. If a timestamp is in the wrong timezone, Bronze keeps it wrong.

**Why keep raw data?** Because if your Silver transform has a bug, you can reprocess from Bronze. If business rules change, you can reprocess from Bronze. Bronze is your insurance policy.

**Analogy:** A security camera recording. You don't edit the footage. You keep the original. If someone asks "what really happened?", you go back to the tape.

### Silver: "Clean it, validate it, trust it"

This is where data engineering happens. The raw data gets:
- **Deduplicated**: Same call recorded twice? Keep one.
- **Timezone-fixed**: UTC timestamps converted to business timezone.
- **Type-corrected**: String "123.45" becomes number 123.45.
- **Joined**: Customer data linked to their orders.
- **Quarantined**: Orphaned records (order with no matching call) go to a dead letter queue, not deleted.

**Analogy:** The water treatment plant. Filter out the dirt, test for contaminants, make it safe. But keep a record of what you filtered out and why.

**The key rule:** Never silently drop data. Quarantine it, log it, alert on it. You can always decide to delete later. You can never get back data you already dropped.

### Where Does Silver Actually Live?

Silver is a concept — a layer of cleaned data. But WHERE it lives depends on your pipeline pattern:

```mermaid
graph TD
    subgraph "Pattern 1: ELT (transform inside warehouse)"
        A1["GCS<br/>Bronze (raw files)"] -->|"bq load"| B1["BigQuery raw dataset<br/>(Bronze tables)"]
        B1 -->|"SQL transform"| C1["BigQuery silver dataset<br/>(Silver tables)"]
        C1 -->|"SQL aggregate"| D1["BigQuery gold dataset<br/>(Gold tables)"]
        
        style C1 fill:#c0c0c0
    end
```

```mermaid
graph TD
    subgraph "Pattern 2: ETL (transform outside warehouse)"
        A2["GCS<br/>Bronze (raw files)"] -->|"PySpark"| B2["GCS silver folder<br/>(cleaned Parquet/Delta)"]
        B2 -->|"bq load or BigLake"| C2["BigQuery<br/>(Gold tables)"]
        
        style B2 fill:#c0c0c0
    end
```

```mermaid
graph TD
    subgraph "Pattern 3: Hybrid (Delta Lake + BigLake)"
        A3["GCS<br/>Bronze (raw files)"] -->|"PySpark"| B3["GCS Delta table<br/>(Silver — ACID, versioned)"]
        B3 -->|"BigLake reads directly"| C3["BigQuery<br/>(queries Silver + builds Gold)"]
        
        style B3 fill:#c0c0c0
    end
```

| Pattern | Where Silver lives | Transform runs in | When to use |
|---|---|---|---|
| **ELT** | BigQuery dataset (`silver`) | BigQuery SQL | Warehouse can handle the transform. Most common today. |
| **ETL** | GCS folder (`gs://bucket/silver/`) | PySpark on Dataproc | Heavy processing, multi-source joins, ML features |
| **Hybrid** | GCS Delta table, BigQuery reads via BigLake | PySpark writes, BigQuery reads | Need ACID on files + SQL analytics on same data |

**In this material, we use ELT** — Silver is a BigQuery dataset. The transform is SQL running inside BigQuery. If you see `silver.calls`, that's a BigQuery table, not a file in GCS.

When the processing is too heavy for SQL (complex joins, ML feature engineering, or the data is too large), you move the Silver step to PySpark and Silver becomes files in GCS. The concept doesn't change — only the location does.

### Gold: "Answer business questions"

Pre-computed answers to the questions people actually ask. Star schema tables optimized for analysis.

**Examples:**
- `mart_daily_campaign`: How did each campaign perform today?
- `mart_hourly_volume`: When do calls peak?
- `mart_conversion`: What's our call-to-order conversion rate?

**Analogy:** A menu at a restaurant. The kitchen (Silver) can make anything. But the menu (Gold) lists the dishes people actually order. Pre-made, consistent, fast to serve.

---

## Why Cloud? Why Not a Local Server?

| Factor | Local Server | Cloud (GCP/AWS) |
|---|---|---|
| **Cost to start** | Buy hardware ($10K-$100K) | Pay per use ($0 to start) |
| **Scale** | Buy more hardware (weeks) | Click a button (minutes) |
| **Maintenance** | Your team patches, updates, fixes | Cloud provider handles it |
| **Disaster recovery** | Build it yourself | Built in |
| **Global availability** | Build data centers | Already there |

The tradeoff: cloud is cheaper to start but can get expensive at scale if you're not careful. That's why cost optimization is a critical pipeline skill (covered in later chapters).

---

## What We're Building

In this material, we build a complete pipeline on Google Cloud Platform:

```mermaid
graph LR
    subgraph Sources
        A["calls.json"]
        B["orders.csv"]
        C["campaigns.csv"]
    end

    subgraph GCP
        D["GCS Bucket<br/>(Bronze)"]
        E["Dataproc / PySpark<br/>(Silver transform)"]
        F["BigQuery<br/>(Gold marts)"]
    end

    subgraph Consumers
        G["Looker Studio<br/>(dashboards)"]
        H["Vertex AI<br/>(ML models)"]
    end

    A --> D
    B --> D
    C --> D
    D --> E
    E --> F
    F --> G
    F --> H
```

Every step is executable. Every step has a "You Should See" checkpoint. Every step explains WHY before HOW.

---

## The Skills This Builds

| What You Learn | Where It Shows Up |
|---|---|
| Ingestion patterns (batch, streaming, CDC) | Every data engineering role |
| Data quality enforcement | Every production system |
| Cloud services (GCS, BigQuery, Dataproc) | GCP certifications, job interviews |
| Medallion architecture | System design interviews |
| Cost optimization | Real-world cloud bills |
| Monitoring and alerting | Production operations |

---

## Quick Links

| Chapter | Topic |
|---|---|
| [01 - Why](01_Why.md) | This page |
| [02 - Concepts](02_Concepts.md) | Cloud services in plain English, how they connect |
| [03 - Hello World](03_Hello_World.md) | Upload data to GCS, query in BigQuery, see a result |
