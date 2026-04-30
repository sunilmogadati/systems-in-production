# Computer Vision — How It Works

**Receptive field, what filters actually learn, training diagnostics for vision, augmentation, transfer learning, and the bugs that only happen with images.**

---

## The Receptive Field — How Much of the Image Each Neuron Sees

A single 3x3 filter sees only 9 pixels. So how does the network ever recognize a whole face?

Answer: **the receptive field grows with depth.** A neuron in Layer 5 sees a small patch from Layer 4. But that Layer 4 patch is itself the result of looking at a bigger patch from Layer 3. By the time you reach Layer 8 or 10, one neuron's receptive field covers most of the original image.

### Calculating Receptive Field

For a stack of conv layers (no pooling), the receptive field grows by `(F - 1)` per layer, where F is the filter size:

| Layer | Filter | Receptive Field on Input |
|---|---|---|
| Conv 1 | 3×3 | **3×3** |
| Conv 2 | 3×3 | **5×5** |
| Conv 3 | 3×3 | **7×7** |
| Conv 4 | 3×3 | **9×9** |

Each 3x3 conv adds 2 pixels to the receptive field on each side. Five stacked 3x3 convs see an 11x11 region — equivalent to a single 11x11 conv but with **far fewer parameters**.

**Pooling doubles the receptive field jump.** A 2x2 max-pool with stride 2 means the next conv layer sees a region twice as big in each direction.

| Layer | Operation | Receptive Field |
|---|---|---|
| Conv 1 (3×3) | — | 3×3 |
| Pool 1 (2×2, stride 2) | doubles spatial | 4×4 |
| Conv 2 (3×3) | — | 8×8 |
| Pool 2 (2×2, stride 2) | doubles spatial | 10×10 |
| Conv 3 (3×3) | — | 18×18 |

So in your Hello World network from [Chapter 03](03_Hello_World.md), by the second pool, each neuron sees ~10x10 pixels of the original 28x28 image — about a third of the digit. By the dense layer, the network sees the whole image.

> **The principle.** Stacking small filters with pooling buys you a large effective receptive field cheaply. This is why VGG and ResNet exclusively use 3x3 filters — they stack to whatever receptive field they need.

---

## What Filters Actually Learn

Train a CNN on faces. Visualize the filters. You will see a hierarchy emerge with no human telling the network what to look for.

| Layer | What the Filters Look Like |
|---|---|
| Layer 1 | Oriented edges (vertical, horizontal, diagonal at various angles), color blobs, simple gradients |
| Layer 2-3 | Corners, T-junctions, simple textures (stripes, dots), curves |
| Layer 4-6 | Object parts — eyes, noses, ears, fur patches, wheels, leaves |
| Layer 7+ | Whole objects — face configurations, full cars, dog breeds |

Tools like **Grad-CAM** and **DeepDream** let you visualize what specific neurons respond to. The hierarchy is consistent across architectures and datasets — train any CNN on natural images and Layer 1 will rediscover edge detectors. The features the network learns are not arbitrary; they are the features the data demands.

> **Why this matters in production.** When debugging a model, "Layer 3 filter 7 is responding to" is a real diagnostic. If your medical CNN's Layer 5 filter learned to respond to **the metadata text printed at the corner of the X-ray** (a real, published failure mode) instead of the lung tissue, the model is exploiting a shortcut. Visualizing what filters respond to is how you catch this.

---

## Reading a Vision Loss Curve

The general rules from [Deep Learning → How It Works](../deep-learning/04_How_It_Works.md) apply: healthy/overfitting/underfitting/diverging are universal. But vision-specific patterns to know:

| Symptom | Likely Cause | Fix |
|---|---|---|
| **Train loss drops to ~0, test loss explodes after epoch 5** | Overfitting on small dataset (common with under 10k images) | Add data augmentation; use transfer learning; reduce capacity |
| **Both losses plateau immediately at a high value** | Wrong normalization; learning rate too low; model on wrong device | Print `images.mean()` and `images.std()` — should be near 0 and 1 after normalization |
| **Loss is NaN after first batch** | Learning rate too high; data has NaN/Inf; un-normalized data with sigmoid output | Reduce learning rate by 10x; check `torch.isnan(images).any()` |
| **First few epochs show no improvement, then sudden drop** | "Warmup" phase common for deep CNNs and transformers | Normal — keep training. If it never improves, the architecture or LR is wrong. |
| **Training accuracy 99%, test accuracy 60%, but loss curves looked similar** | Train/test data leakage — duplicates between sets | Verify no overlap. Image hashes can help. |

---

## Data Augmentation — The Free Data Multiplier

You have 1,000 labeled images. You need 100,000. Augmentation is how you bridge the gap.

**Augmentation creates new training examples by applying small, label-preserving transformations to existing images.** The model sees more variations. It generalizes better. No new labels required.

### The Standard Augmentation Set

| Augmentation | What It Does | When to Use |
|---|---|---|
| **Random horizontal flip** | Mirror image left-to-right | Natural images, animals, objects. **Avoid for digits and letters** (a flipped 3 is not a 3). |
| **Random crop + resize** | Take a smaller crop, scale up | Almost always. Teaches the model that objects can be partially in frame. |
| **Random rotation** | Rotate by ±10-30° | Most natural images. **Avoid for digits** (a rotated 6 looks like a 9). |
| **Color jitter** | Random brightness, contrast, saturation, hue | Outdoor / lighting-variable scenes. Skip for medical (where color is meaningful). |
| **Random erasing / Cutout** | Black out a random rectangle | Forces robustness to occlusion. Cheap and effective. |
| **MixUp** | Blend two images linearly with weighted labels | Strong regularizer. Used in modern training recipes. |
| **CutMix** | Paste a rectangle from one image into another, mix labels accordingly | Modern default for ImageNet-scale training. |

### Augmentation Code (PyTorch)

```python
from torchvision import transforms

train_transform = transforms.Compose([
    transforms.RandomResizedCrop(224),
    transforms.RandomHorizontalFlip(),
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),  # ImageNet stats
])

# Test transform — NO augmentation, just resize and normalize
test_transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])
```

> **Critical rule.** Augmentation only happens at training time. Never augment the test set. Mix this up and your test accuracy becomes meaningless because the test set keeps changing.

> **Hands-on companion.** [Deep Learning Regularization on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Deep_Learning_Regularization.ipynb) — runnable examples of dropout, L2, batch normalization, and data augmentation, with before/after loss curves showing the regularization effect.

---

## Transfer Learning — The Production Cheat Code

This is the most important practical technique in computer vision. **You will use it on almost every real CV project.**

### The Idea

A CNN trained on ImageNet (14 million images, 22,000 classes) has learned features useful for *almost any* vision task — edges, textures, shapes, parts. You can:

1. Take the pretrained network
2. Throw away the final classification layer
3. Replace it with a new layer for your specific task
4. Fine-tune on your (small) dataset

You inherit the feature hierarchy from a model that saw millions of images. You only need a few hundred labeled examples to specialize it.

### Three Strategies

| Strategy | Frozen Layers | When to Use |
|---|---|---|
| **Feature extraction** | Freeze all conv layers, only train the new classifier | Tiny dataset (< 500 images), domain very similar to ImageNet |
| **Fine-tuning** | Train all layers, but with a small learning rate on the conv layers | Medium dataset (1k-100k), domain somewhat different |
| **From scratch** | Train everything from random initialization | Large dataset (> 1M), domain very different (e.g., satellite imagery, medical with unusual modalities) |

### Transfer Learning Code

```python
import torch
import torchvision.models as models
import torch.nn as nn

# Load a CNN pretrained on ImageNet
model = models.resnet18(weights='IMAGENET1K_V1')

# Replace the final layer for your task (e.g., 10 classes)
num_features = model.fc.in_features
model.fc = nn.Linear(num_features, 10)

# Strategy 1: Feature extraction — freeze everything except the new layer
for param in model.parameters():
    param.requires_grad = False
for param in model.fc.parameters():
    param.requires_grad = True

# Strategy 2: Fine-tuning — small LR for old layers, larger LR for new layer
optimizer = torch.optim.Adam([
    {'params': [p for n, p in model.named_parameters() if 'fc' not in n], 'lr': 1e-5},
    {'params': model.fc.parameters(), 'lr': 1e-3},
])
```

### Why Transfer Learning Works So Well

The features learned from ImageNet are almost universal. Edges are edges. Textures are textures. Even medical X-rays have the same low-level structure as natural images. **The first 80% of the network is the same regardless of the task.** Only the last 20% is task-specific. Fine-tuning lets you reuse that 80% for free.

This is why a small startup with 1,000 labeled defect images can ship a production-quality system in weeks, not years. Without transfer learning, every CV project would need ImageNet-scale data. With it, the data barrier collapses.

---

## Common CV-Specific Bugs

Beyond the universal "is the data loaded correctly?" debugging from [Deep Learning → 04](../deep-learning/04_How_It_Works.md#the-debugging-checklist--when-the-model-is-not-learning), vision has its own failure modes.

### Channel Order — RGB vs BGR

OpenCV reads images as BGR (Blue, Green, Red). PyTorch and TensorFlow expect RGB. If you mix the two, the model sees scrambled colors.

```python
import cv2
img = cv2.imread('cat.jpg')        # BGR by default
img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)   # convert to RGB before passing to model
```

**Symptom.** Model trained perfectly, deploys fine on PIL/torchvision, but accuracy collapses when serving with OpenCV.

### Normalization Mismatch Between Train and Inference

The `transforms.Normalize(mean, std)` you used at training time MUST be applied at inference time. With the same mean and std.

**Symptom.** Model gets 95% on test set, gets 30% on production data. Check: are the production images normalized with the same stats?

### Image Size Mismatch

A model trained on 224x224 expects 224x224 at inference. Send 256x256 or 200x200 and the model works (PyTorch will not error) but produces nonsense outputs because the receptive field math breaks.

**Symptom.** Mysterious accuracy drop in production. Check: are inference images resized to the training input size?

### Label Leakage Through Filenames

Your training data folder is `data/train/cat/` and `data/train/dog/`. The labels are in the folder name. If the inference pipeline accidentally sees the file path, the model "predicts" with 100% accuracy by reading the filename.

**Symptom.** Test accuracy looks too good. Check: does inference truly receive only pixels?

### Class Imbalance

You have 10,000 images labeled "healthy" and 200 labeled "tumor." A CNN that always predicts "healthy" gets 98% accuracy and zero clinical value.

**Fix.** Use weighted loss (`pos_weight` in BCEWithLogitsLoss), oversample the minority class, or use focal loss. Track per-class metrics, not just overall accuracy.

---

## A Vision Debugging Workflow

When a CV model is not learning or is performing poorly, work through these in order:

| # | Check | How |
|:---:|---|---|
| 1 | **Visualize a batch.** Are the images actually correct? Are the labels matching? | `plt.imshow(images[0].permute(1,2,0))` — does it look like the right class? |
| 2 | **Check normalization.** | `images.mean()` ≈ 0, `images.std()` ≈ 1 after the transform |
| 3 | **Check shape flow.** Print `x.shape` at every layer. Does the flatten size match `nn.Linear` input? | `print(x.shape)` between layers |
| 4 | **Test on a tiny subset.** Can the model overfit 100 images? If not, the architecture or pipeline is broken. | Train for 50 epochs on 100 images. Loss should hit ~0. |
| 5 | **Visualize predictions.** Plot 10 wrong predictions. Are they obvious failures or subtle ones? | Look for systematic errors (always confuses 3 with 8, etc.) |
| 6 | **Check class balance.** | `Counter(labels)` |
| 7 | **Try transfer learning.** If training from scratch is failing, ResNet pretrained will probably work. | Switch to `models.resnet18(weights='IMAGENET1K_V1')` |

> **The 100-image test is the most underused diagnostic in CV.** A working architecture should always be able to memorize 100 images. If it cannot, the bug is upstream of the architecture choice — in data, normalization, or the training loop itself.

---

**Next:** [05 — Building It](05_Building_It.md) — Architecture choices (ResNet vs EfficientNet vs ViT), optimizer choices, data pipeline patterns, when to fine-tune vs train from scratch.
