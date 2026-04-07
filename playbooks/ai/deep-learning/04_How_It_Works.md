# Deep Learning — How It Works

**Training diagnostics, loss curves, and the patterns that separate a healthy model from a broken one.**

---

## The Doctor's Dashboard

A doctor does not wait for a patient to collapse to know something is wrong. There are vital signs — heart rate, blood pressure, oxygen saturation — and they tell the story in real time. A rising heart rate with falling blood pressure? Internal bleeding. Both stable? The patient is fine. The vitals tell the doctor what to do before the symptoms become obvious.

A **loss curve** is the vital sign of a neural network. It is a plot of the loss (how wrong the model is) over time (training epochs). Two lines: one for training data, one for test data. The shape of those two lines — whether they converge, diverge, plateau, or oscillate — tells an engineer exactly what is happening inside the model and what to do about it.

This is the single most important diagnostic skill in deep learning. Architecture selection, hyperparameter tuning, regularization strategy — all of these decisions are informed by reading the loss curve. An engineer who cannot read a loss curve is flying blind.

---

## The Four Patterns

Every loss curve falls into one of four patterns. Recognizing them instantly is the skill.

### Pattern 1: Healthy Training

```
Train loss:  ████████████▓▓▓▓▒▒▒▒░░░░  (decreasing steadily)
Test loss:   ████████████▓▓▓▓▒▒▒▒░░░░  (decreasing, slightly above train)
Gap:         Small and stable
```

Both lines go down. Test loss stays slightly above training loss — that gap is normal (the model has never seen the test data, so it is always slightly less accurate there). The gap is small and not growing.

**Diagnosis:** The model is learning the real pattern. Keep going.

**Action:** Nothing. This is the goal. Stop training when the test loss plateaus (stops improving).

### Pattern 2: Overfitting

```
Train loss:  ████████████▓▓▓▓▒▒▒▒░░░░  (keeps decreasing)
Test loss:   ████████████▓▓▓▒▒▒░▒▓▓██  (decreases, then INCREASES)
Gap:         Growing — train gets better, test gets worse
```

Training loss keeps dropping — the model is getting better on data it has seen. But test loss hits a minimum, then starts rising. The model is memorizing training data instead of learning the general pattern.

**The analogy:** A student who memorizes the practice test word-for-word. They score 100% on the practice test. The real exam has different questions — they fail. They did not learn the material. They learned the specific test.

**Diagnosis:** Overfitting. The model has more capacity (parameters) than the data requires. It is using that extra capacity to memorize noise.

**Action:**
- Add **dropout** (randomly disable neurons during training — forces the model to learn redundant paths)
- Add **data augmentation** (flip, rotate, crop training images — artificially increases dataset diversity)
- Reduce model size (fewer layers or fewer neurons per layer)
- Get more training data (the best fix, but not always possible)
- **Early stopping** — save the model at the epoch where test loss was lowest, discard everything after

### Pattern 3: Underfitting

```
Train loss:  ████████████████████████  (high, barely moves)
Test loss:   ████████████████████████  (high, barely moves)
Gap:         Small — but both lines are high
```

Both losses are high. Neither is improving much. The model is not learning — not from training data, not generalizing to test data.

**The analogy:** A student trying to understand calculus with only addition skills. The tool is too simple for the problem. No amount of practice helps — the practitioner needs a more powerful framework.

**Diagnosis:** Underfitting. The model is too simple. It does not have enough capacity (layers, neurons, parameters) to capture the pattern in the data. Or the learning rate is too low and the model is barely moving.

**Action:**
- Add more layers or more neurons per layer
- Train for more epochs (maybe it just needs more time)
- Increase the learning rate (the model may be making corrections too small to matter)
- Use a more powerful architecture (CNN instead of MLP for images, Transformer instead of RNN for sequences)
- Check the data — if the data has no pattern, no model will find one

### Pattern 4: Diverging

```
Train loss:  ░░▒▓█▓▒█████▓██▒███████  (oscillating wildly or increasing)
Test loss:   ░▒▓███▓██▒████▓█████████  (same chaos)
Gap:         Irrelevant — nothing is working
```

The loss jumps around erratically, or it increases instead of decreasing. Training is unstable.

**The analogy:** A driver who oversteers. They swerve left to correct, overshoot, swerve right, overshoot again, swerve harder — the car oscillates wider and wider until it goes off the road. The corrections are too large.

**Diagnosis:** The learning rate (η, "AY-tuh") is too high. Each weight update overshoots the optimal value, making the next update overshoot in the opposite direction.

**Action:**
- Reduce the learning rate. If using Adam at 0.001, try 0.0001. If using SGD at 0.01, try 0.001.
- If loss is NaN (Not a Number) or infinity, the learning rate is catastrophically too high. Divide by 10 and try again.
- Check for data issues — are there NaN values in the input? Extremely large feature values? Unnormalized data?

---

## The Diagnostic Table — Quick Reference

| Train Loss | Test Loss | Gap | Diagnosis | Fix |
|:---|:---|:---|:---|:---|
| Decreasing | Decreasing (slightly above) | Small, stable | **Healthy** | Keep going. Stop when test loss plateaus. |
| Decreasing | Decreasing, then **increasing** | Growing | **Overfitting** | Dropout, augmentation, early stopping, more data |
| High, flat | High, flat | Small | **Underfitting** | More capacity, higher learning rate, more epochs, better architecture |
| Wild / increasing | Wild / increasing | Irrelevant | **Diverging** | Lower learning rate, check data for NaN/outliers |

> **The first diagnostic every time:** Plot train loss and test loss. Read the shape. That tells whether the next step is "add complexity," "add regularization," or "fix the learning rate." Getting this wrong sends the debugging in the wrong direction — adding regularization to an underfitting model makes it worse, not better.

---

## The Train-Test Gap — What It Tells

The gap between train accuracy and test accuracy is a thermometer for generalization.

| Train Accuracy | Test Accuracy | Gap | Reading |
|:---|:---|:---|:---|
| 95% | 93% | 2% | Excellent generalization. Ship it. |
| 98% | 85% | 13% | Moderate overfitting. Add regularization. |
| 99.9% | 62% | 38% | Severe overfitting. The model memorized. |
| 55% | 52% | 3% | Both low. Underfitting. The model is too simple. |
| 72% | 71% | 1% | OK performance, good generalization, but accuracy might need to be higher. Try a better architecture. |

The gap is more informative than either number alone. A model with 75% train and 73% test (2% gap) is better-positioned than a model with 99% train and 80% test (19% gap) — even though the second model has higher test accuracy right now. The first model is healthy and can improve with more capacity. The second model is overfitting and will get worse if made more complex.

---

## Learning Rate — The Most Important Number

If there is one hyperparameter (a setting the engineer chooses, not the model learns) that matters more than all others, it is the learning rate.

### What It Controls

The learning rate (η, "AY-tuh") controls how much each weight changes on each training step. Think of it as the step size when walking downhill.

- **η = 0.1 (too high for Adam):** Giant steps. Overshoots the valley. Loss oscillates or explodes.
- **η = 0.001 (typical for Adam):** Moderate steps. Steady progress downhill. Loss decreases smoothly.
- **η = 0.000001 (too low):** Tiny steps. Moving, but barely. Training takes forever. May never reach the valley floor.

### How to Find the Right Value

There is no formula. But there are reliable starting points:

| Optimizer | Typical Starting Learning Rate |
|:---|:---|
| **Adam** | 0.001 (the default in most codebases) |
| **SGD** | 0.01 - 0.1 (higher than Adam because SGD has no built-in adaptation) |
| **AdamW** | 0.001 - 0.0001 (often used with a scheduler that decays the rate) |

**The diagnostic process:**
1. Start with the default (0.001 for Adam)
2. Train for a few epochs and plot the loss curve
3. If loss is **not decreasing** → try doubling the learning rate
4. If loss is **oscillating or increasing** → try halving the learning rate
5. If loss is **decreasing smoothly** → the rate is fine. Keep it.

### Learning Rate Schedules

In production, the learning rate often changes during training:

| Schedule | What It Does | Analogy |
|:---|:---|:---|
| **Step decay** | Multiply by 0.1 every N epochs | Walking fast on flat ground, slowing down near the bottom of the hill where precision matters |
| **Cosine annealing** | Smoothly decrease following a cosine curve | Gradually decelerating — no sudden speed changes |
| **Warmup** | Start very low, gradually increase, then decrease | Warming up before a race — ramp up speed, then cruise, then cool down |

Warmup is standard for Transformer training (the Transformer concepts material covers why). For simple CNNs and MLPs, a flat learning rate with Adam works well enough.

---

## Batch Size — The Other Important Number

**Batch size** is how many training examples the model sees before updating its weights. It is a tradeoff between speed, memory, and generalization.

| Batch Size | Speed | Memory | Gradient Quality | Generalization |
|:---|:---|:---|:---|:---|
| **Small (8-32)** | Slower (more updates per epoch) | Low | Noisy but regularizing — the randomness helps avoid sharp minima | Often better |
| **Medium (64-256)** | Balanced | Moderate | Good estimate of the true gradient | Good. This is the default range. |
| **Large (512-4096)** | Fastest (fewer updates per epoch) | High (may not fit in GPU memory) | Very accurate gradient estimate | Can generalize worse — may converge to sharp minima that do not transfer to new data |

**Default:** 64 for small datasets, 128-256 for larger ones. If the batch does not fit in GPU memory (Out of Memory error), reduce batch size.

**The analogy:** Surveying customers about a product. Ask 1 person (batch size 1) — very noisy, their opinion might be extreme. Ask 1,000 people (batch size 1000) — accurate average, but expensive and slow. Ask 64 people — good enough estimate, affordable, and the slight noisiness actually helps avoid conclusions that are too narrow.

---

## Epochs — When to Stop Training

An **epoch (pronounced "EP-ock")** is one complete pass through the entire training dataset. If the dataset has 60,000 images and the batch size is 64, one epoch is 938 batches (60,000 / 64).

### How Many Epochs?

There is no fixed answer. The answer is: **train until the test loss stops improving.**

- Too few epochs → underfitting. The model has not seen the data enough times.
- Too many epochs → overfitting. The model starts memorizing after the test loss minimum.
- Just right → stop at the epoch where test loss is lowest.

**Early stopping** automates this: track test loss every epoch. If it has not improved for N epochs (called "patience" — typically 5-10), stop training and revert to the weights from the best epoch.

```
Epoch 1:  Test loss = 2.31  ← improving
Epoch 5:  Test loss = 0.42  ← improving
Epoch 10: Test loss = 0.19  ← improving
Epoch 15: Test loss = 0.15  ← best so far ★ (save these weights)
Epoch 20: Test loss = 0.16  ← worse (patience counter: 1)
Epoch 25: Test loss = 0.18  ← worse (patience counter: 2)
Epoch 30: Test loss = 0.21  ← worse (patience counter: 3... stop. Revert to epoch 15 weights.)
```

---

## The Debugging Checklist — When the Model Is Not Learning

When loss is not decreasing or accuracy is stuck, work through this list **in order** — from most common cause to least:

| # | Check | How to Verify | Fix |
|:---|:---|:---|:---|
| 1 | **Is the data loaded correctly?** | Print a batch: `images, labels = next(iter(train_loader))`. Visualize the images. Are the labels correct? | Fix the data pipeline. Garbage in, garbage out. This is the #1 cause of bad models — not bad architecture. |
| 2 | **Is the data normalized?** | Print `images.mean()` and `images.std()`. Should be near 0 and 1. | Add `transforms.Normalize()` with the dataset's mean and std. |
| 3 | **Is the learning rate reasonable?** | Start with 0.001 (Adam). If loss oscillates, halve it. If loss barely moves, double it. | Adjust the learning rate. This fixes more problems than changing the architecture. |
| 4 | **Is the model on the right device?** | Print `next(model.parameters()).device`. Does it match the data's device? | Add `.to(device)` to both model AND data. Mismatched devices cause clear error messages — read them. |
| 5 | **Did you forget `optimizer.zero_grad()`?** | Without it, gradients accumulate across batches. Loss may look normal but training is unstable. | Add `optimizer.zero_grad()` before `loss.backward()`. |
| 6 | **Is the loss function correct?** | Classification → `CrossEntropyLoss`. Regression → `MSELoss`. Binary → `BCEWithLogitsLoss`. | Match the loss to the problem type. Using MSE for classification silently produces bad gradients. |
| 7 | **Is the model too small?** | If BOTH train and test loss are high (underfitting). | Add more layers or more neurons per layer. |
| 8 | **Is the model too large for the data?** | If train loss is near zero but test loss is high (overfitting). | Add dropout, data augmentation, early stopping. Or get more data. |

> **The process mirrors production debugging.** Start broad: is the data even loading? Then narrow down: is the learning rate off? Is the loss function wrong? Then drill into specifics: is the architecture mismatched? Each step eliminates a category of problems. The issue surfaces naturally when the general vicinity is identified and then narrowed. This is the same print-statement drill-down that works for debugging any system — identify the area, then go deeper.

---

**Next:** [05 — Decisions](05_Building_It.md) — Every architectural choice, with tradeoffs. Activation functions, loss functions, optimizers, architecture selection — the decision framework for deep learning projects.
