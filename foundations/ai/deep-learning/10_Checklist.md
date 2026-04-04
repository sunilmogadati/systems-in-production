# Deep Learning — Architect's Decision Checklist

**One-page reference. Every decision, one table.**

---

## Before Starting

| Question | Answer |
|:---|:---|
| Is this a DL (Deep Learning) problem? | Images, text, audio → yes. Tabular / spreadsheet → probably not (try XGBoost first). |
| Is there a pre-trained model that already solves it? | If yes → use it via API or fine-tune it. Do not train from scratch. |
| Do I have enough labeled data? | <1,000 examples per class → try simpler ML or transfer learning. |

## Architecture

| Data Type | Architecture | Start With |
|:---|:---|:---|
| Images | CNN (Convolutional Neural Network) | 2-4 conv layers. Go deeper only if underfitting. |
| Text / sequences | Transformer | Pre-trained (BERT, GPT) + fine-tuning. |
| Tabular | MLP (Multi-Layer Perceptron) — or skip DL | Random Forest / XGBoost first. |
| Audio | CNN on spectrogram or Transformer | Pre-trained (Whisper) if speech. |

## Training Configuration

| Setting | Default | Adjust When |
|:---|:---|:---|
| **Optimizer** | Adam | Fine-tuning → AdamW. Competition → SGD + momentum + scheduler. |
| **Learning rate** | 0.001 (Adam) | Loss oscillating → halve it. Loss not moving → double it. |
| **Batch size** | 64-128 | OOM error → reduce. Large dataset + GPU → increase to 256. |
| **Epochs** | 15-50 + early stopping | Stop when test loss plateaus (patience=5-10). |
| **Activation (hidden)** | ReLU ("REE-loo") | Dead neurons → Leaky ReLU. Transformer → GELU ("GEE-loo"). |
| **Activation (output)** | Softmax (multi-class) / Sigmoid (binary) / None (regression) | Almost never deviate. |
| **Loss** | CrossEntropyLoss (classification) / MSELoss (regression) | Binary → BCEWithLogitsLoss. Outliers → L1Loss. |

## Diagnosing Training Problems

| Train Loss | Test Loss | Gap | Diagnosis | Fix |
|:---|:---|:---|:---|:---|
| ↓ | ↓ (slightly above) | Small | Healthy | Keep going |
| ↓ | ↓ then ↑ | Growing | Overfitting | Dropout, augmentation, early stopping |
| High, flat | High, flat | Small | Underfitting | More layers, more neurons, higher LR |
| Wild / ↑ | Wild / ↑ | — | Diverging | Lower learning rate |

## Regularization (Apply in This Order)

| # | Technique | Default Setting |
|:---|:---|:---|
| 1 | Data augmentation (images) | RandomHorizontalFlip + RandomCrop |
| 2 | Dropout | 0.25 (conv layers), 0.5 (FC layers) |
| 3 | Early stopping | Patience = 5-10 epochs |
| 4 | Batch normalization | After conv layers |
| 5 | Weight decay | 0.01 in AdamW |

## Debugging (Check in This Order)

| # | Check | How |
|:---|:---|:---|
| 1 | Data loaded correctly? | Print a batch. Visualize. Check labels. |
| 2 | Data normalized? | `images.mean()` ≈ 0, `images.std()` ≈ 1 |
| 3 | Learning rate reasonable? | 0.001 for Adam. Oscillating → halve. Flat → double. |
| 4 | Model on right device? | `next(model.parameters()).device` matches data device |
| 5 | `optimizer.zero_grad()` present? | Must appear before `loss.backward()` |
| 6 | Loss function matches problem? | Classification → CrossEntropy. Regression → MSE. |
| 7 | Model too small? | Both train + test loss high → add capacity |
| 8 | Model too large? | Train loss low, test loss high → add regularization |

## Production Deployment

| Concern | Minimum Requirement |
|:---|:---|
| **Serving** | FastAPI or TorchServe behind a load balancer |
| **Model format** | `state_dict` saved, loaded at startup (not retrained per request) |
| **Monitoring** | Latency (p95), error rate, prediction distribution, sample accuracy |
| **Versioning** | Every model traceable to its training data, code, and evaluation results |
| **Security** | Input validation, rate limiting, authentication, audit trail |
| **Drift detection** | Monitor input feature distributions vs training baseline |
| **Fallback** | If model fails → return cached result or fall back to previous version |

## Production Optimization

| Constraint | Technique |
|:---|:---|
| Model too large | Quantization (FP32 → INT8: 4x smaller) |
| Latency too high | Quantization + TorchScript + batched inference |
| GPU cost too high | Batching + quantization + auto-scaling down during low traffic |
| Deploy to mobile/edge | Quantization + pruning + knowledge distillation |

## Governance

| Requirement | Action |
|:---|:---|
| Explainability | SHAP (classification) or Grad-CAM (images) |
| Bias audit | Measure accuracy per demographic subgroup. Counterfactual testing. |
| Regulatory | NIST AI RMF assessment. Model card. EU AI Act conformity (if applicable). |
| Data privacy | Anonymize PII. HIPAA/GDPR compliance. Consider differential privacy. |
| Incident response | Document: what happened, root cause, impact, fix, prevention. |

---

## The 10-Step System Design Framework

Use this for any ML system design question — interviews, architecture reviews, or new projects:

```
 1. Requirements    — Latency, throughput, accuracy, cost, compliance constraints
 2. Data pipeline   — Sources, preprocessing, storage, quality validation
 3. Model selection — Pre-trained vs fine-tuned vs from scratch. Architecture choice.
 4. Training        — Hardware, frequency, experiment tracking, evaluation gates
 5. Serving         — Real-time vs batch vs streaming. API design.
 6. Monitoring      — Accuracy, latency, drift, fairness, business metrics
 7. Scaling         — Auto-scaling, GPU management, cost model
 8. Security        — Input validation, auth, adversarial defense, encryption
 9. Governance      — Versioning, bias audit, regulatory, model cards
10. Iteration       — Retraining triggers, A/B testing, continuous improvement
```

---

## Quick Links

| Chapter | Topic |
|:---|:---|
| [01 — Why](01_Why.md) | The human impact of deep learning |
| [02 — Concepts](02_Concepts.md) | Mental models, analogies, building blocks |
| [03 — Hello World](03_Hello_World.md) | 10 lines of code, 97% accuracy |
| [04 — How It Works](04_How_It_Works.md) | Training diagnostics, loss curves |
| [05 — Decisions](05_Decisions.md) | Every architectural choice with tradeoffs |
| [06 — Real World](06_Real_World.md) | Production case studies (Google Health, Tesla, Midjourney) |
| [07 — System Design](07_System_Design.md) | Scalability, cloud, CI/CD, distributed training |
| [08 — Security & Governance](08_Security_Governance.md) | Adversarial attacks, privacy, bias, compliance |
| [09 — Observability](09_Observability.md) | Monitoring, drift detection, debugging method |
| [10 — Checklist](10_Checklist.md) | This page |

---

**Hands-on:** [Deep Learning & PyTorch Notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Deep_Learning_PyTorch.ipynb) — Build everything from this material with working code.

**Project:** [P1.5 — Deep Learning Image Classifier](https://github.com/sunilmogadati/systems-in-production/blob/main/implementation/projects/P1_5_Deep_Learning_Image_Classifier.md) — Apply it: build 3 architectures on CIFAR-10, diagnose training, compare results.
