# ETL/ELT Patterns - System Design

**Architecture for CDC at scale. How the pieces connect on GCP and AWS. When to stream and when to batch.**

---

## The Full CDC Architecture

A production Change Data Capture (CDC) system has five components:

1. **Source database** — where data originates (PostgreSQL, MySQL, MongoDB, Cloud SQL)
2. **CDC tool** — reads the transaction log and emits change events
3. **Message queue** — buffers events between producer and consumer
4. **Consumer** — reads events, transforms, and writes to the target
5. **Target warehouse** — where analytics queries run (BigQuery, Redshift, Snowflake)

```mermaid
graph LR
    subgraph "Source"
        A["Cloud SQL<br/>(PostgreSQL)"]
    end

    subgraph "Capture"
        B["Datastream<br/>(CDC tool)"]
    end

    subgraph "Buffer"
        C["Pub/Sub<br/>(message queue)"]
    end

    subgraph "Process"
        D["Dataflow<br/>(consumer)"]
    end

    subgraph "Target"
        E["BigQuery<br/>(warehouse)"]
    end

    A -->|"WAL"| B
    B -->|"Change events"| C
    C -->|"Subscribe"| D
    D -->|"MERGE"| E
```

### Why a Message Queue?

You could connect the CDC tool directly to the warehouse. But a message queue in between gives you:

| Benefit | What It Means |
|---|---|
| **Decoupling** | Source and target don't know about each other. Source can be down while the queue buffers events. |
| **Replay** | If the consumer crashes, replay events from the queue. No data loss. |
| **Fan-out** | Multiple consumers can read the same events. Send to BigQuery AND a monitoring system. |
| **Backpressure** | If the consumer is slow, the queue holds events until it catches up. Source isn't affected. |

---

## GCP Architecture

```mermaid
graph TD
    subgraph "Sources"
        S1["Cloud SQL<br/>(calls, orders)"]
        S2["Cloud Storage<br/>(CSV file drops)"]
        S3["External API<br/>(campaign data)"]
    end

    subgraph "Capture Layer"
        C1["Datastream<br/>(log-based CDC)"]
        C2["Cloud Functions<br/>(event trigger on file upload)"]
        C3["Cloud Scheduler<br/>(cron: pull API daily)"]
    end

    subgraph "Buffer"
        Q["Pub/Sub Topics<br/>calls-changes<br/>orders-changes<br/>files-arrived"]
    end

    subgraph "Process"
        P1["Dataflow<br/>(streaming consumer)"]
        P2["Cloud Composer / Airflow<br/>(batch orchestrator)"]
    end

    subgraph "Target"
        T1["BigQuery Silver<br/>(cleaned, merged)"]
        T2["BigQuery Gold<br/>(star schema marts)"]
    end

    S1 --> C1 --> Q
    S2 --> C2 --> Q
    S3 --> C3 --> Q
    Q --> P1 --> T1
    T1 --> P2 --> T2
```

### GCP Service Mapping

| Component | GCP Service | What It Does |
|---|---|---|
| CDC tool | **Datastream** | Reads PostgreSQL/MySQL WAL, emits change events to GCS or Pub/Sub |
| Message queue | **Pub/Sub** | Serverless message queue. Auto-scales. Pay per message. |
| Stream consumer | **Dataflow** | Apache Beam on GCP. Processes streaming events and writes to BigQuery. |
| Batch orchestrator | **Cloud Composer** | Managed Airflow. Schedules Silver→Gold transforms. |
| Warehouse | **BigQuery** | SQL analytics. Supports MERGE natively. |
| Event trigger | **Cloud Functions** | Serverless function triggered by file upload to GCS. |

### Cost Perspective

For a call center processing 500,000 calls per month:

| Service | Estimated Monthly Cost |
|---|---|
| Datastream | $0 (free tier covers up to 500 GB/month) |
| Pub/Sub | ~$5 (low message volume) |
| Dataflow | ~$50-$100 (streaming job) |
| BigQuery | ~$10-$30 (on-demand queries) |
| Cloud Composer | ~$300-$400 (the most expensive piece — managed Airflow environment) |

**The lesson:** Cloud Composer is often the most expensive component. For simple pipelines, consider using Cloud Scheduler + Cloud Functions instead of a full Airflow environment.

---

## AWS Architecture

```mermaid
graph TD
    subgraph "Sources"
        S1["RDS<br/>(PostgreSQL)"]
        S2["S3<br/>(CSV file drops)"]
        S3["External API"]
    end

    subgraph "Capture Layer"
        C1["DMS<br/>(Database Migration Service)"]
        C2["Lambda<br/>(S3 event trigger)"]
        C3["EventBridge<br/>(cron scheduler)"]
    end

    subgraph "Buffer"
        Q["Kinesis Data Streams<br/>or MSK (Managed Kafka)"]
    end

    subgraph "Process"
        P1["Lambda / Glue<br/>(consumer)"]
        P2["MWAA / Step Functions<br/>(batch orchestrator)"]
    end

    subgraph "Target"
        T1["Redshift / Athena<br/>(warehouse)"]
    end

    S1 --> C1 --> Q
    S2 --> C2 --> Q
    S3 --> C3 --> Q
    Q --> P1 --> T1
    T1 --> P2
```

### AWS Service Mapping

| Component | AWS Service | GCP Equivalent |
|---|---|---|
| CDC tool | **DMS** (Database Migration Service) | Datastream |
| Message queue | **Kinesis** or **MSK** (Managed Kafka) | Pub/Sub |
| Stream consumer | **Lambda** or **Glue Streaming** | Dataflow |
| Batch orchestrator | **MWAA** (Managed Airflow) or **Step Functions** | Cloud Composer |
| Warehouse | **Redshift** or **Athena** (on S3) | BigQuery |
| Event trigger | **Lambda** (S3 event) | Cloud Functions |

---

## Streaming vs Batch: When to Use Which

```mermaid
graph TD
    A["How fresh does data need to be?"] --> B{"Seconds?"}
    B -->|"Yes"| C["Streaming CDC<br/>Datastream → Pub/Sub → Dataflow → BigQuery"]
    B -->|"No"| D{"Minutes?"}
    D -->|"Yes"| E["Micro-batch<br/>Query-based CDC every 5 minutes"]
    D -->|"No"| F{"Hours?"}
    F -->|"Yes"| G["Batch Incremental<br/>Nightly watermark-based load"]
    F -->|"No"| H["Full Refresh<br/>Simple. Works at small scale."]
```

| Factor | Streaming CDC | Batch Incremental | Full Refresh |
|---|---|---|---|
| **Freshness** | Seconds | Minutes to hours | Hours |
| **Complexity** | High | Medium | Low |
| **Cost** | Higher (always-on) | Lower (runs periodically) | Lowest (simple job) |
| **Captures deletes** | Yes | No (unless tracked) | Yes (by replacement) |
| **Operational burden** | High (monitor stream lag) | Medium (monitor watermarks) | Low (monitor job success) |
| **Best for** | Real-time dashboards, fraud detection | Standard analytics, daily reports | Small reference tables |

**The practical answer:** Most data engineering teams start with batch incremental and graduate to streaming CDC only when business requirements demand real-time freshness. Don't build streaming complexity for data that's consumed in daily reports.

---

## Hybrid Architecture

Most production systems use both patterns:

```mermaid
graph TD
    subgraph "Real-Time Path (CDC)"
        RT1["High-change tables<br/>(calls, orders)"] --> RT2["Datastream"] --> RT3["Pub/Sub"] --> RT4["Dataflow"] --> RT5["BigQuery Silver<br/>(near real-time)"]
    end

    subgraph "Batch Path (Incremental)"
        B1["Low-change tables<br/>(campaigns, products)"] --> B2["Cloud Scheduler<br/>(daily)"] --> B3["Dataproc PySpark"] --> B4["BigQuery Silver<br/>(daily refresh)"]
    end

    subgraph "Gold Layer (Batch)"
        RT5 --> G["Cloud Composer<br/>(hourly/daily)"]
        B4 --> G
        G --> G1["BigQuery Gold<br/>(star schema marts)"]
    end
```

**The pattern:** Stream the high-volume, frequently changing data (calls, orders). Batch the slow-changing reference data (campaigns, products, agents). Build Gold marts on a schedule regardless.

---

## Quick Links

| Chapter | Topic |
|---|---|
| [06 - Production Patterns](06_Production_Patterns.md) | Late-arriving data, backfill, idempotency |
| [07 - System Design](07_System_Design.md) | This page |
| [08 - Quality Security Governance](08_Quality_Security_Governance.md) | PII, schema drift, audit trails |
| [09 - Observability Troubleshooting](09_Observability_Troubleshooting.md) | Monitoring CDC lag, debugging failures |
