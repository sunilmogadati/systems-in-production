# AI Playbooks

**How to add intelligence to production systems. From prediction to autonomous action.**

Each playbook follows the same 10-chapter structure: Why → Concepts → Hello World → How It Works → Building It → Production Patterns → System Design → Quality / Security / Governance → Observability → Decision Guide. Each has a companion from-scratch notebook with PyTorch / scikit-learn verification.

---

## Foundations

| Playbook | Coverage |
|---|---|
| [Machine Learning](ml/) | Prediction, classification, anomaly detection (21 algorithms) |
| [Deep Learning](deep-learning/) | Neural network foundations — perceptron, backprop, training loop, diagnostics |

## Architecture and Task Domain

| Playbook | Coverage |
|---|---|
| [Computer Vision](computer-vision/) | Vision tasks — CNN, ViT, detection, segmentation. Architecture deep dives for ResNet and U-Net. |
| [Generative](generative/) | Generation — GAN, VAE, Diffusion. Architecture deep dive for GAN. |
| [Sequence Models](sequence-models/) | RNN, LSTM, GRU. Mostly legacy for NLP — still alive in time-series and streaming. Architecture deep dive for RNN/LSTM. |
| [Transformers](transformers/) | Self-attention, encoder/decoder, GPT/BERT — the architecture behind every modern LLM. Architecture deep dive for the Transformer. |
| [NLP](nlp/) | Task-domain entry point for language: classification, NER, generation, translation, embeddings. |

## System Patterns

| Playbook | Coverage |
|---|---|
| [RAG](rag/) | Retrieval-augmented generation — AI grounded in your organization's data |
| [Agents](agents/) | Autonomous AI: ReAct, tool use, multi-step reasoning |

---

## Shared References

| Doc | Purpose |
|---|---|
| [Math for AI](math-for-ai.md) | Conceptual math primer — derivatives, chain rule, gradient descent, dot products. Cross-referenced from the playbooks. |
| [Architecture Glossary](architecture-glossary.md) | Cross-architecture terminology — GAN, RNN/LSTM, U-Net, ResNet, Transformer. Single source for term definitions. |
| [Architecture Math](architecture-math.md) | Worked numerical examples — parameter counting, shape arithmetic, forward-pass numerics across all DL architectures. Companion to Math for AI. |
| [Architecture Reference Card](../../resources/architecture-reference.md) | One-page printable reference — foundations + per-architecture quick-lookup tables. |

---

## Notebooks Index

All notebooks open in Colab. From-scratch notebooks compute every gradient / number by hand and verify against PyTorch or scikit-learn.

### From-Scratch (Math by Hand + Verification)

| Notebook | Playbook / Chapter | Coverage |
|---|---|---|
| [Deep_Learning_From_Scratch](../../implementation/notebooks/Deep_Learning_From_Scratch.ipynb) | [Deep Learning → 02 Concepts](deep-learning/02_Concepts.md) | Pure NumPy MLP. Two hidden neurons, three houses, every gradient by hand. PyTorch autograd verification. |
| [Computer_Vision_From_Scratch](../../implementation/notebooks/Computer_Vision_From_Scratch.ipynb) | [Computer Vision → 02 Concepts](computer-vision/02_Concepts.md) | Manual convolution by NumPy. Single slide, full feature map, pooling, multi-filter, parameter counting. PyTorch `F.conv2d` verification. |
| [Generative_Autoencoder_From_Scratch](../../implementation/notebooks/Generative_Autoencoder_From_Scratch.ipynb) | [Generative → 02 Concepts](generative/02_Concepts.md) | Autoencoder by hand. Encode/decode, full backward through two networks, latent space visualization. PyTorch verification. |
| [Sequence_Models_From_Scratch](../../implementation/notebooks/Sequence_Models_From_Scratch.ipynb) | [Sequence Models → 02 Concepts](sequence-models/02_Concepts.md) | Vanilla RNN by hand. Forward through time, full BPTT backward, weight-sharing accumulation, vanishing-gradient demonstration. PyTorch verification. |
| [Transformer_From_Scratch](../../implementation/notebooks/Transformer_From_Scratch.ipynb) | [Transformers → 02 Concepts](transformers/02_Concepts.md) | Self-attention by hand. Q/K/V projection, scaled dot-product, softmax, multi-head decomposition, causal masking, positional encoding. PyTorch verification. |
| [NLP_From_Scratch](../../implementation/notebooks/NLP_From_Scratch.ipynb) | [NLP → 02 Concepts](nlp/02_Concepts.md) | Classical NLP by hand: BPE tokenization, TF-IDF, naive Bayes, Word2Vec-style embeddings via PCA. scikit-learn verification. |
| [DL_Architecture_Examples](../../implementation/notebooks/DL_Architecture_Examples.ipynb) | [Architecture Math](architecture-math.md) | Runnable numerical examples — parameter counts and shape arithmetic for GAN, RNN, LSTM, U-Net, ResNet, Transformer. |

### Full Implementation in PyTorch / Frameworks

| Notebook | Playbook / Chapter | Coverage |
|---|---|---|
| [Deep_Learning_PyTorch](../../implementation/notebooks/Deep_Learning_PyTorch.ipynb) | [Deep Learning → 03 Hello World](deep-learning/03_Hello_World.md) | Full MLP and CNN in PyTorch. Training loop, diagnostics, evaluation. |
| [Deep_Learning_Regularization](../../implementation/notebooks/Deep_Learning_Regularization.ipynb) | [Deep Learning → 05 Building It](deep-learning/05_Building_It.md), [Computer Vision → 04 How It Works](computer-vision/04_How_It_Works.md) | Dropout, L2, batch norm, augmentation — runnable comparisons with loss curves. |
| [Computer_Vision_CNN](../../implementation/notebooks/Computer_Vision_CNN.ipynb) | [Computer Vision → 03 Hello World](computer-vision/03_Hello_World.md) | Full CNN in PyTorch on MNIST. Convolution, pooling, full pipeline, training, evaluation. |
| [Deep_Learning_Autoencoders_GANs](../../implementation/notebooks/Deep_Learning_Autoencoders_GANs.ipynb) | [Generative → 03 Hello World](generative/03_Hello_World.md) | PyTorch autoencoder, SimpleGAN, and DCGAN on MNIST. |
| [Multimodal_AI](../../implementation/notebooks/Multimodal_AI.ipynb) | (cross-cutting) | Vision-language pipelines. |

### System Patterns

| Notebook | Playbook / Chapter | Coverage |
|---|---|---|
| [RAG_from_Scratch](../../implementation/notebooks/RAG_from_Scratch.ipynb) | [RAG → 01 Why](rag/01_Why.md) | RAG system end-to-end with Ollama. Local, no API keys. |
| [Agents](../../implementation/notebooks/Agents.ipynb) | [Agents → 01 Why](agents/01_Why.md) | ReAct, tool use, multi-step reasoning. |

---

[LinkedIn](https://linkedin.com/in/sunilmogadati) · [Community](https://www.skool.com/deliverymomentum) · [Engagement inquiries](https://calendly.com/sunil-mogadati/connect)
