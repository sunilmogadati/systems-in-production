# Demo Cheat Sheet - The System Shift Presentation

**Use case:** Production Diagnostic Intelligence System
**Narrative:** One system, six components, every role needed.

---

## Before Demos (Slide 8)

Say: "Let me show you a system I'm building. It collects data from databases, application logs, deployment records, runbooks, and APIs. It monitors, diagnoses, and surfaces findings automatically. Each piece I'm about to show you is a component of that system. This is not 6 separate tutorials. This is one system, viewed through 6 lenses."

---

## Demo 1: Star Schema (Data Model)

**What it is in the system:** How you structure incidents, deployments, metrics, and services so they can be queried together.

**Open:** https://github.com/sunilmogadati/systems-in-production/tree/main/foundations/data/star-schema-design

**Show:**
- Open `01_Why.md` - scroll to the 3 AM scenario
- Open `02_Concepts.md` - show the Mermaid diagram (fact table + dimensions)

**Say:** "Before you can diagnose anything, your data has to be structured. This is how you go from a flat table with 300 columns to a clean, queryable system. This is architecture thinking, not code."

**Time:** 2 min

---

## Demo 2: ML Pipeline (Prediction)

**What it is in the system:** Predicts which P3 incident will escalate to P1. Classifies root causes automatically. Learns from past incidents.

**Open:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_ML_Fundamentals.ipynb

**Scroll map (all outputs saved, no need to run):**

| Cell # | What's On Screen | What You Say |
|---|---|---|
| Cell 5 | Architect Decision Checklist | "Before writing any code, these are the decisions. What metric? What baseline? What's good enough?" |
| Cell 89 | Section 9 header | "Now the full pipeline." |
| Cell 90 | Data loaded, 20,640 rows | "Real dataset. 20,000 records. We're predicting values." |
| Cell 91 | Feature engineering: 8 becomes 11 | "Domain knowledge matters. We created new features. These aren't in the raw data. An engineer designed them." |
| Cell 92 | Baseline LinearRegression: R2 = 0.66 | "Start simple. 66% accuracy. That's our benchmark." |
| Cell 93 | Ridge: R2 = 0.66 | "Regularization. No improvement. Tells us overfitting isn't the problem." |
| Cell 94 | GradientBoosting: R2 = 0.84 | "Advanced model. 84%. Significant jump. But notice the train score is 0.95. That gap tells you something." |
| Cell 95 | MODEL COMPARISON TABLE | "Three models, side by side. MLflow tracked every experiment. Not guessing. Measuring." |
| Cell 97 | SHAP charts + pipeline recap | "And this is WHY it won. SHAP shows which features drove each prediction. Transparent, not a black box." |

**Say after:** "In our diagnostic system, this predicts which incidents will escalate. The SHAP chart shows WHY the model flagged something. That's a transparent recommendation, not a guess."

**Time:** 3 min

---

## Demo 3: RAG (Knowledge Retrieval)

**What it is in the system:** When an engineer is debugging at 2 AM, they need answers from runbooks, post-mortems, and docs. RAG makes that searchable by meaning, not keywords.

**Open:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_RAG_from_Scratch.ipynb

**Show:**
- The pipeline flow: ingest documents, create embeddings, retrieve, answer
- The 10-step framework mapping at the end of the notebook

**Say:** "Everyone talks about chatbots. This is what's underneath. In our diagnostic system, this is the knowledge layer. It ingests runbooks, post-mortems, architecture docs. Ask 'what's the fix for connection pool exhaustion' and it finds the answer across all your documentation. No more searching Confluence at 2 AM."

**Check before presenting:** Do outputs show without running? If not, scroll to markdown cells that explain the flow.

**Time:** 2-3 min

---

## Demo 4: GCP Pipeline (Data Ingestion)

**What it is in the system:** Logs, metrics, incidents, deployments arrive in different formats from different systems. This pipeline moves them from raw to clean to analysis-ready.

**Open:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/M08_Cloud_Data_Pipeline_Build.ipynb

**Show:**
- Bronze to Silver to Gold sections
- A BigQuery result or pipeline diagram

**Say:** "In our diagnostic system, this is how data gets from source to usable. Logs come as JSON, metrics as CSV, incidents from ticketing APIs. The pipeline cleans, deduplicates, fixes timezone issues, and structures everything for analysis. Without this, nothing else works."

**Time:** 2 min

---

## Demo 5: Deep Learning (Anomaly Detection)

**What it is in the system:** Memory usage climbing over 5 days. Normal growth or a leak? A neural network trained on historical metrics catches patterns that dashboards miss.

**Open:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Deep_Learning_PyTorch.ipynb

**Show:**
- Training output: loss curves, accuracy going up
- A prediction result

**Say:** "In our diagnostic system, deep learning watches the metrics. CPU, memory, latency, error rates. It learns what normal looks like for each service. When something drifts, it flags it. That search-service memory leak in our dataset? It built up over 5 days before crashing. A trained model catches that on day 2."

**Check before presenting:** Are training outputs saved? If not, show the markdown concept cells.

**Time:** 2 min

---

## Demo 6: AI Agents (Orchestration)

**What it is in the system:** The orchestrator. Receives an alert. Queries the DB. Checks logs. Searches the runbook via RAG. Correlates with recent deployments. Creates a ticket with findings.

**Open:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Agents.ipynb

**Show:**
- Tool-calling flow: agent receives query, picks a tool, executes, returns result

**Say:** "This is where it all comes together. The agent receives an alert, queries the DB, reads the logs, checks the runbook via RAG, correlates with recent deployments, and creates a ticket with findings and recommended actions. That's not a chatbot. That's a diagnostic system. And it needed every skill in this room to build."

**Time:** 2 min

---

## NEW - Demo 7: Production Support Dataset (optional, if time allows)

**What it is:** The actual data the system operates on.

**Open:** VS Code or terminal, navigate to `data/production-support/`

**Show:**
- `services.csv` - "7 microservices, different teams, different tech stacks"
- `incidents.csv` - "15 incidents over 30 days. Some misclassified. Some are the same root cause showing up twice."
- `app_logs.jsonl` - show a few lines. "28,000 log entries. 3% missing request IDs. Broken tracing."
- `runbooks/order-service.md` - "Real runbook. One of these is deliberately outdated."

**Say:** "This is what production looks like as data. There are 10 patterns hidden in here. Can you find them?"

**Time:** 2-3 min (skip if running long)

---

## After All Demos (Slide 15)

Say: "Data pipeline feeds the ML model. ML model flags the anomaly. Deep learning confirms the pattern. RAG retrieves the runbook. The agent ties it all together and creates a ticket. Six components. One system. And it needed every role in this room: data engineering to build the pipeline, a developer to build the agent, QA to validate the outputs, a PM to define what 'good' looks like, DevOps to keep it running."

---

## System Architecture (draw on whiteboard or reference)

```
Logs + DB + Docs + APIs
        |
   DATA PIPELINE (Demo 4: GCP)
   Bronze -> Silver -> Gold
        |
   STAR SCHEMA (Demo 1: Data Model)
   Structured for queries
        |
   +--------+--------+--------+
   |        |        |        |
  ML      DL       RAG     (future)
 (Demo 2) (Demo 5) (Demo 3)
 Predict  Detect   Search
 escalation anomaly runbooks
   |        |        |
   +--------+--------+
            |
       AI AGENT (Demo 6)
       Orchestrate + create ticket
```

---

## Key Lines to Remember

- "This is not 6 separate tutorials. This is one system, viewed through 6 lenses."
- "Six components. One system. Every role in this room was needed to build it."
- "That's not a chatbot. That's a diagnostic system."
- "That's what 'the boundaries are collapsing' actually looks like in practice."
- "The question is not 'will AI take my job?' The question is 'what can I now do that I could not do before?'"

---

## Timing Guide

| Section | Time | Running Total |
|---|---|---|
| Slides 1-7 (talk) | 15 min | 15 min |
| Demo 1: Star Schema | 2 min | 17 min |
| Demo 2: ML Pipeline | 3 min | 20 min |
| Demo 3: RAG | 2-3 min | 23 min |
| Demo 4: GCP Pipeline | 2 min | 25 min |
| Demo 5: Deep Learning | 2 min | 27 min |
| Demo 6: Agents | 2 min | 29 min |
| Demo 7: Dataset (optional) | 2-3 min | 32 min |
| Slides 15-20 (closing) | 8-10 min | 40-42 min |
| Q&A | 10-15 min | 50-55 min |
