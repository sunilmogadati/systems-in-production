# Computer Vision

**How systems learn to see — and how to ship them in production.**

This playbook covers the architectures that turn pixels into predictions: CNNs (Convolutional Neural Networks), Vision Transformers (ViT), and the production systems built on top of them — from medical imaging to autonomous driving to manufacturing inspection.

## Foundations First

This playbook assumes you have read [Deep Learning](../deep-learning/) — perceptron, backprop, training loop, diagnostics. Computer vision adds spatial mechanics on top of those foundations; it does not replace them.

For math refreshers, see [Math for AI](../math-for-ai.md).

## Chapters

| # | Chapter | What You Learn |
|---|---|---|
| 01 | [Why](01_Why.md) | Why CV matters — the radiologist, the farmer, the inspector. Why MLP fails on images. Where CV fits in production. |
| 02 | [Concepts](02_Concepts.md) | Convolution, stride, padding, pooling, depth, 1x1 conv, hierarchical features, output shape formula. |
| 03 | [Hello World](03_Hello_World.md) | Build a working CNN in 30 lines of PyTorch. |
| 04 | [How It Works](04_How_It_Works.md) | Receptive field, training diagnostics for vision, transfer learning, augmentation. |
| 05 | [Building It](05_Building_It.md) | Architecture choices, loss/optimizer choices, data pipeline patterns, label quality. |
| 06 | [Production Patterns](06_Production_Patterns.md) | Tesla Autopilot HydraNet, Google retinal screening, Apple FaceID, manufacturing defect detection. |
| 07 | [System Design](07_System_Design.md) | Serving (Triton, ONNX, TensorRT), edge deployment, batching, GPU economics, multi-model. |
| 08 | [Quality, Security, Governance](08_Quality_Security_Governance.md) | Adversarial inputs, dataset bias, drift, regulatory compliance (medical, automotive). |
| 09 | [Observability & Troubleshooting](09_Observability_Troubleshooting.md) | Measuring CV model quality in production. Per-class metrics, confidence calibration, what to alert on. |
| 10 | [Decision Guide](10_Decision_Guide.md) | "Should I use CV here?" decision table. Production readiness checklist. |

## Architecture Deep Dives

| Doc | Coverage |
|---|---|
| [`architectures/resnet.md`](architectures/resnet.md) | ResNet — residual blocks, identity vs projection shortcuts, basic vs bottleneck variants, the cross-architecture additive-shortcut pattern |
| [`architectures/u-net.md`](architectures/u-net.md) | U-Net — encoder-decoder with skip connections, semantic segmentation, also the backbone of diffusion models |
| `architectures/vision-transformers.md` (coming) | ViT, hybrids (ConvNeXt + attention), when ViT beats CNN |
| `architectures/object-detection.md` (coming) | YOLO, R-CNN family, DETR |

## Hands-On Notebooks

| Notebook | What It Does |
|---|---|
| [Computer_Vision_From_Scratch.ipynb](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Computer_Vision_From_Scratch.ipynb) | Manual convolution by hand in pure NumPy. Single-slide, full feature map, pooling, multi-filter, parameter counting. PyTorch `F.conv2d` verification. |
| [Computer_Vision_CNN.ipynb](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Computer_Vision_CNN.ipynb) | Full CNN in PyTorch on MNIST. Convolution, pooling, training loop, evaluation. |
| [Deep_Learning_Regularization.ipynb](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Deep_Learning_Regularization.ipynb) | Dropout, L2, batch norm, augmentation — the regularizers that fight overfitting. Used by [04 — How It Works](04_How_It_Works.md). |

[LinkedIn](https://linkedin.com/in/sunilmogadati) · [Community](https://www.skool.com/deliverymomentum) · [Engagement inquiries](https://calendly.com/sunil-mogadati/connect)
