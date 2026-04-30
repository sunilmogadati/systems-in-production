# NLP — Natural Language Processing

**The task domain for systems that read, understand, classify, generate, and translate text. NLP is the *application* layer that sits on top of transformer architecture, retrieval systems, and AI agents — pulling them together for language tasks.**

## What This Playbook Is — and Is Not

This is the **task-domain entry point** for NLP, parallel to [`computer-vision/`](../computer-vision/) for vision tasks. It focuses on:

- The NLP task taxonomy (classification, extraction, generation, translation, etc.)
- Tokenization, embeddings, and the language-specific pipeline
- Classical NLP (TF-IDF, naive Bayes) as legacy and foundation
- Modern NLP (transformer fine-tuning, prompting, RAG)
- NLP-specific evaluation (BLEU, ROUGE, perplexity, F1 for entity spans)
- Multilingual considerations
- Production patterns specific to language

This playbook does **not** re-explain transformer architecture. For attention math, encoder/decoder mechanics, multi-head, see [`transformers/`](../transformers/). For retrieval+generation patterns, see [`rag/`](../rag/). For autonomous LLM systems, see [`agents/`](../agents/).

## Foundations First

Before this playbook, read:
- [Deep Learning](../deep-learning/) — backprop, training loop
- [Transformers](../transformers/) — the architecture that powers modern NLP
- [Math for AI](../math-for-ai.md) — derivatives, dot products, softmax

For terminology shared with other architectures, see [Architecture Glossary](../architecture-glossary.md).

## Chapters

| # | Chapter | What You Learn |
|---|---|---|
| 01 | [Why](01_Why.md) | The translator, customer support, the content moderator. NLP task taxonomy. The NLP value at scale. |
| 02 | [Concepts](02_Concepts.md) | Tokenization (BPE, WordPiece), word and sentence embeddings, the NLP pipeline, classical NLP foundations (TF-IDF, n-grams). |
| 03 | [Hello World](03_Hello_World.md) | Build a sentiment classifier two ways: classical (TF-IDF + logistic regression) and modern (BERT fine-tune via Hugging Face). |
| 04 | [How It Works](04_How_It_Works.md) | Embedding geometry, fine-tune vs prompt vs RAG decision, evaluation metrics specific to NLP (BLEU, ROUGE, F1, perplexity, MRR). |
| 05 | [Building It](05_Building_It.md) | Model selection (BERT family, GPT family, T5 family), API vs self-host for NLP, multilingual considerations, prompt engineering patterns. |
| 06 | [Production Patterns](06_Production_Patterns.md) | Google Translate, GitHub Copilot autocomplete, customer support automation, search ranking (BERT in Google), content moderation, NER at scale. |
| 07 | [System Design](07_System_Design.md) | Serving NLP at scale: classification batching, embedding pipelines for retrieval, streaming generation, mixed-mode services. |
| 08 | [Quality, Security, Governance](08_Quality_Security_Governance.md) | Bias in language models, multilingual fairness, prompt injection, hallucination, copyright, regulatory considerations for NLP. |
| 09 | [Observability & Troubleshooting](09_Observability_Troubleshooting.md) | NLP-specific metrics, evaluation harnesses, drift detection for language data, runbooks. |
| 10 | [Decision Guide](10_Decision_Guide.md) | NLP task decision tree. API vs fine-tune vs classical. Production readiness checklist. |

## Hands-On Notebooks

| Notebook | What It Does |
|---|---|
| [NLP_From_Scratch.ipynb](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/NLP_From_Scratch.ipynb) | BPE tokenization by hand, TF-IDF + naive Bayes classifier (classical NLP), Word2Vec-style embeddings intuition. Mostly NumPy. |

## Sibling Playbooks

| Playbook | When to Read |
|---|---|
| [Transformers](../transformers/) | For the architecture mechanics behind modern NLP (attention, encoder/decoder) |
| [Sequence Models](../sequence-models/) | For RNN/LSTM — historically used for NLP, mostly legacy now |
| [RAG](../rag/) | For knowledge-grounded NLP systems (search + generate with citations) |
| [Agents](../agents/) | For autonomous NLP systems that reason and use tools |
| [Computer Vision](../computer-vision/) | For multimodal systems (vision + language) |

## What NLP Powers in 2026

| Domain | Approach |
|---|---|
| Customer support automation | Decoder-only LLMs + RAG over support docs |
| Translation | Encoder-decoder transformers (Google Translate) or LLM with prompts |
| Search ranking | Encoder-only transformers (BERT family) + retrieval |
| Content moderation | Encoder-only classifiers + LLM-as-judge |
| Code completion | Decoder-only LLMs fine-tuned on code |
| Knowledge Q&A | LLM + RAG (see [RAG playbook](../rag/)) |
| Information extraction (NER, relation extraction) | Encoder-only or fine-tuned LLMs |
| Sentiment analysis | Classical (TF-IDF + classifier) for simple cases; encoder-only for nuance |
| Summarization | LLM with prompts; encoder-decoder for structured output |
| Multilingual systems | Multilingual transformers (mT5, XLM-R, multilingual LLMs) |

[Sunil Mogadati](https://www.linkedin.com/in/sunilmogadati) | [Community](https://www.skool.com/deliverymomentum)
