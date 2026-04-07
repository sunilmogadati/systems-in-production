# RAG - Concepts

**Embeddings, vector databases, chunking, and retrieval. Every concept in plain English before you see any code.**

---

## The Core Idea: Meaning as Numbers

How does a computer know that "connection pool exhaustion" and "database ran out of connections" mean the same thing? They share zero words in common.

The answer: **embeddings**.

An embedding converts text into a list of numbers (a vector) that captures its **meaning**, not its words. Texts with similar meaning get similar numbers, even if they use completely different words.

**Analogy:** Think of GPS coordinates. "New York City" and "Manhattan" have different names but nearly identical coordinates (40.7, -74.0). Similarly, "connection pool exhaustion" and "database ran out of connections" have different words but nearly identical embedding vectors.

```
"connection pool exhaustion"       → [0.82, -0.15, 0.67, 0.33, ...]
"database ran out of connections"  → [0.81, -0.14, 0.68, 0.32, ...]  ← very similar
"the weather is nice today"        → [-0.45, 0.91, -0.12, 0.08, ...] ← very different
```

The numbers are close when the meaning is close. That's the foundation of RAG.

---

## Key Concepts (Glossary)

| Term | What It Means | Analogy |
|---|---|---|
| **Embedding** | A list of numbers representing the meaning of a piece of text. Typically 384-3072 numbers. | GPS coordinates for meaning. |
| **Vector** | A list of numbers. "Embedding vector" = the numbers that represent a text's meaning. | A point in space. |
| **Vector Database** | A database optimized for finding similar vectors. Not rows and columns like SQL. | A GPS search: "find all locations within 5 miles of here." |
| **Chunk** | A piece of a document (paragraph, section, or fixed-size block). Documents are split into chunks before embedding. | Tearing a book into individual pages so you can search page by page. |
| **Retrieval** | Finding the most relevant chunks for a given question. | Searching the filing cabinet for the right folder. |
| **LLM** (Large Language Model) | The AI that generates the answer using the retrieved context. Mistral, GPT-4, Claude, Llama. | The expert who reads the retrieved documents and answers your question. |
| **Context window** | How much text an LLM can process at once. Measured in tokens (~0.75 words per token). | The expert's desk. Only so many documents can fit on it at once. |
| **Prompt** | The text you send to the LLM: your question + the retrieved chunks + instructions. | The briefing you give the expert before they answer. |
| **Hallucination** | When the LLM makes up information that's not in the retrieved documents. | The expert guessing when they should say "I don't know." |
| **Semantic search** | Finding documents by meaning (using embeddings). Opposite of keyword search. | Finding a book by what it's about, not by matching title words. |

---

## The 4-Step Pipeline in Detail

```
YOUR DOCUMENTS                    YOUR QUESTION
     |                                 |
     v                                 v
[1. CHUNK]                        [3. EMBED question]
  Split into paragraphs                |
     |                                 v
     v                           [3. RETRIEVE]
[2. EMBED each chunk]              Find similar chunks
  Convert to vectors                   |
     |                                 v
     v                           [4. GENERATE]
[2. STORE in vector DB]            LLM reads chunks
                                   + question
                                       |
                                       v
                                   ANSWER
                                   (with sources)
```

### Step 1: Chunking

Documents are too long to embed as a single piece. You split them into chunks.

| Strategy | How It Works | When to Use |
|---|---|---|
| **Fixed size** | Every 500 characters, split | Simple, works for most cases |
| **By paragraph** | Split on blank lines | Documents with clear paragraph structure |
| **By section** | Split on headers (##, ###) | Markdown, HTML, structured docs |
| **Recursive** | Try paragraph first, then sentence, then character | LangChain default. Handles mixed formats. |

**The tradeoff:** Too small (50 chars) = lost context. Too large (5000 chars) = too much noise in retrieval. Most systems use 200-1000 characters per chunk.

### Step 2: Embedding + Storage

Each chunk gets converted to a vector and stored in a vector database.

**Common embedding models:**

| Model | Dimensions | Speed | Quality | Where It Runs |
|---|---|---|---|---|
| `nomic-embed-text` | 768 | Fast | Good | Local (Ollama) |
| `mxbai-embed-large` | 1024 | Medium | Better | Local (Ollama) |
| `text-embedding-3-small` | 1536 | Fast | Good | OpenAI API ($) |
| `text-embedding-3-large` | 3072 | Medium | Best | OpenAI API ($$) |

**Common vector databases:**

| Database | Type | Best For |
|---|---|---|
| **ChromaDB** | Local, in-process | Learning, prototypes, small datasets |
| **Pinecone** | Cloud managed | Production, no infrastructure to manage |
| **Weaviate** | Self-hosted or cloud | Production, need hybrid search |
| **pgvector** | PostgreSQL extension | Teams already using PostgreSQL |
| **FAISS** | In-memory library | Research, very large datasets, batch processing |

### Step 3: Retrieval

When a question comes in:
1. Convert the question to a vector (same embedding model)
2. Search the vector database for the most similar chunk vectors
3. Return the top-k results (usually k=3 to 5)

**Similarity measures:**

| Measure | How It Works | When to Use |
|---|---|---|
| **Cosine similarity** | Angle between vectors (ignores magnitude) | Most common. Default choice. |
| **Euclidean distance** | Straight-line distance between vectors | When magnitude matters |
| **Dot product** | Combines angle and magnitude | Normalized embeddings |

### Step 4: Generation

The retrieved chunks + the question go into a prompt template:

```
Based on the following context, answer the question.
If the answer is not in the context, say "I don't know."

Context:
{retrieved_chunk_1}
{retrieved_chunk_2}
{retrieved_chunk_3}

Question: {user_question}

Answer:
```

The LLM reads the context and generates an answer. The "if not in the context, say I don't know" instruction reduces hallucination.

---

## How RAG Is Different From Fine-Tuning

| Approach | What Changes | When to Use | Cost |
|---|---|---|---|
| **RAG** | The DATA the model sees (retrieved context) | Knowledge changes frequently. Company-specific data. | Low (no training) |
| **Fine-tuning** | The MODEL itself (retrained weights) | Need to change model behavior/style. Domain-specific language. | High (GPU hours) |
| **Both** | Fine-tuned model + RAG retrieval | Enterprise production systems | Highest |

**Most teams start with RAG.** Fine-tuning is for when RAG isn't enough (the model doesn't understand your domain's language even with context).

---

## Mental Model: The Library Analogy

Think of RAG as a library with a librarian:

| RAG Component | Library Equivalent |
|---|---|
| Documents | Books on the shelves |
| Chunks | Individual pages |
| Embeddings | The catalog card for each page (summarizes what it's about) |
| Vector database | The card catalog (organized by topic, not alphabetically) |
| Question | A patron's question |
| Retrieval | The librarian finding the right pages |
| LLM | The librarian reading the pages and giving you an answer |
| Prompt | The librarian's training: "Only answer from what you read, don't guess" |

The librarian (LLM) is smart but has not read every book. The retrieval system brings the right pages to the librarian's desk. The librarian reads them and answers.

---

## Quick Links

| Chapter | Topic |
|---|---|
| [01 - Why](01_Why.md) | Why RAG matters |
| [02 - Concepts](02_Concepts.md) | This page |
| [03 - Hello World](03_Hello_World.md) | Build a RAG system in 20 lines |
| [04 - How It Works](04_How_It_Works.md) | Deep dive into each step |
| [05 - Building It](05_Building_It.md) | Every tradeoff and choice |
| [06 - Production Patterns](06_Production_Patterns.md) | How production RAG systems work |
| [07 - System Design](07_System_Design.md) | Scaling, caching, hybrid search |
| [08 - Quality, Security, Governance](08_Quality_Security_Governance.md) | Prompt injection, data leakage |
| [09 - Observability & Troubleshooting](09_Observability_Troubleshooting.md) | Measuring quality and cost |
| [10 - Decision Guide](10_Decision_Guide.md) | Decision table and production readiness |
