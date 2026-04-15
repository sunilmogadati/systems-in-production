# Ingestion Patterns - How It Works

**Under the hood: JDBC connection pooling, CDC replication slots, OAuth token lifecycle, stream consumer checkpointing, and MongoDB change streams.**

---

## Database Ingestion Internals

### JDBC / Database Drivers

Every database ingestion uses a driver — a library that speaks the database's wire protocol.

| Database | Python Driver | Java/JDBC Driver | Connection String |
|---|---|---|---|
| PostgreSQL | `psycopg2` / `asyncpg` | `org.postgresql.Driver` | `postgresql://user:pass@host:5432/db` |
| MySQL | `mysql-connector-python` | `com.mysql.cj.jdbc.Driver` | `mysql://user:pass@host:3306/db` |
| SQL Server | `pyodbc` / `pymssql` | `com.microsoft.sqlserver.jdbc.SQLServerDriver` | `mssql+pyodbc://user:pass@host/db` |
| Oracle | `cx_Oracle` / `oracledb` | `oracle.jdbc.OracleDriver` | `oracle://user:pass@host:1521/service` |

### Connection Pooling

Opening a database connection takes 50-200 milliseconds (TCP handshake + TLS + authentication). If your pipeline opens a connection per query, that overhead dominates.

```mermaid
graph TD
    subgraph "Without Pool"
        A1["Query 1"] --> C1["Open conn (150ms)"] --> Q1["Execute (10ms)"] --> CL1["Close"]
        A2["Query 2"] --> C2["Open conn (150ms)"] --> Q2["Execute (10ms)"] --> CL2["Close"]
        A3["Query 3"] --> C3["Open conn (150ms)"] --> Q3["Execute (10ms)"] --> CL3["Close"]
    end

    subgraph "With Pool"
        B1["Query 1"] --> P["Pool (pre-opened connections)"]
        B2["Query 2"] --> P
        B3["Query 3"] --> P
        P --> QQ["Execute (10ms each)"]
    end
```

**Connection pool sizing:** A production OLTP database typically limits connections to 100-500. Your pipeline should use 2-5 connections, not 50. Leave room for the application.

```python
# Connection pooling with psycopg2
from psycopg2 import pool

# Create pool once at startup (2-5 connections)
connection_pool = pool.ThreadedConnectionPool(
    minconn=2,
    maxconn=5,
    host="db-host",
    dbname="callcenter",
    user="pipeline_readonly",
    password="from-secret-manager",
)

# Get connection from pool (reuses existing)
conn = connection_pool.getconn()
try:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM calls WHERE updated_at > %s", (watermark,))
    rows = cursor.fetchall()
finally:
    connection_pool.putconn(conn)  # Return to pool, not close
```

### CDC Replication Slots (PostgreSQL)

Log-based CDC on PostgreSQL uses **replication slots**. A replication slot is a server-side bookmark that tracks which Write-Ahead Log (WAL) entries have been consumed by the CDC tool.

```mermaid
sequenceDiagram
    participant App as Application
    participant PG as PostgreSQL
    participant WAL as WAL (Write-Ahead Log)
    participant Slot as Replication Slot
    participant CDC as CDC Tool (Debezium)

    App->>PG: INSERT INTO calls VALUES (...)
    PG->>WAL: Write WAL entry
    CDC->>Slot: "Give me changes since last checkpoint"
    Slot->>WAL: Read entries from checkpoint
    WAL->>CDC: Change events
    CDC->>CDC: Emit to Kafka/Pub/Sub
    CDC->>Slot: "I've consumed up to LSN 0/1A3B4C"
    Slot->>Slot: Update checkpoint
```

**The replication slot risk:** If the CDC consumer stops (crashes, deployment, network issue), the replication slot keeps accumulating WAL entries. The WAL grows unbounded. Disk fills up. The primary database crashes.

**Mitigation:** Monitor replication slot lag. Alert if lag exceeds 1 hour. Set `max_slot_wal_keep_size` in PostgreSQL to cap WAL retention.

---

## API Ingestion Internals

### OAuth Token Lifecycle

Most production APIs use OAuth 2.0. The token has an expiry (typically 1 hour). Long-running extractions must refresh the token mid-extraction.

```mermaid
sequenceDiagram
    participant Pipeline
    participant AuthServer as Auth Server
    participant API

    Pipeline->>AuthServer: POST /oauth/token (client_id + secret)
    AuthServer->>Pipeline: access_token (expires in 3600s)
    
    Pipeline->>API: GET /calls?cursor=X (Bearer token)
    API->>Pipeline: 200 OK + data
    
    Note over Pipeline: ... 55 minutes later ...
    
    Pipeline->>API: GET /calls?cursor=Y (same token)
    API->>Pipeline: 401 Unauthorized (token expired)
    
    Pipeline->>AuthServer: POST /oauth/token (refresh)
    AuthServer->>Pipeline: new access_token
    
    Pipeline->>API: GET /calls?cursor=Y (new token)
    API->>Pipeline: 200 OK + data
```

**Production pattern:** Wrap token management in a class that auto-refreshes before expiry:

```python
class OAuthSession:
    """Auto-refreshing OAuth session."""
    
    def __init__(self, token_url, client_id, client_secret):
        self.token_url = token_url
        self.client_id = client_id
        self.client_secret = client_secret
        self.token = None
        self.expires_at = 0
    
    def get_token(self):
        """Return valid token, refreshing if needed."""
        if time.time() > self.expires_at - 300:  # Refresh 5 min before expiry
            response = requests.post(self.token_url, data={
                "grant_type": "client_credentials",
                "client_id": self.client_id,
                "client_secret": self.client_secret,
            })
            response.raise_for_status()
            data = response.json()
            self.token = data["access_token"]
            self.expires_at = time.time() + data["expires_in"]
        
        return self.token
```

### Webhook Ingestion (Push-Based APIs)

Some APIs push data to you via webhooks instead of you pulling it.

```mermaid
graph LR
    SRC["Source System"] -->|"POST /webhook"| EP["Your Endpoint<br/>(Cloud Function / Lambda)"]
    EP --> Q["Message Queue<br/>(Pub/Sub / SQS)"]
    Q --> PIPE["Pipeline Consumer"]
    PIPE --> BRONZE["Bronze"]
```

**Webhook challenges:**
- **Verification:** Validate the webhook signature (HMAC) to prevent spoofing
- **Idempotency:** The source may retry delivery. Your endpoint must handle duplicates.
- **Ordering:** Webhooks arrive out of order. Don't assume sequential.
- **Availability:** If your endpoint is down, events are lost (unless the source retries with backoff)

---

## Stream Consumer Internals

### Consumer Checkpointing

A stream consumer reads events and periodically saves its position (checkpoint/offset). On restart, it resumes from the checkpoint.

```mermaid
graph TD
    subgraph "Stream (Kafka Topic)"
        E1["Event 1"] --> E2["Event 2"] --> E3["Event 3"] --> E4["Event 4"] --> E5["Event 5"]
    end

    CP["Checkpoint: Event 3<br/>(last committed)"] -.-> E3

    subgraph "On Restart"
        R["Consumer resumes<br/>from Event 4"]
    end
```

**The checkpoint-before-process vs process-before-checkpoint tradeoff:**

| Strategy | Risk | Result |
|---|---|---|
| Checkpoint THEN process | If processing fails, event is skipped (data loss) | At-most-once delivery |
| Process THEN checkpoint | If crash after process but before checkpoint, event is reprocessed (duplicate) | At-least-once delivery |

**Production default:** Process-then-checkpoint (at-least-once) with idempotent writes downstream. Duplicates are handled by MERGE in the warehouse. Data loss is not recoverable.

---

## MongoDB Change Streams

MongoDB's native CDC mechanism. Watches a collection (or database) for changes and emits events.

```python
from pymongo import MongoClient

client = MongoClient("mongodb://host:27017")
db = client["callcenter"]
collection = db["calls"]

# Open a change stream on the calls collection
# resume_after: resume from last checkpoint (survives restarts)
with collection.watch(resume_after=last_resume_token) as stream:
    for change in stream:
        operation = change["operationType"]  # insert, update, delete, replace
        document = change.get("fullDocument")
        
        # Write to Bronze
        write_to_bronze(change)
        
        # Save checkpoint (the resume token)
        save_checkpoint(change["_id"])
```

**Key difference from relational CDC:** MongoDB change streams emit the full document on update (not just the changed fields), unless you configure `updateLookup`. This means larger payloads but simpler consumer logic.

---

## Quick Links

| Chapter | Topic |
|---|---|
| [03 - Hello World](03_Hello_World.md) | API + database ingestion |
| [04 - How It Works](04_How_It_Works.md) | This page |
| [05 - Building It](05_Building_It.md) | Production ingestion for all five source types |
| [06 - Production Patterns](06_Production_Patterns.md) | Retries, idempotency, schema discovery |
