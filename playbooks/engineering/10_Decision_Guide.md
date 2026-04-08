# Software Engineering Decision Guide

**Every architectural decision is a trade-off. This chapter gives you decision tables for the choices that come up in every production system, a production readiness checklist, and the anti-patterns that cause the most damage.**

---

## Monolith or Microservices?

| Factor | Choose Monolith | Choose Microservices |
|:---|:---|:---|
| Team size | Under 10 engineers | 10+ engineers across multiple teams |
| Codebase age | New project, exploring requirements | Mature system with well-defined boundaries |
| Deployment needs | Ship the whole thing together is fine | Teams need to deploy independently |
| Scaling needs | Uniform load across features | One component needs 10x the resources of others |
| Data model | Shared database is acceptable | Services need their own data stores |
| Operational maturity | No dedicated platform/DevOps team | Have CI/CD, container orchestration, observability in place |
| Latency tolerance | Cannot afford inter-service network hops | Network latency between services is acceptable |

**Default:** Start monolith. Extract services when you feel the pain -- not before.

---

## REST or gRPC or GraphQL?

| Factor | REST | gRPC | GraphQL |
|:---|:---|:---|:---|
| Best for | Public APIs, CRUD operations, broad compatibility | Internal service-to-service, high throughput, streaming | Frontend-driven APIs, mobile apps with bandwidth constraints |
| Serialization | JSON (human-readable, larger) | Protobuf (binary, compact, fast) | JSON |
| Schema/contract | OpenAPI (optional) | .proto files (required, strict) | SDL schema (required, strict) |
| Streaming | No (use WebSocket separately) | Yes (bidirectional) | Subscriptions (via WebSocket) |
| Browser support | Native | Requires grpc-web proxy | Native |
| Learning curve | Low | Medium (protobuf, code generation) | Medium (query language, resolvers) |
| Tooling | Everywhere | Growing but less universal | Good (Apollo, Relay) |

**Default:** REST for external APIs. gRPC for internal service-to-service calls in performance-critical paths. GraphQL when the frontend team is constantly asking for new endpoints because each screen needs a different shape of data.

---

## Docker (Containers) or Serverless (Lambda / Cloud Functions)?

| Factor | Containers (ECS / Cloud Run / K8s) | Serverless (Lambda / Cloud Functions) |
|:---|:---|:---|
| Startup time | Warm (always running or fast spin-up) | Cold starts (100ms to 10s) |
| Execution limit | Unlimited | 15 min (Lambda), 60 min (Cloud Run) |
| Cost at low traffic | Paying for idle instances | Near-zero (pay per invocation) |
| Cost at high traffic | Predictable | Can spike unpredictably |
| State management | Can hold state in memory | Stateless by design |
| ML inference | Good (GPU support, long warm-up OK) | Poor (cold starts kill latency) |
| Long-running jobs | Yes | No (use Step Functions / Workflows instead) |
| Local development | Docker Compose reproduces production | Local testing requires emulators (SAM, Functions Framework) |
| Operational overhead | Manage scaling, health checks, deploys | Managed by the cloud provider |

**Default:** Containers for any service that needs to stay warm, hold state, run ML models, or process long-running requests. Serverless for event handlers, webhooks, lightweight API endpoints, and scheduled tasks.

---

## PostgreSQL or MongoDB or Redis?

| Factor | PostgreSQL | MongoDB | Redis |
|:---|:---|:---|:---|
| Data model | Relational tables, strict schema | Documents (JSON-like), flexible schema | Key-value, sorted sets, streams |
| Transactions | Full ACID | Multi-document (since v4.0), limited | Single-key atomic, no multi-key transactions |
| Query language | SQL | MQL + aggregation pipeline | Commands (GET, SET, HSET, ZADD) |
| Joins | Native, optimized | $lookup (limited, expensive) | Not supported |
| Schema changes | Migrations required (Alembic) | Schema-less, evolve freely | N/A |
| Scaling | Vertical + read replicas | Horizontal (sharding) | Cluster (hash slots) |
| Durability | Disk-first, durable by default | Configurable (WiredTiger) | In-memory, optional persistence (RDB/AOF) |
| Best for | Primary data store, transactional systems, analytics | Catalogs, content management, event stores | Caching, sessions, rate limiting, leaderboards, queues |
| Worst for | Massive write throughput (without sharding) | Complex joins, strict consistency | Large datasets (limited by RAM), primary data store |

**Default:** PostgreSQL as your primary database. Add Redis for caching and ephemeral state. Add MongoDB only if your data is genuinely document-shaped and you do not need joins.

---

## Sync or Async Communication?

| Factor | Synchronous (REST / gRPC) | Asynchronous (Queue / Event) |
|:---|:---|:---|
| Caller needs immediate response? | Yes | No |
| Acceptable if downstream is slow? | No (blocks the caller) | Yes (queue absorbs delay) |
| Downstream can be temporarily down? | No (caller gets an error) | Yes (messages wait in queue) |
| Order matters? | Handled by caller | Use ordered queues (Kafka partitions) |
| Debugging | Simple (request/response) | Complex (trace through queue, consumer logs) |
| Coupling | Tight (caller knows the callee) | Loose (producer does not know consumers) |
| Best for | User-facing API calls, authentication, data reads | Report generation, notifications, analytics, ETL, ML training |

**Rule of thumb:** If the user is waiting for the answer, make it synchronous. If the user can be notified later, make it asynchronous.

---

## Build or Buy?

| Factor | Build | Buy (SaaS / Managed Service) |
|:---|:---|:---|
| Core differentiator? | Yes -- this is your competitive advantage | No -- this is commodity infrastructure |
| Team has expertise? | Yes, and capacity to maintain it | No, or maintaining it distracts from core work |
| Customization needs? | High -- no off-the-shelf solution fits | Low to moderate -- standard solution works |
| Timeline | Have months | Need it this week |
| Ongoing cost | Engineering time (expensive, hidden) | Subscription (visible, predictable) |
| Lock-in risk | Low | Medium to high (migration is painful) |

**Examples:**

| Component | Build | Buy |
|:---|:---|:---|
| Authentication | Almost never | Auth0, Firebase Auth, Cognito |
| Email sending | Never | SendGrid, SES, Postmark |
| Payment processing | Never | Stripe |
| Monitoring | Rarely | Datadog, Grafana Cloud, CloudWatch |
| Vector database | Maybe (pgvector) | Pinecone, Weaviate |
| ML feature store | If you need deep customization | Feast, Tecton, Databricks |
| Diagnostic scoring engine | Yes -- this is your IP | N/A |

---

## When to Add Caching

Add caching when ALL of these are true:

1. The same data is read significantly more often than it is written (10:1 read/write ratio or higher)
2. The data source is slow or expensive (database query > 50ms, external API call, heavy computation)
3. Slightly stale data is acceptable (you can define a TTL)
4. You can define a clear invalidation strategy (TTL, write-through, or event-driven)

**Do NOT add caching when:**
- Data must be real-time (account balances during a transaction)
- Every request is unique (no cache hits)
- The data changes every few seconds (cache is always stale)
- You have not measured whether the data source is actually slow

---

## When to Add a Message Queue

Add a message queue when ANY of these are true:

1. The caller does not need an immediate response (fire and forget)
2. Traffic is bursty and the consumer cannot handle peak load (the queue absorbs spikes)
3. You need guaranteed delivery (at-least-once processing, even if the consumer is temporarily down)
4. Multiple consumers need to react to the same event (fan-out)
5. The producer and consumer need to scale independently

**Do NOT add a message queue when:**
- The system is simple (two services, low traffic) and direct HTTP calls work fine
- You need request-response semantics (the user is waiting)
- You do not have the operational capacity to monitor queue depth, dead letters, and consumer lag

---

## Testing Strategy by System Type

| System Type | Unit Tests | Integration Tests | E2E Tests | Load Tests | Special Tests |
|:---|:---|:---|:---|:---|:---|
| **REST API** | Business logic, validation, transformations | API endpoint + database | Full workflow (submit, process, retrieve) | Concurrent users, throughput | Contract tests (OpenAPI compliance) |
| **Data Pipeline** | Transformation functions, data quality checks | Pipeline stage with real data (small sample) | Full pipeline run (source to destination) | Volume tests (10x expected data) | Schema evolution tests, idempotency tests |
| **ML Model** | Feature engineering, preprocessing | Model training + evaluation | Prediction API (input to output) | Inference latency under load | Accuracy/drift tests, bias tests, reproducibility |
| **Agent System** | Individual tools, routing logic | Tool execution, memory retrieval | Full agent conversation (multi-turn) | Concurrent agent sessions | Guardrail tests (injection, off-topic), cost tests |

---

## Production Readiness Checklist

Use this before any production deployment. Every item is here because its absence has caused an outage.

### Code (5 items)

| # | Item | Verified |
|:---|:---|:---|
| 1 | All environment-specific values are in config/env vars, not hardcoded | |
| 2 | No secrets in code, config files, or git history | |
| 3 | Input validation on every API endpoint (Pydantic or equivalent) | |
| 4 | Error handling: no bare `except:` blocks, errors are logged with context | |
| 5 | Dependencies are pinned to specific versions (lockfile exists) | |

### Testing (5 items)

| # | Item | Verified |
|:---|:---|:---|
| 6 | Unit tests cover core business logic and edge cases | |
| 7 | Integration tests verify database queries and API endpoints work together | |
| 8 | CI pipeline runs all tests on every PR (and blocks merge on failure) | |
| 9 | Load test has been run at expected peak traffic | |
| 10 | ML models: accuracy, latency, and bias tests exist and pass | |

### Security (4 items)

| # | Item | Verified |
|:---|:---|:---|
| 11 | Authentication required on all non-public endpoints | |
| 12 | HTTPS enforced (no HTTP fallback) | |
| 13 | Dependency vulnerability scan passes (Dependabot, safety, or equivalent) | |
| 14 | Database and storage are in private subnets, not publicly accessible | |

### Deployment (3 items)

| # | Item | Verified |
|:---|:---|:---|
| 15 | Automated deployment via CI/CD (no manual SSH deploys) | |
| 16 | Rollback procedure tested (can revert to previous version in under 5 minutes) | |
| 17 | Health check endpoints implemented (`/health/live` and `/health/ready`) | |

### Monitoring (3 items)

| # | Item | Verified |
|:---|:---|:---|
| 18 | Structured logging with request IDs, not print statements | |
| 19 | Metrics dashboard showing RED (rate, errors, duration) for every endpoint | |
| 20 | Alerts configured for error rate spikes and SLO violations, with on-call rotation | |

---

## Anti-Patterns

Patterns that seem reasonable but cause predictable damage in production.

| Anti-Pattern | What Happens | What to Do Instead |
|:---|:---|:---|
| **Premature microservices** | 3 engineers managing 12 services, spending all their time on infra instead of features | Start with a monolith. Extract services only when you have a concrete scaling or team boundary reason. |
| **No tests** | Every deployment is a gamble. "It works on my machine" becomes the team motto. | Write tests before you ship. Start with integration tests if unit tests feel like overhead. |
| **Hardcoded secrets** | API key in a git commit. It is now in the git history forever. Attacker finds it on GitHub within hours. | Use environment variables or a secrets manager. Scan git history with truffleHog or gitleaks. |
| **Log and ignore errors** | `except Exception: pass` or `except Exception as e: logger.error(e)` with no recovery. The system silently corrupts data. | Handle errors explicitly. If you cannot recover, fail loudly. Dead letter queues for async. |
| **No health checks** | Load balancer sends traffic to a container that started but cannot connect to the database. Users see 500 errors. | Implement `/health/ready` that checks all dependencies. Configure the orchestrator to use it. |
| **Caching without invalidation** | Users see stale data for hours. Support tickets pile up. You restart the cache manually. | Define invalidation strategy (TTL, write-through, or event-driven) before adding cache. |
| **Monolith database for everything** | One PostgreSQL instance serves the API, analytics, ML training, and batch jobs. The batch job locks the table and the API goes down. | Read replicas for analytics. Separate databases for workloads with different access patterns. |
| **Alert on everything** | 200 alerts per day. Team ignores all of them. A real P1 gets lost in the noise. | Alert on symptoms (user-facing impact), not causes. Every alert must require human action. |
| **No graceful shutdown** | Container killed mid-request. Half-written database transaction. User sees a 502 and retries, creating duplicate data. | Handle SIGTERM: stop accepting new requests, finish in-flight requests, close connections, exit. |
| **Retry without backoff** | Service is overloaded. All clients retry immediately. Service gets more overloaded. Cascading failure. | Exponential backoff with jitter. Set a max retry count. Use a circuit breaker. |
| **No runbooks** | On-call engineer at 3 AM has never seen this error before. Spends 2 hours figuring out what the previous engineer figured out 6 months ago. | Document common failures and their resolution steps. Link runbooks from alert descriptions. |

---

## Quick Links -- All Chapters

| Chapter | Title |
|:---|:---|
| [01](01_Why.md) | Why |
| [02](02_Concepts.md) | Concepts |
| [03](03_Hello_World.md) | Hello World |
| [04](04_How_It_Works.md) | How It Works |
| [05](05_Building_It.md) | Building It |
| [06](06_Production_Patterns.md) | Production Software Patterns |
| [07](07_System_Design.md) | System Design for AI/Data Services |
| [08](08_Quality_Security_Governance.md) | Quality, Security, and Governance |
| [09](09_Observability_Troubleshooting.md) | Observability and Troubleshooting |
| [10](10_Decision_Guide.md) | **Software Engineering Decision Guide** |
