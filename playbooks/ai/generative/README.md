# Generative Models

**Models that create — images, audio, video, text. GAN, VAE, Diffusion, and the production systems built on top of them.**

This playbook covers the architectures that turn random noise (or text, or other conditioning) into novel data: GAN (Generative Adversarial Network), VAE (Variational Autoencoder), Diffusion. Plus neural style transfer as a bridge between vision and generation, and U-Net as the architectural backbone shared by segmentation and diffusion.

## Foundations First

This playbook assumes you have read:

- [Deep Learning](../deep-learning/) — perceptron, backprop, training loop. Generative models train with the same loop, just with two networks (GAN) or a probabilistic objective (VAE) or an iterative denoising objective (Diffusion).
- [Computer Vision](../computer-vision/) — CNN, convolution, transposed convolution. Most modern generative models use CNN-based generators/decoders/U-Nets.

For math refreshers, see [Math for AI](../math-for-ai.md).

## Chapters

| # | Chapter | What You Learn |
|---|---|---|
| 01 | [Why](01_Why.md) | The case for generative — synthetic data, content creation, scientific simulation. Why this is the hottest area of AI in 2026. |
| 02 | [Concepts](02_Concepts.md) | Generator/Discriminator, latent space, the minimax game, KL divergence (VAE), forward/reverse process (Diffusion). The three families compared. |
| 03 | [Hello World](03_Hello_World.md) | SimpleGAN on MNIST in PyTorch. Generate digits in 50 lines. |
| 04 | [How It Works](04_How_It_Works.md) | Training instability, mode collapse, vanishing gradients, evaluation metrics (FID, IS, CLIP). Diagnostic skills for generative training. |
| 05 | [Building It](05_Building_It.md) | GAN vs VAE vs Diffusion — when to use which. Architecture choices. Training recipes. |
| 06 | [Production Patterns](06_Production_Patterns.md) | Stable Diffusion, Midjourney, Sora, Synthesia, NVIDIA StyleGAN, DeepFake detection. Real deployments, real costs. |
| 07 | [System Design](07_System_Design.md) | Serving generative models — latency, prompt handling, batching, caching, GPU economics. |
| 08 | [Quality, Security, Governance](08_Quality_Security_Governance.md) | Deepfakes, watermarking, content provenance (C2PA), copyright, safety filters, NSFW detection. |
| 09 | [Observability & Troubleshooting](09_Observability_Troubleshooting.md) | Measuring generative quality at scale. FID/CLIP/perceptual metrics in production. |
| 10 | [Decision Guide](10_Decision_Guide.md) | "Should I generate or retrieve?" decision table. GAN vs VAE vs Diffusion vs API. |

## Architecture Deep Dives

| Doc | Coverage |
|---|---|
| [`architectures/gan.md`](architectures/gan.md) | GAN architecture deep dive — concepts, code (SimpleGAN + DCGAN), math (BCE losses, parameter counts), Q&A |
| `architectures/vae.md` (coming) | Vanilla VAE, conditional VAE, β-VAE, VQ-VAE |
| `architectures/diffusion.md` (coming) | DDPM, DDIM, Latent Diffusion, classifier-free guidance |
| `architectures/style-transfer.md` (coming) | Neural style transfer (Gatys), fast style transfer, modern alternatives |
| **U-Net** (used as the diffusion backbone) | See [`computer-vision/architectures/u-net.md`](../computer-vision/architectures/u-net.md) — single canonical home |

## Hands-On Notebooks

| Notebook | What It Does |
|---|---|
| [Generative_Autoencoder_From_Scratch.ipynb](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Generative_Autoencoder_From_Scratch.ipynb) | Autoencoder by hand in pure NumPy. Forward + backward through two networks, every gradient computed manually, PyTorch autograd verification, latent space visualization. **Start here.** |
| [Deep_Learning_Autoencoders_GANs.ipynb](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Deep_Learning_Autoencoders_GANs.ipynb) | PyTorch implementations of autoencoder, SimpleGAN, and DCGAN on MNIST. |

[Sunil Mogadati](https://www.linkedin.com/in/sunilmogadati) | [Community](https://www.skool.com/deliverymomentum)
