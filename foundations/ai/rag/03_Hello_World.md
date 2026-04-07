# RAG - Hello World

**Build a working RAG system. Ingest a document, ask a question, get an answer from the document. See it work before understanding why it works.**

---

## What You Need

- Python 3.8+
- Ollama installed with `mistral` and `nomic-embed-text` models
- ChromaDB and LangChain packages

```bash
# Install Ollama (if not already)
# https://ollama.com/download

# Pull models
ollama pull mistral
ollama pull nomic-embed-text

# Install Python packages
pip install langchain langchain-ollama langchain-community chromadb
```

---

## The Hello World (20 Lines)

```python
from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_ollama import OllamaEmbeddings, OllamaLLM

# Step 1: Load a document
loader = TextLoader("my_document.txt")
docs = loader.load()

# Step 2: Split into chunks
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# Step 3: Embed and store in vector database
db = Chroma.from_documents(chunks, OllamaEmbeddings(model="nomic-embed-text"))

# Step 4: Ask a question
question = "What is the main topic of this document?"
relevant_chunks = db.similarity_search(question, k=3)

# Step 5: Generate answer
context = "\n\n".join([chunk.page_content for chunk in relevant_chunks])
prompt = f"Based on this context, answer the question.\n\nContext:\n{context}\n\nQuestion: {question}\nAnswer:"
answer = OllamaLLM(model="mistral").invoke(prompt)

print(f"Question: {question}")
print(f"Answer: {answer}")
print(f"\nSources used: {len(relevant_chunks)} chunks")
```

**You Should See:** An answer that comes FROM the document, not from the model's training data.

---

## What Just Happened?

```
my_document.txt
      |
      v
[Load] → full text
      |
      v
[Split] → chunks (each ~500 chars)
      |
      v
[Embed] → vectors (numbers representing meaning)
      |
      v
[Store] → ChromaDB (vector database)
      |
      v
[Question] → "What is the main topic?"
      |
      v
[Retrieve] → 3 most relevant chunks
      |
      v
[Generate] → LLM reads chunks + question → answer
```

Five steps. The document went from a file on disk to a searchable knowledge base that an AI can query.

---

## Try It Yourself: Create a Test Document

Create a file called `my_document.txt` with any content. For example:

```
The order-service experienced a database connection pool exhaustion 
on March 8, 2026. The root cause was a missing connection_max_lifetime 
setting in the ORM configuration. When traffic increased during peak 
hours, all 250 connections were consumed and not released. The fix was 
to set connection_max_lifetime=300s, which forces connections to be 
recycled every 5 minutes. A temporary fix of restarting pods was applied 
immediately, and the permanent fix was deployed the next day.

This same issue had occurred on March 5 as a P3 incident but was 
misclassified and not investigated deeply. When it recurred on March 8 
as a P1 outage, the team realized the earlier signal had been missed.
```

Now run the hello world code and ask:
- "What caused the outage on March 8?"
- "What was the fix?"
- "Was there an earlier warning?"

The answers should come from the document, not from the model's general knowledge.

---

## The Test That Proves RAG Is Working

Ask a question that the model could NOT answer from training data alone:

```python
# This question is about YOUR specific document
question = "What was the connection_max_lifetime setting?"

# Without RAG (model answers from memory):
answer_no_rag = OllamaLLM(model="mistral").invoke(question)
print(f"Without RAG: {answer_no_rag}")
# Likely: generic answer or "I don't know"

# With RAG (model answers from document):
chunks = db.similarity_search(question, k=3)
context = "\n\n".join([c.page_content for c in chunks])
prompt = f"Context:\n{context}\n\nQuestion: {question}\nAnswer:"
answer_with_rag = OllamaLLM(model="mistral").invoke(prompt)
print(f"With RAG: {answer_with_rag}")
# Should: "300 seconds (5 minutes)" — from the document
```

**This is the proof.** Without RAG, the model guesses. With RAG, the model answers from your data.

---

## Common First-Time Issues

| Problem | Cause | Fix |
|---|---|---|
| "Ollama not running" error | Ollama server not started | Run `ollama serve` in a terminal |
| Empty or wrong answers | Chunks too large or too small | Try chunk_size=300 or chunk_size=800 |
| "Model not found" error | Model not pulled | Run `ollama pull mistral` and `ollama pull nomic-embed-text` |
| Slow first query | Embedding model loading | First query takes 10-30 seconds. Subsequent queries are fast. |
| Answer ignores the document | Prompt doesn't instruct to use context | Add "Only answer from the context provided" to prompt |

---

## What's Next

You just built a working RAG system. The next chapters explain:
- **How it works:** Why embeddings capture meaning. How similarity search finds relevant chunks.
- **Decisions:** Chunk size, overlap, embedding model, k value, prompt design — every choice and its tradeoff.
- **Real world:** How this scales from a single document to millions.

---

## Quick Links

| Chapter | Topic |
|---|---|
| [01 - Why](01_Why.md) | Why RAG matters |
| [02 - Concepts](02_Concepts.md) | Embeddings, vectors, chunking |
| [03 - Hello World](03_Hello_World.md) | This page |
| [04 - How It Works](04_How_It_Works.md) | Deep dive into each step |
| [05 - Decisions](05_Decisions.md) | Every tradeoff and choice |

**Full hands-on notebook:** [RAG from Scratch on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_RAG_from_Scratch.ipynb)
