# Sequence Models — RNN, LSTM, GRU

**Networks designed to process sequences. RNN (Recurrent Neural Network), LSTM (Long Short-Term Memory), GRU (Gated Recurrent Unit). Mostly legacy for NLP — Transformers dominate that — but still alive and well in time-series forecasting, streaming inference, and edge deployment.**

## Foundations First

This playbook assumes you have read [Deep Learning](../deep-learning/) — perceptron, backprop, training loop, diagnostics. Sequence models add **recurrence and gating** on top of those foundations.

For math refreshers, see [Math for AI](../math-for-ai.md). For terminology shared with other architectures (RNN comparisons, ResNet, etc.), see [Architecture Glossary](../architecture-glossary.md). For worked numerical examples (parameter counts for RNN/LSTM, BPTT walkthroughs), see [Architecture Math](../architecture-math.md).

## Chapters

| # | Chapter | What You Learn |
|---|---|---|
| 01 | [Why](01_Why.md) | Why sequences matter — cardiologist, pilot, translator. The three architectures. Where RNN/LSTM still win in 2026. |
| 02 | [Concepts](02_Concepts.md) | Recurrence, hidden state, BPTT (Backpropagation Through Time), vanishing gradients, LSTM gates, GRU. With a fully worked numerical example. |
| 03 | [Hello World](03_Hello_World.md) | Build a working character-level LSTM in 60 lines of PyTorch. |
| 04 | [How It Works](04_How_It_Works.md) | Vanishing/exploding gradients diagnosed in practice. Gradient clipping. Sequence length tradeoffs. |
| 05 | [Building It](05_Building_It.md) | RNN vs LSTM vs GRU vs Transformer — when to use which. Production training recipes. |
| 06 | [Production Patterns](06_Production_Patterns.md) | Uber demand forecasting, Netflix capacity, IoT anomaly detection, voice assistants, financial trading, embedded sequence models. |
| 07 | [System Design](07_System_Design.md) | Streaming inference, batching variable-length sequences, stateful serving, edge deployment, latency budgets. |
| 08 | [Quality, Security, Governance](08_Quality_Security_Governance.md) | Distribution shift, time-aware splits, regulatory compliance (financial, medical), sensor reliability. |
| 09 | [Observability & Troubleshooting](09_Observability_Troubleshooting.md) | Forecast residuals, drift detection, hidden state health, runbooks for production failures. |
| 10 | [Decision Guide](10_Decision_Guide.md) | "Should I use a recurrent model?" decision table. RNN vs LSTM vs GRU vs Transformer. Production readiness checklist. |

## Architecture Deep Dive

| Doc | Description |
|---|---|
| [`architectures/rnn-lstm.md`](architectures/rnn-lstm.md) | Single-doc reference covering recurrence, BPTT, vanishing/exploding gradients, LSTM gates, training and inference. Concepts + code + math + Q&A in one document. |

## Hands-On Notebooks

| Notebook | What It Does |
|---|---|
| [Sequence_Models_From_Scratch.ipynb](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Sequence_Models_From_Scratch.ipynb) | Vanilla RNN trained by hand in pure NumPy. Forward through 3 timesteps, full BPTT backward, weight-sharing gradient accumulation across time, PyTorch autograd verification, vanishing-gradient demonstration on a longer sequence. |

## Sibling Playbooks

| Playbook | When to Read |
|---|---|
| [Transformers](../transformers/) | The modern alternative — supplanted RNN/LSTM for most NLP work |
| [Deep Learning](../deep-learning/) | For backprop, training mechanics |
| [NLP](../nlp/) (coming) | For language tasks that historically used RNN/LSTM |

[LinkedIn](https://linkedin.com/in/sunilmogadati) · [Community](https://www.skool.com/deliverymomentum) · [Engagement inquiries](https://calendly.com/sunil-mogadati/connect)
