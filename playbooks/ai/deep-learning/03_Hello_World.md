# Deep Learning — Hello World

**See it work first. Understand it after.**

> **Python note:** The code below uses `import`, `class`, `for` loops, and `print`. If any of that is unfamiliar, the [Python for AI notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_Basics.ipynb) covers everything you need. But try running the code first — you may understand more than you think.

---

## What You Are About to See

In the next 60 seconds of reading, you will see a neural network that:
- Takes a photograph of a handwritten digit (a tiny 28x28 pixel image)
- Predicts which digit it is (0 through 9)
- Gets it right 97% of the time
- Trains in under 30 seconds on a laptop

The dataset is **MNIST (Modified National Institute of Standards and Technology, pronounced "em-nist")** — 70,000 handwritten digit images collected from census workers and high school students. It is the "Hello World" of deep learning. Every practitioner has done this.

The model is an **MLP (Multi-Layer Perceptron, "M-L-P")** — the simplest kind of neural network. Three layers. About 110,000 learnable parameters. By comparison, GPT-4 has roughly 1.7 trillion. Perspective.

---

## The Code — All of It

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

# Load 60,000 training images and 10,000 test images
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])
train_data = datasets.MNIST(root='./data', train=True, download=True, transform=transform)
test_data = datasets.MNIST(root='./data', train=False, download=True, transform=transform)
train_loader = DataLoader(train_data, batch_size=64, shuffle=True)
test_loader = DataLoader(test_data, batch_size=1000)

# Define the neural network — 3 layers
class SimpleNet(nn.Module):
    def __init__(self):
        super().__init__()
        self.flatten = nn.Flatten()          # 28x28 image → 784 numbers
        self.layer1 = nn.Linear(784, 128)    # 784 inputs → 128 neurons
        self.layer2 = nn.Linear(128, 64)     # 128 → 64 neurons
        self.layer3 = nn.Linear(64, 10)      # 64 → 10 digit classes
        self.relu = nn.ReLU()                # Activation: keep positives, zero negatives

    def forward(self, x):
        x = self.flatten(x)
        x = self.relu(self.layer1(x))
        x = self.relu(self.layer2(x))
        x = self.layer3(x)                  # No activation on output — cross-entropy handles it
        return x

# Train
model = SimpleNet()
loss_fn = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

for epoch in range(3):
    model.train()
    for images, labels in train_loader:
        predictions = model(images)          # 1. Forward pass
        loss = loss_fn(predictions, labels)  # 2. Compute loss
        optimizer.zero_grad()                # 3. Clear old gradients
        loss.backward()                      # 4. Backward pass
        optimizer.step()                     # 5. Update weights

# Evaluate
model.eval()
correct = 0
total = 0
with torch.no_grad():
    for images, labels in test_loader:
        outputs = model(images)
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()

print(f"Test Accuracy: {100.0 * correct / total:.1f}%")
```

## What You Should See

```
Test Accuracy: 97.3%
```

That is it. That is deep learning.

60,000 images. 3 layers. 110,000 parameters. 3 passes through the data. 30 seconds. 97% accuracy on images the model has never seen.

---

## What Just Happened — Line by Line

### Loading the Data

```python
transform = transforms.Compose([
    transforms.ToTensor(),                    # Convert pixel values from 0-255 integers to 0.0-1.0 floats
    transforms.Normalize((0.1307,), (0.3081,))  # Center around zero using MNIST's known mean and std
])
```

**Why normalize?** Raw pixels range from 0 to 255. Some features would dominate others just because of scale. Normalizing to mean≈0, std≈1 puts all features on equal footing. The model learns faster and more stably. This is the same principle as feature scaling in the ML Fundamentals notebook — it applies to every input, every time.

The numbers 0.1307 and 0.3081 are the pre-computed mean and standard deviation of the entire MNIST dataset. Every MNIST tutorial uses these exact values.

```python
train_loader = DataLoader(train_data, batch_size=64, shuffle=True)
```

**Why batches?** Instead of feeding all 60,000 images at once (too much memory) or one at a time (too slow and noisy), we feed them in **batches of 64.** Each batch gives a good-enough estimate of the gradient while being computationally efficient. The model updates its weights after every batch — that is 938 updates per epoch (60,000 / 64).

**Why shuffle?** If the model sees all the 0s, then all the 1s, then all the 2s, it temporarily "forgets" the earlier digits. Shuffling ensures each batch is a random mix.

### Defining the Network

```python
self.flatten = nn.Flatten()          # 28x28 image → 784 numbers
self.layer1 = nn.Linear(784, 128)    # 784 → 128
self.layer2 = nn.Linear(128, 64)     # 128 → 64
self.layer3 = nn.Linear(64, 10)      # 64 → 10
```

**Why 784?** Each MNIST image is 28 pixels wide × 28 pixels tall = 784 pixels. `nn.Flatten()` reshapes the 2D image into a 1D list of 784 numbers. This is required for an MLP — fully connected layers expect a flat input.

**Why 128 → 64 → 10?** The network compresses information as it goes deeper. 784 features is too many — most are redundant (neighboring pixels are often similar). Layer 1 compresses to 128 features that matter. Layer 2 compresses further to 64. The final layer maps to 10 outputs — one per digit class (0 through 9).

These numbers (128, 64) are not magic. They are choices. You could use 256 → 128 → 10, or 64 → 32 → 10. Bigger networks learn more but risk overfitting. Smaller networks are faster but might underfit. The P1.5 project asks you to experiment with this.

**Why no activation on the last layer?** `nn.CrossEntropyLoss` (pronounced "cross EN-truh-pee") expects **raw scores** (called **logits**, pronounced "LOH-jits") — not probabilities. It applies softmax internally. Adding softmax before cross-entropy would apply it twice, which gives wrong gradients.

### The Training Loop

```python
predictions = model(images)          # 1. Forward pass
loss = loss_fn(predictions, labels)  # 2. Compute loss
optimizer.zero_grad()                # 3. Clear old gradients
loss.backward()                      # 4. Backward pass
optimizer.step()                     # 5. Update weights
```

This is the five-step loop from the Concepts chapter. Here it is in real code. Every deep learning project uses these same five lines — whether training a digit classifier or a language model. The architecture changes. The data changes. These five lines do not.

### Evaluation

```python
model.eval()                          # Switch to evaluation mode (turns off dropout)
with torch.no_grad():                 # Don't compute gradients — saves memory, we're not training
```

**Why `model.eval()`?** Some layers behave differently during training vs evaluation. **Dropout** randomly turns off neurons during training (to prevent memorization) but turns them all on during evaluation. **BatchNorm** uses batch statistics during training but stored statistics during evaluation. Forgetting `model.eval()` before testing is a common bug — your accuracy will be artificially lower.

**Why `torch.no_grad()`?** During training, PyTorch tracks every operation to compute gradients later. During evaluation, you just want predictions — no learning. `torch.no_grad()` tells PyTorch to skip the tracking, which saves memory and speeds things up.

---

## What the Model Learned

Nobody told this network what a "7" looks like. Nobody said "look for a horizontal line at the top and a diagonal line going down-left." The network discovered these patterns by adjusting 110,000 weights across ~2,800 batches (3 epochs × 938 batches) to minimize the cross-entropy loss.

If you visualize the weights of Layer 1 (which the notebook does), you see that different neurons have learned to respond to different patterns — some activate on vertical strokes, some on horizontal, some on curves. The network built its own feature detectors from raw pixels.

This is the fundamental difference between deep learning and traditional programming. You define what "wrong" means. The network figures out everything else.

---

## What It Gets Wrong

97% accuracy means 3% error — about 300 out of 10,000 test images misclassified. The common confusions:

| Predicted | Actual | Why |
|:---|:---|:---|
| 4 | 9 | Both have a vertical stroke. Some handwritten 4s look like 9s with an open top. |
| 3 | 8 | Both have curves. A sloppy 3 with a closed middle looks like an 8. |
| 7 | 1 | A 7 without a crossbar looks like a 1 with a slight lean. |
| 2 | 7 | In some handwriting styles, 2 and 7 share the horizontal stroke and diagonal. |

These failures are not random. They reveal the boundaries of what the model learned. Understanding WHY a model fails is more valuable than celebrating its accuracy — because the failures tell you what to improve.

---

## Try It Yourself

1. **Run the code** in the [Deep Learning notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Deep_Learning_PyTorch.ipynb) — Sections 3-4 walk you through this exact example with visualizations
2. **Change something and see what happens:**
   - Change `nn.Linear(784, 128)` to `nn.Linear(784, 16)` — fewer neurons. Does accuracy drop?
   - Change `num_epochs = 3` to `num_epochs = 10` — more training. Does accuracy improve? Does it plateau?
   - Remove the `nn.ReLU()` activations — what happens to accuracy? (Hint: without non-linearity, the 3-layer network collapses into a single linear model)
3. **Ask yourself:** The model gets 97% on MNIST. Would it get 97% on real-world handwriting from different countries, ages, and writing instruments? Why or why not?

---

## What Comes Next

You have seen it work. Now the question is: how do you make it work **better**, on **harder problems**, in **production**?

| Question | Where You Find the Answer |
|:---|:---|
| What if my images are real photos, not clean digits? | The notebook — CIFAR-10 section (color photos, backgrounds, noise) |
| How do I know if my model is overfitting or underfitting? | [04 — How It Works](04_How_It_Works.md) — training diagnostics and loss curves |
| Which activation function, loss function, and optimizer should I use? | [05 — Decisions](05_Building_It.md) — every choice, with tradeoffs |
| How do Google, Tesla, and hospitals actually deploy this? | [06 — Real World](06_Production_Patterns.md) — production architectures |
| How do I serve this model to 10,000 users? | [07 — System Design](07_System_Design.md) — scalability, cloud, deployment |
| What if someone attacks my model with adversarial images? | [08 — Security & Governance](08_Quality_Security_Governance.md) |
| How do I know if my deployed model is still working? | [09 — Observability](09_Observability_Troubleshooting.md) — monitoring, drift, debugging |
| Give me the one-page decision reference | [10 — Checklist](10_Decision_Guide.md) |

---

**Next:** [04 — How It Works](04_How_It_Works.md) — The mechanics behind what you just saw. Training diagnostics, loss curves, and the patterns that tell you whether your model is healthy or sick.
