# ResNet (Residual Networks)

This reference covers ResNet — the CNN architecture that broke past ~22 layers and made 50–152-layer networks trainable. The single most-deployed CNN family in production today, and the source of an idea (additive residual connections) that now appears in LSTM cell updates, Transformer residual streams, and most modern architectures.

For shared CNN concepts (convolution, pooling, parameter counting), see [Computer Vision → 02 Concepts](../02_Concepts.md). For terminology shared with other architectures, see [Architecture Glossary](../../architecture-glossary.md). For runnable parameter-count and shape-arithmetic examples (including ResNet), see [Architecture Math](../../architecture-math.md).

---

## 1. The Problem ResNet Solves — Degradation

Adding more layers to a plain deep CNN makes accuracy *worse*, even on the training set. This is **not overfitting** — it is an *optimization* problem. A plain 56-layer CNN performs worse than a plain 20-layer CNN on the same task. The deeper network has strictly more capacity, but training cannot exploit it because gradients vanish and the optimizer cannot navigate the deeper loss surface.

ImageNet ILSVRC progression illustrates the depth wall:

| Year | Architecture | Depth | Top-5 Error |
|---|---|---:|---:|
| 2012 | AlexNet | 8 layers | 16.4% |
| 2014 | VGG-19 | 19 layers | 7.3% |
| 2014 | GoogLeNet (Inception) | 22 layers | 6.7% |
| **2015** | **ResNet-152** | **152 layers** | **3.57%** |

ResNet (He, Zhang, Ren, Sun — Microsoft Research, 2015) introduced **residual connections** to make networks 50–152 layers deep trainable.

---

## 2. The Residual Block

The core idea is a wire from input directly to output, bypassing some layers:

```
Input x ─────────────────────┐
   │                          │
   ↓                          │
   Conv 3×3 → BN → ReLU       │  (this is F(x))
   ↓                          │
   Conv 3×3 → BN              │
   ↓                          ↓
   ────────── + ──────────────
   ↓
   ReLU
   ↓
Output y = F(x) + x
```

The block learns the **residual** `F(x) = H(x) − x` — the *difference* between the desired output and the input — rather than the full mapping `H(x)`.

### Why This Helps Optimization

If the optimal layer is the identity (do nothing), the residual block can learn `F(x) = 0` trivially — this is the easy default. **Adding more blocks can never hurt**, because the network can always set them to identity. A plain CNN has no such guarantee — adding layers can make optimization harder.

### Why This Helps Gradient Flow

The gradient of the loss with respect to the input `x`:

```
∂L/∂x = ∂L/∂y · (1 + ∂F/∂x)
```

The `+1` is a **parameter-free direct path** for the gradient back to `x`. Even if `∂F/∂x ≈ 0` (the layers in the block contribute little), the gradient still flows through the shortcut. **No vanishing through the residual stream** — the central reason ResNet trains 152 layers where plain CNNs cap at ~20.

---

## 3. Identity vs Projection Shortcuts

The shortcut `+ x` only works if the input and output have the same shape. Two cases:

| Shortcut Type | When | Implementation |
|---|---|---|
| **Identity** | Input and output have the same channel count and spatial size | `y = F(x) + x` — pure addition, **zero parameters** |
| **Projection** | Channel count or spatial size changes (e.g., stride 2 downsampling, channel doubling) | `y = F(x) + W_s · x`, where `W_s` is a 1×1 convolution to match shapes |

The projection shortcut adds a small number of parameters (a 1×1 conv plus optional BN). The identity shortcut adds none.

---

## 4. Block Variants

### Basic Block — Used in ResNet-18 and ResNet-34

```
Conv 3×3 → BN → ReLU → Conv 3×3 → BN → (+ shortcut) → ReLU
```

Two stacked 3×3 convolutions with a shortcut around them.

### Bottleneck Block — Used in ResNet-50, 101, 152

```
Conv 1×1 → BN → ReLU → Conv 3×3 → BN → ReLU → Conv 1×1 → BN → (+ shortcut) → ReLU
```

A 1×1 convolution first reduces the channel count (e.g., 256 → 64), then a 3×3 convolution operates on the reduced representation, then a final 1×1 convolution restores the channel count (64 → 256). The 3×3 convolution — the expensive part — sees fewer channels, making the block much cheaper at the same effective channel width.

**Why bottleneck.** At a given channel width (e.g., 256), a basic block uses ~1.18M parameters. A bottleneck block at the same effective width uses ~70K. **~17× fewer parameters** for similar capacity. This is why deep ResNet variants (50/101/152) all use bottleneck blocks.

---

## 5. The Full ResNet Architecture Pattern

Every ResNet variant follows the same pattern:

```
Input image
  ↓
Conv 7×7, stride 2  (initial wide-receptive-field convolution)
  ↓
MaxPool 3×3, stride 2
  ↓
Stage 1: residual blocks at 64 channels
  ↓
Stage 2: residual blocks at 128 channels (first block uses stride 2 to downsample)
  ↓
Stage 3: residual blocks at 256 channels (first block uses stride 2)
  ↓
Stage 4: residual blocks at 512 channels (first block uses stride 2)
  ↓
Global Average Pool
  ↓
Linear → softmax over class scores
```

**The pattern:** periodically double the number of filters and use stride 2 in the first block of each stage to downsample. Within a stage, all blocks have the same channel count and spatial size, with identity shortcuts.

The variants differ in the number of blocks per stage:

| Variant | Stage 1 | Stage 2 | Stage 3 | Stage 4 | Total |
|---|---:|---:|---:|---:|---:|
| ResNet-18 (basic) | 2 | 2 | 2 | 2 | 18 |
| ResNet-34 (basic) | 3 | 4 | 6 | 3 | 34 |
| ResNet-50 (bottleneck) | 3 | 4 | 6 | 3 | 50 |
| ResNet-101 (bottleneck) | 3 | 4 | 23 | 3 | 101 |
| ResNet-152 (bottleneck) | 3 | 8 | 36 | 3 | 152 |

---

## 6. Worked Parameter Counts

### Basic Block — 64 in/out, no bias, with BN

```
Conv1 (3×3, 64→64, no bias):    9 · 64 · 64   = 36,864
BN1:                             2 · 64        =     128
Conv2 (3×3, 64→64, no bias):    9 · 64 · 64   = 36,864
BN2:                             2 · 64        =     128
Identity shortcut:                              =       0  (parameter-free)
TOTAL                                            =  73,984
```

The shortcut adds zero parameters because the shapes match.

### Basic Block with Projection Shortcut — 64 → 128, stride 2

When the block changes channel count from 64 to 128 with stride 2:

```
Conv1 (3×3, 64→128, s=2, no bias):     9 · 64 · 128   =  73,728
BN1:                                     2 · 128        =     256
Conv2 (3×3, 128→128, no bias):         9 · 128 · 128  = 147,456
BN2:                                     2 · 128        =     256
Projection shortcut (1×1, 64→128, s=2): 64 · 128       =   8,192
BN_shortcut:                             2 · 128        =     256
TOTAL                                                    = 230,144
```

The projection shortcut adds ~8.4K parameters — the cost of the shape change.

### Bottleneck Block — C=256

```
Conv 1×1 (256→64):     1·1·256·64    = 16,384
BN:                     2 · 64         =    128
Conv 3×3 (64→64):      9 · 64 · 64    = 36,864
BN:                     2 · 64         =    128
Conv 1×1 (64→256):     1·1·64·256    = 16,384
BN:                     2 · 256        =    512
Identity shortcut:                       =      0
TOTAL                                    = 70,400
```

Compare against a basic block at the same channel count (256): `2 · (9·256²) + 2·(2·256) = 1,180,672 + 1,024 = 1,181,696` parameters. **The bottleneck saves ~17×** at equal effective width.

For more worked examples including additional ResNet variants, see [Architecture Math → Type 1: Parameter Counting](../../architecture-math.md#type-1-parameter-counting).

---

## 7. The Cross-Architecture Idea — Additive Shortcuts Are Everywhere

ResNet's residual connection is a specific instance of a much broader pattern: **add the input back to the output to preserve gradient flow**. The same idea appears in:

| Architecture | Skip Operation | Where in the Network |
|---|---|---|
| **ResNet** | `F(x) + x` (add) | Within each block, same shape |
| **LSTM** | `C_t = f_t ⊙ C_{t-1} + i_t ⊙ C̃_t` (add) | Cell state through time |
| **Transformer** | `x + Sublayer(x)` (add) | Residual stream around MHA and FFN |
| **U-Net** | `torch.cat([up, skip], dim=1)` (**concatenate**) | Encoder → decoder, cross-block |

ResNet, LSTM, and Transformer all use **additive** shortcuts. They preserve gradient flow through depth (ResNet), through time (LSTM), and through both (Transformer).

U-Net is the exception — it **concatenates** rather than adds. The goal is different: U-Net fuses encoder and decoder feature maps to preserve spatial detail; ResNet preserves gradient flow within a block. See [U-Net](u-net.md) for that architecture.

This pattern — additive shortcuts to preserve gradient flow through depth — is one of the most important architectural ideas in modern deep learning. It is what makes 100+-layer networks trainable.

---

## 8. Common Questions

**What problem does ResNet solve?** Degradation in deep networks. Plain CNNs past ~20 layers train *worse*, not better. ResNet enables 100+ layer networks to train successfully.

**Why does the residual connection help training?** The gradient `∂L/∂x = ∂L/∂y · (1 + ∂F/∂x)` has a direct unattenuated path back via the `+1` term. Even if the residual `∂F/∂x ≈ 0`, gradient still flows.

**Does the shortcut add parameters?** No, for identity shortcuts (when shapes match). Only projection shortcuts (when channel count or spatial size changes) add a small 1×1 conv.

**Compare ResNet's skip with U-Net's skip.** ResNet adds (within block, gradient flow). U-Net concatenates (cross-block, spatial detail). Different operations, different purposes.

**Compare ResNet to LSTM.** Both use additive shortcuts. ResNet preserves gradient through depth; LSTM through time. Both make deep / long-sequence training possible by giving gradients a direct path back.

**Why does ResNet use BatchNorm everywhere?** Standard CNN regularizer. Stabilizes activation distributions across the deep stack — without BN, the 100+-layer ResNets would not train reliably even with the residual shortcuts. BN is in every `Conv → BN → ReLU` block.

**When to choose ResNet variant?** ResNet-18/34 for small datasets and edge deployment. ResNet-50 is the workhorse for production fine-tuning. ResNet-101/152 when you have very large datasets and compute budget — diminishing returns past 50 layers for most tasks.

---

## 9. ResNet in Production (2026)

ResNet remains one of the two or three most deployed CNN architectures despite being a decade old. It is used as the **backbone** for object detectors (Faster R-CNN, Mask R-CNN), feature extractor for transfer learning, and baseline for almost any image-classification project.

For most vision projects in 2026:
- **Default starting point**: pretrained ResNet50, fine-tune on your data
- **When you need accuracy**: EfficientNet-B3 or ConvNeXt-Tiny (modernized CNNs that beat ResNet at the cost of more compute)
- **When you need speed at the edge**: MobileNetV3 or EfficientNet-Lite
- **When you have billions of images**: Vision Transformer (ViT) — see the [Transformers playbook](../../transformers/)

The residual connection idea has outlived ResNet itself. It now lives inside every Transformer block, every modern CNN, and most architectures shipped to production.
