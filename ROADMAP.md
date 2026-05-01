# Roadmap

This file tracks the parts of the repo that are intentionally on a phased path. Items marked **(coming)** elsewhere in the repo correspond to the entries below — they are scoped, not abandoned.

## Cloud Pipelines — AWS and Azure equivalents

The pipeline material is GCP-canonical today. The concepts (Bronze-Silver-Gold, lakehouse formats, ELT vs ETL, ingestion patterns, orchestration) are cloud-agnostic; the runnable hands-on path is GCP because that is the deepest-tested deployment.

AWS and Azure equivalents are tracked, in this order:

1. **AWS Pipeline notebook** — Redshift / Athena + S3 + EMR + DMS, with the same Bronze-Silver-Gold structure as the GCP notebook.
2. **AWS pages** under `playbooks/data/pipelines/ingestion/`, `etl-elt/`, `cloud/` — sidebar treatment, not full re-derivations.
3. **Azure Pipeline notebook** — Synapse + ADLS + Data Factory.
4. **Azure pages** alongside the AWS pages.

Until those land, the GCP material is canonical and should be read as the reference implementation. Most concepts transfer directly; the differences are surface-level (service names, IAM model, console workflow).

## SQL Playbook — Chapters 06–10

`playbooks/data/data-design/sql/` is complete through chapter 05 (Building It). Chapters 06–10 (Production Patterns, System Design, Quality / Security / Governance, Observability, Decision Guide) are scoped and on the same 10-chapter framework as the AI playbooks.

## Pure-Python ingestion patterns (no cloud)

A "no cloud" ingestion track — pure Python against APIs, files, and databases — is on the roadmap. Most teams need this before they need a managed cloud service.

## Compliance and Hybrid

`playbooks/data/pipelines/compliance/` and `playbooks/data/pipelines/hybrid/` are placeholders today. They are scoped but lower priority than the AWS/Azure parity work above.

---

If you are tracking something specific, open an issue on the GitHub repo or reach out via the channels in any playbook footer.
