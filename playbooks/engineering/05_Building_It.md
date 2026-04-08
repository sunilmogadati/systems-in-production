# Building a Complete Production Service

This chapter builds a RAG prediction service from scratch. Not a sketch. A complete, deployable system with project structure, validation, tests, Docker, and CI/CD. Every step builds on the previous one. By the end, you have a production service you can deploy today.

This is the same pattern whether you are deploying an ML model, a RAG endpoint, or a diagnostic agent. The architecture is identical. Only the processing logic changes.

---

## The Complete Project Structure

Here is what you are building:

```
rag-service/
    src/
        __init__.py
        main.py                  # FastAPI application entry point
        config.py                # Configuration from environment variables
        routes/
            __init__.py
            predict.py           # Prediction/RAG endpoints
            health.py            # Health and readiness checks
        models/
            __init__.py
            requests.py          # Pydantic request models
            responses.py         # Pydantic response models
        services/
            __init__.py
            rag_service.py       # Business logic: retrieval + generation
            embedding_service.py # Embedding generation
        repositories/
            __init__.py
            vector_store.py      # Vector store data access
            document_store.py    # Document metadata storage
        middleware/
            __init__.py
            auth.py              # Authentication middleware
            logging.py           # Request/response logging
    tests/
        __init__.py
        unit/
            __init__.py
            test_rag_service.py
            test_embedding_service.py
            test_models.py
        integration/
            __init__.py
            test_api.py
            test_vector_store.py
    Dockerfile
    docker-compose.yml
    Makefile
    requirements.txt
    requirements-dev.txt
    .env.example
    .github/
        workflows/
            ci.yml
```

```mermaid
flowchart TD
    subgraph "API Layer"
        MAIN[main.py] --> ROUTES[routes/]
        ROUTES --> PREDICT[predict.py]
        ROUTES --> HEALTH[health.py]
    end

    subgraph "Validation"
        PREDICT --> REQ[models/requests.py]
        PREDICT --> RES[models/responses.py]
    end

    subgraph "Business Logic"
        PREDICT --> RAG[services/rag_service.py]
        RAG --> EMB[services/embedding_service.py]
    end

    subgraph "Data Access"
        RAG --> VS[repositories/vector_store.py]
        RAG --> DS[repositories/document_store.py]
    end

    subgraph "Cross-Cutting"
        MAIN --> AUTH[middleware/auth.py]
        MAIN --> LOG[middleware/logging.py]
        MAIN --> CFG[config.py]
    end
```

---

## Step 1: Configuration

Configuration comes from environment variables. Never hardcode secrets, URLs, or model paths.

**`src/config.py`**

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    # Application
    app_name: str = "rag-service"
    debug: bool = False

    # Database
    database_url: str = "postgresql://localhost:5432/ragdb"

    # Vector Store
    vector_store_url: str = "http://localhost:6333"
    vector_store_collection: str = "documents"

    # LLM
    llm_api_key: str = ""
    llm_model: str = "claude-sonnet-4-20250514"
    llm_max_tokens: int = 1024

    # Embedding
    embedding_model: str = "text-embedding-3-small"
    embedding_dimension: int = 1536

    # Auth
    api_key_header: str = "X-API-Key"
    valid_api_keys: list[str] = []

    class Config:
        env_file = ".env"


settings = Settings()
```

**`.env.example`** (committed to Git, actual `.env` is gitignored)

```
DATABASE_URL=postgresql://user:pass@localhost:5432/ragdb
VECTOR_STORE_URL=http://localhost:6333
LLM_API_KEY=your-key-here
VALID_API_KEYS=["key1","key2"]
DEBUG=true
```

---

## Step 2: Pydantic Models for Request/Response Validation

**`src/models/requests.py`**

```python
from pydantic import BaseModel, Field


class QueryRequest(BaseModel):
    query: str = Field(
        ...,
        min_length=1,
        max_length=2000,
        description="The question to answer"
    )
    top_k: int = Field(
        default=5,
        ge=1,
        le=20,
        description="Number of documents to retrieve"
    )
    include_sources: bool = Field(
        default=True,
        description="Whether to include source documents in response"
    )


class PredictionRequest(BaseModel):
    features: dict[str, float] = Field(
        ...,
        description="Feature name to value mapping"
    )
    model_version: str = Field(
        default="latest",
        description="Model version to use for prediction"
    )
```

**`src/models/responses.py`**

```python
from pydantic import BaseModel, Field


class Source(BaseModel):
    document_id: str
    title: str
    relevance_score: float
    snippet: str


class QueryResponse(BaseModel):
    answer: str
    sources: list[Source] = []
    model_version: str
    tokens_used: int


class PredictionResponse(BaseModel):
    prediction: str | float
    confidence: float = Field(..., ge=0.0, le=1.0)
    model_version: str


class HealthResponse(BaseModel):
    status: str
    version: str
    dependencies: dict[str, str]
```

---

## Step 3: Repositories (Data Access Layer)

Repositories isolate data access. Your service layer never talks directly to a database or vector store.

**`src/repositories/vector_store.py`**

```python
import httpx
from src.config import settings


class VectorStoreRepository:
    def __init__(self, client: httpx.AsyncClient):
        self.client = client
        self.base_url = settings.vector_store_url
        self.collection = settings.vector_store_collection

    async def search(self, vector: list[float], top_k: int = 5) -> list[dict]:
        """Search for similar documents by vector."""
        response = await self.client.post(
            f"{self.base_url}/collections/{self.collection}/points/search",
            json={
                "vector": vector,
                "limit": top_k,
                "with_payload": True,
            },
        )
        response.raise_for_status()
        results = response.json().get("result", [])
        return [
            {
                "document_id": str(r["id"]),
                "title": r["payload"].get("title", ""),
                "content": r["payload"].get("content", ""),
                "score": r["score"],
            }
            for r in results
        ]

    async def health_check(self) -> bool:
        """Check if the vector store is reachable."""
        try:
            response = await self.client.get(f"{self.base_url}/healthz")
            return response.status_code == 200
        except Exception:
            return False
```

---

## Step 4: Services (Business Logic Layer)

The service layer contains your core logic -- retrieval, generation, prediction. It depends on repositories for data access and is independent of the HTTP layer.

**`src/services/embedding_service.py`**

```python
import httpx
from src.config import settings


class EmbeddingService:
    def __init__(self, client: httpx.AsyncClient):
        self.client = client

    async def embed(self, text: str) -> list[float]:
        """Generate an embedding vector for the given text."""
        response = await self.client.post(
            "https://api.openai.com/v1/embeddings",
            headers={"Authorization": f"Bearer {settings.llm_api_key}"},
            json={
                "model": settings.embedding_model,
                "input": text,
            },
        )
        response.raise_for_status()
        return response.json()["data"][0]["embedding"]
```

**`src/services/rag_service.py`**

```python
import logging
from src.models.requests import QueryRequest
from src.models.responses import QueryResponse, Source
from src.repositories.vector_store import VectorStoreRepository
from src.services.embedding_service import EmbeddingService
from src.config import settings

logger = logging.getLogger(__name__)


class RAGService:
    def __init__(
        self,
        vector_store: VectorStoreRepository,
        embedding_service: EmbeddingService,
        llm_client,
    ):
        self.vector_store = vector_store
        self.embedding_service = embedding_service
        self.llm_client = llm_client

    async def answer(self, request: QueryRequest) -> QueryResponse:
        """Full RAG pipeline: embed -> retrieve -> generate."""

        # Step 1: Embed the query
        logger.info(f"Embedding query: {request.query[:50]}...")
        query_vector = await self.embedding_service.embed(request.query)

        # Step 2: Retrieve relevant documents
        logger.info(f"Retrieving top {request.top_k} documents")
        documents = await self.vector_store.search(query_vector, top_k=request.top_k)

        # Step 3: Build context from retrieved documents
        context = "\n\n".join(
            f"[{doc['title']}]\n{doc['content']}" for doc in documents
        )

        # Step 4: Generate answer using LLM
        prompt = self._build_prompt(request.query, context)
        llm_response = await self.llm_client.generate(
            prompt=prompt,
            max_tokens=settings.llm_max_tokens,
        )

        # Step 5: Build response
        sources = []
        if request.include_sources:
            sources = [
                Source(
                    document_id=doc["document_id"],
                    title=doc["title"],
                    relevance_score=doc["score"],
                    snippet=doc["content"][:200],
                )
                for doc in documents
            ]

        return QueryResponse(
            answer=llm_response.text,
            sources=sources,
            model_version=settings.llm_model,
            tokens_used=llm_response.usage.total_tokens,
        )

    def _build_prompt(self, query: str, context: str) -> str:
        return f"""Answer the following question based on the provided context.
If the context does not contain enough information, say so clearly.

Context:
{context}

Question: {query}

Answer:"""
```

---

## Step 5: Routes (API Layer)

Routes are thin. They validate input, call the service, and return the response. No business logic lives here.

**`src/routes/health.py`**

```python
from fastapi import APIRouter, Depends
from src.models.responses import HealthResponse

router = APIRouter(tags=["health"])


@router.get("/health", response_model=HealthResponse)
async def health_check(vector_store=Depends(get_vector_store)):
    vs_healthy = await vector_store.health_check()
    return HealthResponse(
        status="healthy" if vs_healthy else "degraded",
        version="1.0.0",
        dependencies={
            "vector_store": "healthy" if vs_healthy else "unhealthy",
        },
    )
```

**`src/routes/predict.py`**

```python
from fastapi import APIRouter, Depends, HTTPException
from src.models.requests import QueryRequest
from src.models.responses import QueryResponse
from src.services.rag_service import RAGService

router = APIRouter(tags=["predictions"])


@router.post("/ask", response_model=QueryResponse)
async def ask(request: QueryRequest, rag_service: RAGService = Depends(get_rag_service)):
    try:
        return await rag_service.answer(request)
    except Exception as e:
        logger.error(f"RAG pipeline failed: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Failed to generate answer")
```

---

## Step 6: FastAPI Application

**`src/main.py`**

```python
import logging
from contextlib import asynccontextmanager
import httpx
from fastapi import FastAPI

from src.config import settings
from src.routes import health, predict
from src.middleware.auth import AuthMiddleware
from src.middleware.logging import LoggingMiddleware

logging.basicConfig(
    level=logging.DEBUG if settings.debug else logging.INFO,
    format="%(asctime)s %(name)s %(levelname)s %(message)s",
)
logger = logging.getLogger(__name__)

# Shared HTTP client, created at startup, closed at shutdown
http_client: httpx.AsyncClient | None = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    global http_client
    http_client = httpx.AsyncClient(timeout=30.0)
    logger.info(f"Starting {settings.app_name}")
    yield
    # Shutdown
    await http_client.aclose()
    logger.info(f"Shutting down {settings.app_name}")


app = FastAPI(
    title=settings.app_name,
    lifespan=lifespan,
)

# Middleware (applied in reverse order)
app.add_middleware(LoggingMiddleware)
app.add_middleware(AuthMiddleware, api_keys=settings.valid_api_keys)

# Routes
app.include_router(health.router)
app.include_router(predict.router, prefix="/v1")
```

---

## Step 7: Unit Tests

Unit tests verify individual functions in isolation. Dependencies are mocked.

**`tests/unit/test_rag_service.py`**

```python
import pytest
from unittest.mock import AsyncMock, MagicMock
from src.services.rag_service import RAGService
from src.models.requests import QueryRequest


@pytest.fixture
def mock_vector_store():
    store = AsyncMock()
    store.search.return_value = [
        {
            "document_id": "doc-1",
            "title": "Test Document",
            "content": "This is test content about machine learning.",
            "score": 0.95,
        }
    ]
    return store


@pytest.fixture
def mock_embedding_service():
    service = AsyncMock()
    service.embed.return_value = [0.1] * 1536
    return service


@pytest.fixture
def mock_llm_client():
    client = AsyncMock()
    client.generate.return_value = MagicMock(
        text="Machine learning is a subset of AI.",
        usage=MagicMock(total_tokens=150),
    )
    return client


@pytest.fixture
def rag_service(mock_vector_store, mock_embedding_service, mock_llm_client):
    return RAGService(
        vector_store=mock_vector_store,
        embedding_service=mock_embedding_service,
        llm_client=mock_llm_client,
    )


@pytest.mark.asyncio
async def test_answer_returns_response(rag_service):
    request = QueryRequest(query="What is machine learning?")
    response = await rag_service.answer(request)

    assert response.answer == "Machine learning is a subset of AI."
    assert len(response.sources) == 1
    assert response.sources[0].document_id == "doc-1"
    assert response.tokens_used == 150


@pytest.mark.asyncio
async def test_answer_without_sources(rag_service):
    request = QueryRequest(query="What is ML?", include_sources=False)
    response = await rag_service.answer(request)

    assert response.sources == []


@pytest.mark.asyncio
async def test_answer_calls_services_in_order(
    rag_service, mock_embedding_service, mock_vector_store, mock_llm_client
):
    request = QueryRequest(query="Test query")
    await rag_service.answer(request)

    # Verify the pipeline executed in order
    mock_embedding_service.embed.assert_called_once_with("Test query")
    mock_vector_store.search.assert_called_once()
    mock_llm_client.generate.assert_called_once()
```

**`tests/unit/test_models.py`**

```python
import pytest
from pydantic import ValidationError
from src.models.requests import QueryRequest


def test_query_request_valid():
    req = QueryRequest(query="What is RAG?")
    assert req.query == "What is RAG?"
    assert req.top_k == 5  # default
    assert req.include_sources is True  # default


def test_query_request_empty_query():
    with pytest.raises(ValidationError):
        QueryRequest(query="")


def test_query_request_top_k_out_of_range():
    with pytest.raises(ValidationError):
        QueryRequest(query="Valid query", top_k=100)


def test_query_request_top_k_minimum():
    with pytest.raises(ValidationError):
        QueryRequest(query="Valid query", top_k=0)
```

---

## Step 8: Integration Tests

Integration tests verify that components work together with real (or near-real) dependencies.

**`tests/integration/test_api.py`**

```python
import pytest
from httpx import AsyncClient, ASGITransport
from src.main import app


@pytest.fixture
async def client():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client


@pytest.mark.asyncio
async def test_health_endpoint(client):
    response = await client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] in ["healthy", "degraded"]
    assert "version" in data


@pytest.mark.asyncio
async def test_ask_endpoint_requires_auth(client):
    response = await client.post("/v1/ask", json={"query": "What is RAG?"})
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_ask_endpoint_validates_input(client):
    response = await client.post(
        "/v1/ask",
        json={"query": ""},
        headers={"X-API-Key": "test-key"},
    )
    assert response.status_code == 422


@pytest.mark.asyncio
async def test_ask_endpoint_success(client):
    response = await client.post(
        "/v1/ask",
        json={"query": "What is machine learning?", "top_k": 3},
        headers={"X-API-Key": "test-key"},
    )
    assert response.status_code == 200
    data = response.json()
    assert "answer" in data
    assert "sources" in data
```

---

## Step 9: Dockerfile (Multi-Stage Build)

A multi-stage build keeps the final image small. The first stage installs dependencies. The second stage copies only what is needed to run.

**`Dockerfile`**

```dockerfile
# Stage 1: Build dependencies
FROM python:3.11-slim AS builder
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim
WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /install /usr/local

# Copy application code
COPY src/ src/

# Non-root user for security
RUN useradd --create-home appuser
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Why multi-stage? The builder stage may include compilers and build tools needed for packages like numpy. The runtime stage only has the compiled packages. This can reduce image size by 50% or more.

---

## Step 10: Docker Compose

**`docker-compose.yml`**

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      qdrant:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ragdb
      POSTGRES_USER: raguser
      POSTGRES_PASSWORD: ragpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U raguser -d ragdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:6333/healthz || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  qdrant_data:
```

One command starts everything:

```bash
docker compose up --build
```

**You Should See:** Three containers starting -- your app, PostgreSQL, and Qdrant (vector store). The app waits for both dependencies to be healthy before starting.

---

## Step 11: GitHub Actions CI/CD Pipeline

**`.github/workflows/ci.yml`**

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint
        run: ruff check src/ tests/

      - name: Type check
        run: mypy src/

      - name: Unit tests
        run: pytest tests/unit/ -v --tb=short

      - name: Integration tests
        run: pytest tests/integration/ -v --tb=short
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t rag-service:${{ github.sha }} .

      - name: Tag as latest
        run: docker tag rag-service:${{ github.sha }} rag-service:latest

      # In a real deployment, push to a container registry:
      # - name: Push to ECR/GCR/Docker Hub
      #   run: docker push $REGISTRY/rag-service:${{ github.sha }}
```

Every push to `main` or every pull request triggers this pipeline. Tests must pass before the Docker image is built. The image is tagged with the Git commit SHA for traceability.

---

## Step 12: Makefile

A Makefile gives your team a consistent interface for common commands. Nobody needs to remember Docker flags or pytest options.

**`Makefile`**

```makefile
.PHONY: install test lint run build up down clean

install:
	pip install -r requirements.txt
	pip install -r requirements-dev.txt

test:
	pytest tests/unit/ -v --tb=short

test-integration:
	pytest tests/integration/ -v --tb=short

test-all:
	pytest tests/ -v --tb=short

lint:
	ruff check src/ tests/
	mypy src/

format:
	ruff format src/ tests/

run:
	uvicorn src.main:app --reload --host 0.0.0.0 --port 8000

build:
	docker build -t rag-service .

up:
	docker compose up --build -d

down:
	docker compose down

logs:
	docker compose logs -f app

clean:
	docker compose down -v
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type d -name .pytest_cache -exec rm -rf {} +
```

Usage:

```bash
make install       # Set up your local dev environment
make test          # Run unit tests
make lint          # Check code quality
make run           # Run locally with hot reload
make up            # Start everything in Docker
make logs          # Tail application logs
make down          # Stop all containers
```

---

## The Architecture

```mermaid
flowchart TD
    CLIENT[Client] --> API[FastAPI\nsrc/main.py]

    subgraph "API Layer"
        API --> MW[Middleware\nauth + logging]
        MW --> ROUTES[Routes\nhealth.py + predict.py]
    end

    subgraph "Validation"
        ROUTES --> PYDANTIC[Pydantic Models\nrequests.py + responses.py]
    end

    subgraph "Business Logic"
        ROUTES --> SVC[RAG Service\nrag_service.py]
        SVC --> EMB[Embedding Service\nembedding_service.py]
    end

    subgraph "Data Access"
        SVC --> REPO_VS[Vector Store Repo\nvector_store.py]
        SVC --> REPO_DB[Document Store Repo\ndocument_store.py]
    end

    subgraph "Infrastructure"
        REPO_VS --> QDRANT[(Qdrant\nVector Store)]
        REPO_DB --> PG[(PostgreSQL\nMetadata)]
        EMB --> LLM_API[LLM API\nEmbedding + Generation]
    end
```

**Why this structure matters:**

- **Routes are thin.** They handle HTTP concerns only. No business logic. If you swap FastAPI for another framework, only routes change.
- **Services contain logic.** The RAG pipeline lives here. It is testable in isolation because it depends on interfaces (repositories), not infrastructure.
- **Repositories isolate data access.** Swap Qdrant for Pinecone by writing a new repository class. The service layer does not change.
- **Pydantic models are the contract.** They define exactly what goes in and what comes out. Clients and developers can rely on this contract.
- **Configuration is external.** The same code runs locally, in staging, and in production. Only environment variables change.

---

## Dependencies File

**`requirements.txt`**

```
fastapi==0.115.0
uvicorn==0.30.0
pydantic==2.9.0
pydantic-settings==2.5.0
httpx==0.27.0
sqlalchemy==2.0.35
asyncpg==0.30.0
python-jose==3.3.0
```

**`requirements-dev.txt`**

```
pytest==8.3.0
pytest-asyncio==0.24.0
ruff==0.6.0
mypy==1.11.0
coverage==7.6.0
```

---

## What You Have Now

After completing this chapter, you have:

1. A well-structured FastAPI application with separated concerns
2. Pydantic validation on all inputs and outputs
3. A service layer with business logic isolated from HTTP
4. Repository pattern for data access abstraction
5. Authentication middleware
6. Unit tests with mocked dependencies
7. Integration tests against the real API
8. A multi-stage Dockerfile that produces a small, secure image
9. Docker Compose for local development with all dependencies
10. A GitHub Actions pipeline that tests and builds on every push
11. A Makefile for consistent developer experience

This is not a demo. This is a deployable production service. The same structure works for:

- **ML model serving** -- Replace RAGService with a PredictionService that loads and runs a model
- **RAG endpoints** -- Exactly what we built
- **Diagnostic agents** -- Replace the RAG pipeline with an agent loop that calls tools and returns structured diagnostics
- **Data pipeline APIs** -- Replace services with pipeline triggers and status endpoints

The pattern is the same. The processing logic is what changes.

---

## Quick Links

| Chapter | Title |
|---|---|
| [01 -- Why](01_Why.md) | Software Engineering for Production Systems -- Why It Matters |
| [02 -- Concepts](02_Concepts.md) | Software Engineering Concepts for AI/Data Systems |
| [03 -- Hello World](03_Hello_World.md) | Notebook to API in 10 Minutes |
| [04 -- How It Works](04_How_It_Works.md) | How Production Services Work |
| [05 -- Building It](05_Building_It.md) | Building a Complete Production Service |
| [06 -- Production Patterns](06_Production_Patterns.md) | Production Software Patterns |
| [07 -- System Design](07_System_Design.md) | System Design for AI/Data Servicesads |
| [08 -- CI/CD](08_Quality_Security_Governance.md) | Automated Pipelines from Commit to Production |
| [09 -- Observability and Troubleshooting](09_Observability_Troubleshooting.md) | Observability for Models, Pipelines, and Agents |
| [10 -- Decision Guide](10_Decision_Guide.md) | Production Patterns for Reliable Systems |
