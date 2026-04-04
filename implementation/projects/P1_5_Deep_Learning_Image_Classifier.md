# Project P1.5: Deep Learning Image Classifier

**Week:** 3 (Deep Learning and PyTorch)
**Time estimate:** 6-8 hours
**Prerequisites:** P1 (ML Predictor), Deep Learning and PyTorch notebook, Math for AI notebook

---

## Objective

Build a **PyTorch image classifier** that recognizes real-world objects from photographs. You will design a neural network architecture, train it, diagnose training problems using loss curves, and compare multiple approaches to find the best model.

This project bridges the gap between "I understand the theory" and "I can build and debug a neural network." Every technique you use here — training loops, loss curves, regularization, architecture comparison — applies directly to the Transformer and Fine-Tuning sections.

---

## The Dataset

**CIFAR-10 (Canadian Institute for Advanced Research, 10 classes)** — 60,000 color photographs, 32x32 pixels, 10 classes.

| Class ID | Class Name | Examples |
|:---|:---|:---|
| 0 | airplane | Commercial jets, military planes |
| 1 | automobile | Cars, sedans, SUVs |
| 2 | bird | Various species |
| 3 | cat | House cats, various poses |
| 4 | deer | Deer in nature |
| 5 | dog | Various breeds |
| 6 | frog | Frogs, close-up |
| 7 | horse | Horses in various settings |
| 8 | ship | Ships, boats on water |
| 9 | truck | Trucks, delivery vehicles |

Split: 50,000 training + 10,000 test images. Built into `torchvision.datasets.CIFAR10` — no manual download needed.

> **Why CIFAR-10 and not MNIST?** MNIST (handwritten digits) is too easy — even a simple MLP gets 97%+. CIFAR-10 has real-world complexity: color images, backgrounds, visual similarity between classes (cat vs dog, automobile vs truck). This is where architecture choices actually matter.

---

## Requirements

Your project must include ALL of the following:

### 1. Data Exploration
- [ ] Load CIFAR-10 and display sample images from each class (labeled)
- [ ] Show the class distribution (is it balanced?)
- [ ] Print image dimensions and data types
- [ ] Display the normalization statistics you will use (mean, std per channel)

### 2. Architecture Comparison (minimum 3 models)
Build and train at least 3 different architectures:

- [ ] **Model A: MLP (Multi-Layer Perceptron)** — flatten the image, fully connected layers only. This is your baseline.
- [ ] **Model B: Simple CNN (Convolutional Neural Network)** — 2-4 convolutional layers + pooling + fully connected. Your main model.
- [ ] **Model C: CNN with regularization** — same as Model B but add dropout, batch normalization, and data augmentation.

For each model:
- [ ] Define as an `nn.Module` subclass with clear comments
- [ ] Print the architecture and parameter count
- [ ] Train for the same number of epochs (minimum 15) with the same optimizer settings
- [ ] Record training loss, training accuracy, test loss, and test accuracy per epoch

### 3. Training Diagnostics (the core skill)
- [ ] Plot **loss curves** (train + test) for all 3 models on the same chart or side-by-side
- [ ] Plot **accuracy curves** (train + test) for all 3 models
- [ ] Identify which models show overfitting, underfitting, or healthy training
- [ ] Calculate the **train-test accuracy gap** for each model and interpret it
- [ ] Write a 1-paragraph diagnosis for each model: "Model A shows [pattern] because [reason]. The fix would be [action]."

### 4. Experiments (minimum 2)
Run at least 2 of these experiments and document the results:

- [ ] **Learning rate experiment:** Train Model B with 3 different learning rates (e.g., 0.1, 0.001, 0.00001). Plot all 3 loss curves. Which works best? Why?
- [ ] **Chunk size experiment (data augmentation):** Train Model B with and without data augmentation (RandomHorizontalFlip, RandomCrop). Compare test accuracy. How much does augmentation help?
- [ ] **Architecture depth experiment:** Compare a shallow CNN (2 conv layers) versus a deeper CNN (4-6 conv layers). Does deeper always mean better?
- [ ] **Optimizer experiment:** Compare SGD vs Adam on the same architecture. Which converges faster? Which reaches higher accuracy?

### 5. Per-Class Analysis
- [ ] Generate a **confusion matrix** for your best model
- [ ] Identify the 2-3 most confused class pairs (e.g., cat vs dog, automobile vs truck)
- [ ] Show example misclassifications — display the image, the predicted class, and the true class
- [ ] Explain in plain English why the model confuses these classes

### 6. Code Quality
- [ ] All code in a Jupyter notebook or Python scripts
- [ ] The training loop extracted into a **reusable function** (not copy-pasted 3 times)
- [ ] The plotting code extracted into a **reusable function**
- [ ] Comments explaining WHY, not just WHAT (25%+ comment density)
- [ ] GPU (Graphics Processing Unit) support: model and data moved to the best available device (CUDA, MPS, or CPU)
- [ ] A `requirements.txt` listing all dependencies
- [ ] A `.gitignore`

### 7. README
- [ ] Problem statement
- [ ] Architecture descriptions (with parameter counts)
- [ ] Results table: all 3 models, all metrics, side by side
- [ ] Best model and why
- [ ] Loss curve screenshots or embedded images
- [ ] How to run

### 8. Reflection Journal
Answer in a separate `REFLECTION.md`:

1. Which architecture won and why? Was it what you expected?
2. What did the loss curves tell you that the accuracy numbers did not?
3. What was the most common type of misclassification? Why do you think it happens?
4. If you had to improve your best model's accuracy by 5%, what would you try next?
5. How does this compare to P1 (scikit-learn)? What was easier or harder in PyTorch?

---

## Step-by-Step Guide

### Step 1: Set up your project
```bash
mkdir p1-5-image-classifier && cd p1-5-image-classifier
python -m venv .venv
source .venv/bin/activate
pip install torch torchvision matplotlib seaborn scikit-learn jupyter
pip freeze > requirements.txt
```

### Step 2: Explore the data
Load CIFAR-10. Display samples from each class. Understand what 32x32 pixels actually looks like (tiny, noisy — even humans make mistakes on these).

### Step 3: Build Model A (MLP baseline)
Flatten the image to 3072 inputs. 2-3 fully connected layers. Train it. Record the history. This will likely get ~50-55% accuracy. That is the point — to show why CNNs exist.

### Step 4: Build Model B (Simple CNN)
Use `nn.Conv2d` layers with ReLU and MaxPool. 2-4 conv layers → flatten → fully connected → output. Train it. You should see a clear improvement over the MLP.

### Step 5: Build Model C (CNN + regularization)
Copy Model B. Add `nn.Dropout` between layers, `nn.BatchNorm2d` after conv layers, and data augmentation in the transforms. Train it. The train-test gap should shrink.

### Step 6: Plot diagnostics
All 3 models, same chart. This is the most important deliverable — it shows you can read training behavior.

### Step 7: Run experiments
Pick 2 experiments. Document results with plots and plain-English interpretation.

### Step 8: Confusion matrix and misclassifications
Use `sklearn.metrics.confusion_matrix` and `seaborn.heatmap`. Show the images the model got wrong.

### Step 9: Write README, reflection, push to GitHub

```bash
git init && git add . && git commit -m "P1.5: CIFAR-10 image classifier with training diagnostics"
gh repo create p1-5-image-classifier --public --push
```

---

## Grading Rubric

| Category | Points | What We Look For |
|:---|:---|:---|
| Data exploration | 10 | Samples visualized, class distribution shown, normalization explained |
| 3 architectures (MLP, CNN, CNN+regularization) | 20 | Correct nn.Module structure, parameter counts documented, all trained same epochs |
| Training diagnostics (loss curves, accuracy curves, diagnosis) | 25 | All 3 models plotted, overfitting/underfitting identified, written diagnosis per model |
| Experiments (minimum 2 with documented results) | 15 | Clear setup, results plotted, plain-English interpretation |
| Per-class analysis (confusion matrix, misclassifications) | 10 | Matrix visualized, confused pairs identified, example images shown with explanation |
| Code quality (reusable functions, comments, GPU support) | 10 | Training function reused across models, WHY comments, device-aware code |
| README + Reflection journal | 10 | Clear results table, honest reflection, forward-looking (what would you try next?) |
| **Total** | **100** | |

> **Note:** Training diagnostics is worth 25 points — the most of any category. This is deliberate. The ability to read loss curves and diagnose training problems is the skill that separates someone who can train a model from someone who can debug one. Debugging is what you will do in production.

---

## The Interview Story

> "I built three image classifiers on CIFAR-10 — an MLP, a basic CNN, and a CNN with regularization — to understand when and why architecture matters. The MLP topped out at 53% because it flattens spatial structure. The basic CNN reached 72% but showed overfitting — training accuracy hit 95% while test accuracy plateaued. Adding dropout, batch normalization, and data augmentation closed the gap and pushed test accuracy to 79%. The most confused classes were cat vs dog and automobile vs truck — makes sense, they share visual features at 32x32 resolution. The key takeaway was that loss curves told me more than accuracy numbers. The MLP was underfitting (both losses high). The basic CNN was overfitting (train loss dropping, test loss rising). The regularized CNN was healthy (both losses decreasing together). I used the same diagnostic patterns when debugging a RAG pipeline later in the program."

---

## Reference: External Projects to Study First

Before building your own, study how others approached similar problems:

| Repo | What to Study |
|:---|:---|
| [pytorch/examples — MNIST](https://github.com/pytorch/examples/tree/main/mnist) | The canonical 140-line PyTorch training script. Read it first — it has the exact pattern you will use. |
| [bentrevett/pytorch-image-classification](https://github.com/bentrevett/pytorch-image-classification) | 5 progressive notebooks (MLP → LeNet → AlexNet → VGG → ResNet). Work through notebook 1 (MLP) and 2 (LeNet) before building your own. |
| [kuangliu/pytorch-cifar](https://github.com/kuangliu/pytorch-cifar) | 10+ architecture implementations on CIFAR-10 with accuracy benchmarks. Use as a reference catalog after you build your own. |

A hands-on guide for cloning and running these repos is available at: [Deep Learning Projects Setup Guide](https://github.com/sunilmogadati/systems-in-production/blob/main/implementation/guides/Deep_Learning_Projects_Setup_Guide.md)

> **Study the external repos, but build your own from scratch.** The goal is not to copy — it is to internalize the pattern. If you can build an nn.Module, write a training loop, plot loss curves, and diagnose what went wrong without looking at someone else's code, you own the skill.

---

## Deliverables

Push to GitHub with this structure:

```
p1-5-image-classifier/
  README.md
  REFLECTION.md
  requirements.txt
  .gitignore
  notebooks/
    exploration.ipynb         # Data visualization and exploration
    modeling.ipynb            # All 3 architectures, training, diagnostics
    experiments.ipynb         # Learning rate / augmentation / depth experiments
  src/                        # (optional) Reusable modules
    models.py                 # nn.Module class definitions
    train.py                  # Training function
    utils.py                  # Plotting, evaluation helpers
```

---

## How P1.5 Connects to What Comes Next

| What You Build Here | Where It Goes |
|:---|:---|
| nn.Module class pattern | the Transformer section: You will build Transformer blocks as nn.Module subclasses |
| Training loop (5-step pattern) | the Transformer section: Same loop trains nanoGPT. Same loop trains everything. |
| Loss curve diagnostics | the Transformer section: You will use loss curves to debug a language model |
| Regularization (dropout, data augmentation) | the Fine-Tuning section: LoRA/QLoRA fine-tuning is fundamentally a regularization strategy |
| Architecture comparison methodology | the Cloud & MLOps section (P2): Comparing LLM models on cost, latency, and quality |
| Confusion matrix analysis | the RAG section+: Understanding where RAG and agents fail follows the same pattern |
