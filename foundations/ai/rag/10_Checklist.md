# RAG - Checklist

**The decision tables, readiness checklists, and quick reference cards you need when designing, building, and shipping a RAG system to production.**

---

## Should I Use RAG?

Not every AI application needs RAG. This decision table helps you choose the right approach BEFORE you build anything.

### Decision Table

| Situation | Use RAG? | Why |
|---|---|---|
| Knowledge changes frequently (weekly or faster) | **Yes** | Fine-tuned models freeze knowledge at training time. RAG retrieves the latest documents on every query. |
| Answers must cite specific sources | **Yes** | RAG naturally returns which chunks were used. Fine-tuned models cannot point to a source document. |
| Company-specific or private data | **Yes** | Private data should not be baked into a model's weights. RAG keeps data in a controlled knowledge base with access controls. |
| Users ask questions about a large document corpus (thousands of pages) | **Yes** | RAG scales to millions of chunks. Stuffing everything into a prompt does not. |
| Accuracy matters more than speed | **Yes** | RAG grounds the answer in retrieved evidence. Without it, the LLM (Large Language Model, pronounced "L-L-M") relies on training data that may be outdated or wrong. |
| General knowledge questions ("What is photosynthesis?") | **No** | The LLM already knows this from training data. RAG adds latency and cost with no benefit. |
| Creative writing, brainstorming, ideation | **No** | These tasks benefit from the LLM's broad training data and creativity, not from constrained retrieval. |
| Simple classification ("Is this email spam?") | **No** | Classification does not require document retrieval. Fine-tuning or prompting is sufficient. |
| Domain-specific language the model does not understand | **Consider fine-tuning** | If the model struggles with your domain's terminology (e.g., specialized medical or legal jargon), fine-tuning teaches the model the language. RAG provides the facts; fine-tuning provides the vocabulary. |
| You need the model to behave differently (tone, format, reasoning style) | **Consider fine-tuning** | RAG does not change how the model writes. Fine-tuning changes the model's behavior. |
| Both: domain language AND up-to-date facts | **RAG + fine-tuning** | Fine-tune for the domain. RAG for the facts. This is the combination used by many production systems. |

### The One-Sentence Rule

**If the answer depends on YOUR data (not the world's data), you probably need RAG.**

---

## Production Readiness Checklist

Use this table before launching a RAG system to production. Every item should be addressed, even if the answer is "not applicable" or "deferred to v2."

### Data

| Item | Status | Notes |
|---|---|---|
| Documents are indexed and up to date | | How many documents? When was the last ingestion? |
| Freshness strategy defined | | How often are new documents indexed? Manual or automated? |
| PII (Personally Identifiable Information) scanned and handled | | What PII exists? Redacted, masked, or access-controlled? |
| Document sources are authenticated and trusted | | Who can add documents? Is there an approval workflow? |
| Stale document policy defined | | What happens to documents older than X days? Archived? Flagged? |

### Retrieval

| Item | Status | Notes |
|---|---|---|
| Chunk size tested and justified | | What size? What overlap? Why? (See quick reference below) |
| k value (number of retrieved chunks) tuned | | What k? Tested with different values? |
| Embedding model selected and documented | | Which model? Why? (See quick reference below) |
| Hybrid search evaluated | | Is keyword + semantic search needed? Tested? |
| Metadata filtering implemented | | By department, date, document type, access level? |
| Re-ranking evaluated | | Is a cross-encoder re-ranker needed? Tested impact on quality? |
| Retrieval quality benchmarked | | Precision@k, recall@k, MRR (Mean Reciprocal Rank) on labeled test set? |

### Generation

| Item | Status | Notes |
|---|---|---|
| Prompt template tested and versioned | | Is the template in version control? Tested with edge cases? |
| Grounding instructions in prompt | | Does the prompt tell the LLM to use ONLY the retrieved context? |
| "I don't know" behavior defined | | What happens when context is insufficient? Safe fallback message? |
| Hallucination rate measured | | What percentage of answers contain unsupported claims? |
| Citation format defined | | How are sources shown to the user? Clickable links? |
| LLM model selected and documented | | Which model? Why? Cost per query? |

### Security

| Item | Status | Notes |
|---|---|---|
| Access control on retrieval | | Role-based metadata filtering at query time? |
| Prompt injection defense | | Input sanitization? Prompt hardening? Output validation? |
| PII masking in responses | | Output scanned for PII before returning to user? |
| Document poisoning defense | | Source verification? Content validation at ingestion? |
| Audit trail | | Every query, retrieval, and response logged? |
| Compliance requirements identified | | HIPAA (Health Insurance Portability and Accountability Act)? GDPR (General Data Protection Regulation)? SOC 2 (System and Organization Controls 2)? |

### Observability

| Item | Status | Notes |
|---|---|---|
| Retrieval quality tracked | | Precision, recall, hit rate measured regularly? |
| Answer quality tracked | | Faithfulness scored via LLM-as-judge or human review? |
| Latency monitored (P50, P90, P99) | | Dashboard with percentile bands? Alerts on degradation? |
| Cost per query tracked | | Embedding + LLM + infrastructure cost per query? |
| User feedback collected | | Thumbs up/down? Corrections? Escalation tracking? |
| Drift detection in place | | Knowledge freshness, query evolution, model changes? |
| Alerting configured | | What triggers a page? What triggers a Slack notification? |

### Scale

| Item | Status | Notes |
|---|---|---|
| Caching strategy defined | | Semantic cache for repeated/similar queries? Cache TTL (Time to Live)? |
| Concurrent user capacity tested | | How many simultaneous queries can the system handle? |
| Index update strategy defined | | Full re-index vs. incremental? Downtime during re-index? |
| Vector database sized appropriately | | Current document count? Projected growth? Memory requirements? |
| LLM API rate limits understood | | Provider rate limits? Retry strategy? Fallback model? |
| Cost projection for target scale | | Cost at 10x, 100x current query volume? |

---

## The 10-Step RAG System Design Framework

When designing a RAG system from scratch, work through these 10 steps in order. Each step builds on the previous one.

### Step 1: Requirements

| Question | Why It Matters |
|---|---|
| What questions will users ask? | Determines the scope of documents to index and the retrieval strategy. |
| How accurate must answers be? | Determines whether you need re-ranking, human review, or citation requirements. |
| What latency is acceptable? | Determines model choice (local vs. API), caching strategy, and architecture. |
| Who are the users? What are their roles? | Determines access control requirements and metadata filtering. |
| What compliance requirements apply? | Determines data handling, encryption, audit, and vendor selection. |

### Step 2: Data Pipeline

| Question | Why It Matters |
|---|---|
| Where do documents live? | Confluence, S3, SharePoint, Git, databases -- each needs a different connector. |
| How often do documents change? | Determines ingestion frequency (batch vs. real-time). |
| What formats are the documents? | PDF, HTML, Markdown, code -- each needs a different parser. |
| How large is the corpus? | 1,000 documents is different from 1,000,000. Affects infrastructure choices. |

### Step 3: Embedding Model

Choose based on your use case. See the quick reference card below.

### Step 4: Index Strategy

| Decision | Options | Tradeoff |
|---|---|---|
| Chunk size | 256, 512, 1024 tokens | Smaller = precise retrieval, less context. Larger = more context, noisier retrieval. |
| Chunk overlap | 0%, 10%, 20% | More overlap = fewer boundary losses, more storage. |
| ANN (Approximate Nearest Neighbor) algorithm | HNSW (Hierarchical Navigable Small World), IVF (Inverted File Index) | HNSW = faster search, more memory. IVF = less memory, slower. |
| Vector database | ChromaDB, Pinecone, Weaviate, pgvector, FAISS (Facebook AI Similarity Search) | See quick reference card below. |

### Step 5: Retrieval

| Decision | Options | Tradeoff |
|---|---|---|
| Retrieval method | Vector only, keyword only, hybrid | Hybrid catches both semantic matches and exact keyword matches. |
| Number of chunks (k) | 3, 5, 10, 20 | More chunks = more context for LLM, but also more noise and higher cost. |
| Re-ranking | None, cross-encoder, Cohere Rerank | Improves precision at the cost of 50-200 ms latency. |
| Metadata filtering | None, role-based, date-based, source-based | Filters reduce search space and enforce access control. |

### Step 6: Generation

| Decision | Options | Tradeoff |
|---|---|---|
| LLM | Claude, GPT-4o, Llama, Mistral | Capability vs. cost vs. latency vs. data privacy. |
| Prompt template | Minimal, production (grounded), chain-of-thought | More structured prompts = better grounding, longer prompts. |
| Response format | Free text, structured JSON, citations required | Structured = easier to validate, less natural. |

### Step 7: Scaling

| Decision | Options | Tradeoff |
|---|---|---|
| Caching | None, exact match, semantic cache | Reduces cost and latency for repeated queries. Stale cache risk. |
| Horizontal scaling | Single instance, replicated vector DB, load-balanced LLM calls | More infra complexity, higher availability. |
| Index updates | Full rebuild, incremental, real-time | Full rebuild = simplest but has downtime. Real-time = complex but always fresh. |

### Step 8: Security

Apply the threat model from Chapter 08. At minimum: input validation, access control on retrieval, PII scanning on output, audit logging.

### Step 9: Governance

| Decision | Options | Tradeoff |
|---|---|---|
| Data retention | Keep all queries, keep summaries only, delete after 30 days | Compliance vs. debugging capability. |
| Model versioning | Pin to specific version, auto-update, A/B test | Stability vs. staying current. |
| Prompt versioning | Version control all templates, track which version served each query | Adds overhead but critical for debugging regressions. |
| Change management | Document every change to models, prompts, data sources | SOC 2 requirement; also just good engineering. |

### Step 10: Iteration

RAG systems are never "done." Plan for continuous improvement:

1. Review user feedback weekly
2. Run retrieval benchmarks monthly
3. Update the knowledge base as documents change
4. Test new embedding models and LLMs quarterly
5. Refine prompts based on failure patterns

---

## Quick Reference Cards

### Chunk Size Selection

| Document Type | Recommended Chunk Size | Recommended Overlap | Why |
|---|---|---|---|
| Technical documentation | 512 tokens | 10-15% | Balanced: enough context for technical concepts without diluting specifics |
| FAQ / Knowledge base | 256 tokens | 5-10% | Short, self-contained answers; smaller chunks match question-answer pairs |
| Legal / regulatory documents | 1024 tokens | 15-20% | Legal clauses need full context; cross-references span paragraphs |
| Code files | 512 tokens (by function/class) | 10% | Semantic chunking by function boundaries is better than token-based splitting |
| Conversational transcripts | 256-512 tokens | 10% | Speaker turns are natural boundaries; smaller chunks isolate specific exchanges |
| Research papers | 512-1024 tokens | 15% | Sections are self-contained but reference each other |

**When in doubt:** Start with 512 tokens, 10% overlap. Measure retrieval quality. Adjust.

### Embedding Model Selection

| Model | Dimensions | Max Tokens | Strengths | Best For |
|---|---|---|---|---|
| OpenAI text-embedding-3-small | 1536 | 8191 | Low cost, good quality, fast | Production systems where cost matters |
| OpenAI text-embedding-3-large | 3072 | 8191 | Highest quality (OpenAI), supports dimension reduction | When accuracy is critical and budget allows |
| Cohere embed-v3 | 1024 | 512 | Strong multilingual, supports search/classification modes | Multilingual knowledge bases |
| nomic-embed-text | 768 | 8192 | Open source, runs locally, long context | Local/private deployments, prototyping |
| BGE (BAAI General Embedding) large | 1024 | 512 | Open source, strong benchmarks | Self-hosted production on GPU |
| Voyage AI voyage-3 | 1024 | 32000 | Very long context, strong on code | Codebases, long-document retrieval |

**When in doubt:** Start with `text-embedding-3-small` (API) or `nomic-embed-text` (local). Upgrade if retrieval quality benchmarks show a gap.

### Vector Database Selection

| Database | Hosted/Self-Hosted | Best For | Metadata Filtering | Hybrid Search |
|---|---|---|---|---|
| ChromaDB | Self-hosted | Prototyping, small-medium datasets | Yes | No (vector only) |
| Pinecone | Hosted (managed) | Production, auto-scaling, minimal ops | Yes | Yes (sparse + dense) |
| Weaviate | Both | Production, hybrid search, multi-modal | Yes | Yes (BM25 + vector) |
| pgvector | Self-hosted (PostgreSQL extension) | Teams already on PostgreSQL, moderate scale | Yes (SQL WHERE clauses) | Yes (with pg_trgm) |
| FAISS | Self-hosted (library, not a database) | Research, maximum flexibility, very large scale | No (application layer) | No |
| Qdrant | Both | Production, filtering-heavy workloads, Rust performance | Yes (rich filtering) | Yes |
| Milvus | Both | Very large scale (billions of vectors), distributed | Yes | Yes |

**When in doubt:** ChromaDB for prototyping. Pinecone or Weaviate for managed production. pgvector if you are already on PostgreSQL and want fewer moving parts.

### LLM Selection for RAG Generation

| Model | Strengths for RAG | Latency | Cost (Relative) | Context Window |
|---|---|---|---|---|
| Claude Sonnet | Strong instruction following, long context, good at grounding | Medium | Medium | 200K tokens |
| Claude Haiku | Fast, cheap, good for simple Q&A | Low | Low | 200K tokens |
| GPT-4o | Strong reasoning, good at synthesis | Medium | Medium | 128K tokens |
| GPT-4o-mini | Fast, cheap, good for high-volume | Low | Low | 128K tokens |
| Llama 3.1 (70B) | Open source, runs on-premise, no data leaves your infra | Depends on hardware | Infrastructure cost only | 128K tokens |
| Mistral Large | Strong European language support, good reasoning | Medium | Medium | 128K tokens |

**When in doubt:** Claude Sonnet or GPT-4o for quality-critical applications. Claude Haiku or GPT-4o-mini for high-volume, cost-sensitive applications. Llama for on-premise data privacy requirements.

---

## The RAG System Design Cheat Sheet

One page. Pin it to the wall.

```
BEFORE YOU BUILD
  1. Do I need RAG? (See decision table above)
  2. What documents? How many? How often do they change?
  3. Who are the users? What can they see?
  4. What accuracy, latency, and cost are acceptable?

BUILDING
  5. Chunk documents (start: 512 tokens, 10% overlap)
  6. Embed chunks (start: text-embedding-3-small or nomic-embed-text)
  7. Store in vector DB (start: ChromaDB for prototype, Pinecone/Weaviate for prod)
  8. Retrieve top-k chunks (start: k=5, tune from there)
  9. Generate with grounded prompt (ONLY use retrieved context, cite sources)
  10. Add access control metadata at ingestion, filter at query time

SHIPPING
  11. Measure retrieval quality (precision, recall, hit rate)
  12. Measure answer quality (faithfulness via LLM-as-judge)
  13. Set up latency and cost dashboards
  14. Implement input validation and output scanning
  15. Collect user feedback (thumbs up/down, corrections)

MAINTAINING
  16. Index new documents on schedule
  17. Review feedback weekly, fix top failure patterns
  18. Re-benchmark monthly
  19. Watch for drift (knowledge, query, embedding)
  20. Iterate: prompts, chunk sizes, models, k values
```

---

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---|---|---|
| Skipping retrieval evaluation | Teams focus on the LLM and assume retrieval "just works" | Benchmark retrieval BEFORE tuning generation. Most bad answers are retrieval failures. |
| Using different embedding models for indexing and querying | Upgraded the model but did not re-index | Always re-index when changing embedding models. Vectors from different models are incompatible. |
| Chunk size never tested | Default of 500 tokens used without validation | Test 256, 512, and 1024 on your data. Measure retrieval quality for each. |
| No access control on retrieval | "We'll add it later" | Add metadata filtering from day one. Retrofitting access control is painful and error-prone. |
| Prompt template not versioned | Template edited in code without tracking | Put templates in version control. Log which version served each query. |
| No "I don't know" behavior | LLM makes up answers when context is insufficient | Explicitly instruct the LLM to decline when context is insufficient. Test this case. |
| Ignoring cost until the bill arrives | Prototype cost was trivial; production cost is 100x | Estimate per-query cost BEFORE launch. Set budget alerts. |
| Knowledge base never refreshed | Initial load was thorough; nobody set up ongoing ingestion | Automate document ingestion on a schedule. Monitor knowledge base freshness. |

---

## Chapter Summary: The Full RAG Curriculum

This 10-chapter series covers RAG from first principles to production readiness.

| Chapter | What You Learned |
|---|---|
| [01 - Why](01_Why.md) | Why LLMs need retrieval. The filing cabinet analogy. Where RAG fits in production systems. |
| [02 - Concepts](02_Concepts.md) | Embeddings, vector databases, chunking, retrieval -- all in plain English. |
| [03 - Hello World](03_Hello_World.md) | A working RAG system in 20 lines of code. The minimum viable pipeline. |
| [04 - How It Works](04_How_It_Works.md) | Why embeddings capture meaning. Cosine similarity. ANN algorithms (HNSW, IVF). The full query flow. |
| [05 - Decisions](05_Decisions.md) | Chunk size, embedding model, retrieval strategy, LLM choice. Every tradeoff with data. |
| [06 - Real World](06_Real_World.md) | How GitHub Copilot, Perplexity, Glean, Notion AI, and others use RAG in production. |
| [07 - System Design](07_System_Design.md) | Scaling RAG: caching, hybrid search, re-ranking, multi-tenant architecture. |
| [08 - Security](08_Security.md) | Prompt injection, data leakage, PII, document poisoning, compliance (HIPAA, GDPR, SOC 2). |
| [09 - Observability](09_Observability.md) | Retrieval quality, answer quality, latency, cost. LLM-as-judge. Drift detection. Debugging method. |
| [10 - Checklist](10_Checklist.md) | This page. Decision tables, production readiness checklist, quick reference cards. |

**Hands-on notebook:** [RAG from Scratch on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_RAG_from_Scratch.ipynb) -- builds everything from ingestion to evaluation, running locally with Ollama.

---

## Quick Links

| Chapter | Topic |
|---|---|
| [01 - Why](01_Why.md) | Why RAG matters |
| [02 - Concepts](02_Concepts.md) | Embeddings, vectors, chunking |
| [03 - Hello World](03_Hello_World.md) | Build a RAG system in 20 lines |
| [04 - How It Works](04_How_It_Works.md) | Embeddings, similarity, ANN algorithms |
| [05 - Decisions](05_Decisions.md) | Every tradeoff and choice |
| [06 - Real World](06_Real_World.md) | How production RAG systems work |
| [07 - System Design](07_System_Design.md) | Scaling, caching, hybrid search |
| [08 - Security](08_Security.md) | Prompt injection, data leakage |
| [09 - Observability](09_Observability.md) | Measuring quality and cost |
| [10 - Checklist](10_Checklist.md) | This page |

**Hands-on notebook:** [RAG from Scratch on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_RAG_from_Scratch.ipynb)
