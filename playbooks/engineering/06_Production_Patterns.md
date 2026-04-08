# Production Software Patterns

**The difference between a working prototype and a production system is what happens when things go wrong. These ten patterns are how production systems survive network failures, overloaded databases, cascading outages, and traffic spikes. Every pattern here exists because someone learned the hard way what happens without it.**

---

## Retry with Exponential Backoff

The single most important production pattern. When a network call fails, you try again -- but you wait longer between each attempt so you do not hammer a service that is already struggling.

**Analogy: Knocking on a Door.**
If nobody answers, you knock again after a few seconds. You do not stand there hammering continuously -- that annoys everyone and guarantees you will not get in. You wait 2 seconds, then 4, then 8. If they still do not answer after a few tries, you leave a note (dead letter queue) and walk away.

```python
import time
import random
import httpx

def call_with_retry(url: str, max_retries: int = 3) -> httpx.Response:
    """Call an external service with exponential backoff and jitter."""
    for attempt in range(max_retries):
        try:
            response = httpx.get(url, timeout=5.0)
            response.raise_for_status()
            return response
        except (httpx.HTTPStatusError, httpx.ConnectError) as e:
            if attempt == max_retries - 1:
                raise  # Final attempt -- propagate the error
            # Exponential backoff: 1s, 2s, 4s + random jitter
            wait = (2 ** attempt) + random.uniform(0, 1)
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {wait:.1f}s")
            time.sleep(wait)
```

**Key rules:**
- Add jitter (randomness) so all clients do not retry at the exact same moment
- Set a maximum number of retries -- infinite retries are a denial-of-service attack on yourself
- Only retry on transient errors (timeouts, 503s). Do not retry on 400 Bad Request -- it will fail every time
- Use a dead letter queue for requests that exhaust all retries

```mermaid
flowchart TD
    A[Request] --> B{Success?}
    B -->|Yes| C[Return Response]
    B -->|No| D{Retries Left?}
    D -->|Yes| E[Wait: 2^attempt + jitter]
    E --> A
    D -->|No| F[Send to Dead Letter Queue]
    F --> G[Alert / Manual Review]
```

---

## Circuit Breaker

Stop calling a service that is failing. If a downstream dependency is down, continuing to send requests wastes your resources, increases latency for your users, and prevents the failing service from recovering.

**Analogy: Electrical Circuit Breaker.**
When your house draws too much current, the breaker trips and cuts power -- preventing a fire. You fix the problem, then flip the breaker back on. Software circuit breakers work the same way.

**Three states:**

| State | What Happens | Transitions To |
|:---|:---|:---|
| **Closed** (normal) | Requests pass through. Failures are counted. | Half-Open (when failure count hits threshold) |
| **Open** (tripped) | All requests are immediately rejected. No calls to the downstream service. | Half-Open (after a timeout period) |
| **Half-Open** (testing) | One test request is allowed through. | Closed (if test succeeds) or Open (if test fails) |

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open : Failure threshold exceeded
    Open --> HalfOpen : Timeout expires
    HalfOpen --> Closed : Test request succeeds
    HalfOpen --> Open : Test request fails
```

```python
import time
from dataclasses import dataclass, field

@dataclass
class CircuitBreaker:
    failure_threshold: int = 5
    reset_timeout: float = 30.0  # seconds
    _failure_count: int = field(default=0, init=False)
    _state: str = field(default="closed", init=False)
    _last_failure_time: float = field(default=0.0, init=False)

    def call(self, func, *args, **kwargs):
        if self._state == "open":
            if time.time() - self._last_failure_time > self.reset_timeout:
                self._state = "half-open"
            else:
                raise RuntimeError("Circuit is open -- request rejected")

        try:
            result = func(*args, **kwargs)
            if self._state == "half-open":
                self._state = "closed"
                self._failure_count = 0
            return result
        except Exception as e:
            self._failure_count += 1
            self._last_failure_time = time.time()
            if self._failure_count >= self.failure_threshold:
                self._state = "open"
            raise
```

**Common mistake:** Setting the reset timeout too short. If the downstream service needs 2 minutes to recover, a 5-second timeout means you keep hitting it during recovery.

---

## Bulkhead

Isolate failures so one component does not bring down everything. If your payment service is slow, it should not consume all your threads and make your health check unresponsive.

**Analogy: Compartments on a Ship.**
A ship has watertight compartments (bulkheads). If one compartment floods, the others stay dry and the ship stays afloat. Without bulkheads, one hole sinks the whole ship.

**Implementation approaches:**
- **Thread pool isolation:** Each downstream service gets its own thread pool. If the payment service pool is exhausted, the order service pool still has threads available.
- **Connection limits:** Cap the number of concurrent connections to any single dependency.
- **Separate process/container:** Run critical services in isolated containers so a memory leak in one does not starve another.

```python
import asyncio
from asyncio import Semaphore

# Each service gets its own semaphore (connection limit)
payment_semaphore = Semaphore(10)   # Max 10 concurrent payment calls
search_semaphore = Semaphore(50)    # Max 50 concurrent search calls
notification_semaphore = Semaphore(20)

async def call_payment_service(request):
    async with payment_semaphore:
        # If all 10 slots are taken, this blocks instead of
        # consuming shared resources
        return await make_payment_request(request)
```

---

## Message Queues

Decouple producers and consumers. Instead of Service A calling Service B directly (and waiting), Service A puts a message on a queue. Service B processes it when ready. If Service B is slow or down, messages wait in the queue instead of causing timeouts.

| Queue | Best For | Key Feature | Managed Option |
|:---|:---|:---|:---|
| **Apache Kafka** | High-throughput event streaming, log aggregation | Durable, ordered, replayable | Confluent Cloud, AWS MSK |
| **RabbitMQ** | Task distribution, work queues | Flexible routing, acknowledgments | CloudAMQP, Amazon MQ |
| **AWS SQS** | Simple cloud-native queuing | Zero ops, auto-scaling | (Is the managed option) |
| **Google Pub/Sub** | GCP-native event distribution | Global, auto-scaling | (Is the managed option) |

**When to use a message queue:**
- The producer does not need the result immediately (async processing)
- You need to handle traffic spikes (the queue absorbs the burst)
- The consumer is slower than the producer
- You need guaranteed delivery (at-least-once processing)

**When NOT to use a message queue:**
- The caller needs an immediate response (use sync HTTP instead)
- The data is so time-sensitive that even seconds of delay is unacceptable
- You have a simple two-service system and adding a queue doubles your infrastructure

---

## Event-Driven Architecture

Instead of services calling each other directly, they emit events. Other services subscribe to events they care about. This removes tight coupling: the order service does not need to know about the notification service, the analytics service, or the inventory service.

```mermaid
flowchart LR
    subgraph Producers
        A[Order Service]
        B[User Service]
    end

    subgraph Event Bus
        C[Message Broker<br/>Kafka / Pub/Sub / SNS]
    end

    subgraph Consumers
        D[Notification Service]
        E[Analytics Service]
        F[Inventory Service]
        G[ML Feature Pipeline]
    end

    A -->|order.created| C
    A -->|order.completed| C
    B -->|user.signed_up| C
    C --> D
    C --> E
    C --> F
    C --> G
```

**Key principles:**
- Events describe what happened ("order.created"), not what to do ("send_email"). The producer does not care what consumers do with the event.
- Events are immutable. Once emitted, they are facts. You can add new events but never change the schema of existing ones without a versioning strategy.
- Every consumer must be idempotent. Messages can be delivered more than once. Processing the same event twice must produce the same result.

**Common mistake:** Event spaghetti. When every service emits events and every other service subscribes to everything, you end up with invisible dependencies that are harder to trace than direct calls.

---

## Caching

Store frequently accessed data closer to the caller. Every database query, API call, or computation you can avoid is latency saved and load reduced.

```mermaid
flowchart TD
    A[Client Request] --> B{CDN Cache?}
    B -->|Hit| C[Return Cached Response]
    B -->|Miss| D{Application Cache?<br/>Redis / In-Memory}
    D -->|Hit| E[Return Cached Data]
    D -->|Miss| F[Query Database]
    F --> G[Store in Cache]
    G --> E
```

| Cache Layer | Latency | Best For | Example |
|:---|:---|:---|:---|
| **CDN** (CloudFront, Cloudflare) | ~5 ms | Static assets, API responses by geography | Product images, public API data |
| **Application cache** (Redis) | ~1 ms | Session data, computed results, rate limit counters | User profiles, ML prediction results |
| **In-memory** (Python dict, `functools.lru_cache`) | ~0.01 ms | Hot data within a single process | Configuration, lookup tables |
| **Database cache** (query cache, materialized views) | ~10 ms | Complex query results that rarely change | Aggregation dashboards |

**Cache invalidation -- the hard problem:**
- **TTL (Time-to-Live):** Data expires after N seconds. Simple but stale data is possible.
- **Write-through:** Update the cache every time you update the database. Consistent but slower writes.
- **Event-driven invalidation:** When data changes, emit an event that clears the cache. Best consistency but requires infrastructure.

**What to cache:** Data that is read far more often than it is written. User profiles (read 100x per write), product catalog, ML model predictions for common inputs.

**What NOT to cache:** Data that must be real-time (account balances, inventory counts in a flash sale), data that changes every request, sensitive data without proper encryption.

---

## Connection Pooling

Opening a new database connection or HTTP connection for every request is expensive (TCP handshake, TLS negotiation, authentication). A connection pool maintains a set of open connections that are reused across requests.

```python
# Database connection pool with SQLAlchemy
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://user:pass@localhost/mydb",
    pool_size=20,          # Keep 20 connections open
    max_overflow=10,       # Allow 10 more under load (30 total max)
    pool_timeout=30,       # Wait 30s for a connection before error
    pool_recycle=1800,     # Recycle connections after 30 minutes
)

# HTTP connection pool with httpx
import httpx

# Reuse this client across requests -- do not create a new one per call
client = httpx.Client(
    base_url="https://api.example.com",
    limits=httpx.Limits(
        max_connections=100,
        max_keepalive_connections=20,
    ),
    timeout=10.0,
)
```

**Common mistake:** Creating a new database connection inside every request handler. This works in development (5 requests/second) and falls apart in production (500 requests/second) when you hit the database connection limit.

---

## Rate Limiting

Protect your service from overload -- whether from a misbehaving client, a DDoS attack, or an honest traffic spike. Without rate limiting, one bad actor can consume all your resources.

**Common strategies:**

| Strategy | How It Works | Best For |
|:---|:---|:---|
| **Token bucket** | Tokens refill at a fixed rate. Each request costs one token. | API rate limits (100 req/min) |
| **Sliding window** | Count requests in a rolling time window. | Smooth traffic shaping |
| **Fixed window** | Count requests per time interval (per minute, per hour). | Simple implementation |
| **Concurrency limit** | Limit simultaneous in-flight requests. | Protecting expensive operations (ML inference) |

```python
# Simple rate limiter using Redis
import redis
import time

r = redis.Redis()

def is_rate_limited(client_id: str, max_requests: int = 100, window: int = 60) -> bool:
    """Allow max_requests per window seconds per client."""
    key = f"rate_limit:{client_id}"
    current = r.get(key)
    if current and int(current) >= max_requests:
        return True
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    pipe.execute()
    return False
```

**Return proper status codes:** `429 Too Many Requests` with a `Retry-After` header. Do not return 500 -- the client needs to know they are being throttled, not that your service is broken.

---

## Background Jobs

Not every task needs to complete before the HTTP response is sent. Report generation, email sending, ML model training, image processing -- these can run in the background.

| Tool | Best For | How It Works |
|:---|:---|:---|
| **Celery** (Python) | Task queues for Python services | Workers pull tasks from Redis/RabbitMQ |
| **Cloud Tasks** (GCP) | Serverless task execution | HTTP-triggered, auto-retries, rate limiting |
| **SQS + Lambda** (AWS) | Serverless queue processing | Lambda polls SQS, processes messages |
| **Cloud Scheduler / EventBridge** | Cron-style scheduled jobs | Trigger functions on a schedule |

```python
# Celery task definition
from celery import Celery

app = Celery("tasks", broker="redis://localhost:6379/0")

@app.task(bind=True, max_retries=3, default_retry_delay=60)
def generate_diagnostic_report(self, client_id: str):
    """Generate a report in the background. Retries on failure."""
    try:
        data = fetch_client_data(client_id)
        report = analyze_delivery_health(data)
        store_report(client_id, report)
        send_notification(client_id, "Report ready")
    except Exception as exc:
        self.retry(exc=exc)

# Trigger from your API handler
@app.post("/reports")
async def request_report(client_id: str):
    generate_diagnostic_report.delay(client_id)
    return {"status": "processing", "message": "Report will be ready in ~2 minutes"}
```

---

## Health Checks and Graceful Shutdown

Production services must tell the orchestrator (Kubernetes, ECS, Cloud Run) whether they are healthy and ready to receive traffic. Without health checks, traffic gets routed to broken instances.

**Two types of health checks:**

| Check | Purpose | Example Response |
|:---|:---|:---|
| **Liveness** (`/health/live`) | Is the process running? | `{"status": "alive"}` -- if this fails, restart the container |
| **Readiness** (`/health/ready`) | Can it serve requests? | `{"status": "ready", "db": "connected", "cache": "connected"}` -- if this fails, stop sending traffic but do not restart |

```python
from fastapi import FastAPI
import asyncpg

app = FastAPI()
db_pool = None

@app.get("/health/live")
async def liveness():
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness():
    try:
        async with db_pool.acquire() as conn:
            await conn.fetchval("SELECT 1")
        return {"status": "ready", "db": "connected"}
    except Exception:
        return JSONResponse(status_code=503, content={"status": "not ready"})
```

**Graceful shutdown:** When the orchestrator sends SIGTERM, your service should:
1. Stop accepting new requests
2. Finish processing in-flight requests (within a timeout)
3. Close database connections and flush logs
4. Exit cleanly

**Common mistake:** No graceful shutdown. The orchestrator kills the container mid-request, leaving database transactions half-committed and users seeing 502 errors.

---

## Pattern Reference Table

| Pattern | When to Use | What It Prevents | Common Mistake |
|:---|:---|:---|:---|
| **Retry + backoff** | Any network call to an external service | Transient failures causing permanent errors | Retrying without backoff (hammering a struggling service) |
| **Circuit breaker** | Calling a service that may go down | Cascading failures, resource exhaustion | Reset timeout too short (re-hitting during recovery) |
| **Bulkhead** | Multiple downstream dependencies | One slow service consuming all resources | No isolation -- one dependency brings down everything |
| **Message queue** | Producer does not need immediate response | Tight coupling, lost messages during outages | Adding queues to simple request/response flows |
| **Event-driven** | Multiple services react to the same event | Services needing to know about each other | Event spaghetti -- invisible dependencies |
| **Caching** | Read-heavy data, expensive computations | Unnecessary database/API load | Caching data that must be real-time |
| **Connection pooling** | Any database or HTTP client usage | Connection exhaustion under load | Creating new connections per request |
| **Rate limiting** | Public APIs, expensive operations | DDoS, resource exhaustion, runaway clients | Returning 500 instead of 429 |
| **Background jobs** | Long-running tasks (reports, emails, ML) | Slow API responses, request timeouts | No retry logic on background tasks |
| **Health checks** | Every production service, no exceptions | Traffic routed to broken instances | Liveness check that queries the database (restart loops) |

---

## Quick Links -- All Chapters

| Chapter | Title |
|:---|:---|
| [01](01_Why.md) | Why |
| [02](02_Concepts.md) | Concepts |
| [03](03_Hello_World.md) | Hello World |
| [04](04_How_It_Works.md) | How It Works |
| [05](05_Building_It.md) | Building It |
| [06](06_Production_Patterns.md) | **Production Software Patterns** |
| [07](07_System_Design.md) | System Design for AI/Data Services |
| [08](08_Quality_Security_Governance.md) | Quality, Security, and Governance |
| [09](09_Observability_Troubleshooting.md) | Observability and Troubleshooting |
| [10](10_Decision_Guide.md) | Software Engineering Decision Guide |
