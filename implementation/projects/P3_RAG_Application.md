# Project P3: RAG Application

**Week:** 8 (Context Engineering and RAG)
**Time estimate:** 10-15 hours
**Prerequisites:** RAG from Scratch notebook (all 29 sections), Deep Learning and PyTorch notebook, P1 (ML Predictor), P1.5 (Image Classifier)

---

## Objective

Build a **production-quality document Q&A system** using RAG (Retrieval-Augmented Generation). Your system will ingest real documents, store them in a vector database, retrieve relevant chunks, generate answers with source citations, and defend against prompt injection attacks.

This is not another tutorial. You already built a RAG system in the notebook (Section 6-27). This project adds what the notebook intentionally left out: **your own documents, automated evaluation, security, and production structure.**

P3 is also the foundation for P4 (Week 10), where you will swap repos with a partner and extend their code. Build it like someone else will read and modify it — because they will.

---

## The Dataset

**You choose your own documents.** The RAG system must work on a domain YOU care about. Options:

| Domain | Source | Why It Works |
|:---|:---|:---|
| Company documentation | Internal wikis, policy docs, handbooks | Real enterprise use case — this is what RAG is built for |
| Technical documentation | API docs, framework guides, man pages | High-precision Q&A with verifiable answers |
| Academic papers | ArXiv papers on a topic you are studying | Complex, dense content — tests chunking and retrieval quality |
| Legal / compliance | Publicly available regulations, terms of service | Long documents, specific questions, citation matters |
| Personal knowledge base | Your notes, blog posts, book summaries | Practical — you will actually use this after the program |

**Minimum requirements:**
- At least **5 documents** (PDF, Markdown, or text)
- At least **50 pages** of total content (not 5 one-page documents)
- Content you can verify — you need to know if the answers are correct

> **Do not use the Monopoly/Ticket to Ride PDFs from the tutorial repos.** Those are toy datasets. Choose something where wrong answers have consequences — that forces you to care about evaluation and guardrails.

---

## Requirements

### 1. Document Ingestion Pipeline
- [ ] Load documents from a `data/` directory (support at least 2 formats: PDF + Markdown, or PDF + text)
- [ ] Split into chunks using `RecursiveCharacterTextSplitter`
- [ ] Document your chunking strategy: what chunk size? What overlap? Why?
- [ ] Store chunks in **ChromaDB** (or another vector database — document your choice)
- [ ] Implement a `--reset` flag to clear and rebuild the database
- [ ] Print ingestion statistics: number of documents loaded, total chunks created, average chunk size

### 2. Embedding and Retrieval
- [ ] Use a local embedding model (Ollama `nomic-embed-text` or `mxbai-embed-large`, or sentence-transformers)
- [ ] Implement **similarity search** (baseline)
- [ ] Implement **MMR (Maximal Marginal Relevance)** search — reduces redundancy in retrieved chunks
- [ ] Make `top_k` (number of chunks retrieved) configurable
- [ ] Return source citations with every answer (which document, which page/section)

### 3. Query Pipeline
- [ ] Use a local LLM (Large Language Model) via Ollama (Mistral, Llama 3, or similar)
- [ ] Build a prompt template that includes:
  - System instruction ("Answer based only on the provided context")
  - Retrieved chunks as context
  - The user's question
- [ ] Support both **single query** mode (CLI argument) and **interactive** mode (REPL — Read-Evaluate-Print Loop)
- [ ] Handle the "no relevant context" case gracefully — the system should say "I don't know" instead of hallucinating

### 4. Evaluation with RAGAS
- [ ] Create a **test set** of at least **15 question-answer pairs** covering:
  - 5 straightforward questions (answer is directly in one chunk)
  - 5 multi-hop questions (answer requires combining information from 2+ chunks)
  - 5 unanswerable questions (answer is NOT in the documents)
- [ ] Run **RAGAS (RAG Assessment)** evaluation on your test set, measuring:
  - **Context Relevance** — are the retrieved chunks relevant to the question?
  - **Answer Faithfulness** — is the answer grounded in the retrieved context (not hallucinated)?
  - **Answer Relevance** — does the answer actually address the question?
- [ ] Report all 3 scores with interpretation: what do they mean? Where is the system weakest?
- [ ] If any score is below 0.7, diagnose the problem and document what you would fix

> **If RAGAS is difficult to set up** (it can be with local models), you may substitute with the LLM-as-judge pattern from the tutorial repos (Section 12 of the RAG notebook). Document your evaluation approach either way.

### 5. Security — Prompt Injection Defense
- [ ] Implement **input validation** — detect and reject obvious prompt injection attempts:
  - "Ignore all previous instructions"
  - "You are now a different AI"
  - System prompt extraction attempts ("What are your instructions?")
- [ ] Implement **output filtering** — check that the response does not contain sensitive content or system prompt leakage
- [ ] Create at least **5 prompt injection test cases** and show your defenses working
- [ ] Document your defense strategy: what do you catch? What could still get through?

> **This is not about building perfect security.** It is about demonstrating that you understand the attack surface. In production, you would use a framework like Guardrails AI, NeMo Guardrails, or LlamaGuard. For P3, hand-written checks are fine.

### 6. Code Quality
- [ ] **Modular structure** — separate files for ingestion, retrieval, generation, evaluation, and security
- [ ] All code has WHY comments (25%+ comment density)
- [ ] A reusable `query_rag()` function that the test suite calls
- [ ] Configuration via `.env` file or command-line arguments (not hardcoded model names, chunk sizes, etc.)
- [ ] A `requirements.txt` with pinned versions
- [ ] A `.gitignore` (exclude `__pycache__/`, `.venv/`, `chroma/`, large data files)

> **Build it like someone else will modify it.** In P4, a classmate will receive your repo, read your code, and extend it. If your code is a single 500-line script with no comments, they will have a bad time — and your P4 peer review score will reflect it.

### 7. README
- [ ] Problem statement: what domain, what documents, what questions it answers
- [ ] Architecture diagram (text-based is fine):
  ```
  Documents → Loader → Chunker → Embedder → ChromaDB
                                                ↓
  User Question → Embedder → Similarity Search → Top-K Chunks → Prompt Template → LLM → Answer + Sources
  ```
- [ ] Setup instructions (clone, install, pull Ollama models, ingest, query)
- [ ] Example queries and expected answers
- [ ] Evaluation results (RAGAS scores or LLM-as-judge results)
- [ ] Known limitations

### 8. Reflection Journal
Answer in a separate `REFLECTION.md`:

1. What chunking strategy did you choose and why? What would you change if you had 10x more data?
2. What was the hardest question for your system to answer correctly? Why?
3. What would break if you deployed this to 100 concurrent users tomorrow?
4. How did RAGAS (or your evaluation method) change your understanding of the system's quality?
5. What architectural decisions did you make that a partner might disagree with in P4?

---

## Step-by-Step Guide

### Step 1: Set up your project
```bash
mkdir p3-rag-application && cd p3-rag-application
python -m venv .venv
source .venv/bin/activate
pip install langchain langchain-community chromadb pypdf sentence-transformers ragas matplotlib jupyter python-dotenv
pip freeze > requirements.txt
```

### Step 2: Gather your documents
Collect at least 5 documents (50+ pages total). Place them in a `data/` directory. If using proprietary data, ensure you have permission.

### Step 3: Build the ingestion pipeline
Start with the pattern from the RAG notebook (Sections 7-10). Extract it into a standalone script (`ingest.py` or `populate_database.py`). Add the `--reset` flag and ingestion statistics.

### Step 4: Build the query pipeline
Start with the pattern from the RAG notebook (Section 11). Extract it into a standalone script (`query.py`). Add source citations and the "no relevant context" handler.

### Step 5: Write your test set
This is the hardest part. You need 15 question-answer pairs where you KNOW the correct answer. Write them by reading your own documents. Include the 5 unanswerable questions — these test whether the system hallucinates.

### Step 6: Run RAGAS evaluation
Install RAGAS and run your test set. Record the scores. If a score is below 0.7, investigate: is it a retrieval problem (wrong chunks) or a generation problem (right chunks, wrong answer)?

### Step 7: Add prompt injection defense
Add input validation to your query pipeline. Test it with the 5 injection attempts. Document what you catch and what you do not.

### Step 8: Refactor for modularity
If your code is still in one file, split it. Your partner in P4 will thank you.

### Step 9: Write README, reflection, push to GitHub
```bash
git init && git add . && git commit -m "P3: RAG document Q&A with RAGAS evaluation and prompt injection defense"
gh repo create p3-rag-application --public --push
```

---

## Grading Rubric

| Category | Points | What We Look For |
|:---|:---|:---|
| Document ingestion pipeline (loading, chunking, storage, reset, stats) | 15 | Works on your documents, chunking strategy documented, reset flag |
| Retrieval (similarity + MMR, configurable top_k, source citations) | 15 | Both search modes work, citations point to correct sources |
| Query pipeline (prompt template, interactive mode, graceful "I don't know") | 10 | Clean prompt, handles out-of-scope questions |
| RAGAS evaluation (15 test questions, 3 scores, interpretation) | 20 | Diverse test set (easy + multi-hop + unanswerable), scores reported and interpreted |
| Prompt injection defense (5 test cases, input + output filtering) | 10 | At least basic injection detection, documented limitations |
| Code quality (modular, WHY comments, config via .env, requirements.txt) | 15 | Partner-ready code — someone else can read, run, and modify it |
| README + Reflection journal | 15 | Architecture diagram, setup instructions, honest reflection on limitations |
| **Total** | **100** | |

> **Evaluation is worth 20 points — the most of any category.** In production, a RAG system without evaluation is a liability. You do not know if it works. You do not know when it breaks. You do not know what to fix. The test set is the single most valuable artifact you produce in this project.

---

## The Interview Story

> "I built a RAG system for [your domain] using LangChain, ChromaDB, and a local LLM via Ollama. The corpus was [X] documents, [Y] pages total. I evaluated it with a 15-question test set covering factual lookups, multi-hop reasoning, and unanswerable questions. My RAGAS scores were [context relevance: X, faithfulness: Y, answer relevance: Z]. The weakest area was [X] because [reason] — the retrieval was pulling [problem]. I also added prompt injection defense — basic input validation that catches 'ignore previous instructions' patterns and system prompt extraction attempts. If I were taking this to production, the first thing I would add is [guardrails framework / reranking / hybrid search / authentication] because [reason]. My classmate extended this system in P4 and added [what they built]."

---

## Reference: External Projects to Study First

Before building your own, study how others approached RAG:

| Repo | What to Study |
|:---|:---|
| [pixegami/rag-tutorial-v2](https://github.com/pixegami/rag-tutorial-v2) | The architecture pattern (ingest → query → test). Your P3 extends this with evaluation, security, and real documents. |
| [sbj1198/local-llm-rag-chromadb](https://github.com/sbj1198/local-llm-rag-chromadb) | The `.env` config pattern and standalone vector search script. Good model for modularity. |

A hands-on guide for cloning and running these repos is available at: [RAG Projects Setup Guide](https://github.com/sunilmogadati/systems-in-production/blob/main/implementation/guides/RAG_Projects_Setup_Guide.md)

> **The tutorial repos are your starting point, not your ceiling.** They have no evaluation, no security, no modularity. P3 adds all of those — and uses your own documents, not board game rulebooks.

---

## How P3 Connects to What Comes Next

| What You Build Here | Where It Goes |
|:---|:---|
| Modular, well-documented code | **P4 (Week 10):** A classmate will clone your P3 and extend it. They will read every line. |
| RAGAS evaluation test set | **P4:** Your partner runs the same test set after their changes. Scores should improve, not regress. |
| ChromaDB + LangChain pipeline | **P4:** Partner adds reranking, hybrid search, or query classification on top of your pipeline |
| Prompt injection defense | **P5 (Week 12):** Enterprise integration adds PII (Personally Identifiable Information) redaction, RBAC (Role-Based Access Control), and formal security assessment |
| Reflection on architectural decisions | **P4:** Partner documents what they would have done differently — your decisions become their constraints |
| The entire RAG system | **P6 (Week 16):** Capstone builds on the same platform — adds agents, voice/chat UI, and 1-week production operation |

---

## Deliverables

Push to GitHub with this structure:

```
p3-rag-application/
  README.md
  REFLECTION.md
  requirements.txt
  .env.example                # Template (no real keys) — document every variable
  .gitignore
  data/                       # Your documents (or instructions to obtain them)
  src/
    ingest.py                 # Document loading, chunking, vector DB storage
    query.py                  # Retrieval + generation pipeline
    security.py               # Prompt injection detection, output filtering
    evaluate.py               # RAGAS evaluation runner
  tests/
    test_questions.json       # Your 15 question-answer pairs
    test_injection.py         # Prompt injection test cases
  results/
    evaluation_report.md      # RAGAS scores + interpretation
```

> **Why this structure?** Each file has one job. Your partner in P4 can find the retrieval logic without reading the ingestion code. The test set is separate from the code so it can be run independently. The `.env.example` tells them what configuration they need without exposing your actual settings.
