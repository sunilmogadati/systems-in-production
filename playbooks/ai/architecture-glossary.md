# AI Architecture Glossary

A cross-architecture reference covering terminology for GAN, RNN/LSTM, U-Net, ResNet, and Transformers, plus shared foundational concepts. Definitions are tight (1–3 sentences) and use plain English. Cross-cuts the [generative/](generative/), [computer-vision/](computer-vision/), [sequence-models/](sequence-models/), and [transformers/](transformers/) playbooks — each one references this document.

For the math primer that defines derivatives, chain rule, and gradient descent, see [Math for AI](math-for-ai.md). For worked numerical examples (parameter counts, shape arithmetic, by architecture), see [Architecture Math](architecture-math.md).

---

## Part 1 — Foundations (apply to all topics)

### Numbers and Shapes
- **Scalar** — a single number.
- **Vector** — a 1-D list of numbers, e.g., `[1, 2, 3]`.
- **Matrix** — a 2-D grid of numbers (rows × columns).
- **Tensor** — a generalization to any number of dimensions; in PyTorch, every input/output is a tensor.
- **Shape** — the size of a tensor along each dimension, e.g., `(batch=64, channels=3, height=224, width=224)`.

### Network Anatomy
- **Neuron** — a single unit that computes a weighted sum of its inputs plus a bias, then applies an activation function.
- **Layer** — a group of neurons that all process the same input together.
- **Hidden layer** — any layer between input and output. Networks with multiple hidden layers are "deep."
- **Fully-connected (FC) / Linear / Dense layer** — every input is connected to every output by a weight; output = `W·x + b`.
- **Weight** — a learnable number that scales an input.
- **Bias** — a learnable number added after the weighted sum; lets the layer shift its output independently of input.
- **Parameters** — collective name for all weights and biases. The total count is what we mean by "model size."
- **Hyperparameters** — settings chosen before training (learning rate, batch size, number of layers); not learned.

### Forward and Backward Pass
- **Forward pass** — compute outputs by feeding inputs through the network layer-by-layer.
- **Loss function (objective, cost)** — a number measuring how wrong the model's output is. Lower = better.
- **Backward pass / backpropagation** — apply the chain rule to compute the gradient of the loss with respect to every weight, layer-by-layer in reverse.
- **Gradient** — for one parameter, the partial derivative of the loss with respect to that parameter; tells us which direction to nudge it.
- **Chain rule** — calculus rule used to propagate gradients through composed functions (i.e., through layers).

```
FORWARD PASS  (compute outputs):

  input ──▶ Layer 1 ──▶ Layer 2 ──▶ Layer 3 ──▶ output ──▶ loss
            (W1, b1)    (W2, b2)    (W3, b3)

BACKWARD PASS  (compute gradients via chain rule):

           ◀── ∂L/∂W1 ◀── ∂L/∂W2 ◀── ∂L/∂W3 ◀── ∂L/∂out ◀── loss

UPDATE  (optimizer applies gradients):

  W1 ← W1 - η·∂L/∂W1     W2 ← W2 - η·∂L/∂W2     W3 ← W3 - η·∂L/∂W3
```

### Training Basics
- **Optimizer** — algorithm that uses gradients to update parameters. Examples: SGD, Adam.
- **Learning rate (η)** — how big a step the optimizer takes in the gradient direction.
- **Epoch** — one full pass through the entire training dataset.
- **Batch / minibatch** — a small group of training examples processed together (e.g., 64 images). Trades off compute and gradient noise.
- **Iteration / step** — one parameter update (one batch's worth).

### Optimizers
- **SGD (Stochastic Gradient Descent)** — basic update: `θ ← θ - η·g`.
- **Momentum** — keeps a running average of gradients; smooths the trajectory.
- **AdaGrad** — gives each parameter its own learning rate that shrinks based on the sum of past squared gradients. Good for sparse features.
- **RMSProp** — like AdaGrad but uses an exponential moving average of squared gradients (doesn't shrink forever).
- **Adam** — combines momentum (1st moment) and RMSProp (2nd moment). Default choice for most deep learning.
- **AdamW** — Adam with decoupled weight decay; modern default for transformers.
- **Hyperparameters** — `β1` controls momentum decay, `β2` controls squared-gradient decay, `ε` prevents division by zero.

### Weight Initialization
- **Xavier (Glorot)** — variance `2/(fan_in + fan_out)`. For sigmoid/tanh networks.
- **He (Kaiming)** — variance `2/fan_in`. For ReLU networks.
- **DCGAN init** — `N(0, 0.02²)` for conv weights, `N(1, 0.02²)` for BN scale.
- **Forget-gate bias = 1** — special LSTM init; makes the network start by remembering rather than forgetting.

### Activation Functions
- **Activation function** — a non-linear function applied element-wise to a layer's output. Without it, stacking layers would still be a linear function.
- **Sigmoid** — `1/(1+e^-z)`, output range `(0, 1)`. Used for gates and binary outputs.
- **Tanh (hyperbolic tangent)** — output range `(-1, +1)`. Used in RNN hidden state, LSTM candidate, GAN G output.
- **ReLU (Rectified Linear Unit)** — `max(0, z)`. Most common; sparse activations, simple gradient. Can have "dead neurons."
- **LeakyReLU** — like ReLU but with a small slope for negative inputs (e.g., 0.2). Prevents dead neurons.
- **GELU (Gaussian Error Linear Unit)** — smooth ReLU variant; standard in BERT, GPT-2+.
- **Softmax** — turns a list of numbers into a probability distribution: `e^z_i / Σ e^z_j`. Used for multi-class outputs and attention weights.
- **Saturation** — when an activation flattens out (sigmoid near 0 or 1, tanh near ±1), gradients become near zero.

### Loss Functions
- **MSE (Mean Squared Error)** — `(1/n)·Σ(ŷ - y)²`. Used for regression and autoencoders.
- **BCE (Binary Cross-Entropy)** — `-y·log(p) - (1-y)·log(1-p)`. For binary classification and GANs.
- **BCEWithLogitsLoss** — sigmoid + BCE in one numerically stable operation. Used in GANs to avoid `log(0)`.
- **Cross-Entropy (CE / Categorical CE)** — `-log(p_true_class)`. For multi-class classification (RNN, U-Net, transformers).
- **Dice loss** — `1 - 2|P∩T|/(|P|+|T|)`. Measures region overlap; robust to class imbalance.
- **Logits** — raw, unnormalized output scores (before sigmoid or softmax). PyTorch's `*WithLogits*` losses take logits directly for stability.

### Common Layers and Operations
- **Convolution (Conv2d)** — slides a small filter across the input, computing a weighted sum at each position. Filter dimensions = `(k_h, k_w, C_in)` per output channel.
- **Kernel / Filter** — the small weight grid used by a convolution (e.g., 3×3).
- **Stride** — how many pixels the filter jumps each step. Stride > 1 shrinks output spatially.
- **Padding** — extra zeros (or other values) added around input borders to control output size. "Same padding" keeps spatial size unchanged with stride 1.
- **Channel** — depth dimension of a tensor; e.g., RGB images have 3 channels.
- **Pooling** — fixed (non-learnable) downsampling. Max-pool keeps the largest value in each window; avg-pool keeps the mean.
- **Transposed convolution (deconvolution, fractionally-strided conv)** — learnable upsampling. Output is spatially larger than input. NOT the inverse of convolution.
- **1×1 convolution** — a convolution with kernel size 1×1; does no spatial mixing, just a per-pixel linear layer over channels.
- **Receptive field** — the region of the input that affects one output unit. Grows deeper into the network.

```
CONVOLUTION  (shrinks spatial — N=6, k=3, s=1, p=0):

  input 6×6                     output 4×4
  ┌─────────────┐               ┌─────────┐
  │ ▓ ▓ ▓ . . . │               │ * * * * │
  │ ▓ ▓ ▓ . . . │ ────slide───▶ │ * * * * │
  │ ▓ ▓ ▓ . . . │   3×3 kernel  │ * * * * │
  │ . . . . . . │               │ * * * * │
  │ . . . . . . │               └─────────┘
  │ . . . . . . │               N_out = (6 + 0 - 3)/1 + 1 = 4
  └─────────────┘

TRANSPOSED CONVOLUTION  (grows spatial — N=2, k=3, s=2, p=0):

  input 2×2                     output 5×5
  ┌─────┐                       ┌───────────┐
  │ ▓ ▓ │ ────expand───▶        │ * * * * * │
  │ ▓ ▓ │   each input cell     │ * * * * * │
  └─────┘   places the kernel   │ * * * * * │
            then strides by 2   │ * * * * * │
                                │ * * * * * │
                                └───────────┘
                                N_out = (2-1)·2 - 0 + 3 = 5
```

### Normalization and Regularization
- **BatchNorm** — re-centers and re-scales activations across the batch dimension. Stabilizes CNN training.
- **LayerNorm** — normalizes across the feature dimension within each token, independent of batch. Used in transformers.
- **Dropout** — randomly zeros out a fraction of activations during training; turned off at inference. Regularization technique.
- **Regularization** — anything that discourages overfitting (dropout, weight decay, data augmentation).
- **Data augmentation** — synthetically expanding the training set with transformations (flips, rotations, elastic deformation).

### Common Failure Modes
- **Overfitting** — model fits training data too well; doesn't generalize.
- **Underfitting** — model can't capture the training pattern; loss stays high on both train and test.
- **Vanishing gradient** — gradients shrink to near-zero through deep networks or long sequences; early layers stop learning.
- **Exploding gradient** — gradients blow up (often NaN); training destabilizes.
- **Gradient clipping** — fix for exploding: rescale the gradient if its norm exceeds a threshold (`g ← g·θ/‖g‖`).
- **Dead neuron** — a ReLU neuron stuck in the negative region; gradient = 0; never updates.
- **Mode collapse** — generative model produces only a narrow set of outputs.

### Learning Paradigms
- **Supervised learning** — training data has labels; loss compares prediction to label.
- **Unsupervised learning** — no labels; the model learns structure from data alone.
- **Reinforcement learning** — agent interacts with environment, receives reward/penalty; learns a policy.
- **Generative model** — learns `p(x)` or `p(x, y)`; can produce new samples.
- **Discriminative model** — learns `p(y | x)`; classifies but cannot generate.

### Encoding Schemes
- **One-hot encoding** — represent a discrete label as a vector with 1 in the correct slot and 0 elsewhere. Wasteful for large vocabularies.
- **Embedding** — a learned dense vector for each discrete item (word, token, ID); similar items get similar vectors.

### Important Math Concepts
- **Dot product** — multiply corresponding entries of two vectors and sum. Measures similarity. Identical vectors give a big dot product; opposite vectors give a small or negative one.
- **Matrix multiplication** — bulk dot products: `(A·B)[i][j] = Σ_k A[i][k]·B[k][j]`. If `A` is `(m×k)` and `B` is `(k×n)`, then `A·B` is `(m×n)`.
- **Element-wise (Hadamard) product** — multiply corresponding entries; written `⊙`. Both operands must be the same shape. Used in LSTM gate operations: `f_t ⊙ C_{t-1}`.
- **Transpose** — swap rows and columns of a matrix; written `Aᵀ`. Shape `(m, n)` becomes `(n, m)`.
- **Norm (L2)** — magnitude of a vector: `‖v‖ = √(Σ v_i²)`. Used in gradient clipping.
- **Eigenvalues / singular values** — decomposition of a matrix into stretching factors. Vanishing/exploding gradient happens when the spectral norm (largest singular value) of the recurrent matrix is < 1 or > 1.
- **Toeplitz matrix** — a banded matrix where each diagonal is constant. Convolution can be expressed as multiplication by a Toeplitz matrix; transposed convolution = multiplication by its transpose.

### Tensor Operations (PyTorch idioms)
These come up in code-trace questions, especially for multi-head attention and reshape-heavy operations.
- **`.view(*shape)`** — reshape a contiguous tensor without copying. Requires the new shape to be compatible with strides; will error on non-contiguous tensors.
- **`.reshape(*shape)`** — like `view`, but copies if needed (always works).
- **`.transpose(dim1, dim2)`** — swap two specified dimensions. Often used as `Q.transpose(-2, -1)` to flip the last two dims for `Q·Kᵀ`.
- **`.permute(*dims)`** — reorder all dimensions arbitrarily. More general than transpose.
- **`.contiguous()`** — return a memory-contiguous version (often needed before `view` after a transpose).
- **`.unsqueeze(dim)` / `.squeeze(dim)`** — add or remove a dim of size 1. Used for broadcasting alignment.
- **Broadcasting** — when two tensors have different but compatible shapes, the smaller one is virtually expanded to match. E.g., `(B, 1, D) + (B, T, D)` works.
- **`@` operator** — matrix multiplication (equivalent to `torch.matmul`). For 2D, `A @ B` is matrix multiply; for batched, it's batched matmul over the last two dims.
- **`einsum`** — `torch.einsum("bij,bjk->bik", A, B)` is batched matmul; explicit and often clearer for multi-axis operations.
- **`torch.cat([a, b], dim=k)`** — concatenate along dimension `k`. U-Net skip connections use `dim=1` (channel dim).
- **`torch.stack([a, b], dim=k)`** — concatenate along a NEW dim. Different from cat.

### Computational Graph
PyTorch builds a directed acyclic graph (DAG) during the forward pass: nodes are tensor operations, edges are tensor flows. When `loss.backward()` is called, it walks this graph in reverse order and accumulates gradients into each parameter's `.grad` field via the chain rule. Key concepts:
- **`.requires_grad = True`** — marks a tensor for tracking; PyTorch will record operations on it.
- **`with torch.no_grad():`** — context manager that disables graph tracking. Used in inference, GAN D-step (so G's path is not recorded), or any place gradients aren't needed. Saves memory.
- **`.detach()`** — returns a new tensor disconnected from the graph. Common alternative to `no_grad()`.
- **Graph rebuilt every forward pass** — PyTorch is "define-by-run" (dynamic graph), unlike TensorFlow 1.x's static graphs. Each iteration creates a fresh graph.
- **`zero_grad()`** — needed before each backward pass because PyTorch *accumulates* gradients into `.grad`. Without zeroing, gradients from previous batches would carry over.

---

## Part 2 — GAN Terminology

### Two-Player Setup
- **Generator (G)** — the "forger" network that maps random noise to fake samples.
- **Discriminator (D)** — the "detective" network that classifies inputs as real or fake.
- **Critic** — alternative name for D in Wasserstein GANs (outputs a continuous score, not a probability).
- **Adversarial training** — training G and D against each other; G tries to fool D, D tries to catch G.
- **Minimax objective** — `min_G max_D V(D, G)`; D maximizes, G minimizes the same expression.

### Inputs and Latent Space
- **Noise vector (z)** — a random list of numbers fed to G as the seed for generation. Typically 100-dimensional.
- **Latent space** — the abstract vector space the noise lives in. "Latent" means hidden — the dimensions are abstract knobs the network learns to use.
- **Latent dimension** — design choice for noise vector length (e.g., 100 in introductory GANs, 512–1024 in production).
- **Standard normal distribution** — bell curve centered at 0 with variance 1; written `N(0, 1)`.
- **Uniform distribution on [-1, 1]** — every value in the range equally likely; outside the range, impossible.
- **Latent-space interpolation** — slowly morphing between two outputs by interpolating their `z` vectors. A popular GAN demo.

### Outputs and Labels
- **Fake** — any sample produced by G.
- **Real** — a sample drawn from the training dataset.
- **Target** — the label used in BCE loss. Real → 1, fake → 0.
- **Auto-generated labels** — what makes GAN training "unsupervised at the system level": labels come from the source of each sample, not human annotation.

### Loss Variants
- **D loss** — `-log D(x) - log(1 - D(G(z)))`; D wants both terms minimized (i.e., `D(x)→1, D(G(z))→0`).
- **G loss (saturating)** — `log(1 - D(G(z)))`; vanishes when D wins early. Avoid.
- **G loss (non-saturating)** — `-log D(G(z))`; gives stronger gradients early in training. Used in code.
- **BCE summed (not averaged)** — D's total loss is the sum of real_loss and fake_loss, not their mean.

### Training Procedure
- **D-step** — freeze G, update D on a batch of real + fake images.
- **G-step** — freeze D's optimizer (but D's gradients still compute), update G to fool D.
- **`torch.no_grad()`** — context manager that tells PyTorch not to track gradients; used during D-step around fake-image generation so G's weights don't update.
- **Two optimizers** — D and G have separate optimizers; each updates only its own parameters.
- **Two backward passes per batch** — D-step has one, G-step has another. Distinguishes GAN training from regular CNN training.

```
GAN TRAINING (per minibatch — TWO separate forward+backward passes)

  ┌─── D-STEP (G frozen via no_grad) ──────────────────────────┐
  │                                                            │
  │   real batch ──────▶ D ──▶ logits ──▶ BCE(target=1) = L_real
  │                                                          │ │
  │   z ──▶ G ─(no_grad)─▶ fakes ──▶ D ──▶ BCE(target=0) = L_fake
  │                                                          │ │
  │                                              L_D = L_real + L_fake (SUM)
  │                                                          │ │
  │                                              backward    │ │
  │                                                          ▼ │
  │                                              d_optim.step() ── updates D only
  └────────────────────────────────────────────────────────────┘

  ┌─── G-STEP (D not frozen, but only G updated) ──────────────┐
  │                                                            │
  │   z ──▶ G ──▶ fakes ──▶ D ──▶ logits ──▶ BCE(target=1) = L_G
  │                                                          │ │
  │                                              backward    │ │
  │                       (gradient flows D → G; D's grads discarded)
  │                                                          ▼ │
  │                                              g_optim.step() ── updates G only
  └────────────────────────────────────────────────────────────┘

  KEY: target=1 in G-step is the GOAL (fool D), not the truth.
       D and G never update simultaneously.
```

### Equilibrium and Failure Modes
- **Nash equilibrium** — `p_G = p_data` and `D*(x) = 0.5` everywhere. The discriminator is forced to guess.
- **Mode collapse** — G discovers one (or a few) outputs that fool D and produces only those, ignoring data diversity.
- **Oscillation** — G and D chase each other in cycles without converging.
- **Saturation** — G's loss curve flattens (gradient near zero), training stalls.

### DCGAN
- **DCGAN (Deep Convolutional GAN)** — convolutional GAN architecture from Radford, Metz, Chintala (2015).
- **Strided convolution** — a Conv2d with stride > 1; shrinks spatial output; replaces pooling in DCGAN's discriminator.
- **Transposed convolution** — learnable upsampling; replaces pooling in DCGAN's generator. Sometimes called "deconvolution" (misleading name).
- **DCGAN's five rules** — (1) replace pooling with strided/transposed convs, (2) BatchNorm except G output and D input, (3) no FC hidden layers, (4) ReLU in G + LeakyReLU in D, (5) init from `N(0, 0.02²)`.

### Inference and Toeplitz
- **After training** — discard D; deploy G alone for sample generation.
- **Toeplitz matrix view of convolution** — convolution can be written as matrix multiplication `Y = C·X` where C is a banded Toeplitz matrix.
- **Transposed convolution = `C^T·X`** — multiplication by the transpose. Not the inverse: `C^T·C·X ≠ X`.

### GAN Variants (briefly — less common in production, but worth recognizing)
- **Wasserstein GAN (WGAN)** — replaces BCE with the *Earth Mover's distance*. D becomes a "critic" that outputs a continuous score (no sigmoid). Training is more stable. Original WGAN clips weights; WGAN-GP uses a gradient penalty.
- **Conditional GAN (cGAN)** — both G and D receive an extra conditioning input (e.g., a class label). G produces samples *of that class*; D scores realness *given* that class.
- **LSGAN (Least Squares GAN)** — replaces BCE with mean-squared error. Reduces saturation problems.
- **StyleGAN** — high-resolution face generation; introduces style mixing and adaptive instance norm. Cultural milestone, not central here.
- **CycleGAN** — image-to-image translation without paired data; uses two G/D pairs and a cycle-consistency loss.

### GAN Inference and Sampling
- **Sampling** — draw `z ~ N(0, I)` (or `U[-1, 1]`), pass through G, get a sample. Each new `z` produces a different sample.
- **Latent walk** — slowly interpolate `z` between two points; observe smooth morphing between outputs. Demonstrates that the latent space is structured.
- **Truncation trick** — draw `z` and then shrink it toward zero; produces higher-quality but lower-diversity samples (used in BigGAN, StyleGAN).

---

## Part 3 — RNN / BPTT / LSTM Terminology

### Sequences and Recurrence
- **Sequence** — an ordered list where the order matters (words in a sentence, frames in a video, time-series data).
- **Time step (t)** — one position in the sequence.
- **Recurrent** — uses its own previous output as part of the next input.
- **Recurrent neural network (RNN)** — processes a sequence one step at a time, maintaining a hidden state.
- **Hidden state (h_t)** — the network's running summary of everything seen so far. Updated at each step.
- **Weight sharing across time** — the same weight matrices (`W_xh`, `W_hh`, `W_hy`) are used at every time step. Why param count is independent of sequence length.

### Vanilla RNN Specifics
- **Forward equation**: `h_t = tanh(W_xh·x_t + W_hh·h_{t-1} + b_h)`.
- **Output**: `ŷ_t = softmax(W_hy·h_t + b_y)` (with cross-entropy loss).
- **Initial state**: `h_0 = 0` typically.
- **Tanh in hidden update** — bounded output prevents h from exploding through repeated multiplications.

```
RNN UNROLLED OVER TIME

  x_1            x_2            x_3            x_T
   │              │              │              │
   │ W_xh         │ W_xh         │ W_xh         │ W_xh
   ▼              ▼              ▼              ▼
 ┌─────┐ W_hh  ┌─────┐ W_hh  ┌─────┐ W_hh  ┌─────┐
 │ h_1 │ ────▶ │ h_2 │ ────▶ │ h_3 │ ─...─▶│ h_T │
 └─────┘       └─────┘       └─────┘       └─────┘
   │              │              │              │
   │ W_hy         │ W_hy         │ W_hy         │ W_hy
   ▼              ▼              ▼              ▼
  ŷ_1            ŷ_2            ŷ_3            ŷ_T

  ▲ THE SAME W_xh, W_hh, W_hy MATRICES ARE REUSED AT EVERY STEP
    (weight sharing across time → param count independent of T)

  Loss = Σ_t cross_entropy(ŷ_t, y_t)
  BPTT = backprop through this unrolled graph (chain rule across time)
```

### Training Recurrent Networks
- **Backpropagation Through Time (BPTT)** — unroll the recurrent network across time steps into one big computation graph, then run regular backprop.
- **Unrolling** — visualizing the network laid out flat across time, with the same weights repeated.
- **Truncated BPTT** — backprop only `k` steps instead of the full sequence; saves memory at the cost of long-range learning.
- **Gradient accumulation across time** — `dL/dW = Σ_t (dL_t/dW)`; the same weight matrix is used at each step, so its gradient is the sum of contributions.

### Vanishing and Exploding Gradients
- **Vanishing gradient (RNN context)** — `∂h_t/∂h_{t-1} = W_hh^T·diag(tanh'(...))`; multiplied repeatedly, magnitude decays geometrically. Long-range dependencies are lost.
- **Exploding gradient** — same chain, but if singular values > 1, magnitude grows geometrically; gradient becomes NaN.
- **Gradient clipping** — fix for exploding: if `‖g‖ > θ`, rescale to `g·θ/‖g‖`. Typical `θ = 5`.
- **Why vanilla RNNs can't learn long-range deps** — by the time the gradient flows back many steps, it has decayed to nearly zero.

### LSTM (Long Short-Term Memory)
- **LSTM** — a recurrent architecture that adds a separate "cell state" with explicit gates to control memory flow. Solves vanishing gradient.
- **Cell state (C_t)** — a memory channel that flows through time mostly untouched.
- **Hidden state (h_t)** — like vanilla RNN's hidden state; also exposed as the layer's output.
- **Gate** — a sigmoid-output vector that scales another signal element-wise. Values near 1 keep, values near 0 drop.
- **Forget gate (f_t)** — decides what fraction of the previous cell state to keep.
- **Input gate (i_t)** — decides what fraction of the new candidate to write.
- **Output gate (o_t)** — decides what part of the cell state to expose as the hidden state.
- **Candidate (C̃_t)** — a tanh-bounded vector of new content proposed for the cell state.
- **Cell update equation**: `C_t = f_t ⊙ C_{t-1} + i_t ⊙ C̃_t`. **Additive**, not multiplicative.
- **Hidden update equation**: `h_t = o_t ⊙ tanh(C_t)`.
- **Why LSTM solves vanishing gradient** — `∂C_t/∂C_{t-1} = f_t`; when `f_t ≈ 1`, gradient flows backward through time without geometric decay.
- **Why sigmoid for gates** — fractions in `[0, 1]` are interpretable as "fraction-to-keep / fraction-to-write."
- **Why tanh for candidate** — bounded `[-1, +1]` prevents cell state from exploding.
- **Forget-bias init = 1** — `f_t ≈ 1` early in training; cell state is preserved by default; gradient flows clean.

```
LSTM CELL INTERNALS (one time step)

                     ┌─── CELL STATE HIGHWAY ─────────────────▶ C_t
                     │                              ▲
                     │                              │
                     │   ⊗(forget)         ⊕(add)   │
  C_{t-1} ───────────┼────▶ × ─────────────▶ + ─────┘
                     │      ▲                ▲
                     │      │                │
                     │      f_t              i_t ⊗ C̃_t
                     │      │                │
                     │   ┌──┴───┐         ┌──┴───┐
                     │   │  σ   │         │  σ   │   ┌────────┐
   x_t ──┐           │   │ W_f  │         │ W_i  │   │  tanh  │
         │           │   └──────┘         └──────┘   │  W_c   │
   h_{t-1} ──┐       │      ▲                ▲       └────────┘
         │   │       │      │                │            ▲
         ▼   ▼       │      │                │            │
       concat ───────┼──────┴────────────────┴────────────┘
       z_t = [h_{t-1}; x_t]
                     │                                   ┌────────┐
                     │                                   │  tanh  │
                     │   ┌──────┐                        └────────┘
                     │   │  σ   │ ── o_t                      ▲
                     │   │ W_o  │                             │
                     │   └──────┘                             │
                     │      ▲                                 │
                     │      │                                 │
                     │   z_t (same concat above)              │
                     │                                        │
                     │                                  o_t ⊗ tanh(C_t) ──▶ h_t
                     │                                        ▲
                     └────────────────────────────────────────┘

KEY: The cell-state path (top horizontal) carries memory across time
     with only multiplicative gating (f_t) and additive writes (i_t·C̃_t).
     ∂C_t/∂C_{t-1} = f_t  →  when f_t≈1, gradient flows clean (no vanishing).
```

### Variants and Code
- **GRU (Gated Recurrent Unit)** — simpler LSTM variant with 2 gates (reset, update); merges cell and hidden states into one. ~25% fewer parameters than LSTM. Often performs comparably; sometimes preferred for shorter sequences.
- **GRU equations** (for reference):
  - `r_t = σ(W_r·[h_{t-1}; x_t])` — reset gate (how much past to use)
  - `z_t = σ(W_z·[h_{t-1}; x_t])` — update gate (how much new vs old)
  - `h̃_t = tanh(W·[r_t ⊙ h_{t-1}; x_t])` — candidate
  - `h_t = (1 - z_t) ⊙ h_{t-1} + z_t ⊙ h̃_t` — convex combination of old and new
- **Peephole connection** — gates can also see the cell state directly. Rare in modern code; original LSTMs used them, modern impls usually don't.
- **`nn.RNN`, `nn.LSTM`, `nn.GRU`** — PyTorch's built-in recurrent layers; output is `(out, hidden)`. For LSTM, hidden is `(h_n, c_n)` tuple.
- **`nn.utils.clip_grad_norm_`** — PyTorch helper to clip gradients in place.

### Multi-Layer and Bidirectional RNN
- **Stacked / multi-layer RNN** — `nn.LSTM(num_layers=N)` stacks N RNN layers vertically. The hidden state of each layer becomes the input to the next. Captures hierarchical features. Param count ≈ N × (single-layer LSTM params), with the lower layers using `D` and upper layers using `H` as input dim.
- **Bidirectional RNN** — runs *two* RNNs in opposite directions (one left-to-right, one right-to-left) and concatenates their outputs. Each output position has access to both past and future context. Doubles the parameter count. Used in BERT-style understanding tasks (but not for generation, since future tokens aren't available at inference time).

### Encoder-Decoder (seq2seq with RNNs)
The pre-Transformer architecture for translation-like tasks. An *encoder* RNN reads the source sequence and produces a final hidden state (a "context vector" summarizing the entire input). A *decoder* RNN takes that context as its initial hidden state and generates the target sequence one token at a time.
- **Bottleneck problem** — the entire source sequence's information is squeezed into a single fixed-size vector. Long sequences degrade badly.
- **Attention (Bahdanau, 2014)** — fix for the bottleneck. At each decoder step, compute attention weights over the encoder's hidden states, then take a weighted sum. The decoder can "look at" different parts of the source as needed. This was the precursor to Transformer attention.
- **Bahdanau (additive) vs Luong (multiplicative) attention** — two early attention formulas. Luong's `score = h_dec · h_enc` (a dot product) is what eventually generalized to scaled dot-product attention in Transformers.
- **Teacher forcing** — during training, feed the *true* target token at each decoder step (instead of the model's previous prediction). Speeds up training but creates a train/test mismatch.

### Inference Techniques (for generative RNN/Transformer models)
- **Greedy decoding** — at each step, pick the token with highest probability. Fast but often suboptimal.
- **Beam search** — keep the `k` best partial sequences at each step (the "beam"). Higher-quality output. Common for translation.
- **Top-k sampling** — sample from the top `k` most probable tokens (zeroing out the rest first).
- **Top-p / nucleus sampling** — sample from the smallest set of tokens whose cumulative probability exceeds `p` (e.g., 0.9). Adapts to confidence.
- **Temperature** — divide logits by `T` before softmax. `T < 1` makes the distribution sharper (more deterministic); `T > 1` makes it flatter (more random); `T = 1` is unchanged.

---

## Part 4 — U-Net Terminology

### Task Framing
- **Image segmentation** — assigning a class label to every pixel rather than one label to the whole image.
- **Semantic segmentation** — segmentation where pixels of the same class get the same label (e.g., all "cat" pixels labeled the same, even if there are multiple cats).
- **Instance segmentation** — distinguishes individual instances (cat #1 vs cat #2). Different task; not what U-Net does by default.
- **Pixel-wise classification** — equivalent way to describe semantic segmentation.
- **WHAT vs WHERE** — classifiers know WHAT (object identity) but lose WHERE (spatial location). Segmentation needs both.

### Architecture
- **U-Net** — encoder-decoder architecture with skip connections, originally for biomedical image segmentation (Ronneberger et al. 2015).
- **Encoder path (contracting path)** — sequence of `DoubleConv → MaxPool` stages; halves spatial size, doubles channels each stage.
- **Decoder path (expanding path)** — sequence of `ConvTranspose → concat skip → DoubleConv` stages; doubles spatial, halves channels.
- **Bottom (bottleneck)** — the deepest layer; highest semantic level, lowest spatial resolution. Not really a bottleneck in U-Net because skips bypass it.
- **DoubleConv** — the building block: two 3×3 convolutions with ReLU between (modern: with BatchNorm).
- **U shape** — the visual layout: encoder goes down, decoder goes up, skip connections bridge across.

```
U-NET — THE "U" SHAPE WITH SKIP CONNECTIONS

  input (1×H×W)                                          output (N_classes×H×W)
       │                                                          ▲
       ▼                                                          │
   DoubleConv → 64    ─────── skip c1 (concat) ───────▶  DoubleConv ← 64
       │                                                          ▲
       ▼ MaxPool 2×2                                  ConvTranspose 2×2
       │                                                          │
   DoubleConv → 128   ────── skip c2 (concat) ───────▶  DoubleConv ← 128
       │                                                          ▲
       ▼ MaxPool 2×2                                  ConvTranspose 2×2
       │                                                          │
   DoubleConv → 256   ────── skip c3 (concat) ───────▶  DoubleConv ← 256
       │                                                          ▲
       ▼ MaxPool 2×2                                  ConvTranspose 2×2
       │                                                          │
   DoubleConv → 512   ────── skip c4 (concat) ───────▶  DoubleConv ← 512
       │                                                          ▲
       ▼ MaxPool 2×2                                  ConvTranspose 2×2
       │                                                          │
   DoubleConv → 1024 ──────────────────────────────────────────────┘
                          (bottom — deepest features)

Encoder: spatial HALVES, channels DOUBLE.
Decoder: spatial DOUBLES, channels HALVE.
Skip connection: concat along channel dim (dim=1), NOT add.
Final: 1×1 Conv to N_classes channels (per-pixel classifier).
```

### Skip Connections (the central insight)
- **Skip connection** — a wire that carries activations from an early layer directly to a later layer, bypassing intermediate layers.
- **Concat (along channel dim)** — `torch.cat([up, skip], dim=1)`. Channels stack, spatial unchanged.
- **Why concat, not add** — concatenation preserves both signals as separate channels; addition mixes them and loses information.

### Layers and Operations
- **MaxPool2d** — non-learnable downsampler that takes the largest value in each window. 2×2 with stride 2 halves spatial dimensions.
- **ConvTranspose2d** — learnable upsampling. Different shape formula than Conv2d: `N_out = (N_in - 1)·s - 2p + f`.
- **1×1 convolution** — final layer of U-Net; per-pixel linear classifier with no spatial mixing.
- **Same padding** — `p = (f-1)/2`; keeps spatial size unchanged with stride 1.
- **Valid padding** — `p = 0`; output is smaller (original 2015 paper used this; modern impls use same padding).

### Loss Functions
- **Cross-Entropy (per pixel)** — standard multi-class loss applied at every pixel position.
- **Dice loss** — `1 - 2|P∩T|/(|P|+|T|)`; measures region overlap; robust to class imbalance.
- **Class imbalance** — when one class dominates (e.g., 95% of pixels are background); pixel-wise CE incentivizes predicting the dominant class.
- **CE + Dice combo** — common in medical imaging; combines pixel-level accuracy with region-level overlap.

### Training
- **Elastic deformation** — data augmentation by warping the image with smooth random distortions; the U-Net paper's key trick for tiny biomedical datasets.
- **He / Kaiming initialization** — standard for the ReLU-heavy U-Net.

### Connection to Theory
- **Fully convolutional network (FCN)** — no fully-connected layers anywhere; works on any input size. U-Net is FCN.
- **Toeplitz matrix view of conv** — convolution as matrix multiplication; transposed convolution is multiplication by the transpose, not the inverse.
- **U-Net vs autoencoder** — both have encoder-decoder structure. Autoencoder has no skip connections → blurry outputs. U-Net's skips inject high-resolution detail back into the decoder.

### Segmentation Metrics
- **IoU (Intersection over Union, also called Jaccard index)** — `|P ∩ T| / |P ∪ T|`. The fraction of "pixels you got right" out of "all pixels claimed by either side." Ranges 0 to 1; 1 = perfect match.
- **Dice coefficient** — `2|P ∩ T| / (|P| + |T|)`. Closely related to IoU but uses the SUM rather than the union in the denominator. Always ≥ IoU. Same as F1 score.
- **Pixel accuracy** — fraction of correctly classified pixels. Misleading under class imbalance (always-background gets high accuracy on a 95%-background image).
- **Mean IoU (mIoU)** — IoU averaged over classes; standard for multi-class segmentation benchmarks (Pascal VOC, Cityscapes).

### U-Net Variants and Extensions (briefly)
- **3D U-Net** — replaces 2D convs with 3D for volumetric data (CT, MRI scans).
- **Attention U-Net** — adds attention gates on the skip connections, letting the decoder weight which encoder features matter most.
- **U-Net++ / Nested U-Net** — denser skip pathways, more decoder stages.
- **TransUNet** — replaces the U-Net's bottom layer with a transformer encoder for better global context.
- **nnU-Net** — "no new U-Net": automated configuration of vanilla U-Net per dataset; surprisingly hard to beat in medical imaging.

---

## Part 4.5 — ResNet Terminology

### The problem and the fix
- **Degradation problem** — adding more layers to a plain deep CNN makes accuracy *worse*, even on the training set. Not overfitting — an *optimization* problem. Plain 56-layer CNN performs worse than 20-layer CNN.
- **ResNet (Residual Network)** — He, Zhang, Ren, Sun, 2015. Introduced residual connections to enable training of networks 50–152 layers deep.
- **ImageNet ILSVRC 2015** — ResNet-152 won with 3.57% top-5 error. First architecture to break past ~22 layers (VGG-19, GoogLeNet-22 were the prior leaders).

### Core concepts
- **Residual block** — a small group of layers wrapped with an identity shortcut. Output is `y = F(x) + x` where `F(x)` is what the layers compute and `x` is the input.
- **Residual function** `F(x) = H(x) − x` — the *difference* between what we want (`H(x)`, the desired output) and what we already have (`x`, the input). The block learns this difference, not the full mapping.
- **Identity shortcut (skip connection)** — the wire from input directly to output. Parameter-free; just addition.
- **Why it works** — (1) gradient flow: `∂L/∂x = ∂L/∂y · (1 + ∂F/∂x)`; the `+1` keeps gradient flowing even if `∂F/∂x ≈ 0`. (2) Optimization: identity (`F = 0`) is the trivial solution — adding blocks can never hurt.
- **Identity vs projection shortcut** — identity when shapes match; projection (`y = F(x) + W_s·x` with a 1×1 conv) when channel count or spatial size changes.
- **Signal-amplification intuition** — without shortcuts, the input signal "thins out" through depth. The shortcut re-injects the input, keeping it "loud" at later layers.

### Block structure
- **Basic block** (ResNet-18, 34) — two stacked 3×3 convs: `Conv 3×3 → BN → ReLU → Conv 3×3 → BN → (+ shortcut) → ReLU`.
- **Bottleneck block** (ResNet-50, 101, 152) — sandwich `1×1 → 3×3 → 1×1` to reduce, process, then restore channel count. Cheaper than a basic block at the same channel width.
- **Architecture pattern** — start with `7×7` conv on input image, then stacks of residual blocks. Periodically double the number of filters and use stride 2 for downsampling.
- **Common depths** — ResNet-18, ResNet-34, ResNet-50, ResNet-101, ResNet-152.

### Cross-architecture connections (the central insight)
ResNet's "additive shortcut to preserve gradient flow" is the same idea used in:
- **LSTM cell update** — `C_t = f_t ⊙ C_{t-1} + i_t ⊙ C̃_t`. Additive update preserves gradient through *time*; ResNet preserves it through *depth*.
- **Transformer residual stream** — `x + Sublayer(x)` around every sub-block (MHA, FFN). Stacked transformer blocks rely on this to train deep.
- **Distinct from U-Net's skip** — U-Net concatenates (along channel dim), not adds. Different goal: U-Net preserves spatial detail by fusing encoder and decoder feature maps; ResNet preserves gradient flow within a block.

### Common questions
- "What problem does ResNet solve?" → degradation in deep networks; can train 100+ layers.
- "Why does the residual connection help training?" → gradient has a direct unattenuated path back via the `+1` term in `∂L/∂x = ∂L/∂y · (1 + ∂F/∂x)`.
- "Does the shortcut add parameters?" → No — identity shortcut is a pure addition. Only projection shortcuts (when shapes change) add a small 1×1 conv.
- "Compare ResNet's skip with U-Net's skip." → ResNet adds (within block, gradient flow); U-Net concatenates (cross-block, spatial detail). Different purposes.
- "Compare ResNet to LSTM." → Both use additive shortcuts. ResNet preserves gradient through depth; LSTM through time.
- "Why does ResNet use BatchNorm?" → standard CNN regularizer; stabilizes activation distributions across the deep stack. BN is standard in every Conv → BN → ReLU block in ResNet.

---

## Part 5 — Transformers Terminology

### Tokens and Embeddings
- **Token** — one unit of input: a word, sub-word, or character.
- **Tokenization** — splitting raw text into tokens.
- **Vocabulary** — the fixed set of tokens the model knows (typically 30k–50k).
- **Vocabulary size (V)** — size of the vocabulary.
- **One-hot encoding** — representing a token as a vector with 1 in its slot and 0 elsewhere; wasteful, no notion of similarity.
- **Embedding** — a learned dense vector for each token. The embedding layer is a lookup table of shape `(V × d_model)`.
- **Token embedding** — the dense vector representing a single token.
- **Embedding layer** — the learnable matrix that maps token IDs to vectors.
- **Contextual embedding** — the output of a transformer layer for one token; the token's representation refined by attention to neighbors.
- **Static embedding** — context-independent (e.g., Word2Vec); same vector regardless of surrounding tokens.
- **Word2Vec, GloVe** — older embedding methods that produce static vectors.

### Tokenization Strategies
- **Word-level** — one token per word. Simple but huge vocabularies and out-of-vocabulary (OOV) problems. Rare words → `<UNK>`.
- **Character-level** — one token per character. Tiny vocab (~256), no OOV, but very long sequences and weak per-token signal.
- **Sub-word tokenization** — modern compromise. Common words stay whole; rare words get split into known sub-pieces. No OOV problem. The three big algorithms:
  - **BPE (Byte-Pair Encoding)** — start with characters, repeatedly merge the most frequent adjacent pair into a new token. Used by GPT.
  - **WordPiece** — similar to BPE but uses likelihood-based merging. Used by BERT.
  - **SentencePiece** — operates on raw bytes/Unicode without language-specific tokenization. Used by T5, Llama.
- **Special tokens** — `[CLS]`, `[SEP]`, `[MASK]`, `[PAD]`, `<s>`, `</s>`. Reserved IDs in the vocab for control purposes.

### Positional Information
- **Permutation-invariant** — self-attention treats input as an unordered set; shuffling tokens gives the same output reshuffled.
- **Positional encoding** — a position-dependent vector added to the embedding so the network knows token order.
- **Sinusoidal encoding** — original Vaswani 2017 choice: `PE(pos, 2i) = sin(pos / 10000^(2i/d))`, `PE(pos, 2i+1) = cos(...)`. Linearly recoverable for relative positions.
- **Learned positional embedding** — train a separate position vector per slot (BERT, GPT-2).
- **RoPE (Rotary Positional Embedding)** — applies a position-dependent rotation; used in Llama, Mistral.
- **ALiBi** — adds a position-dependent linear bias to attention scores; extrapolates well to longer sequences.

### Attention Mechanism
- **Attention** — a mechanism that lets each position in a sequence look directly at every other position and weight how much each one matters.
- **Self-attention** — attention where queries, keys, and values all come from the same input sequence.
- **Cross-attention** — attention where queries come from one source (decoder) and keys/values from another (encoder); used in encoder-decoder models.
- **Query (Q)** — vector representing what a token is looking for. Computed as `Q = X·W_Q`.
- **Key (K)** — vector representing what a token offers; matched against queries. `K = X·W_K`.
- **Value (V)** — vector representing what a token actually delivers. `V = X·W_V`.
- **Attention scores** — `Q·Kᵀ`, the matrix of pairwise similarities.
- **Scaled dot-product attention** — `softmax(QKᵀ / √d_k)·V`.
- **`√d_k` scaling** — prevents dot products from growing with `d_k`; without it, softmax saturates and gradients vanish.
- **Attention weights** — softmax of scaled scores; row-wise probability distribution over keys.
- **`d_k`** — dimension of each query/key vector. `d_v` is the value dim (usually equal to `d_k`).

```
SELF-ATTENTION FLOW (one head)

  Input X (n × d_model)
   │
   ├──▶ × W_Q ──▶ Q (n × d_k)  ──┐
   │                              │
   ├──▶ × W_K ──▶ K (n × d_k)  ──┤
   │                              ▼
   │                         Q · Kᵀ   ──▶ ÷ √d_k ──▶ softmax(dim=-1)
   │                       (n × n)                          │
   │                                                         │
   │                                                  weights (n × n)
   │                                                         │
   └──▶ × W_V ──▶ V (n × d_v)  ──────────────────▶ weights · V
                                                             │
                                                             ▼
                                                      output (n × d_v)

  Steps: 1. Project X into Q, K, V via three learned matrices
         2. Score = Q · Kᵀ  (pairwise similarity, n×n matrix)
         3. Scale by √d_k   (prevents softmax saturation)
         4. Softmax row-wise (each row = prob distribution over keys)
         5. Weighted sum of V vectors

  Same shape in (n, d_model) and out (n, d_v). Each output token is a
  context-aware blend of all input token Values.
```

### Multi-Head Attention
- **Multi-head attention (MHA)** — run several attention computations in parallel, each over a slice of the dimensions.
- **Attention head** — one of the parallel attention computations.
- **`h`** — number of heads.
- **Per-head dimension** — `d_k = d_model / h`.
- **W_O (output projection)** — final linear matrix that combines the concatenated head outputs.
- **Why multiple heads** — different heads can specialize (one attends to syntactic neighbors, another to long-range references, another to semantic clusters).

```
MULTI-HEAD ATTENTION (split → parallel attention → concat → project)

  Input X (n × d_model)
       │
       ▼ project with W_Q, W_K, W_V
       │
       ▼ reshape + transpose → split d_model into h heads
       │
   ┌───┴────┬────────┬────────┬─── ... ───┬────────┐
   ▼        ▼        ▼        ▼            ▼        ▼
 head 1   head 2   head 3   head 4      head h-1  head h
 (d_k)    (d_k)    (d_k)    (d_k)        (d_k)    (d_k)
   │        │        │        │            │        │
   │        │        │        │            │        │
 attn ▼  attn ▼   attn ▼   attn ▼        attn ▼   attn ▼
   │        │        │        │            │        │
   └────────┴────────┴────────┴─── ... ────┴────────┘
                          │
                          ▼ concat back to d_model
                          │
                          ▼ × W_O (final projection)
                          │
                          ▼
                   output (n × d_model)

  d_k = d_model / h   (e.g., d_model=512, h=8 → d_k=64)
  Total params: 4 d_model²  (W_Q, W_K, W_V, W_O each d_model × d_model)
```

### Masking
- **Causal mask (autoregressive mask)** — triangular mask preventing each position from looking at future positions. Required during training of decoder-only models so they can't "cheat" by looking ahead.
- **Padding mask** — mask that hides padding tokens (added to make sequences in a batch the same length) from attention.
- **Mask polarity gotcha** — manual implementations often use `1 = allow, 0 = block`; PyTorch's `nn.MultiheadAttention.attn_mask` uses `True = block`. Opposite conventions.

### Encoder Block
- **Encoder block** — one stack-unit of the encoder: self-attention sub-block + FFN sub-block, each wrapped in residual + LayerNorm.
- **Position-wise feed-forward network (FFN)** — a 2-layer MLP applied independently at each position. Hidden dim is typically `4 × d_model`.
- **Residual connection** — additive shortcut: `output = x + Sublayer(x)`. Lets gradients flow around any sub-block.
- **LayerNorm** — normalizes across the feature dimension within each token, independent of batch.
- **Pre-LN** — `x + Sublayer(LayerNorm(x))`. Modern (GPT-2+, Llama). Stable for deep stacks.
- **Post-LN** — `LayerNorm(x + Sublayer(x))`. Original 2017. Needs warmup.

```
TRANSFORMER ENCODER BLOCK (Pre-LN, modern)

  input x ─┬────────────────────────────┐  (residual stream)
           │                            │
           ▼                            │
       LayerNorm                        │
           │                            │
           ▼                            │
   Multi-Head Self-Attention            │
           │                            │
           ▼                            │
           ⊕ ◀──────────────────────────┘   (add — residual)
           │
           ├────────────────────────────┐  (residual stream)
           │                            │
           ▼                            │
       LayerNorm                        │
           │                            │
           ▼                            │
          FFN  (Linear → GELU → Linear) │
           │                            │
           ▼                            │
           ⊕ ◀──────────────────────────┘   (add — residual)
           │
           ▼
        output  (same shape as input: n × d_model)

  Stack N such blocks (N=6 in original, 12 in BERT-base, 96 in GPT-3).

  Per-block params: ≈ 12 d_model²   (4d² MHA + 8d² FFN)
```

```
FULL TRANSFORMER FORWARD PASS

  tokens (ids)
    │
    ▼ Embedding lookup       (V × d_model lookup)
  embeddings (n × d_model)
    │
    + Positional Encoding    (sin/cos or learned)
    │
    ▼
  ┌──────────────────────────┐
  │  Encoder Block × N       │   ← repeat
  │  (MHA + FFN + residuals) │
  └──────────────────────────┘
    │
    ▼
  Final LayerNorm
    │
    ▼
  output (n × d_model)        ── for BERT: classify with pooled [CLS]
                              ── for GPT: project to vocab, softmax → next token
```

### Decoder Block
- **Decoder block** — same structure as encoder but with a *masked* self-attention sub-block (causal mask) and a *cross-attention* sub-block.
- **Cross-attention** — Q comes from decoder, K and V from encoder output. Lets decoder query the encoder's representations.
- **Autoregressive** — generating one token at a time, each conditioned on previous outputs. Decoder-only models do this.

### Architecture Variants
- **Encoder-only (BERT family)** — bidirectional attention; full context visible. Used for understanding tasks (classification, NER, question answering).
- **Decoder-only (GPT family, Llama, Mistral)** — causal attention only. Used for generation and modern chat models.
- **Encoder-decoder (original Transformer, T5, BART)** — encoder reads source, decoder generates target with cross-attention. Used for translation and summarization.

```
THREE TRANSFORMER VARIANTS

  ENCODER-ONLY (BERT)              DECODER-ONLY (GPT)
                                   
  ┌───────────────┐                ┌───────────────┐
  │ Encoder × N   │                │ Decoder × N   │
  │ (bidirectional│                │ (causal mask: │
  │  attention)   │                │  no peek      │
  │               │                │  ahead)       │
  └───────────────┘                └───────────────┘
        │                                 │
        ▼                                 ▼
   classification               next-token generation
   (CLS pooled)                  (autoregressive)
   

  ENCODER-DECODER (original Transformer, T5, BART)
                                   
  ┌───────────────┐                ┌─────────────────────┐
  │ Encoder × N   │ ──── K, V ───▶ │ Decoder × N         │
  │ (reads source)│                │ - masked self-attn  │
  │               │                │ - cross-attn (K,V   │
  └───────────────┘                │   from encoder)     │
                                   │ - FFN               │
        source seq                 └─────────────────────┘
                                            │
                                            ▼
                                      target seq
                                      (translation,
                                       summarization)
```

### Training
- **Masked language modeling** — BERT's pre-training objective: predict randomly masked tokens.
- **Next-token prediction (causal LM)** — GPT's pre-training objective: predict the next token given the previous ones.
- **Label smoothing** — replace one-hot targets with a softened distribution (e.g., 0.9 on correct, 0.01 on others). Prevents over-confidence.
- **Warmup + inverse-square-root decay** — Vaswani 2017 LR schedule: `lr = d_model^(-0.5) · min(step^(-0.5), step · warmup^(-1.5))`.
- **Adam (Vaswani)** — `β1=0.9, β2=0.98, ε=1e-9`. Note `β2=0.98` is non-default.

### Complexity and Trade-offs
- **Complexity O(n²·d)** — time and memory grow quadratically with sequence length. Main bottleneck for long contexts.
- **Context window** — the maximum sequence length the model was trained on (e.g., BERT 512, GPT-3 2048, GPT-4 8k–128k, Claude 200k+, Gemini 1M+). Going beyond it requires special techniques (e.g., RoPE extrapolation, sliding window attention).
- **Why transformers replaced RNNs** — (1) parallel training across positions, (2) direct long-range dependencies (no decay), (3) better scaling with data and compute.
- **Why LayerNorm not BatchNorm** — sequences are variable-length and batch statistics across tokens would be unstable. LayerNorm normalizes per-token.

### Attention Optimizations & Variants (modern but worth recognizing)
- **KV cache** — at inference, decoder-only models reuse the K and V projections of past tokens (they don't change). Each new token only computes its own Q,K,V and reads the cached K,V. Critical for fast generation; memory cost grows with sequence length.
- **Flash Attention** — exact attention rewritten to be IO-aware, fusing softmax with matmul to avoid materializing the full `n × n` matrix in memory. Same outputs as standard attention, much faster.
- **Sparse attention** — restrict attention to a subset of positions (sliding window, strided, blocked). Trades expressivity for `O(n·log n)` or `O(n)` complexity.
- **Linear attention** — approximates softmax via kernel decomposition. `O(n·d²)` instead of `O(n²·d)`.
- **Grouped Query Attention (GQA), Multi-Query Attention (MQA)** — multiple Q heads share K and V projections. Used in Llama 2/3, Mistral, GPT-4. Saves memory and inference compute.

### Pre-training and Fine-tuning Paradigm
- **Pre-training** — train a model on a huge generic corpus with a self-supervised objective (masked LM for BERT, next-token for GPT). Produces a general-purpose foundation model.
- **Fine-tuning** — take a pre-trained model and continue training on a smaller task-specific dataset.
- **Instruction tuning** — fine-tune on `(instruction, response)` pairs to make the model follow natural-language instructions.
- **RLHF (Reinforcement Learning from Human Feedback)** — further refine the model using a reward model trained on human preference data. Used in ChatGPT, Claude.
- **Few-shot / zero-shot learning** — provide examples in the prompt itself (few-shot) or no examples at all (zero-shot). Discovered as an emergent property of large pre-trained models.
- **In-context learning** — the model learns the task pattern from the prompt without weight updates. Surprising emergent ability of LLMs.
- **LoRA (Low-Rank Adaptation)** — fine-tune by adding small rank-decomposition matrices to attention weights instead of training all parameters. Cheap, effective.

### Generation / Inference (decoder-only models)
- **Greedy decoding** — pick max-probability token. Fast, often suboptimal.
- **Beam search** — track top `k` partial sequences. Quality-first; common for translation.
- **Sampling** — draw from the predicted distribution (introduces variation).
- **Top-k sampling** — sample from top `k` most-probable tokens.
- **Top-p (nucleus) sampling** — sample from smallest set whose cumulative prob ≥ p.
- **Temperature** — divide logits by `T` before softmax. Lower = more deterministic; higher = more random.
- **Repetition penalty** — downweight tokens that already appeared. Reduces echoing.

### Code Snippets You Should Recognize
- **`Q @ K.transpose(-2, -1)`** — computes the attention scores in batched form.
- **`scores.masked_fill(mask == 0, float('-inf'))`** — applies a mask before softmax (manual style).
- **`torch.softmax(scores, dim=-1)`** — row-wise softmax over the last dimension.
- **`torch.tril(torch.ones(n, n))`** — creates a lower-triangular causal mask.
- **`Q.view(B, N, h, d_k).transpose(1, 2)`** — reshapes for multi-head attention; the split into heads is a reshape + transpose, not separate matrices.

---

## Cross-Topic Concept Map

If you see a term in the exam, here's where it likely belongs:

| Term | Topic | Page in reference card |
|---|---|---|
| Tanh, ReLU, sigmoid, softmax | Foundations | Page 1 |
| BCE, CE, Dice, MSE | Foundations | Page 1 |
| Adam, SGD, AdaGrad | Foundations | Page 1 |
| `(in+1)·out`, `(k·k·C+1)·C_out` | Foundations | Page 1 |
| Conv2d / ConvTranspose shape formula | Foundations | Page 1 |
| Toeplitz | Foundations + GAN + U-Net | Pages 1, 2, 3 |
| Generator / Discriminator / minimax / mode collapse | GAN | Page 2 |
| DCGAN / strided conv / transposed conv | GAN + U-Net | Pages 2, 3 |
| Latent space / noise vector | GAN | Page 2 |
| Hidden state / cell state / gates | RNN/LSTM | Page 3 |
| BPTT / vanishing gradient / clipping | RNN/LSTM | Page 3 |
| Encoder / decoder / skip connection / DoubleConv | U-Net | Page 3 |
| Dice loss / 1×1 conv / fully convolutional | U-Net | Page 3 |
| Q / K / V / attention / softmax | Transformers | Page 4 |
| Multi-head / FFN / LayerNorm / positional encoding | Transformers | Page 4 |
| Encoder-only / decoder-only / encoder-decoder | Transformers | Page 4 |
| Causal mask / cross-attention | Transformers | Page 4 |
| Pre-LN / Post-LN | Transformers | Page 4 |

---

## Architecture Comparison Matrix

A side-by-side reference for the four architectures covered.

| Aspect | GAN (SimpleGAN/DCGAN) | RNN / LSTM | U-Net | Transformer |
|---|---|---|---|---|
| **Task family** | Generation (unsupervised) | Sequence modeling | Per-pixel classification (supervised) | Sequence modeling / generation |
| **Input shape** | Noise `z` (B, 100) | Sequence (B, T, D) | Image (B, C, H, W) | Sequence (B, n, d_model) |
| **Output shape** | Image (B, C, H, W) | Sequence outputs (B, T, V) | Mask (B, N_classes, H, W) | Sequence (B, n, d_model) |
| **Core building block** | Linear or Conv/ConvT layers | Recurrent cell (RNN/LSTM/GRU) | DoubleConv (3×3 Conv → BN → ReLU)×2 | MHA + FFN with residuals |
| **Hidden activation** | LeakyReLU(0.2) (or ReLU in DCGAN G) | tanh | ReLU | ReLU/GELU (FFN), softmax (attn) |
| **Output activation** | Tanh (G), raw logit (D) | softmax (per step) or none | none (logits) | none (logits) |
| **Loss** | BCE (D); BCE non-saturating (G) | CrossEntropy (sum over time) | CE per pixel + Dice (binary imbalanced) | CrossEntropy (next-token or class) |
| **Parallelizable across time/space?** | ✓ (no time axis) | ✗ Sequential — must wait for `h_{t-1}` | ✓ | ✓ Within sequence |
| **Long-range dependencies** | n/a | Decay geometrically (LSTM mitigates) | Captured via deep encoder | Direct via attention (constant path length) |
| **Param count formula** | Σ Linear or Σ Conv params | 2HC + H² (RNN); 4H(H+D) (LSTM) | Σ DoubleConv blocks | 12·d² per encoder block |
| **Optimizer** | Adam β₁=0.5 (DCGAN) | Adam (or AdaGrad in HW-4) | Adam, lr ~1e-4 | Adam β₂=0.98 + warmup |
| **Why this architecture exists** | Generate without explicit p(x) formula | Handle ordered variable-length sequences | Recover spatial detail lost by pooling | Replace recurrence with parallel attention |
| **Famous failure mode** | Mode collapse, oscillation | Vanishing gradient | Class imbalance (use Dice) | Quadratic memory in seq length |
| **Originally from (year)** | Goodfellow 2014; Radford 2015 (DCGAN) | Elman 1990; Hochreiter 1997 (LSTM) | Ronneberger 2015 | Vaswani 2017 |

## When to use which architecture

| Task | Best fit | Why |
|---|---|---|
| Generate realistic-looking images | GAN, diffusion model | No labels needed; adversarial loss captures realism |
| Translate / summarize text | Encoder-decoder Transformer (T5, BART) | Variable-length input → variable-length output |
| Classify text | Encoder-only Transformer (BERT) | Bidirectional attention sees full context |
| Generate text (chatbot, code) | Decoder-only Transformer (GPT, Llama) | Causal mask enables autoregressive generation |
| Time-series prediction | LSTM or small Transformer | Both work; LSTM cheap on short series, Transformer better with more data |
| Segment medical images | U-Net | Precise per-pixel output with high-resolution preservation |
| Object detection | Faster R-CNN, YOLO, DETR | (Out of scope for this exam) |
| Style transfer | CycleGAN, StyleGAN | Unpaired image-to-image |

## Quick Recall Tips

**GAN training rhythm**: D-step (sum, no_grad), G-step (target=1, grad flows). Two of everything per minibatch.

**LSTM gate order**: f, i, o + C̃. Output sequence: f → i → C̃ → C → o → h. (Or memorize via "Forget what you Inputed, Output a Candidate Cell-state Hidden.")

**U-Net rule of thumb**: encoder doubles channels, halves spatial. Decoder halves channels, doubles spatial. Skips reach across at matching levels.

**Transformer step shapes**: input `(n, d)` → Q, K, V `(n, d_k)` → scores `(n, n)` → weights `(n, n)` → output `(n, d_v)`. Same shape in and out.

**Conv vs ConvTranspose**: Conv shrinks (`floor((N+2p-f)/s)+1`); ConvT grows (`(N-1)s-2p+f`). Different formulas — most common exam trap.

**Param count quick formulas**: Linear `(in+1)·out`. Conv `(k²·C_in+1)·C_out`. RNN `2HC + H²`. LSTM `4H(H+D)`. MHA `4d²`. FFN `8d²`. Transformer block `12d²`.

**Activation choice cheat**: LSTM gates → sigmoid (need fraction); LSTM candidate / RNN h → tanh (bounded symmetric); CNN/U-Net → ReLU; DCGAN G out → Tanh; GAN G hidden → ReLU; GAN D hidden → LeakyReLU.

**The "additive update" fix**: LSTM solves vanishing gradient because cell-state update is `f·C + i·C̃` (additive) — gradient `∂C_t/∂C_{t-1} = f_t` doesn't decay when f≈1. ResNet's `x + F(x)` works for the same reason. Transformer's residual stream too.

**The 3 things attention does in one breath**: (1) Q·Kᵀ similarity → (2) ÷√d_k, softmax → probability dist → (3) weighted sum of V. Same shape in / out.

**The 5 DCGAN rules in one breath**: (1) strided convs replace pooling, (2) BN everywhere except boundaries, (3) no FC hidden, (4) ReLU in G + LeakyReLU in D, (5) init `N(0, 0.02²)`.

**Why each loss function is used where**:
- BCE → binary problems, GAN.
- CrossEntropy → multi-class problems with one correct label.
- MSE → regression, autoencoder reconstruction.
- Dice → segmentation with class imbalance.
- The non-saturating GAN G loss (`-log D(G(z))`) is preferred because the saturating one has near-zero gradients early.
