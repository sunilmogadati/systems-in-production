# Pipelines

**How to build, operate, and evolve data pipelines on any cloud.**

| Topic | What You Learn |
|---|---|
| [Ingestion](ingestion/) | How data enters the pipeline — OLTP databases, APIs, streams, NoSQL, files |
| [Cloud Pipeline](cloud/) | Bronze → Silver → Gold architecture, GCS/S3/ADLS, automation, orchestration |
| [ETL/ELT Patterns](etl-elt/) | Full refresh, incremental, CDC, MERGE/upsert, dead letter queues |
| [Lakehouse Formats](lakehouse/) | Delta Lake, Apache Iceberg, Hudi — ACID transactions on files |
| [Data Quality](data-quality/) | Automated quality gates — Great Expectations, dbt tests, Soda, Dataplex, Glue DQ |
| [Hybrid](hybrid/) | On-prem → cloud data movement, scheduling handoffs, network architecture |
| [Compliance](compliance/) | Regulated data — SAFE Room, DLP, PHI/PII, HIPAA/PCI-DSS/GDPR controls |
| [Semantic Layer](semantic-layer/) | Starburst, dbt Semantic Layer, LookML — where business definitions live |

**Suggested order:** Ingestion → Cloud Pipeline → ETL/ELT Patterns → Lakehouse Formats → Data Quality → Hybrid → Compliance → Semantic Layer

All theory is cloud-agnostic. Cloud-specific implementation is in the notebooks.

## Hands-On

| Notebook | Cloud | Open in Colab |
|---|---|---|
| [ETL/ELT Patterns](../../../implementation/notebooks/ETL_ELT_Patterns.ipynb) | None (pure Python) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/ETL_ELT_Patterns.ipynb) |
| [Delta Lake Hello World](../../../implementation/notebooks/Delta_Lake_Hello_World.ipynb) | None (local PySpark) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Delta_Lake_Hello_World.ipynb) |
| [GCP Full Pipeline](../../../implementation/notebooks/GCP_Full_Pipeline.ipynb) | GCP | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/GCP_Full_Pipeline.ipynb) |
| [GCP Pipeline Automation](../../../implementation/notebooks/GCP_Pipeline_Automation.ipynb) | GCP | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/GCP_Pipeline_Automation.ipynb) |
| [GCP Lakehouse](../../../implementation/notebooks/GCP_Lakehouse.ipynb) | GCP | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/GCP_Lakehouse.ipynb) |
| AWS Pipeline | AWS | Coming soon |
| Azure Pipeline | Azure | Coming soon |

[Community](https://www.skool.com/deliverymomentum) | [Book a 1:1](https://calendly.com/sunil-mogadati/connect)
