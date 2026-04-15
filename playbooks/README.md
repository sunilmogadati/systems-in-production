# Playbooks

**Reference architectures and operating frameworks for building intelligent production systems.**

Do not try to read everything. Follow one path until you can build something. Then come back for the next one.

Each playbook follows the same 10-chapter framework:

```
01 Why           — What problem this solves and why it matters
02 Concepts      — Core ideas in plain English with diagrams
03 Hello World   — See it work in minutes
04 How It Works  — What happens under the hood
05 Building It   — Build something real
06 Production    — How industry does it
07 System Design — Architecture at scale
08 Quality       — Security, governance, data quality
09 Observability — Monitoring, debugging, troubleshooting
10 Decision Guide— "Should I use this?" with tradeoff tables
```

---

## The Builder's Path

Follow this sequence. Each step builds on the previous one.

### Step 1: The Language
| Playbook | What You Get | Time |
|---|---|---|
| **[Python](python/)** | The language everything runs in. If you know Java/JS, start at chapter 04. | 1-2 days |

### Step 2: Data Foundation
| Playbook | What You Get | Time |
|---|---|---|
| **[Data Design](data/data-design/)** | SQL, data modeling, star schema — how to structure and query data | 2-3 days |
| **[PySpark](data/pyspark/)** | Distributed data processing at scale | 1-2 days |
| **[Pipelines](data/pipelines/)** | Cloud pipelines, ETL/ELT patterns, lakehouse formats | 2-3 days |

### Step 3: Intelligence Layer
| Playbook | What You Get | Time |
|---|---|---|
| **[Machine Learning](ai/ml/)** | Prediction, classification, anomaly detection (21 algorithms) | 2-3 days |
| **[Deep Learning](ai/deep-learning/)** | Neural networks, CNNs, training diagnostics | 2-3 days |
| **[RAG](ai/rag/)** | AI that answers from your organization's data | 1-2 days |
| **[Agents](ai/agents/)** | AI that takes actions, not just answers | 1-2 days |

### Step 4: Ship It
| Playbook | What You Get | Time |
|---|---|---|
| **[Software Engineering](engineering/)** | APIs, testing, Docker, CI/CD, production patterns | 2-3 days |

### Reference
| Resource | What It Is |
|---|---|
| **[Data to Model Pipeline](Data_to_Model_Pipeline.md)** | How DE feeds ML |

---

## How to Use Each Playbook

**Read the concept chapter on GitHub** (diagrams render here), then **run the notebook on Colab** (one click, no setup).

Every playbook links to its matching notebooks. Every notebook links back to its playbook.

---

## Get Help

**Community:** [DeliveryMomentum on Skool](https://www.skool.com/deliverymomentum) — ask questions, share what you're building, discuss real systems.

**1:1:** [Book a conversation with Sunil](https://calendly.com/sunil-mogadati/connect) — 20 minutes, focused on your specific situation.
