# Math for AI — A Working Reference

**The handful of math primitives that show up in every AI system, explained in plain English. Every function, every loss, every training loop reduces to this set.**

> **Hands-on companion:** [Math for AI on Colab](https://colab.research.google.com/github/sunilmogadati/ai-engineer-accelerator/blob/main/AI_Engineer_Accelerator_Math_for_AI.ipynb) — runnable notebook with worked examples, plots, and an Architect's Mental Model checklist.

---

## Why This Document Exists

You can ship AI without knowing the math, the same way you can drive a car without knowing how the engine works — until something breaks.

When training stalls, when the loss explodes, when the gradients vanish — these are not framework bugs. They are the math telling you something. The engineers who ship reliably are the ones who can read the symptoms.

This is the working reference. Not a textbook. Not exhaustive. Just the parts you actually need to:

- Read modern ML/DL papers without losing the thread
- Debug a training run that is misbehaving
- Choose between activation functions, loss functions, optimizers
- Move on to architecture-specific math (attention in [Transformers](transformers/), convolutions in [Computer Vision](computer-vision/), etc.)

The notebook above runs every concept here with code and plots. Use them together.

---

## What's Covered

| Section | What You Learn | Where It Shows Up |
|---|---|---|
| [Functions and Shapes](#functions-and-shapes) | Linear, polynomial, exponential, log, sin/cos. The basic shapes. | Activation functions, loss curves, learning rate schedules |
| [Derivatives — Single Variable](#derivatives--single-variable) | `d/dx` rules — the slope at a point | Simple regression, basic gradient descent |
| [Partial Derivatives — Many Variables](#partial-derivatives--many-variables) | `∂` notation — slope when many things are changing | Every neural network: loss depends on millions of weights |
| [The Chain Rule](#the-chain-rule) | How to compose derivatives across stages | Backpropagation in every neural network |
| [Gradient Descent](#gradient-descent) | The update rule that powers all training | Every optimizer (SGD, Adam, AdamW) |
| [Vectors and Matrices (briefly)](#vectors-and-matrices-briefly) | Dot products, matrix multiplication | The forward pass at scale |

---

## Functions and Shapes

Different functions produce different output shapes. Recognizing the shape is half the battle when reading code or debugging a model.

### Linear

```
y = m·x + b
```

Straight line. Same change for every step. The simplest possible relationship.

| Where you see it | Why |
|---|---|
| Linear regression | The model is literally this equation |
| The output of any neuron *before* activation | `z = w·x + b` is linear |
| The output layer of a regression network | No activation = linear output |

### Polynomial

```
y = x², x³, x⁴, ...
```

Curved. The exponent amplifies how fast it grows.

| Where you see it | Why |
|---|---|
| MSE (Mean Squared Error) loss | `L = (y_pred − y)²` — the squared term punishes big errors more than small ones |
| Polynomial regression | A more flexible regression curve |

### Exponential

```
y = eˣ   or   y = 2ˣ
```

Slow start, then explosive growth. The growth rate depends on the *current value*.

| Where you see it | Why |
|---|---|
| Softmax | Uses `eˣ` to convert raw scores into probabilities that sum to 1 |
| Loss explosion | When training diverges, the loss often grows exponentially before going to NaN (Not a Number) |

### Logarithmic

```
y = log(x)
```

Fast early growth, then flattening. Diminishing returns.

| Where you see it | Why |
|---|---|
| Cross-entropy loss | Uses `log(p)` — heavy penalty for confidently wrong predictions |
| Learning rate schedules | Log decay is a common shape — fast at first, then careful |

### Sine and Cosine

```
y = sin(x),   y = cos(x)
```

Periodic waves. Repeat forever.

| Where you see it | Why |
|---|---|
| Positional encoding in Transformers | Sine/cosine waves encode token position into the input |
| Cosine learning rate schedules | Smooth speed up and slow down across training |

---

## Derivatives — Single Variable

A **derivative** measures the slope of a function at a point. "If I nudge `x` by a tiny amount, how much does `y` change?"

Notation: `dy/dx` (read as "dee-y dee-x") or `f'(x)` (read as "eff-prime of x").

### The Rules You Need

| Function | Derivative | Why It's Useful |
|---|---|---|
| constant `c` | `0` | A flat line has no slope |
| `x` | `1` | The slope of `y = x` is always 1 |
| `a·x` | `a` | A line `y = 5x` has slope 5 everywhere |
| `x²` | `2x` | The parabola gets steeper as `x` grows |
| `x³` | `3x²` | Higher powers grow even faster |
| `xⁿ` | `n·xⁿ⁻¹` | The general rule for any power |
| `eˣ` | `eˣ` | Special: the exponential is its own derivative |
| `ln(x)` | `1/x` | Slope of the natural log starts steep, flattens |
| `sin(x)` | `cos(x)` | Slope of a wave is the wave shifted 90° |
| `cos(x)` | `−sin(x)` | Same idea, opposite sign |

These rules cover the math you read in ~95% of AI papers.

### Why Derivatives Matter for Training

Training a neural network = **find the weights that minimize loss**. The derivative of the loss tells you which way to move each weight.

- Derivative is **positive** → loss is rising → moving the weight up makes it worse → move it **down**.
- Derivative is **negative** → loss is falling → moving the weight up makes it better → move it **up**.

That is gradient descent in one sentence.

---

## Partial Derivatives — Many Variables

When a function depends on **many** variables, you need a way to ask: "if I nudge ONLY this one variable and hold the rest fixed, how does the output change?"

That is a **partial derivative**. Different notation, same operation.

| Symbol | When to use | Reads as |
|---|---|---|
| `dy/dx` | `y` depends on only `x` | "dee-y dee-x" |
| `∂L/∂w₁` | `L` depends on `w₁, w₂, ..., wₙ` (many variables) — only nudge `w₁` | "partial-L partial-w-one" or "del-L del-w-one" |

### Why Neural Networks Always Use ∂

A neural network's loss is a function of every weight in the model:

```
L = function(w₁, b₁, w₂, b₂, ..., wₙ, bₙ, v₁, v₂, ..., c)
```

GPT-4 has ~1.7 trillion of these. To train, we need:

> If I change ONLY `w₁` (and nothing else), what happens to `L`?

That is exactly what `∂L/∂w₁` answers. So neural networks always use `∂`, never `d`.

The math is identical to single-variable derivatives. You treat every variable EXCEPT the one you are differentiating with respect to as a constant. The d/dx rules table above applies term by term.

---

## The Chain Rule

Most variables in a neural network do not directly produce the loss — they pass through a chain of transformations.

```
w₁  →  z₁  →  h₁  →  y_pred  →  L
```

To find `∂L/∂w₁`, walk the chain backward and multiply the partial derivatives at each step:

```
∂L/∂w₁ = (∂L/∂y_pred) · (∂y_pred/∂h₁) · (∂h₁/∂z₁) · (∂z₁/∂w₁)
```

### Plain English

> "How sensitive is the loss to a tiny nudge in `w₁`? Walk the chain: how sensitive `L` is to `y_pred`, times how sensitive `y_pred` is to `h₁`, times how sensitive `h₁` is to `z₁`, times how sensitive `z₁` is to `w₁`. Multiply them all."

Each link is small and easy. The product gives you the gradient.

### Why It's Magic in Practice

This is **backpropagation**. Every neural network library — PyTorch, TensorFlow, JAX — implements automatic differentiation that builds this chain automatically as you write the forward pass. When you call `loss.backward()`, the framework walks the chain and computes every `∂L/∂w` for every weight. You do not write the math by hand once you understand it.

**For the worked-by-hand version with real numbers:** [DL → 02_Concepts: The Math, Step by Step](deep-learning/02_Concepts.md#the-math-step-by-step--a-worked-example) — three houses, two hidden neurons, every gradient computed explicitly.

---

## Gradient Descent

Once you have the gradient, the update rule is:

```
new_weight = old_weight − learning_rate · gradient
```

That is the entire algorithm. Three components:

| Component | Symbol | What It Does |
|---|---|---|
| Old weight | `w` | The current value |
| Learning rate | `α` ("alpha") or `η` ("eta") | How big a step to take |
| Gradient | `∂L/∂w` | Which direction reduces loss |

### Why The Minus Sign

The gradient points in the direction of **steepest increase** of loss. We want to **decrease** loss. So we go the opposite way — hence the minus sign.

- Subtract a *positive* gradient (loss is rising) → weight goes down.
- Subtract a *negative* gradient (loss is falling) → weight goes up.

Either way, the loss goes down.

### Variants

Same update rule, different ways of computing the gradient:

| Optimizer | What's Different |
|---|---|
| **SGD (Stochastic Gradient Descent)** | Uses one batch at a time. Fast, noisy. |
| **SGD with Momentum** | Remembers past gradients, builds up speed in consistent directions |
| **Adam (Adaptive Moment Estimation)** | Adapts the learning rate per-weight based on history |
| **AdamW** | Adam with proper weight decay (regularization) — used for fine-tuning |

All four obey the same `new = old − α · gradient` skeleton. The variants change *how the gradient is smoothed or scaled* before applying. See [DL → 02_Concepts: Optimizers](deep-learning/02_Concepts.md#optimizers--the-weight-adjusters) for the full comparison.

---

## Vectors and Matrices (Briefly)

Real neural networks do not nudge one weight at a time. They process many inputs and many weights at once using **vectors** (1D arrays of numbers) and **matrices** (2D arrays).

### Dot Product

The fundamental operation. Multiply corresponding elements, then add them all up.

```
[1, 2, 3] · [4, 5, 6] = 1·4 + 2·5 + 3·6 = 32
```

A neuron with three inputs and three weights does exactly this:

```
output = w · x + b   (where w and x are vectors)
```

### Matrix Multiplication

A whole layer of neurons in one operation. If a layer has 100 neurons taking 50 inputs, the weights form a 100×50 matrix. Multiplying that matrix by a 50-element input vector gives all 100 neuron outputs at once.

```
output = W · x + b   (where W is a matrix, x and b are vectors)
```

This is what GPUs (Graphics Processing Units) are good at. Thousands of these multiplications in parallel is exactly what training a deep network requires. That is why framework code says `torch.matmul(W, x)` — it dispatches to the GPU and runs every neuron's computation at once.

For deeper linear algebra, work through the [Math for AI notebook on Colab](https://colab.research.google.com/github/sunilmogadati/ai-engineer-accelerator/blob/main/AI_Engineer_Accelerator_Math_for_AI.ipynb).

---

## Where This Math Shows Up

| Playbook | What math from this doc applies |
|---|---|
| [ML](ml/) | Derivatives, gradient descent for logistic regression and gradient-boosted trees |
| [Deep Learning](deep-learning/) | All sections — partial derivatives, chain rule, gradient descent are the core |
| [Transformers](transformers/) (when built) | Chain rule (backprop), softmax (`eˣ`), positional encoding (sin/cos), attention (matrix multiplication) |
| [RAG](rag/) | Cosine similarity (dot product on normalized vectors), embedding spaces |
| [Agents](agents/) | Indirect — but every model the agent calls is built on this |

---

## What's NOT Covered Here

This is a working reference, not a complete textbook. For deeper treatment of:

- **Probability and statistics** (Bayes' theorem, distributions, expected value) — needed for reinforcement learning, generative models, uncertainty quantification
- **Linear algebra in depth** (eigenvalues, SVD, vector spaces) — needed for PCA, advanced architecture analysis
- **Information theory** (entropy, KL divergence) — needed for understanding loss functions like cross-entropy at depth
- **Calculus of variations** — needed for advanced theoretical derivations

…go to a textbook. The [Math for AI Colab notebook](https://colab.research.google.com/github/sunilmogadati/ai-engineer-accelerator/blob/main/AI_Engineer_Accelerator_Math_for_AI.ipynb) covers the next layer up with worked examples.

---

**Hands-on:** [Math for AI on Colab](https://colab.research.google.com/github/sunilmogadati/ai-engineer-accelerator/blob/main/AI_Engineer_Accelerator_Math_for_AI.ipynb) — run every concept with code, plots, and the Architect's Mental Model checklist.
