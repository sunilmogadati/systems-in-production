# Running RAG Projects from GitHub — Hands-On Guide

**Purpose:** Clone, run, and test real RAG projects so you see how a working system behaves before building your own.

**Prerequisites:**
- Python 3.9+ installed (`python3 --version`)
- Git installed (`git --version`)
- Ollama installed ([ollama.com/download](https://ollama.com/download)) — for Projects 1, 2, and 4
- 8+ GB RAM recommended for local LLM inference
- Completed the [RAG from Scratch notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/RAG_from_Scratch.ipynb) (Section 29 references these projects)

**Time estimate:** ~30-45 minutes for Project 1 (the primary one). Budget extra time for model downloads on first run.

---

## Project 1: pixegami/rag-tutorial-v2 (Start Here)

**Why this one first:** Our RAG notebook is based on this architecture. Same pattern — load PDFs, chunk, embed, store in ChromaDB, query with an LLM. Plus it has automated tests.

**Stack:** LangChain + ChromaDB + Ollama (Mistral)
**Cost:** Free (runs entirely on your machine)

### Step 1: Clone the repo

```bash
cd ~/projects    # or wherever you keep code
git clone https://github.com/pixegami/rag-tutorial-v2.git
cd rag-tutorial-v2
```

### Step 2: Look at the project structure

```bash
ls -la
```

You should see:

```
README.md
requirements.txt
data/                          # PDF documents (board game rulebooks)
populate_database.py           # Load → chunk → embed → store
query_data.py                  # Ask questions → retrieve → generate
get_embedding_function.py      # Embedding model configuration
test_rag.py                    # Automated tests (LLM-as-judge)
```

> **Mental model:** Map each file to what you built in the notebook.
> `populate_database.py` = our `load_documents()` + `split_into_chunks()` + `store_chunks()`.
> `query_data.py` = our `query_rag()`.
> `test_rag.py` = our `test_query()` with the LLM-as-judge pattern.

### Step 3: Create a virtual environment and install dependencies

```bash
python3 -m venv venv
source venv/bin/activate       # On Windows: venv\Scripts\activate
pip install -r requirements.txt
pip install langchain-community
```

> **Why the extra install?** The repo's `requirements.txt` lists `langchain` but not `langchain-community`, which contains the Ollama and ChromaDB integrations. Without it, you will get `ModuleNotFoundError` when you try to run.

### Step 4: Fix LangChain import changes (required)

This repo was written for an older version of LangChain. The library has since reorganized its modules. You need to update 3 files or the scripts will crash with `ModuleNotFoundError`.

> **Why this happens:** LangChain moved classes from `langchain.*` to `langchain_community.*` and `langchain_core.*` in 2024. Open source projects on GitHub don't always keep up. Knowing how to fix import breakage in someone else's code is a real-world skill.

**File 1: `get_embedding_function.py`** — Replace the entire file contents with:

```python
from langchain_community.embeddings.ollama import OllamaEmbeddings


def get_embedding_function():
    embeddings = OllamaEmbeddings(model="nomic-embed-text")
    return embeddings
```

This does two things: switches from AWS Bedrock to local Ollama embeddings, and uses the correct import path.

**File 2: `populate_database.py`** — Find the imports at the top and replace them:

```python
# BEFORE (broken with newer LangChain):
from langchain.document_loaders.pdf import PyPDFDirectoryLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain.schema.document import Document
from get_embedding_function import get_embedding_function
from langchain.vectorstores.chroma import Chroma

# AFTER (working imports):
from langchain_community.document_loaders import PyPDFDirectoryLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document
from get_embedding_function import get_embedding_function
from langchain_community.vectorstores import Chroma
```

**File 3: `query_data.py`** — Find the imports at the top and replace them:

```python
# BEFORE (broken with newer LangChain):
from langchain.vectorstores.chroma import Chroma
from langchain.prompts import ChatPromptTemplate
from langchain_community.llms.ollama import Ollama

# AFTER (working imports):
from langchain_community.vectorstores import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_community.llms.ollama import Ollama
```

> **No changes needed in `test_rag.py`** — it imports from `query_data.py`, so fixing that file covers the tests too.

After saving all 3 files, you are ready to proceed.

### Step 5: Pull the Ollama models (free, local)

```bash
ollama pull nomic-embed-text    # Embedding model (~275 MB)
ollama pull mistral             # Chat model (~4.1 GB)
```

> **First time?** These downloads are one-time. After this, the models load in seconds.

Verify they are available:

```bash
ollama list
```

You should see both `nomic-embed-text` and `mistral` in the list.

### Step 6: Start Ollama (if not already running)

```bash
ollama serve
```

> If you see `Error: listen tcp 127.0.0.1:11434: bind: address already in use`, Ollama is already running. That is fine — move on.

Open a **new terminal tab** for the next steps (keep Ollama running in this one). Or if Ollama was already running, stay in your current terminal.

### Step 7: Look at the data

```bash
ls data/
```

You should see PDF files (board game rulebooks like Monopoly, Ticket to Ride). Open one to see what the RAG system will answer questions about.

### Step 8: Populate the database (ingest the PDFs)

```bash
python populate_database.py --reset
```

> The `--reset` flag clears any existing database before loading. Safe to use on first run too.

**You should see** output like:

```
✨ Clearing Database
Number of existing documents in DB: 0
👉 Adding new documents: 40
```

You will also see deprecation warnings about `OllamaEmbeddings` and `Chroma` classes. **These are cosmetic — the code works fine.** The warnings mean LangChain has newer versions of these classes in separate packages (`langchain-ollama`, `langchain-chroma`). Ignore them for now.

> **What just happened:** The script loaded both PDFs from `data/` (Monopoly and Ticket to Ride rulebooks), split them into 40 chunks using `RecursiveCharacterTextSplitter` with chunk_size=800, generated embeddings with `nomic-embed-text`, and stored everything in a local ChromaDB database.

Verify the database was created:

```bash
ls chroma/
```

### Step 9: Ask a question

```bash
python query_data.py "How much total money does a player start with in Monopoly?"
```

**You should see** something like:

```
Response: A player starts with $2500 in Monopoly.
Sources: ['data/monopoly.pdf:2:0', 'data/monopoly.pdf:0:1', 'data/monopoly.pdf:1:2', ...]
```

Two things to notice:
1. **Sources** — the system tells you exactly which PDF pages it retrieved (e.g., `monopoly.pdf:2:0` = page 2, chunk 0). This is the retrieval step.
2. **The answer may be wrong** — the correct Monopoly answer is $1,500, but the LLM might say $2,500. The retrieval found the right pages, but the LLM misread the context. This is a **generation problem, not a retrieval problem** — exactly the debugging pattern from Section 13 of our notebook.

> **Try these too:**
> ```bash
> python query_data.py "How do you win at Ticket to Ride?"
> python query_data.py "How many points does the longest continuous train get in Ticket to Ride?"
> python query_data.py "What happens when you land on Free Parking?"
> ```

### Step 10: Run the automated tests

```bash
pytest test_rag.py -v
```

**You should see** something like:

```
test_rag.py::test_monopoly_rules FAILED                                  [ 50%]
test_rag.py::test_ticket_to_ride_rules PASSED                            [100%]

FAILED test_rag.py::test_monopoly_rules - AssertionError: assert False
=================== 1 failed, 1 passed in 8.59s ====================
```

**Yes, a test failure is expected.** This is a teaching moment, not a bug in your setup:

- **test_monopoly_rules FAILED** — The test asks "How much money does a player start with?" and expects "$1,500". But Mistral answered "$2,500". The LLM-as-judge (a second Mistral call) compared the two and correctly returned `false`. The retrieval worked (right PDF pages), but the LLM hallucinated the dollar amount. This is a **generation failure** — the exact debugging scenario from Section 13 of our notebook.

- **test_ticket_to_ride_rules PASSED** — The test asks about longest train points, expects "10 points", and the LLM answered correctly. The judge confirmed `true`.

> **What just happened:** Each test asks a question, gets the RAG answer, then uses Mistral as a **judge** to verify the answer matches the expected response. This is the LLM-as-judge pattern from Section 12 of our notebook. Two LLM calls per test: one to answer, one to evaluate.

Open `test_rag.py` and read it — it is short and clean (under 50 lines). Notice how each test has:
- A question
- An expected answer (the "ground truth")
- An LLM call that judges whether the RAG output matches

> **Think about it:** The Monopoly test failure shows why evaluations matter. If you only ran the query manually and saw "$2,500", you might not know it was wrong unless you checked the rulebook yourself. The automated test caught it. In production, this is the difference between shipping a hallucination and catching it before users see it.

### Step 11: Break it (learn by experimenting)

Try these experiments to build intuition:

**Experiment A — Change chunk size:**
Open `populate_database.py`, find the `RecursiveCharacterTextSplitter`, and change `chunk_size` from 800 to 200. Then:
```bash
python populate_database.py --reset    # --reset clears the old database
python query_data.py "What are the rules for buying property in Monopoly?"
```
Did the answer get worse? Better? Why?

**Experiment B — Ask an out-of-scope question:**
```bash
python query_data.py "What is the capital of France?"
```
The system retrieves 5 Monopoly chunks (it has no choice — there is no relevance threshold in the code). But instead of hallucinating, Mistral responds: *"The provided text does not contain any information about the capital of France."*

Why? Look at the prompt template in `query_data.py`. The key phrase is **"Answer the question based only on the following context."** That word "only" is doing the heavy lifting — it is a **prompt-level guardrail**. The LLM was smart enough to recognize the context was irrelevant and refuse to guess.

This is not guaranteed with every model or question. A weaker model might ignore the instruction and hallucinate anyway. For production, you would add a **code-level guardrail** too — check the similarity score and reject low-confidence retrievals before the LLM ever sees them.

**Experiment C — Swap the LLM model:**
Open `query_data.py`, find where it creates the Ollama LLM, and change `mistral` to `llama3`. Then:
```bash
ollama pull llama3              # Download if you haven't
python query_data.py "How do you win at Ticket to Ride?"
```
Compare the answer quality. Different models, same retrieval — the generation step matters.

### Step 12: Clean up (optional)

```bash
deactivate                      # Exit the virtual environment
cd ..
# rm -rf rag-tutorial-v2        # Only if you want to delete it entirely
```

### What You Learned

| Concept | Where You Saw It |
|---------|-----------------|
| Document loading | `populate_database.py` — `PyPDFDirectoryLoader` reads every PDF in `data/` |
| Chunking | `populate_database.py` — `RecursiveCharacterTextSplitter` (800 chars, 80 overlap) |
| Embeddings | `get_embedding_function.py` — Ollama `nomic-embed-text` (local, free) |
| Vector store | ChromaDB persisted to `chroma/` directory (40 chunks from 2 PDFs) |
| Retrieval + generation | `query_data.py` — similarity search (top 5 chunks) → LLM prompt → answer with sources |
| Automated testing | `test_rag.py` — LLM-as-judge pattern (Mistral evaluates its own answers) |
| Database reset | `--reset` flag clears and rebuilds from scratch |
| Import breakage | LangChain reorganized its modules — you fixed 3 files. This happens with fast-moving libraries. |
| Generation vs retrieval failures | Monopoly test: retrieval found the right pages, but the LLM hallucinated the dollar amount. Retrieval was correct; generation was wrong. |
| Why evaluations matter | The automated test caught a hallucination ($2,500 instead of $1,500) that a manual spot-check might miss. |

---

## Project 2: sbj1198/local-llm-rag-chromadb (Zero-Cost Local RAG)

**Why this one:** Fully local, zero API keys, very similar to our notebook setup. Good for cross-referencing your own code.

**Stack:** Ollama (llama3:8b + mxbai-embed-large) + LangChain + ChromaDB
**Cost:** Free

### Step 1: Clone and set up

```bash
cd ~/projects
git clone https://github.com/sbj1198/local-llm-rag-chromadb.git
cd local-llm-rag-chromadb

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

> **Good news:** Unlike Project 1, this repo already uses the newer `langchain_community` and `langchain_core` imports. No import fixes needed — the code runs as-is.

### Step 2: Pull the Ollama models

If you already ran Project 1, you have `mistral`. You can use that instead of downloading `llama3:8b` (saves ~4.7 GB). Otherwise:

```bash
ollama pull llama3:8b            # Chat model (~4.7 GB) — or use mistral if you have it
ollama pull mxbai-embed-large    # Embedding model (~670 MB)
```

### Step 3: Create the .env file

```bash
cat > .env << 'EOF'
EMBEDDINGS_MODEL=mxbai-embed-large
CHAT_MODEL=mistral
CHROMA_DIR=./chroma_data
COLLECTION_NAME=docs
TOP_K=5
EOF
```

> Change `CHAT_MODEL=mistral` to `CHAT_MODEL=llama3:8b` if you want to use the repo's original model.

### Step 4: Fix the data URLs (required)

Open `load_data.py` and look at the `urls` list. The default URLs point to Microsoft Azure documentation that has since moved — they now return **404: Not Found**. If you run the script without fixing this, it will happily load "404: Not Found" as your document content. The script will appear to succeed, but your queries will return nonsense.

> **This is a common problem with open source tutorials.** External URLs break over time. Always verify your data actually loaded real content, not error pages.

Replace the URLs in `load_data.py` with working ones. For example, to load the Ollama and ChromaDB documentation (relevant to what you are learning):

```python
urls = [
    "https://raw.githubusercontent.com/ollama/ollama/main/README.md",
    "https://raw.githubusercontent.com/chroma-core/chroma/main/README.md",
]
```

Or use any Markdown/HTML URLs you want — the loader supports web pages via `WebBaseLoader`.

**Also fix the chunk size.** In the same file, find:

```python
markdown_splitter = MarkdownTextSplitter(chunk_size=1500, chunk_overlap=200)
```

Change it to:

```python
markdown_splitter = MarkdownTextSplitter(chunk_size=500, chunk_overlap=50)
```

> **Why:** The `mxbai-embed-large` embedding model has a limited context window. Chunks of 1500 characters can exceed it, causing an error: `"the input length exceeds the context length"`. Reducing to 500 keeps chunks within the model's limit. This is the kind of mismatch you will hit in production — your chunk size must fit your embedding model's context window.

### Step 5: Load the data

```bash
python3 load_data.py
```

**You should see** something like:

```
📥 Downloading documents from ['https://raw.githubusercontent.com/ollama/ollama/main/README.md', ...]
✅ Loaded 66 chunks from 2 docs into Chroma at './chroma_data'
```

You will see deprecation warnings (same as Project 1). Ignore them.

Verify:
```bash
ls chroma_data/
```

> **Sanity check:** If the directory exists but your queries later return garbage answers, the data source may be broken. Delete `chroma_data/` and reload with verified URLs. If you get `"the input length exceeds the context length"`, your chunks are too large for the embedding model — reduce `chunk_size` in `load_data.py`.

### Step 6: Ask questions

```bash
python3 rag_chain.py
```

This starts an interactive Q&A session. Type a question and press Enter. Type `quit` or `exit` to stop.

Try questions that match the documents you loaded. If you used the Ollama and ChromaDB READMEs:
```
> What models does Ollama support?
> How do I install ChromaDB?
> What is the default embedding function in Chroma?
```

### Step 7: Test similarity search standalone

```bash
python3 vector_search.py
```

> **Why this is useful:** This lets you see the retrieval step in isolation — what chunks come back for a query, without the LLM generation step. When debugging RAG, always check retrieval first (Section 13 of our notebook).
>
> The `vector_search.py` script shows you the raw similarity scores. If these scores are low or the chunks look irrelevant, the problem is in **retrieval** (wrong embeddings, bad chunking, missing data). If scores are high and chunks look good but the final answer is wrong, the problem is in **generation** (the LLM is misinterpreting good context).

### What's Different from Project 1

| Aspect | pixegami/rag-tutorial-v2 | sbj1198/local-llm-rag-chromadb |
|--------|-------------------------|-------------------------------|
| LLM | Mistral | llama3:8b |
| Embeddings | nomic-embed-text | mxbai-embed-large |
| Config | Hardcoded in scripts | `.env` file (cleaner) |
| Interface | Single query via CLI args | Interactive REPL (Read-Evaluate-Print Loop) |
| Tests | Yes (pytest + LLM-judge) | No |
| Standalone search | No | Yes (`vector_search.py`) |

---

## Project 3: Faridghr/Simple-RAG-Chatbot (Web UI with Streamlit)

**Why this one:** Shows how to put a web interface on a RAG system. This is the "next step" after a CLI-only project — the kind of demo that impresses in interviews.

**Stack:** LangChain + ChromaDB + Streamlit + OpenAI + Pinecone
**Cost:** Requires OpenAI API key (paid) + Pinecone API key (free tier available)

> **Heads up — paid API keys required.** This project cannot run without an OpenAI API key ($5+ minimum credit) and a Pinecone API key (free tier works). It also has outdated LangChain imports (same issue as Project 1) and a hardcoded OpenAI org ID in the source that you would need to remove. If you do not have these keys, **read the code without running it** — the Streamlit UI patterns and the RAG/UI separation architecture are the valuable lessons here, not the execution.

### Step 1: Clone and set up

```bash
cd ~/projects
git clone https://github.com/Faridghr/Simple-RAG-Chatbot.git
cd Simple-RAG-Chatbot

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Step 2: Get your API keys

**OpenAI:** Go to [platform.openai.com/api-keys](https://platform.openai.com/api-keys), create a key. You need credits on your account.

**Pinecone:** Go to [app.pinecone.io](https://app.pinecone.io), sign up (free tier works), get your API key from the dashboard.

### Step 3: Create the .env file

```bash
cat > .env << 'EOF'
OPENAI_API_KEY=sk-your-key-here
PINECONE_API_KEY=your-pinecone-key-here
EOF
```

### Step 4: Add PDF documents

Place your PDF files in `src/materials/`. The chatbot will answer questions from these documents.

### Step 5: Run the Streamlit app

```bash
cd src
streamlit run streamlitMain.py
```

**You should see:**

```
You can now view your Streamlit app in your browser.
Local URL: http://localhost:8501
```

Open that URL in your browser. You will see a chat interface. Upload a PDF or use the pre-loaded ones. Ask a question in the chat box.

### Step 6: Study the architecture

While the app runs, read these files:

1. `src/RAG_ChatBot.py` — The RAG pipeline (load → embed → store → retrieve → generate)
2. `src/streamlitMain.py` — The Streamlit UI layer

> **Key observation:** The RAG logic (`RAG_ChatBot.py`) is completely separate from the UI (`streamlitMain.py`). This separation is a production pattern — you can swap Streamlit for FastAPI without touching the RAG code.

### What's Different from Projects 1-2

| Aspect | Projects 1-2 | This project |
|--------|-------------|-------------|
| Interface | Terminal (Command Line Interface) | Web browser (Streamlit) |
| Vector DB | ChromaDB (local files) | Pinecone (cloud-hosted) |
| LLM | Local (Ollama) | Cloud (OpenAI gpt-3.5-turbo) |
| Embeddings | Local (Ollama/sentence-transformers) | Local (HuggingFace sentence-transformers) |
| Cost | Free | OpenAI API charges per query |

> **Interview angle:** "I compared local versus cloud RAG stacks. Local (Ollama + ChromaDB) is free and private but limited by your hardware. Cloud (OpenAI + Pinecone) scales better but costs money and sends data off-machine. The right choice depends on the use case — healthcare data stays local, customer support scales to the cloud."

---

## Project 4: pixegami/langchain-rag-tutorial (OpenAI Version)

**Why this one:** The v1 of Project 1 — same author, same architecture, but uses OpenAI instead of Ollama. Good for comparing local versus cloud LLM approaches side by side.

**Stack:** LangChain + ChromaDB + OpenAI
**Cost:** Requires OpenAI API key (paid)

> **Heads up — paid API key required.** This project requires an OpenAI API key for both embeddings and chat. If you do not have one, **skip this entirely** — you already ran Project 1 (the improved v2 by the same author). The main value of this project is comparing local versus cloud LLM latency and answer quality. You can understand that difference conceptually without spending money.

### Step 1: Clone and set up

```bash
cd ~/projects
git clone https://github.com/pixegami/langchain-rag-tutorial.git
cd langchain-rag-tutorial

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install "unstructured[md]"
```

> **macOS note:** If you get errors with `onnxruntime`, try: `pip install onnxruntime` separately. On Apple Silicon, you may need: `conda install onnxruntime -c conda-forge`.

### Step 2: Set your OpenAI API key

```bash
export OPENAI_API_KEY="sk-your-key-here"
```

### Step 3: Create the database

```bash
python create_database.py
```

**You should see** output about loading Markdown documents (Alice in Wonderland text), splitting into chunks, and storing in ChromaDB.

### Step 4: Query

```bash
python query_data.py "How does Alice meet the Mad Hatter?"
```

**You should see** an answer sourced from the Alice in Wonderland text, with source document references.

### Step 5: Compare with Project 1

Ask the same question through both systems (on their respective documents) and compare:
- Response latency (local Ollama vs OpenAI API)
- Answer quality
- Code structure differences

---

## Project 5: umbertogriffo/rag-chatbot (Production Reference)

**Why this one:** Full-stack RAG application — FastAPI backend, React frontend, multiple LLM backends, pytest suite, Poetry dependency management. This is what a production RAG system looks like.

**Stack:** llama-cpp-python + sentence-transformers + ChromaDB + FastAPI + React
**Cost:** Free (local models via llama-cpp-python)

> **Difficulty: High.** This project has strict requirements that will block most setups:
> - **Python 3.12 only** — `pyproject.toml` says `>=3.12,<3.13`. If you have 3.11 or 3.13+, Poetry will refuse to install. You would need `pyenv` to install 3.12 alongside your system Python.
> - **Poetry 2.x** — not pip. Different dependency manager, different workflow.
> - **llama-cpp-python** — compiles C++ code with Metal (macOS) or CUDA (NVIDIA) flags. This is the step most likely to fail on student machines (missing compilers, wrong SDK version).
> - **Node.js 22+ and Yarn** — for the React frontend.
>
> **Recommendation:** Read the code structure and Makefile to understand what production looks like. Only attempt to run it if you already have Python 3.12, Poetry, and Node.js installed and are comfortable troubleshooting C++ compilation errors.

### Step 1: Check prerequisites

```bash
python3 --version               # Must be 3.12.x
node --version                  # Must be 22.x+
yarn --version                  # Must be 1.22+
poetry --version                # Must be 2.x+ (install: pip install poetry)
```

If any of these are missing, install them before continuing.

### Step 2: Clone and set up

```bash
cd ~/projects
git clone https://github.com/umbertogriffo/rag-chatbot.git
cd rag-chatbot
cp .env.example .env
```

Edit `.env` to configure the model (default `llama-3.2` is a good starting point).

### Step 3: Install dependencies

**macOS (Apple Silicon):**
```bash
make setup_metal
```

**Linux/Windows with NVIDIA GPU:**
```bash
make setup_cuda
```

> This installs all Python dependencies via Poetry and downloads the selected model. It may take several minutes.

### Step 4: Prepare documents

Place your Markdown files in the `docs/` directory. Then build the vector database:

```bash
python chatbot/memory_builder.py --chunk-size 1000 --chunk-overlap 50
```

### Step 5: Run the backend

```bash
cd backend
PYTHONPATH=.:../chatbot uvicorn main:app --reload
```

The API is now running at `http://localhost:8000`. Test it:

```bash
curl http://localhost:8000/health
```

### Step 6: Run the frontend

In a new terminal:

```bash
cd frontend
yarn install
yarn dev
```

Open `http://localhost:5173` in your browser. You should see a chat interface.

### Step 7: Run the tests

```bash
make test
```

### What Makes This "Production Grade"

| Feature | Tutorial Projects (1-4) | This Project |
|---------|------------------------|-------------|
| Dependency management | `requirements.txt` | Poetry with lockfile |
| API | None (scripts) | FastAPI with health checks |
| Frontend | None or Streamlit | React + Vite |
| Tests | None or basic | pytest with mocks and coverage |
| Model support | 1 model hardcoded | 14+ models configurable via `.env` |
| Code quality | Scripts | Linting (ruff), pre-commit hooks |
| Build system | Manual pip install | Makefile with targets for each platform |

> **You do not need to build at this level for your P3 project.** But knowing what production looks like helps you make better architectural decisions — and gives you something to reference in interviews when they ask "how would you take this to production?"

---

## Troubleshooting

### Ollama Issues

| Problem | Solution |
|---------|---------|
| `ollama: command not found` | Install from [ollama.com/download](https://ollama.com/download) |
| `Error: listen tcp 127.0.0.1:11434: bind: address already in use` | Ollama is already running — this is fine, continue |
| Model download is slow | Models are 275 MB to 4.7 GB. Use a stable connection. Downloads resume if interrupted. |
| `Error: model 'mistral' not found` | Run `ollama pull mistral` first |
| Out of memory (OOM) during inference | Close other apps. If your machine has <8 GB RAM, use smaller models (`phi3` instead of `llama3:8b`). |

### Python / pip Issues

| Problem | Solution |
|---------|---------|
| `ModuleNotFoundError: No module named 'X'` | Make sure your virtual environment is activated: `source venv/bin/activate` |
| pip install fails on macOS (onnxruntime) | Try: `pip install onnxruntime` separately. On Apple Silicon: `conda install onnxruntime -c conda-forge` |
| `chromadb` install fails | Upgrade pip first: `pip install --upgrade pip` |
| Permission denied | Never use `sudo pip install`. Use virtual environments instead. |

### General

| Problem | Solution |
|---------|---------|
| "No documents found" or empty results | Check that data files (PDFs/Markdown) are in the correct directory. Re-run the populate/load script. |
| Bad answers despite correct retrieval | The LLM is the issue, not retrieval. Try a different model or adjust the prompt template. |
| Bad answers with wrong source chunks | Retrieval problem. Try smaller `chunk_size`, increase `top_k`, or check that your documents loaded correctly. |
| ChromaDB errors after changing settings | Delete the database directory (`chroma/` or `chroma_data/`) and re-populate. |

---

## Reality Check: What We Found Testing All 5 (March 2026)

We cloned and attempted to run all 5 projects. Here is what actually happened:

| # | Repo | Code Runs? | Data Works? | Blocker |
|---|------|-----------|-------------|---------|
| 1 | pixegami/rag-tutorial-v2 | After 3 import fixes | PDFs included and work | LangChain reorganized imports in 2024 |
| 2 | sbj1198/local-llm-rag-chromadb | Yes, no import fixes needed | Default URLs return 404 + chunk size too large | Microsoft moved their Azure docs; chunk_size=1500 exceeds mxbai-embed-large context |
| 3 | Faridghr/Simple-RAG-Chatbot | Old imports (same as #1) | Text file included | Requires OpenAI + Pinecone API keys |
| 4 | pixegami/langchain-rag-tutorial | Not tested | Markdown included | Requires OpenAI API key |
| 5 | umbertogriffo/rag-chatbot | Not tested | Markdown in `docs/` | Requires Python 3.12 exact, Poetry, llama-cpp-python compilation |

**Only Project 1 ran end-to-end** (after fixes). Project 2 was close but needed a URL swap. Projects 3-5 have hard blockers.

**The lesson here is the lesson.** Open source tutorials go stale. Libraries reorganize. URLs break. API keys get required. Python versions get pinned. The ability to diagnose why a cloned project does not work — and fix it — is a real-world engineering skill. You will do this constantly in your career.

When you hit a problem:
1. **Read the error message.** `ModuleNotFoundError` tells you exactly which import path is wrong.
2. **Check the dates.** When was the repo last updated? When did the library change?
3. **Check the data.** If the code runs but answers are wrong, inspect what actually got loaded — it might be a 404 page.
4. **Check the dependencies.** Does the repo require API keys, specific Python versions, or paid services?

---

## Summary: Which Project to Run When

| Your Goal | Run This |
|-----------|---------|
| **First hands-on RAG experience** | Project 1 (pixegami/rag-tutorial-v2) |
| **See retrieval in isolation** | Project 2 (sbj1198/local-llm-rag-chromadb) — `vector_search.py` |
| **See a web UI on RAG** | Project 3 (Faridghr/Simple-RAG-Chatbot) — Streamlit |
| **Compare local vs cloud LLMs** | Project 1 (local) vs Project 4 (OpenAI) |
| **See production architecture** | Project 5 (umbertogriffo/rag-chatbot) — read the code |
| **Build your P3 project** | Start from Project 1's structure, add your documents, add your improvements |

---

## Next Step

After running at least Project 1:

1. Go back to the [RAG from Scratch notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/RAG_from_Scratch.ipynb), Section 29 ("Your Turn — Build Your Own")
2. Pick a domain (your own documents)
3. Use Project 1's file structure as your starting template
4. Add your own improvements — this becomes your P3 project

> **The goal is not to copy these projects.** The goal is to see how a working RAG system behaves — what good output looks like, what bad output looks like, and what breaks when you change things. Then build your own.
