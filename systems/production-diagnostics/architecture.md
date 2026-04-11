# Production Diagnostic Intelligence System

## Architecture

How to diagnose, correct, and improve production systems by collecting data from databases, logs, documents, and APIs.

---

## System Overview

```mermaid
graph TD
    subgraph Data Sources
        LOGS["Application Logs<br/>(JSON, 28K+ entries)"]
        DB["Databases<br/>(incidents, metrics, services)"]
        DEPLOY["Deployment Records<br/>(versions, rollbacks)"]
        DOCS["Runbooks & Docs<br/>(markdown, Confluence)"]
        API["APIs & Alerts<br/>(monitoring, ticketing)"]
        CODE["Code Repositories<br/>(source code, configs)"]
        GIT["Git History<br/>(commits, PRs, blame)"]
    end

    subgraph Data Pipeline
        BRONZE["Bronze Layer<br/>Raw data, untouched"]
        SILVER["Silver Layer<br/>Cleaned, deduped, timezone-fixed"]
        GOLD["Gold Layer<br/>Star schema, analysis-ready"]
    end

    subgraph Intelligence Layer
        ML["ML Pipeline<br/>Predict escalation<br/>Classify root cause"]
        DL["Deep Learning<br/>Anomaly detection<br/>on time-series metrics"]
        RAG["RAG<br/>Search runbooks by meaning<br/>Retrieve past fixes"]
    end

    subgraph Action Layer
        AGENT["AI Agent<br/>Orchestrates all components<br/>Creates ticket with findings"]
    end

    subgraph Consumer Layer
        DASH["Dashboards & Reports<br/>(Looker Studio, Tableau)"]
        AIANA["AI Analytics<br/>(Natural language queries,<br/>chat with your data)"]
        TICKET["Ticket Created<br/>What happened<br/>What's related<br/>Likely root cause<br/>Recommended action"]
    end

    LOGS --> BRONZE
    DB --> BRONZE
    DEPLOY --> BRONZE
    DOCS --> RAG
    API --> BRONZE
    CODE --> RAG
    GIT --> BRONZE

    BRONZE --> SILVER
    SILVER --> GOLD

    GOLD --> ML
    GOLD --> DL
    GOLD --> AGENT
    GOLD --> DASH
    GOLD --> AIANA

    ML --> AGENT
    DL --> AGENT
    RAG --> AGENT
    RAG --> AIANA

    AGENT --> TICKET
```

---

## Components

### 1. Data Pipeline (Bronze to Silver to Gold)

Ingests data from multiple sources in different formats. Cleans, deduplicates, fixes quality issues, and structures for analysis.

| Layer | What Happens | Example |
|---|---|---|
| Bronze | Raw ingestion, no transformation | `app_logs.jsonl` as-is |
| Silver | Cleaned, deduped, timezone-fixed, typed | Missing request IDs flagged, timestamps normalized |
| Gold | Star schema, joined, analysis-ready | Incidents joined with deployments, services, metrics |

**Quality issues in the data (intentional):**
- 3% of logs missing request_id (broken distributed tracing)
- Some timestamps without timezone indicator
- Incident severity misclassification (P3 that should be P2)

---

### 2. Star Schema (Data Model)

Structures incidents, deployments, metrics, and services into fact and dimension tables for queryable analysis.

```mermaid
erDiagram
    fact_incidents {
        string incident_id PK
        timestamp created_at
        timestamp resolved_at
        int time_to_resolve_minutes
        string severity
        string status
        string root_cause
    }
    dim_services {
        string service_name PK
        string team
        string language
        string database
        string deployment_target
        int sla_p99_ms
    }
    dim_deployments {
        string deploy_id PK
        timestamp deploy_time
        string version
        string deployer
        string status
        boolean rollback
    }
    fact_metrics {
        timestamp metric_time
        float cpu_percent
        float memory_percent
        float disk_percent
        int active_connections
        float error_rate_percent
        int p99_latency_ms
    }

    fact_incidents ||--o{ dim_services : "service"
    fact_metrics ||--o{ dim_services : "service"
    dim_deployments ||--o{ dim_services : "service"
```

---

### 3. ML Pipeline (Prediction and Classification)

Trains models to predict incident escalation and classify root causes.

**Approach:**
- Baseline model (simple, fast)
- Regularized model (Ridge)
- Advanced model (GradientBoosting)
- Compare side-by-side via MLflow
- Explain with SHAP (which features drive predictions)

**Use cases in this system:**
- Predict which P3 incident will escalate to P1
- Classify root cause from incident description + metrics
- Identify services at risk based on metric trends

---

### 4. Deep Learning (Anomaly Detection)

Neural networks trained on historical infrastructure metrics to detect patterns that dashboards and static thresholds miss.

**Example from the dataset:**
- Search-service memory usage climbs steadily over 5 days (days 10-14)
- Static threshold (90%) only triggers on day 14 when it crashes
- A trained model detects the upward drift on day 11, three days before the crash

**Patterns detected:**
- Slow-burn resource exhaustion (memory leaks, disk fill)
- Periodic anomalies (batch job conflicts at 2-3 AM)
- Correlated failures across services

---

### 5. RAG (Knowledge Retrieval)

Retrieval-Augmented Generation searches runbooks, post-mortems, and documentation by meaning, not keywords.

**Flow:**
1. Ingest documents (runbooks, post-mortems, architecture docs)
2. Create embeddings (semantic vector representation)
3. On query, retrieve most relevant chunks
4. Generate answer using LLM with retrieved context

**Example:**
- Query: "What's the fix for order-service connection pool exhaustion?"
- RAG finds: order-service runbook, section on DB connection pool, past incident INC-1005
- Returns: "Set connection_max_lifetime=300s in database config. This was identified March 5. Temporary fix: restart pods."

---

### 6. AI Agent (Orchestrator)

The agent ties all components together. It receives an alert and executes a diagnostic workflow:

```mermaid
sequenceDiagram
    participant Alert as Alert System
    participant Agent as AI Agent
    participant DB as Database
    participant Logs as Log Store
    participant Metrics as Metrics Store
    participant RAG as RAG Knowledge Base
    participant Ticket as Ticket System

    Alert->>Agent: order-service error rate spike
    Agent->>DB: Any deployments in last 2 hours?
    DB-->>Agent: DEP-777 (v2.4.1, 2 hours ago)
    Agent->>Logs: Error patterns in order-service?
    Logs-->>Agent: "Failed to acquire DB connection" (150x)
    Agent->>Metrics: Connection pool and CPU trends?
    Metrics-->>Agent: Connections: 248/250, CPU: 85%
    Agent->>RAG: order-service connection pool fix?
    RAG-->>Agent: Set connection_max_lifetime=300s
    Agent->>DB: Has this happened before?
    DB-->>Agent: INC-1005 (March 5, same root cause)
    Agent->>Ticket: Create P1 ticket
    Note over Ticket: Title: DB connection pool exhaustion (recurring)<br/>Root cause: connection_max_lifetime not set after v2.4.1<br/>Related: INC-1005, DEP-777<br/>Action: Set connection_max_lifetime=300s, restart pods
```

---

## Hidden Patterns in the Dataset

The production support dataset contains 10 intentional patterns for diagnostic practice:

| # | Pattern | Diagnostic Skill |
|---|---|---|
| 1 | Bad deployment March 8 causes order-service errors | Correlate deployments with error spikes |
| 2 | Search-service memory leak builds over 5 days, crashes day 15 | Detect slow-burn failures |
| 3 | Same DB pool root cause: P3 on day 5, then P1 on day 20 | Recognize recurring root causes |
| 4 | Payment-service errors spike daily at 2-3 AM | Identify time-based patterns |
| 5 | Auth-service runbook references old Redis config | Detect outdated documentation |
| 6 | Notification-service disk fills over days 18-21 | Resource exhaustion trends |
| 7 | 3% of logs missing request_id | Broken distributed tracing |
| 8 | Some timestamps missing timezone | Data quality issues |
| 9 | Day-4 incident misclassified as P3 | Severity triage failures |
| 10 | Day-10 memory leak classified P4, ignored until crash | Ignored warnings becoming outages |

---

## Dataset

All data is in `data/production-support/`:

| File | Records | Description |
|---|---|---|
| `services.csv` | 7 | Service registry (name, team, tech stack, SLA) |
| `incidents.csv` | 15 | Incidents over 30 days |
| `deployments.csv` | 36 | Deployment history with rollbacks |
| `infra_metrics.csv` | 60,480 | CPU, memory, disk, latency (5-min intervals) |
| `app_logs.jsonl` | 28,189 | Application logs (JSON lines) |
| `runbooks/` | 4 | Service runbooks (markdown) |
