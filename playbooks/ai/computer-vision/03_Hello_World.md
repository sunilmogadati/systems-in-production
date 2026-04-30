# Computer Vision — Hello World

**Before theory, see it work. A complete CNN that recognizes handwritten digits, in 30 lines of PyTorch.**

---

## Why Hello World First

Read enough theory and you start to forget that CNNs actually run. They are not philosophical objects. They are code. Eight lines of model definition, ten lines of training loop, two minutes of compute, 99% accuracy on MNIST.

This chapter shows you that CNN. Then [04 — How It Works](04_How_It_Works.md) explains *why* it works.

---

## What You Will Build

A CNN that reads handwritten digits (MNIST — 28x28 grayscale images of 0-9) and classifies them. Target accuracy: **98% or better.** Training time: **under 2 minutes on a CPU, seconds on a GPU.**

Architecture (the same pipeline from [02 — Concepts](02_Concepts.md#the-full-cnn-pipeline--one-picture)):

```
Input (28×28×1) → Conv 3×3 (8 filters) → ReLU → MaxPool 2×2
                → Conv 3×3 (16 filters) → ReLU → MaxPool 2×2
                → Flatten → Dense (128) → ReLU → Dense (10) → Softmax
```

---

## The Code

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

# === DATA ===
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))   # MNIST mean and std
])
train_data = datasets.MNIST('./data', train=True, download=True, transform=transform)
test_data  = datasets.MNIST('./data', train=False, transform=transform)
train_loader = DataLoader(train_data, batch_size=64, shuffle=True)
test_loader  = DataLoader(test_data, batch_size=1000)

# === MODEL ===
class CNN(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(1, 8, kernel_size=3, padding=1)    # 28×28×1 → 28×28×8
        self.conv2 = nn.Conv2d(8, 16, kernel_size=3, padding=1)   # 14×14×8 → 14×14×16
        self.fc1   = nn.Linear(16 * 7 * 7, 128)                   # flatten → 128
        self.fc2   = nn.Linear(128, 10)                           # 128 → 10 classes

    def forward(self, x):
        x = F.max_pool2d(F.relu(self.conv1(x)), 2)   # 28×28×8 → 14×14×8
        x = F.max_pool2d(F.relu(self.conv2(x)), 2)   # 14×14×16 → 7×7×16
        x = x.view(x.size(0), -1)                    # flatten
        x = F.relu(self.fc1(x))
        return self.fc2(x)

# === TRAIN ===
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = CNN().to(device)
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
loss_fn = nn.CrossEntropyLoss()

for epoch in range(3):
    for images, labels in train_loader:
        images, labels = images.to(device), labels.to(device)
        predictions = model(images)              # 1. Forward
        loss = loss_fn(predictions, labels)      # 2. Loss
        optimizer.zero_grad()                    # 3. Zero gradients
        loss.backward()                          # 4. Backward
        optimizer.step()                         # 5. Update
    print(f"Epoch {epoch+1}: loss = {loss.item():.4f}")

# === EVALUATE ===
model.eval()
correct = 0
with torch.no_grad():
    for images, labels in test_loader:
        images, labels = images.to(device), labels.to(device)
        preds = model(images).argmax(dim=1)
        correct += (preds == labels).sum().item()

print(f"Test accuracy: {100 * correct / len(test_data):.2f}%")
```

That is the entire system. **30 lines for the model + training loop, 10 lines for data and evaluation.**

---

## What You Should See

```
Epoch 1: loss = 0.0743
Epoch 2: loss = 0.0489
Epoch 3: loss = 0.0312

Test accuracy: 98.84%
```

Three epochs. ~99% accuracy. ~16,000 parameters total.

---

## Compare to MLP — Why CNN Wins

Train the same task with a fully-connected MLP (Multi-Layer Perceptron) of similar parameter count:

| Architecture | Parameters | Test Accuracy |
|---|---:|---:|
| **MLP** (784 → 128 → 10) | 101,770 | ~96.5% |
| **CNN** (above) | ~16,400 | **~98.8%** |

The CNN has **6x fewer parameters** and **higher accuracy.** That is the weight-sharing dividend in numbers.

The gap widens dramatically on harder vision tasks. On CIFAR-10 (32x32 color photos):

| Architecture | Test Accuracy |
|---|---:|
| MLP | ~53% (basically guessing for hard classes) |
| CNN | ~78% (realistic baseline) |
| Modern CNN with augmentation | ~95% |

For real images (224x224 color, like ImageNet), MLP simply does not work. CNN is the floor, not the ceiling.

---

## What Just Happened — The Five Steps

The training loop you wrote is the same five steps from [Deep Learning → 02 Concepts → The Training Loop](../deep-learning/02_Concepts.md#the-training-loop--five-steps-you-will-use-for-everything):

| Step | Code | What It Does |
|---|---|---|
| **1. Forward** | `predictions = model(images)` | Image flows through conv layers, pools, flattens, dense layers — produces 10 scores per image |
| **2. Loss** | `loss = loss_fn(predictions, labels)` | Cross-entropy — how confidently wrong is the model? |
| **3. Zero gradients** | `optimizer.zero_grad()` | PyTorch accumulates by default; clear before each batch |
| **4. Backward** | `loss.backward()` | Autograd computes ∂L/∂w for every weight in every layer (conv + dense) |
| **5. Update** | `optimizer.step()` | Adam takes one gradient step on every weight |

The mechanics are identical to the MLP from the deep-learning playbook. **What changed is the architecture, not the training loop.** Convolution is a layer type. The loop does not care.

---

## What Could Go Wrong — Top 3 First-Run Bugs

### 1. Shape Mismatch at the Flatten Layer

The most common CNN bug. The flatten size depends on the input size and every conv/pool layer above it.

If you change the input to 32x32 but leave `nn.Linear(16 * 7 * 7, 128)`, you will get:

```
RuntimeError: shape '[64, 784]' is invalid for input of size [64, 1024]
```

**Fix.** Walk through the shape at every layer manually using the [output formula](02_Concepts.md#the-output-shape-formula) from Chapter 02. Or temporarily print `x.shape` in `forward()` before the `view()`. Real engineers print shapes; pretending you can compute them in your head is how bugs ship.

### 2. Forgetting `model.eval()` Before Testing

If you have `Dropout` or `BatchNorm` layers and you do not call `model.eval()` before evaluation, those layers behave as if you were training — randomly dropping neurons or using batch statistics. Test accuracy will be artificially low. Always call `model.eval()` for inference. (We did, on line 50.)

### 3. Wrong Normalization

MNIST has known pixel statistics: mean ≈ 0.1307, std ≈ 0.3081. The `transforms.Normalize` line uses those. **Skip normalization and the model trains, just slower and to lower accuracy.** Use the wrong values (e.g., 0.5/0.5) and the model still works but is mildly handicapped.

For CIFAR-10 and ImageNet, the normalization values are different. Always use the dataset-specific values. `torchvision` provides them: `torchvision.transforms.Normalize.MNIST_MEAN_STD` etc.

---

## Run It Yourself

The full hands-on experience — including loss-curve plots, per-class accuracy, confusion matrix, and visualization of what each filter learned — lives in the notebook:

**[Computer Vision CNN on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Computer_Vision_CNN.ipynb)**

For the **forward pass demystified by hand** (manual convolution, single-slide computation, full feature map by NumPy), see also:

**[Computer Vision From Scratch on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Computer_Vision_From_Scratch.ipynb)** — every patch multiplied by hand, verified against `F.conv2d`.

The notebook has 34 cells covering: minimal CNN (this code), convolution mechanics, output formula, multiple filters, 1x1 convolutions, pooling, the full pipeline, and a comparison to MLP. Run on Colab in your browser — no install needed.

---

## What's Next

You have a working CNN. Now the questions get richer:

- **Why does this model achieve 98%?** What did Layer 1 learn? What did Layer 2 learn? — see [04 — How It Works](04_How_It_Works.md)
- **How big does the receptive field get?** When does Layer 4 see "the whole digit"? — also Chapter 04
- **What if I have 500 medical images instead of 60,000 MNIST digits?** — transfer learning, also Chapter 04
- **What architecture should I actually use in production?** ResNet? EfficientNet? MobileNet? — see [05 — Building It](05_Building_It.md)

---

**Next:** [04 — How It Works](04_How_It_Works.md) — Receptive field, training diagnostics for vision, transfer learning, augmentation strategies.
