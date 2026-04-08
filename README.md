# Systems in Production

**Real-world systems that break — and how to diagnose, fix, and improve them.**

This is not a tutorial collection. This is documentation of how production systems actually work — where they fail, what patterns hold up, and how decisions are made under real constraints.

---

## Start Here

### → [See a real system](systems/call-intelligence/)
A production diagnostic system that unifies fragmented data sources, automates investigation across databases, logs, and documents, and surfaces operational issues before they compound. Architecture, data flow, failure points, and design decisions.

### → [Understand the patterns](patterns/)
Architecture patterns extracted from production systems: Bronze-Silver-Gold data pipelines, multi-system reconciliation, AI-derived feature engineering, feedback loops that make systems improve over time.

### → [Learn where it breaks](failures/)
Where and why systems fail in production — not in theory, but from real incidents. Flat table architectures that create duplicate data. ML models that fail because the features don't carry signal. Cross-system joins that silently drop records.

### → [Explore how decisions are made](decisions/)
The decisions behind the architecture: batch vs streaming, star schema vs querying source tables directly, SQL vs Spark vs BigQuery. Not what to choose — but how to think about choosing.

### → [Explore the playbooks](playbooks/)
Reference architectures and operating frameworks for Cloud, Data, and AI. Each topic follows: why it matters, how it works, how to build it, production patterns, system design, quality and security, observability, and a decision guide.

### → [Run the code](implementation/)
30 Jupyter notebooks, pipeline implementations, and project specifications. Open any notebook in Google Colab — no local setup needed.

### → [Read field notes](case-notes/)
Short observations from production: what broke, what was missed, what was recovered.

---

## Implementation — Notebooks

Click any Colab badge to open and run.

### Data Engineering

| Notebook | Open in Colab |
|:---|:---|
| [GCP Full Pipeline](implementation/notebooks/GCP_Full_Pipeline.ipynb) — Bronze → Silver → Gold on BigQuery | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/GCP_Full_Pipeline.ipynb) |
| [Analytics Pipeline](implementation/pipelines/Analytics_Pipeline.ipynb) — Bronze → Silver → Gold in Pandas | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/pipelines/Analytics_Pipeline.ipynb) |
| [Data Modeling](implementation/notebooks/Data_Modeling.ipynb) — Star schema, SCD, partitioning | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Data_Modeling.ipynb) |
| [Advanced SQL](implementation/notebooks/Advanced_SQL.ipynb) — Window functions, CTEs | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Advanced_SQL.ipynb) |

### Machine Learning & AI

| Notebook | Open in Colab |
|:---|:---|
| [ML Pipeline](implementation/pipelines/ML_Pipeline.ipynb) — Feature engineering → train → evaluate → SHAP | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/pipelines/ML_Pipeline.ipynb) |
| [ML Fundamentals](implementation/notebooks/ML_Fundamentals.ipynb) — Bias-variance, regularization, SHAP, MLflow | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/ML_Fundamentals.ipynb) |
| [Deep Learning & PyTorch](implementation/notebooks/Deep_Learning_PyTorch.ipynb) — Neural networks, training diagnostics | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Deep_Learning_PyTorch.ipynb) |
| [RAG from Scratch](implementation/notebooks/RAG_from_Scratch.ipynb) — Retrieval-augmented generation | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/RAG_from_Scratch.ipynb) |
| [Agents](implementation/notebooks/Agents.ipynb) — ReAct, tool use, multi-step reasoning | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Agents.ipynb) |

### Foundations

| Notebook | Open in Colab |
|:---|:---|
| [Python for AI](implementation/notebooks/Python_for_AI.ipynb) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_for_AI.ipynb) |
| [Database Essentials](implementation/notebooks/Database_Essentials.ipynb) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Database_Essentials.ipynb) |
| [Math for AI](implementation/notebooks/Math_for_AI.ipynb) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Math_for_AI.ipynb) |

---

## The Dataset

All pipelines use a **call center analytics dataset** — synthetic data modeled on production DRTV (Direct Response Television) operations. Includes calls, campaigns, orders, products, and intentional data quality issues (duplicates, timezone bugs, missing values).

The same dataset powers both the data pipeline (Bronze → Silver → Gold) and the ML pipeline (feature engineering → model training → evaluation). This mirrors production reality: the data engineering pipeline feeds the AI system.

---

## Author

**Sunil Mogadati** — 25+ years building and leading complex systems end-to-end. I fix systems that don't respond to more tools or more people — especially when AI is involved.

[LinkedIn](https://linkedin.com/in/sunilmogadati) · [GitHub](https://github.com/sunilmogadati)
