# Project P2: AI Eval & Cost

**Week:** 7 (Cloud & MLOps Essentials)
**Time estimate:** 8-10 hours
**Prerequisites:** P1 (ML Predictor), P1.5 (Image Classifier), Cloud & MLOps week content

---

## Objective

Build an **LLM (Large Language Model) evaluation harness** that compares multiple AI models on three dimensions: **quality, latency, and cost.** You will measure how well different models answer the same set of questions, how fast they respond, and how much each query costs — then recommend the best model for a given use case.

This is the project where you stop being a model user and start being a model evaluator. In production, the question is never "does this model work?" — it is "which model gives the best quality at an acceptable cost and latency for THIS use case?"

---

## Why This Project Matters

Companies choosing between GPT-4, Claude, Mistral, Llama, and dozens of other models need data, not opinions. Marketing pages claim every model is the best. Your evaluation harness provides the evidence:

- **Quality:** Does the model actually answer correctly? Does it hallucinate?
- **Latency:** Can it respond in under 2 seconds? What about under 200ms?
- **Cost:** At 10,000 queries per day, what is the monthly bill?

The engineer who can run this analysis and present the results with data is the one who gets asked to make the architectural decision.

---

## The Evaluation Dataset

Create a **benchmark test set** of at least **30 question-answer pairs** across 3 categories:

### Category 1: Factual Q&A (10 questions)
Questions with a single, verifiable correct answer.
- "What is the capital of Australia?" → "Canberra"
- "In what year was Python first released?" → "1991"
- "What does HTTP stand for?" → "HyperText Transfer Protocol"

### Category 2: Reasoning / Multi-Step (10 questions)
Questions that require the model to think, not just recall.
- "If a train travels 60 mph for 2.5 hours, how far does it go?" → "150 miles"
- "What is the next number in the sequence: 2, 6, 12, 20, 30, ?" → "42"
- "A company's revenue grew 15% year over year. If it was $10M last year, what is this year's revenue?" → "$11.5M"

### Category 3: Summarization / Generation (10 questions)
Open-ended questions where you evaluate quality, not exact match.
- "Summarize the key differences between SQL and NoSQL databases in 3 sentences."
- "Explain gradient descent to a 10-year-old."
- "Write a Python function that reverses a string. Include a docstring."

> **Why 3 categories?** Different models excel at different tasks. A model that aces factual recall may struggle with reasoning. A model optimized for speed may give shallow summaries. Your harness should reveal these differences.

---

## Requirements

### 1. Model Comparison (minimum 3 models)
- [ ] Compare at least **3 models**. Choose from:

| Model | Access Method | Approximate Cost |
|:---|:---|:---|
| **Mistral 7B** (via Ollama) | Local, free | $0 (your hardware) |
| **Llama 3 8B** (via Ollama) | Local, free | $0 (your hardware) |
| **GPT-3.5-turbo** (via OpenAI API) | Cloud API, paid | ~$0.002 per 1K tokens |
| **GPT-4o-mini** (via OpenAI API) | Cloud API, paid | ~$0.00015 per 1K input tokens |
| **Claude 3 Haiku** (via Anthropic API) | Cloud API, paid | ~$0.00025 per 1K input tokens |
| **Gemma 2 9B** (via Ollama) | Local, free | $0 (your hardware) |

> **Budget-friendly option:** Use 2 local models (Mistral + Llama 3 via Ollama) + 1 cloud model (GPT-3.5-turbo or GPT-4o-mini — costs pennies for 30 queries). If you cannot use any paid API (Application Programming Interface), use 3 local Ollama models of different sizes.

### 2. Quality Evaluation
For each model on each question:
- [ ] **Factual Q&A:** Exact match or LLM-as-judge (does the answer match the expected answer?)
- [ ] **Reasoning:** LLM-as-judge with a rubric ("Is the reasoning correct? Is the final answer correct?")
- [ ] **Summarization/Generation:** At least 2 metrics from:
  - **BLEU (Bilingual Evaluation Understudy)** — measures n-gram overlap between generated and reference text
  - **ROUGE (Recall-Oriented Understudy for Gisting Evaluation)** — measures recall of reference text in generated text
  - **LLM-as-judge** — use a separate model to score quality (1-5 scale with rubric)
  - **Human evaluation** — you manually rate each response (most reliable, most time-consuming)
- [ ] Compute an **overall quality score** per model (weighted average across categories)

### 3. Latency Measurement
For each model on each question:
- [ ] Measure **time to first token** (TTFT — Time to First Token) if streaming, or **total response time** if not
- [ ] Compute latency statistics per model: **median, p95 (95th percentile), max**
- [ ] Plot a **latency distribution** (histogram or box plot) for each model

> **Why p95, not average?** Average hides outliers. A model with 200ms average but occasional 5-second spikes will frustrate users. P95 tells you: "95% of responses are faster than this threshold."

### 4. Cost Analysis
For each model on each question:
- [ ] Count **input tokens** and **output tokens** (use the model's tokenizer or estimate at ~4 chars per token)
- [ ] Calculate **cost per query** using the model's pricing
- [ ] Project **monthly cost** at different query volumes: 100/day, 1,000/day, 10,000/day
- [ ] Build a **cost comparison table** across all models

For local models (Ollama), estimate cost as:
- Hardware cost per hour (e.g., GPU instance cost if running in cloud) / queries per hour = cost per query
- Or document as "$0 (local hardware)" if running on your own machine

### 5. Hallucination Detection
- [ ] For each factual Q&A response, flag whether the model **hallucinated** (gave a confident but wrong answer)
- [ ] Calculate **hallucination rate** per model (% of factual questions where the model was confidently wrong, not just uncertain)
- [ ] Show examples of hallucinations with the model's response vs the correct answer

### 6. Semantic Caching (Bonus)
- [ ] Implement a simple cache: if the same question (or a semantically similar question) is asked again, return the cached answer instead of calling the model
- [ ] Use embedding similarity to detect "close enough" questions (threshold at cosine similarity > 0.95)
- [ ] Measure **cache hit rate** on a repeated query set
- [ ] Calculate **cost savings** from caching

> **This is a bonus.** Worth extra credit if implemented, not required for a passing grade.

### 7. Experiment Tracking
- [ ] Log all evaluation runs to **MLflow** or **Weights & Biases (W&B)**
- [ ] Each run should record: model name, quality scores, latency stats, cost per query, hallucination rate
- [ ] Show a screenshot or export of the experiment tracking dashboard comparing all models

### 8. Results Report
Create a `results/evaluation_report.md` with:
- [ ] **Summary table** — all models, all metrics, side by side
- [ ] **Recommendation** — which model for which use case:
  - "For a customer-facing chatbot (latency matters), use [X]"
  - "For a batch analysis pipeline (quality matters, latency does not), use [Y]"
  - "For a startup on a budget (cost matters most), use [Z]"
- [ ] **Visualizations** — at least 3 charts (quality comparison, latency distribution, cost projection)
- [ ] **Trade-off analysis** — there is no single "best" model. The best model depends on what you are optimizing for.

### 9. Code Quality
- [ ] Modular structure: separate files for evaluation logic, metrics, cost calculation, visualization
- [ ] WHY comments (25%+)
- [ ] Configuration via `.env` or config file (model names, API keys, thresholds)
- [ ] `requirements.txt` with pinned versions
- [ ] `.gitignore`

### 10. README + Reflection Journal
**README:**
- [ ] What models were compared and why
- [ ] How to run the evaluation (setup, install, configure, execute)
- [ ] Summary of results with key findings
- [ ] Screenshots of experiment tracking dashboard

**REFLECTION.md:**
1. Which model won overall? Was it the one you expected?
2. What surprised you most about the latency or cost results?
3. Where did the models hallucinate and why do you think it happened?
4. If you were advising a company choosing between these models, what would you tell them?
5. How would this evaluation change for a domain-specific use case (medical, legal, finance)?

---

## Step-by-Step Guide

### Step 1: Set up your project
```bash
mkdir p2-ai-eval-cost && cd p2-ai-eval-cost
python -m venv .venv
source .venv/bin/activate
pip install openai anthropic tiktoken mlflow rouge-score nltk matplotlib seaborn pandas jupyter python-dotenv
pip freeze > requirements.txt
```

Pull local models:
```bash
ollama pull mistral
ollama pull llama3
```

### Step 2: Build your test set
Write 30 question-answer pairs in a JSON file. This is the foundation — spend time on it.

```json
[
  {
    "id": 1,
    "category": "factual",
    "question": "What is the capital of Australia?",
    "expected_answer": "Canberra",
    "evaluation_method": "exact_match"
  },
  {
    "id": 11,
    "category": "reasoning",
    "question": "If a train travels 60 mph for 2.5 hours, how far does it go?",
    "expected_answer": "150 miles",
    "evaluation_method": "llm_judge"
  }
]
```

### Step 3: Build the evaluation runner
A function that takes a model name and a test set, sends each question to the model, records the response, latency, and token count.

### Step 4: Build the quality scorer
Score each response using exact match, BLEU/ROUGE, or LLM-as-judge depending on the category.

### Step 5: Build the cost calculator
Count tokens, multiply by pricing. Project to daily/monthly volumes.

### Step 6: Run evaluations
Run all models on all questions. Log to MLflow/W&B.

### Step 7: Build visualizations and write the report
### Step 8: Write README, reflection, push to GitHub

```bash
git init && git add . && git commit -m "P2: LLM evaluation harness — quality, latency, cost comparison"
gh repo create p2-ai-eval-cost --public --push
```

---

## Grading Rubric

| Category | Points | What We Look For |
|:---|:---|:---|
| Model comparison (3+ models, all 30 questions) | 15 | All models evaluated on all questions, responses saved |
| Quality evaluation (exact match + LLM-judge + BLEU/ROUGE) | 20 | Correct metrics per category, overall quality score per model |
| Latency measurement (median, p95, distribution plot) | 10 | Accurate timing, per-model distribution visualized |
| Cost analysis (per-query + monthly projections) | 10 | Token counting, pricing applied, projection table |
| Hallucination detection (rate per model + examples) | 10 | Hallucinations correctly identified and counted |
| Experiment tracking (MLflow/W&B with all metrics logged) | 10 | Dashboard screenshot showing model comparison |
| Results report (summary table, recommendation, 3+ charts) | 15 | Clear trade-off analysis, use-case-specific recommendations |
| Code quality + README + Reflection | 10 | Modular, WHY comments, reproducible, honest reflection |
| **Total** | **100** | |
| **Bonus: Semantic caching** | **+10** | Working cache with hit rate and cost savings measured |

---

## The Interview Story

> "I built an LLM evaluation harness that compared Mistral 7B, Llama 3 8B, and GPT-3.5-turbo across 30 questions in three categories: factual recall, multi-step reasoning, and text generation. GPT-3.5-turbo won on quality (89% accuracy on factual, best ROUGE scores on summarization) but at $0.002 per query and 800ms median latency. Mistral 7B running locally had 78% factual accuracy but zero cost and 200ms latency. The surprise was hallucination rates: GPT-3.5-turbo hallucinated on 2 of 10 factual questions with high confidence, while Mistral hallucinated on 3 but with lower confidence (easier to detect). My recommendation: for a customer-facing chatbot, use GPT-3.5-turbo with a confidence threshold. For internal batch processing, use Mistral locally — the cost savings at 10,000 queries per day is $600/month. I logged everything to MLflow so the team can rerun the evaluation when new models release."

---

## How P2 Connects to What Comes Next

| What You Build Here | Where It Goes |
|:---|:---|
| LLM-as-judge evaluation pattern | **P3 (the RAG section):** RAGAS evaluation uses the same judge pattern on RAG responses |
| Latency benchmarking | Extension: load testing with Locust/k6 |
| Cost modeling | **the LLMOps section (LLMOps):** Token cost dashboards, rate limiting, caching strategies |
| Hallucination detection | **P3 (the RAG section):** RAG faithfulness scores measure the same thing in a retrieval context |
| Experiment tracking (MLflow/W&B) | **the LLMOps section:** Production MLOps integrates experiment tracking into CI/CD pipelines |
| Model comparison methodology | **Every future project:** You will always need to justify model choices with data, not opinions |

---

## Deliverables

Push to GitHub with this structure:

```
p2-ai-eval-cost/
  README.md
  REFLECTION.md
  requirements.txt
  .env.example
  .gitignore
  data/
    test_questions.json        # Your 30 question-answer pairs
  src/
    evaluate.py                # Main evaluation runner
    metrics.py                 # Quality scoring (exact match, BLEU, ROUGE, LLM-judge)
    cost.py                    # Token counting, pricing, projections
    cache.py                   # (bonus) Semantic caching
    visualize.py               # Charts and plots
  results/
    evaluation_report.md       # Full results with recommendations
    charts/                    # Saved chart images
  notebooks/
    analysis.ipynb             # Interactive exploration of results
```
