# Ingestion Patterns - System Design

**Ingestion architecture at scale. Build vs buy. How Airbyte, Fivetran, and custom pipelines compare. Multi-source ingestion platform design.**

---

## The Ingestion Platform

At scale, you don't build one pipeline per source. You build an **ingestion platform** — a system that manages connections, extractions, scheduling, monitoring, and schema tracking for all sources.

```mermaid
graph TD
    subgraph "Sources"
        S1["PostgreSQL"]
        S2["REST API<br/>(CRM)"]
        S3["Kafka<br/>(events)"]
        S4["MongoDB"]
        S5["SFTP<br/>(vendor files)"]
    end

    subgraph "Ingestion Platform"
        CONN["Connection Manager<br/>(credentials, endpoints)"]
        SCHED["Scheduler<br/>(per-source cadence)"]
        EXTRACT["Extractors<br/>(one per source type)"]
        SCHEMA["Schema Registry<br/>(track drift)"]
        MONITOR["Monitoring<br/>(lag, errors, volume)"]
    end

    subgraph "Landing Zone"
        BRONZE["Bronze<br/>(object storage)<br/>Raw, immutable"]
    end

    S1 --> CONN --> EXTRACT
    S2 --> CONN
    S3 --> CONN
    S4 --> CONN
    S5 --> CONN
    SCHED --> EXTRACT
    EXTRACT --> BRONZE
    EXTRACT --> SCHEMA
    EXTRACT --> MONITOR
```

---

## Build vs Buy

The most consequential ingestion decision: custom code or a managed tool?

### The Landscape

| Tool | Type | Sources | Cost Model | Strength |
|---|---|---|---|---|
| **Airbyte** | Open source (self-hosted or cloud) | 300+ connectors | Free (self-hosted) or per-row (cloud) | Connector library, open source |
| **Fivetran** | Fully managed SaaS | 300+ connectors | Per Monthly Active Row (MAR) | Zero maintenance, reliability |
| **Meltano** | Open source (Singer-based) | 200+ taps | Free | Extensible, GitOps-friendly |
| **Stitch** | Managed SaaS (by Talend) | 100+ connectors | Per row | Simple, acquired by Talend |
| **Custom code** | Your own Python/Spark | Whatever you build | Engineering time | Full control, exact fit |
| **Cloud-native CDC** | Datastream / DMS / Data Factory | Databases only | Per GB replicated | Managed, tight cloud integration |

### Decision Matrix

```mermaid
graph TD
    A["How many sources?"] --> B{"< 5 sources?"}
    B -->|"Yes"| C{"Standard sources?<br/>(Postgres, Salesforce, etc.)"}
    C -->|"Yes"| D["Fivetran or Airbyte Cloud<br/>Don't write custom code"]
    C -->|"No — internal/custom APIs"| E["Custom code<br/>(no connector exists)"]
    B -->|"No — 5+ sources"| F{"Budget for SaaS?"}
    F -->|"Yes"| G["Fivetran<br/>(managed, reliable, expensive)"]
    F -->|"No"| H["Airbyte self-hosted<br/>(open source, you run it)"]
    
    I["Database CDC specifically?"] --> J["Cloud-native<br/>(Datastream / DMS / Data Factory)"]
```

### When to Build Custom

| Build Custom When | Use Managed When |
|---|---|
| Source is an internal API with custom auth | Source is a standard SaaS (Salesforce, Stripe) |
| Data volume > 100M rows/day (cost of managed at scale) | Data volume < 10M rows/day |
| You need sub-second latency | Minutes-to-hours latency is acceptable |
| You need custom transformation at ingestion time | Raw extraction + downstream transform is fine |
| You have a platform team to maintain it | You don't want to maintain infrastructure |

### The Hybrid Approach (Most Common at Scale)

```mermaid
graph TD
    subgraph "Managed (Fivetran/Airbyte)"
        M1["SaaS sources<br/>(Salesforce, Stripe, HubSpot)"]
        M2["Standard DBs<br/>(Postgres, MySQL)"]
    end

    subgraph "Custom Code"
        C1["Internal APIs<br/>(custom auth, custom schema)"]
        C2["Legacy systems<br/>(mainframe, FTP)"]
    end

    subgraph "Cloud-Native CDC"
        N1["High-volume OLTP<br/>(production database)"]
    end

    M1 --> BRONZE["Bronze"]
    M2 --> BRONZE
    C1 --> BRONZE
    C2 --> BRONZE
    N1 --> BRONZE
```

Use managed tools for standard sources (80% of connectors). Write custom code for internal APIs and edge cases (15%). Use cloud-native CDC for high-volume database replication (5%). All three land in the same Bronze layer.

---

## Multi-Source Architecture by Cloud

### GCP

```mermaid
graph TD
    subgraph "Sources"
        S1["Cloud SQL"]
        S2["REST APIs"]
        S3["Pub/Sub"]
        S4["GCS Files"]
    end

    subgraph "Ingestion"
        I1["Datastream<br/>(CDC)"]
        I2["Cloud Functions<br/>(API extraction)"]
        I3["Dataflow<br/>(stream consumer)"]
        I4["Cloud Functions<br/>(file trigger)"]
    end

    subgraph "Bronze"
        B["GCS Bucket<br/>(raw/source_name/date/)"]
    end

    S1 --> I1 --> B
    S2 --> I2 --> B
    S3 --> I3 --> B
    S4 --> I4 --> B
```

### AWS

```mermaid
graph TD
    subgraph "Sources"
        S1["RDS"]
        S2["REST APIs"]
        S3["Kinesis"]
        S4["S3 Files"]
    end

    subgraph "Ingestion"
        I1["DMS<br/>(CDC)"]
        I2["Lambda + EventBridge<br/>(API extraction)"]
        I3["Lambda / Glue<br/>(stream consumer)"]
        I4["Lambda<br/>(S3 trigger)"]
    end

    subgraph "Bronze"
        B["S3 Bucket<br/>(raw/source_name/date/)"]
    end

    S1 --> I1 --> B
    S2 --> I2 --> B
    S3 --> I3 --> B
    S4 --> I4 --> B
```

### Azure

```mermaid
graph TD
    subgraph "Sources"
        S1["Azure SQL"]
        S2["REST APIs"]
        S3["Event Hubs"]
        S4["ADLS Files"]
    end

    subgraph "Ingestion"
        I1["Data Factory<br/>(CDC)"]
        I2["Azure Functions + Logic Apps<br/>(API extraction)"]
        I3["Stream Analytics<br/>(stream consumer)"]
        I4["Azure Functions<br/>(blob trigger)"]
    end

    subgraph "Bronze"
        B["ADLS Gen2<br/>(raw/source_name/date/)"]
    end

    S1 --> I1 --> B
    S2 --> I2 --> B
    S3 --> I3 --> B
    S4 --> I4 --> B
```

---

## Bronze Folder Structure

Regardless of cloud or source type, all data lands in a consistent Bronze structure:

```
bronze/
├── calls/                    # Source: PostgreSQL CDC
│   ├── date=2026-04-13/
│   ├── date=2026-04-14/
│   └── date=2026-04-15/
├── crm_contacts/             # Source: Salesforce API
│   ├── extract_2026-04-13.jsonl
│   └── extract_2026-04-14.jsonl
├── click_events/             # Source: Kafka stream
│   ├── hour=2026-04-15-14/
│   └── hour=2026-04-15-15/
├── vendor_reports/           # Source: SFTP file drop
│   ├── report_2026-04-13.csv
│   └── report_2026-04-14.csv
└── mongo_orders/             # Source: MongoDB Change Stream
    ├── date=2026-04-13/
    └── date=2026-04-14/
```

**Rules:**
- One folder per source
- Partitioned by date (or hour for high-volume)
- Raw format preserved (JSON stays JSON, CSV stays CSV)
- Never modify Bronze data — it's the audit trail

---

## Quick Links

| Chapter | Topic |
|---|---|
| [06 - Production Patterns](06_Production_Patterns.md) | Retries, idempotency, credential rotation |
| [07 - System Design](07_System_Design.md) | This page |
| [08 - Quality Security Governance](08_Quality_Security_Governance.md) | Source credentials, PII at ingestion |
| [09 - Observability Troubleshooting](09_Observability_Troubleshooting.md) | Ingestion monitoring |
