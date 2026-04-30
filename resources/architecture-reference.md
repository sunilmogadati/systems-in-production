# Deep Learning Architecture Reference

*A one-page reference card covering foundations + GAN, RNN/LSTM, U-Net, ResNet, Transformer architectures. Designed for printable / quick-lookup use.*

*The companion narrative documents are in `playbooks/ai/`. The companion glossary is at [`playbooks/ai/architecture-glossary.md`](../playbooks/ai/architecture-glossary.md). The companion worked examples are at [`playbooks/ai/architecture-math.md`](../playbooks/ai/architecture-math.md).*

<style>
@page { margin: 0.2in; size: letter; }
body { font-size: 6.3pt; line-height: 1.1; column-count: 2; column-gap: 0.14in; column-rule: 1px solid #ccc; font-family: -apple-system, sans-serif; margin: 0; }
h1 { column-span: all; font-size: 12pt; margin: 0 0 4pt; border-bottom: 1px solid #000; }
h2 { column-span: all; font-size: 11pt; margin: 6pt 0 3pt; background: #ddd; padding: 2pt 5pt; }
h3 { font-size: 10pt; margin: 4pt 0 2pt; color: #224; font-weight: bold; }
h4 { font-size: 9.5pt; margin: 3pt 0 1pt; }
p { margin: 2pt 0; }
table { font-size: 6.5pt; width: 100%; border-collapse: collapse; margin: 1pt 0; }
th, td { border: 1px solid #999; padding: 1.5pt 3pt; text-align: left; vertical-align: top; }
th { background: #eee; }
pre { font-size: 6.3pt; line-height: 1.1; background: #f6f6f6; padding: 1pt 2pt; margin: 1pt 0; white-space: pre-wrap; border-left: 2px solid #888; }
code { font-size: 6.7pt; background: #f0f0f0; padding: 0 1pt; }
ul, ol { margin: 2pt 0 2pt 14pt; padding: 0; }
li { margin: 0; }
hr { border: none; border-top: 1px dashed #999; margin: 3pt 0; }
</style>

## FOUNDATIONS

### Activations
| Name | Formula | Range | Where used |
|---|---|---|---|
| Sigmoid σ | `1/(1+e⁻ᶻ)` | (0,1) | LSTM gates, binary out |
| Tanh | `(eᶻ-e⁻ᶻ)/(eᶻ+e⁻ᶻ)` | (-1,+1) | RNN h, LSTM C̃, GAN G out |
| ReLU | `max(0,z)` | [0,∞) | CNN, U-Net, FFN, DCGAN G |
| LeakyReLU(α) | `z if z>0 else αz` | ℝ | DCGAN D, SimpleGAN |
| GELU | smooth ReLU | ℝ | BERT, GPT-2+ FFN |
| Softmax | `eᶻⁱ/Σeᶻⱼ` | (0,1), Σ=1 | multi-class out, attn weights |

### Losses
| Name | Formula | Use |
|---|---|---|
| MSE | `(1/n)Σ(ŷ-y)²` | regression, AE |
| BCE | `-y·log p - (1-y)·log(1-p)` | binary, GAN |
| BCEWithLogits | sigmoid+BCE in one stable op | GAN |
| CrossEntropy | `-log p_true` (softmax inside) | RNN, U-Net per-pixel |
| Dice | `1 - 2|P∩T|/(|P|+|T|)` | imbalanced segmentation |

### Optimizers
```
SGD:        θ ← θ - η·g
Momentum:   v ← β·v + g;     θ ← θ - η·v
AdaGrad:    accum_g² += g²;  θ ← θ - η·g/√(accum+ε)
RMSProp:    EMA of g²;       θ ← θ - η·g/√(EMA+ε)
Adam:       m=β₁·m+(1-β₁)·g;  v=β₂·v+(1-β₂)·g²;  θ ← θ-η·m̂/(√v̂+ε)
AdamW:      Adam with decoupled weight decay (modern default)

Defaults:  Adam(lr=1e-3, β₁=0.9, β₂=0.999, ε=1e-8)
DCGAN:     Adam(lr=2e-4, β₁=0.5)         ← lower β₁ stabilizes
Vaswani:   Adam(β₁=0.9, β₂=0.98, ε=1e-9) + warmup + inv-sqrt LR
```

### Initialization
| Scheme | Variance | Use |
|---|---|---|
| Xavier/Glorot | `2/(fan_in+fan_out)` | tanh, sigmoid |
| He/Kaiming | `2/fan_in` | ReLU nets (CNN, U-Net) |
| DCGAN | `N(0,0.02²)` Conv; `N(1,0.02²)` BN γ | DCGAN |
| Forget-bias | init to 1 | LSTM (start remembering) |

### Param-Count Formulas

| Layer | Params (with bias) | Notes |
|---|---|---|
| Linear | `(in + 1) · out` | +1 = bias term |
| Conv2d | `(k_h · k_w · C_in + 1) · C_out` | bias optional |
| ConvTranspose2d | `(k_h · k_w · C_in + 1) · C_out` | same formula as Conv2d |
| BatchNorm | `2 · C` | γ scale + β shift per channel |
| LayerNorm | `2 · D` | γ + β per feature |
| Embedding | `V · D` | lookup table |
| Vanilla RNN | `HD + H² + CH + H + C` | when D=C: `≈ 2HC + H²` |
| LSTM | `4H(H+D) + 4H` | 4 gate matrices + 4 biases |
| MHA (Q,K,V,O) | `4 · d²` | four square projections |
| FFN (d_ff = 4d) | `8 · d²` | 2 layers, 4× expansion |
| Per Transformer block | `12 · d²` | MHA + FFN |

**Variable key:**
- `in / out` — input / output dimension of a Linear layer
- `k_h, k_w` — kernel height, width (often equal, e.g., 3×3)
- `C_in, C_out` — input / output channel count of a Conv layer
- `C` — channel count (BatchNorm)
- `D` — input dim of a token (RNN/LSTM); also used as feature dim for LayerNorm
- `H` — hidden state dim (RNN/LSTM)
- `V` — vocabulary size
- `d` (or `d_model`) — transformer model dim
- `d_ff` — FFN hidden dim, typically `4·d_model`
- `d_k = d_v = d_model / h` — per-head dim in multi-head attention
- `h` — number of attention heads

### Shape Formulas (DIFFERENT — common trap)
```
Conv2d:         N_out = floor((N_in+2p-f)/s) + 1     SHRINKS
ConvTranspose:  N_out = (N_in-1)·s - 2p + f          GROWS
Same padding:   p = (f-1)/2     (3×3→p=1, 5×5→p=2, 7×7→p=3)
Pool 2×2 s=2:   halves spatial, depth unchanged, 0 params

Receptive field (RF) — input region affecting one output unit:
   RF_out = RF_in + (f-1)·jump_in   ;   jump_out = jump_in · s
   Stack of two 3×3 convs (s=1): RF = 5×5  (same as one 5×5 — fewer params)
```

### Normalization & Regularization
| | Normalizes over | Best for |
|---|---|---|
| BatchNorm | batch + spatial; per channel | CNN |
| LayerNorm | feature dim; per token | Transformer |
| Dropout | random zero (rate p), off at inference | regularization |

### Conv2d vs ConvTranspose2d (when each is used)
Both appear repeatedly across architectures. They use **different shape formulas** — common gotcha.

**Conv2d** — slides a small filter across the input; each output cell is a weighted sum of the input cells the filter covers. Output is the same size or smaller. Used in:
- CNN classifiers (encoder side)
- DCGAN's discriminator (with stride 2 to downsample)
- U-Net's encoder path (in DoubleConv blocks)

**ConvTranspose2d** — places each input cell's contribution onto a larger output canvas, multiplied by the kernel. Output is larger. Used in:
- DCGAN's generator (each ConvT step doubles spatial size)
- U-Net's decoder path (one ConvT per up-stage before concat-with-skip)

Example shapes: Conv2d 6×6 → `(6-3)/1+1 = 4×4`. ConvTranspose2d 2×2 → `(2-1)·2+3 = 5×5`.

### Vanishing / Exploding Gradients
| Cause | Symptom | Fix |
|---|---|---|
| Deep, sigmoid/tanh | grad → 0 | ReLU, residuals, BN, init |
| RNN long seq | geom decay | LSTM (additive), clip |
| Saturating GAN G loss | early no signal | non-saturating G |
| Spike → NaN | exploding | clip: `g·θ/‖g‖` if `‖g‖>θ` |

### Toeplitz / matrix-form of conv (used in DCGAN G + U-Net decoder)
**Idea:** convolution is matrix multiplication. Flatten the input into a vector `X`. Build a special matrix `C` whose rows are "the kernel slid to position 0, 1, 2, ...". Then `Y = C · X`. Most of `C` is zeros (banded sparsity); each diagonal holds one kernel weight — that's the **Toeplitz** pattern.

**Worked example (1D conv, no pad, stride 1):**
- Kernel: `[a, b, c]`
- Input: `X = [x₀, x₁, x₂, x₃]` (length 4)
- Output length: `4 - 3 + 1 = 2`

`C` (shape 2×4):
```
 row 0: [a, b, c, 0]   →  y₀ = a·x₀ + b·x₁ + c·x₂
 row 1: [0, a, b, c]   →  y₁ = a·x₁ + b·x₂ + c·x₃
```

**Transposed conv** = multiply by `Cᵀ` (shape 4×2). Each input cell `y_i` is *spread* across multiple output cells via the kernel weights:
```
 Cᵀ rows:  [a,0], [b,a], [c,b], [0,c]
```
So `Cᵀ · y` produces a length-4 vector — bigger than the input. That's how transposed conv "grows" data.

**Key insight:** `Cᵀ · C ≠ I`. Transposed convolution does NOT undo a convolution — it just redistributes. DCGAN's G and U-Net's decoder use it as a **learned upsampler**: the network learns kernel weights that produce useful upsampling, not exact inversion.

### Worked example (Foundations) — Linear param count
*Used in: SimpleGAN, FFN inner layers, output classification heads.*
- Single layer: `Linear(128, 64)` with bias → `(128+1)·64 = 8,256` params.
- Full SimpleGAN G (`100→32→64→128→784`, 4 layers with bias):
  `101·32 + 33·64 + 65·128 + 129·784 = 3,232 + 2,112 + 8,320 + 101,136 = 114,800`.

### Worked example (Foundations) — Conv2d output shape
*Used in: CNN classifiers, U-Net encoder, DCGAN discriminator.*
- Input 32×32, kernel 5, p=0, s=1 → `floor((32-5)/1)+1 = 28`. Output 28×28.
- Input 28×28, kernel 3, p=1, s=2 → `floor((28+2-3)/2)+1 = floor(13.5)+1 = 14`.

### Concept terms (foundations)
- **Scalar/Vector/Matrix/Tensor** — 0/1/2/n-D number arrays.
- **Parameter** — weight or bias the optimizer learns.
- **Hyperparameter** — chosen before training (lr, batch size).
- **Logit** — raw output before sigmoid/softmax (numerically stable losses use logits).
- **Saturation** — activation flattens (sigmoid near 0/1) → grad ≈ 0.
- **Receptive field** — input region affecting one output unit.
- **One-hot** — vector with single 1, rest 0; large + no similarity.
- **Embedding** — learned dense vector for each discrete item.
- **Generative** — learns p(x); can sample. **Discriminative** — learns p(y|x).
- **Supervised** — labels given. **Unsupervised** — no labels.

## GAN

### Architecture (SimpleGAN, MNIST 28×28)
| | Shape flow | Activations |
|---|---|---|
| **G** | z(100) → 32 → 64 → 128 → 784 → reshape (1,28,28) | hidden: LeakyReLU(0.2)+Drop(0.3); **out: Tanh** |
| **D** | (784) → 128 → 64 → 32 → 1 logit | hidden: LeakyReLU(0.2)+Drop(0.3); **out: raw logit** |

G is **bottleneck-then-expand** (100→32 contracts, then doubles). Tanh range `[-1,+1]` matches real images rescaled `(x*2)-1`.

### DCGAN — 5 Rules + Hyperparams
1. Replace pooling: strided Conv2d (D), ConvTranspose2d (G).
2. BatchNorm in both — except G output, D input.
3. No FC hidden layers.
4. ReLU in G (out: Tanh), LeakyReLU(0.2) in D.
5. Init `~ N(0, 0.02²)`; BN scale `γ ~ N(1, 0.02²)`.

`Adam(lr=2e-4, β₁=0.5, β₂=0.999)` — β₁=0.5 stabilizes adversarial.

**DCGAN Generator skeleton** (z=100 → 64×64 image, 3 channels):
```python
nn.Sequential(
    nn.ConvTranspose2d(100,  512, 4, 1, 0, bias=False),  # 1→4
    nn.BatchNorm2d(512), nn.ReLU(True),
    nn.ConvTranspose2d(512,  256, 4, 2, 1, bias=False),  # 4→8
    nn.BatchNorm2d(256), nn.ReLU(True),
    nn.ConvTranspose2d(256,  128, 4, 2, 1, bias=False),  # 8→16
    nn.BatchNorm2d(128), nn.ReLU(True),
    nn.ConvTranspose2d(128,   64, 4, 2, 1, bias=False),  # 16→32
    nn.BatchNorm2d(64),  nn.ReLU(True),
    nn.ConvTranspose2d( 64,    3, 4, 2, 1, bias=False),  # 32→64
    nn.Tanh())                                            # ← out in [-1,+1]
```
Discriminator is the mirror: `Conv2d(stride=2)` blocks with LeakyReLU(0.2), no BN at input layer, no FC at end.

### SimpleGAN training loop (PyTorch — exam code)
```python
bce = nn.BCEWithLogitsLoss()
d_optim = optim.Adam(D.parameters(), lr=2e-4, betas=(0.5, 0.999))
g_optim = optim.Adam(G.parameters(), lr=2e-4, betas=(0.5, 0.999))

for real, _ in loader:                       # real shape: (B, 1, 28, 28)
    B = real.size(0)
    real = (real * 2) - 1                    # rescale [0,1] → [-1,+1]
    ones, zeros = torch.ones(B,1), torch.zeros(B,1)

    # ---- D-STEP (freeze G via no_grad) ----
    d_optim.zero_grad()
    loss_real = bce(D(real), ones)
    with torch.no_grad():
        z = torch.randn(B, 100)
        fakes = G(z)
    loss_fake = bce(D(fakes), zeros)
    loss_D = loss_real + loss_fake           # ← SUM, not mean
    loss_D.backward(); d_optim.step()

    # ---- G-STEP (D's grads computed but discarded) ----
    g_optim.zero_grad()
    z = torch.randn(B, 100)
    fakes = G(z)                             # grad tracked this time
    loss_G = bce(D(fakes), ones)             # target=1 = GOAL, not truth
    loss_G.backward(); g_optim.step()
```

### Math
```
Minimax:    min_G max_D V(D,G) = E_x[log D(x)] + E_z[log(1-D(G(z)))]
D loss:     L_D = -log D(x) - log(1-D(G(z)))
G loss:     L_G^nonsat = -log D(G(z))    (used in code, strong grads)
            L_G^sat    = log(1-D(G(z)))   (vanishes early — avoid)
Optimal D:  D*(x) = p_data(x) / (p_data(x) + p_G(x))
Nash:       p_G = p_data, D*(x) = 0.5 everywhere
```

### Param Counts
```
G: 100→32   101·32   =  3,232    D: 784→128  785·128 = 100,480
   32→64    33·64    =  2,112       128→64   129·64  =   8,256
   64→128   65·128   =  8,320       64→32     65·32  =   2,080
   128→784  129·784  =101,136       32→1      33·1   =      33
   TOTAL G           =114,800       TOTAL D          = 110,849
```
G ≈ D — mirror pyramids.

### Failure Modes
| Failure | Cause | Fix |
|---|---|---|
| Mode collapse | G outputs narrow set | label smoothing, BN |
| Vanishing grad | sat G loss → 0 | non-saturating G |
| Oscillation | moving-target loss | β₁=0.5, balanced lr |

### Worked example (GAN) — BCE loss for D and G
Given `D(real)=0.9, D(fake)=0.2`:
- `L_D = -log(0.9) - log(1-0.2) = 0.105 + 0.223 =` **0.328** (D doing well)
- `L_G^nonsat = -log(0.2) =` **1.609** (G failing, D detects fakes)
- If G improves so `D(fake)=0.75`: `L_G = -log(0.75) =` **0.288** (G winning)

### Worked example (DCGAN) — first ConvTranspose upsampling step
`(z=100, 1, 1) → ConvT(out=512, k=4, s=1, p=0)` → `(1-1)·1 - 0 + 4 = 4` → output `(512, 4, 4)`. The noise vector becomes a 4×4 spatial map with 512 channels, ready for further upsampling.

### Concept terms (GAN)
- **Latent space / noise vector z** — abstract input, hundred-dim, sampled from N(0,1) or U[-1,1].
- **Real / fake** — sample source determines auto-generated label.
- **Target=1 in G-step** — G's *goal*, not truth.
- **Critic** — alt name for D (Wasserstein GANs).
- **After training** — keep G, discard D.
- **Toeplitz tie** — DCGAN's transposed conv = matrix-mul by Cᵀ (not inverse of conv).

## RNN / LSTM + U-NET

### Vanilla RNN
```
h_t = tanh(W_xh·x_t + W_hh·h_{t-1} + b_h)
ŷ_t = softmax(W_hy·h_t + b_y)
loss = Σ_t cross_entropy(ŷ_t, y_t)
```
Params = `HD + H² + CH + H + C`. When D=C: **`≈ 2HC + H²`**.
W_hh is REUSED every time step → weight sharing across time.

### How an RNN processes a sequence
At every time step `t`, the RNN takes two inputs: the current token `x_t` (via `W_xh`) and the previous hidden state `h_{t-1}` (via `W_hh`). It combines them, applies tanh, and produces the new hidden state `h_t`. Optionally, `h_t` is also passed through `W_hy` to produce an output `ŷ_t` for that step.

**Key idea:** the same three weight matrices (`W_xh`, `W_hh`, `W_hy`) are reused at every time step — this is *weight sharing across time*. That's why the parameter count is independent of the sequence length T. The network can process sequences of any length with the same parameters.

To train it, "unroll" the network across all T time steps into one big computation graph and run regular backprop on it. This is **BPTT** — Backpropagation Through Time.

### LSTM (4 gates + cell state)
Concatenate prev hidden + current input: `z_t = [h_{t-1}; x_t]` (size H+D).
```
f_t  = σ(W_f · z_t + b_f)             forget gate    [0,1]
i_t  = σ(W_i · z_t + b_i)             input gate     [0,1]
o_t  = σ(W_o · z_t + b_o)             output gate    [0,1]
C̃_t  = tanh(W_c · z_t + b_c)          candidate      [-1,+1]
C_t  = f_t ⊙ C_{t-1} + i_t ⊙ C̃_t      cell (ADDITIVE)
h_t  = o_t ⊙ tanh(C_t)                hidden state
```
Params = **`4H(H+D) + 4H`**. Forget-bias init=1 → start by remembering.
**Why no vanishing:** `∂C_t/∂C_{t-1} = f_t` (additive); when `f_t ≈ 1`, gradient flows clean.

### How an LSTM cell works
Picture the cell state `C_t` as a memory channel that flows horizontally through time, **mostly untouched**. Every step decides three things based on the current input `x_t` and previous hidden state `h_{t-1}` (concatenated as `z_t`):

1. **Forget**: how much of `C_{t-1}` to keep? The forget gate `f_t` (sigmoid) outputs a value between 0 and 1 for every cell-state slot. `C_{t-1}` is multiplied by `f_t` element-wise.
2. **Write**: what new content to add? The input gate `i_t` (sigmoid) decides how much, and the candidate `C̃_t` (tanh) decides what. Their product is added to the cell state.
3. **Read**: what to output as the hidden state? The output gate `o_t` (sigmoid) gates a tanh-squashed view of the new cell state, producing `h_t`.

Because the cell-state update is **additive** (`C_t = f_t ⊙ C_{t-1} + i_t ⊙ C̃_t`), the gradient `∂C_t/∂C_{t-1}` equals `f_t`. When `f_t ≈ 1`, gradients flow through time without geometric decay. That's the whole reason LSTMs solve the vanishing gradient problem.

### BPTT idea
Forward T steps and accumulate loss. Backward in reversed time: `dW = Σ_t (dL_t/dW)`. Vanishing/exploding from repeated `dh_t/dh_{t-1}` multiplication. Fixes:
- **Clip:** if `‖g‖ > θ`, set `g ← g · θ/‖g‖` (typ θ=5).
- **Truncated BPTT:** backprop only k steps (saves memory; trades long-range learning).

### BPTT loop (vanilla RNN — full forward + backward)
```python
# Forward pass over a sequence of length T
hs = {-1: np.zeros(H)}
loss = 0
for t in range(T):
    hs[t]  = np.tanh(Wxh @ x[t] + Whh @ hs[t-1] + bh)
    ys[t]  = Why @ hs[t] + by
    ps[t]  = softmax(ys[t])
    loss  += -np.log(ps[t][targets[t]])      # cross-entropy

# Backward pass — BPTT through time (reversed!)
dWxh, dWhh, dWhy = (np.zeros_like(W) for W in [Wxh, Whh, Why])
dh_next = np.zeros(H)
for t in reversed(range(T)):                  # ← REVERSED time
    dy        = ps[t].copy(); dy[targets[t]] -= 1   # dL/dy
    dWhy     += dy @ hs[t].T
    dh        = Why.T @ dy + dh_next                # incoming grad
    dh_raw    = (1 - hs[t]**2) * dh                 # tanh' = 1 - tanh²
    dWxh     += dh_raw @ x[t].T              # ← +=  accumulate
    dWhh     += dh_raw @ hs[t-1].T           # ← +=  across time
    dh_next   = Whh.T @ dh_raw

for d in [dWxh, dWhh, dWhy, dbh, dby]:
    np.clip(d, -5, 5, out=d)                  # clip gradients in place
```
**Common gotchas:** `+=` (not `=`) on dW accumulators · `reversed(range(T))` not `range(T)` · `(1 - hs[t]**2)` is tanh derivative.

### Worked example (LSTM/RNN) — reverse-derivation drills
**LSTM dimensions from W_x shape:** `W_x` shape `(400, 356)` for the combined gate matrix.
- Row dim = `4H = 400` → **H = 100**
- Col dim = `H + D = 356` → **D = 256**

**Vanilla RNN vocab from total params:** `H = 100`, total = 1.6M, `D = C`.
- `2HC + H² = 1,600,000`  →  `200·C + 10,000 = 1,600,000`  →  **C = 7,950**

---

### How U-Net works
Two paths shaped like a U. The **encoder** (left side, going down) applies four stages of `DoubleConv → MaxPool`, each stage halving the spatial size and doubling the channel count: `64 → 128 → 256 → 512`. At the bottom is one more DoubleConv producing 1024 channels at the smallest spatial size.

The **decoder** (right side, going up) reverses the encoder. At each stage it does `ConvTranspose2d (stride 2)` to double the spatial size, then **concatenates** that upsampled feature map with the matching encoder map (the "skip connection"), then applies a DoubleConv that halves the channel count. This pattern repeats four times.

**Skip connections** are the key insight. The encoder's pooling discards spatial detail. The skip wires bring the encoder's high-resolution feature maps directly to the decoder, so the decoder doesn't have to reconstruct them from scratch. Concatenation (along the channel dim) preserves both signals as separate channels — addition would mix and lose them.

**Final layer:** a `Conv2d 1×1` that converts the 64-channel feature map into one channel per class — a per-pixel classifier with no spatial mixing.

### DoubleConv (U-Net's building block) — exam code
The repeating unit at every encoder + decoder level. Two 3×3 convs with same padding, BN, and ReLU. Spatial size is preserved; only channel count changes.
```python
class DoubleConv(nn.Module):
    def __init__(self, in_c, out_c):
        super().__init__()
        self.net = nn.Sequential(
            nn.Conv2d(in_c, out_c, 3, padding=1, bias=False),
            nn.BatchNorm2d(out_c), nn.ReLU(inplace=True),
            nn.Conv2d(out_c, out_c, 3, padding=1, bias=False),
            nn.BatchNorm2d(out_c), nn.ReLU(inplace=True))
    def forward(self, x): return self.net(x)
```
Original 2015 paper used valid padding (p=0) and no BN. Modern impls use same padding (p=1) and add BN — also drops the bias from Conv2d since BN's β subsumes it.

### U-Net decoder block (skip connection in code)
```python
x = self.up(x)                          # ConvTranspose2d: doubles spatial
x = torch.cat([x, skip], dim=1)         # ← THE skip — concat on channel dim
x = self.double_conv(x)
# Final layer outside the loop:
self.out = nn.Conv2d(64, n_classes, kernel_size=1)   # 1×1 conv (logits)
```

### Loss (and why Dice handles imbalance)
- Multi-class: `nn.CrossEntropyLoss()` per pixel.
- Binary imbalanced: `BCEWithLogitsLoss` + Dice (combo). **Dice robust to class imbalance.**

**Worked Dice example:** image with 5% foreground, 95% background.
- "Predict all background" → `|P∩T| = 0` → Dice = 0, `L_Dice = 1` (max loss). Strong signal.
- Pixel-wise CE on the same prediction → `L_CE ≈ -log(0.95) ≈ 0.05` per pixel (low loss). CE rewards the lazy "all background" answer; Dice punishes it.

### Worked example (U-Net) — shape & param count
- ConvT k=4, s=2, p=1 on 8×8 → `(8-1)·2 - 2 + 4 = 16` ✓ exact doubling.
- DoubleConv(128, 64) with bias = `(9·128+1)·64 + (9·64+1)·64 = 73,792 + 36,928` = **110,720** params.
- 4-level encoder needs input divisible by `2⁴ = 16`.

### Concept terms (RNN/LSTM/U-Net)
- **Hidden state h_t** — running summary of seen so far.
- **Cell state C_t** — LSTM's memory channel; flows mostly untouched.
- **Gate** — sigmoid output [0,1] that scales another signal.
- **BPTT** — unroll in time, then chain-rule backward.
- **Skip connection** — wire from early to late layer; concat in U-Net.
- **1×1 conv** — per-pixel linear classifier; no spatial mixing.
- **Fully convolutional** — no FC layers; works on any input size.
- **Elastic deformation** — U-Net's key augmentation (smooth random warps).

## RESNET

### ResNet (He et al., 2015) — CNN architecture
**Problem:** plain deep CNNs (50+ layers) suffer the **degradation problem** — accuracy gets *worse* with more layers (vanishing gradient + hard to optimize identity). ImageNet 2015: shallow → AlexNet (8) → VGG-19 → GoogLeNet (22) → **ResNet-152 (3.57% top-5 error)** broke past ~22 layers.

**Residual block:** output `y = F(x) + x`. The `+ x` is a **parameter-free identity shortcut** wiring input directly to output, bypassing the conv layers. The block learns the residual `F(x) = H(x) − x`, not the full mapping. Trivial "do nothing" → `F = 0` is the easy default. Adding more blocks can never hurt.

**Why it works:** `∂L/∂x = ∂L/∂y · (1 + ∂F/∂x)`. The `+1` keeps gradient flowing back via the shortcut — **no vanishing through the residual stream**. Identity shortcut adds **zero parameters**; only an addition.

**Shortcut types:**
- **Identity** — same shape: `y = F(x) + x`. Parameter-free.
- **Projection** — when stride > 1 or channels change: `y = F(x) + W_s·x` (`W_s` is a 1×1 conv).

**Basic block (ResNet-18/34):** `Conv 3×3 → BN → ReLU → Conv 3×3 → BN → (+ shortcut) → ReLU`. Architecture: `7×7` conv on input → stacks of residual blocks → periodically double filters, stride 2 to downsample.

**Bottleneck block (ResNet-50/101/152):** `1×1 → 3×3 → 1×1`. The 1×1 convs squeeze/restore channels so the 3×3 acts on fewer. Cheaper at same channel width.

**Signal-amplification intuition:** input signal "thins" through depth; the shortcut "makes it loud" at later layers — forward signal and backward gradient both get through. Why ResNet trains 152 layers where plain CNNs cap at ~20.

**Cross-architecture pattern — additive-shortcut trick is everywhere:**
| Architecture | Skip operation | Where |
|---|---|---|
| **ResNet** | `F(x) + x` (add) | within block, same shape |
| **LSTM** | `f_t ⊙ C_{t-1} + i_t ⊙ C̃_t` (add) | cell-state through time |
| **Transformer** | `x + Sublayer(x)` (add) | residual stream around MHA, FFN |
| **U-Net** | `torch.cat([up, skip], dim=1)` (**CONCAT**) | encoder→decoder, cross-block |

ResNet, LSTM, Transformer all use additive shortcuts to fix gradient flow. U-Net concatenates instead — different goal (fuse encoder + decoder feature maps to preserve spatial detail).

**Worked param count — basic block, C=64 in/out (no bias, with BN):** `2·(9·64²) + 2·(2·64) = 73,728 + 256 =` **73,984**.

## TRANSFORMERS

### How a Transformer processes a sequence
1. **Tokens → vectors.** Token IDs (integers) are passed through an embedding layer, a learnable `V × d_model` lookup table that returns one dense vector per token.
2. **Add positional encoding.** Self-attention treats input as a set (no order), so a position-dependent vector is *added* to each embedding to inject token order.
3. **Run through N encoder blocks.** Each block has two sub-blocks: multi-head self-attention (token mixing) and a feed-forward network (per-token transformation). Each sub-block is wrapped with a residual connection (add the input back to the output) and a LayerNorm.
4. **Final LayerNorm.** Produces output of shape `(B, n, d_model)` — same shape as the input embeddings.

For BERT-style classification: pool `[CLS]` token's output and feed to a classifier. For GPT-style generation: project the last token's output to vocabulary size and softmax to predict the next token.

### Variants
| | Attention | Use |
|---|---|---|
| Encoder-only (BERT) | bidirectional | classify, NER, QA |
| Decoder-only (GPT, Llama) | causal mask | generation |
| Encoder-decoder (T5, BART) | enc bidir + dec causal + cross | translation |

### How self-attention works (6 steps)
1. **Project** input `X` (shape `n × d_model`) into three matrices: `Q = X·W_Q`, `K = X·W_K`, `V = X·W_V`. Each token plays three roles — query (what it's asking), key (what it offers), value (what it delivers).
2. **Score every pair**: compute `Q · Kᵀ`, an `n × n` matrix where entry `(i, j)` is the dot product of token `i`'s query with token `j`'s key — a similarity score.
3. **Scale** by `√d_k`: prevents dot products from growing with `d_k`, which would cause softmax to saturate and gradients to vanish.
4. **Softmax** row-wise: each row becomes a probability distribution over all tokens. Token `i`'s row tells how much it attends to each other token.
5. **Weighted sum**: multiply weights by `V`. Each output token = blend of all `V` vectors, weighted by attention.
6. **Output** has the same shape as input (`n × d_v`). Each token now has a *context-aware* representation — its own value mixed with relevant neighbors.

**Formula:** `Attention(Q,K,V) = softmax(Q·Kᵀ/√d_k)·V`

### Why √d_k scaling — variance argument
Suppose `q, k` have entries with mean 0, variance 1, dim `d_k`. Then `q·k = Σ q_i·k_i` has variance `d_k`. So as `d_k` grows, dot products grow ~`√d_k`. Without scaling, softmax inputs grow → softmax saturates (one entry near 1, rest near 0) → gradient through softmax ≈ 0. Dividing by `√d_k` keeps the variance ≈ 1 regardless of d_k.

### How multi-head attention works
Instead of one big attention computation over `d_model`, split the dimensions into `h` parallel "heads," each of size `d_k = d_model / h`. Each head runs its own attention computation independently. The outputs from all heads are concatenated back to size `d_model`, then passed through one final linear projection `W_O`.

Why: different heads can specialize. One might attend to syntactic neighbors, another to long-range references, another to semantic clusters. Same total parameter budget for Q/K/V (still `3·d²`) plus the output projection (`d²`) gives `4·d²` total.

In code, the split is implemented as a *reshape + transpose* of one big projection matrix — not separate matrices per head.

### How an encoder block is wired
Each block has two sub-blocks, each wrapped in `residual + LayerNorm`:

**Sub-block 1 — Self-Attention.** Input `x` is normalized (LayerNorm), passed through multi-head attention, then *added back* to the original `x` (residual connection).

**Sub-block 2 — FFN.** Result of sub-block 1 is normalized again, passed through a 2-layer MLP (Linear → GELU → Linear), then *added back* to the input of sub-block 2.

The "residual stream" — the un-modified input flowing alongside each sub-block — is what allows transformers to be stacked very deep. Gradients flow cleanly along the residual path.

**Pre-LN vs Post-LN:**
- **Pre-LN** (modern, GPT-2+, Llama): `x + Sublayer(LN(x))`. Stable for deep stacks; usually no warmup needed.
- **Post-LN** (Vaswani 2017): `LN(x + Sublayer(x))`. Needs LR warmup to train stably.

### Decoder block
Generative models replace encoder self-attention with **masked self-attention** (causal mask blocks future positions). Two flavors:

- **Encoder-decoder decoder** (T5, BART, original Transformer) — **3 sub-blocks**:
  1. Masked self-attention (decoder attends to its own past)
  2. Cross-attention (`Q` from decoder, `K, V` from encoder output)
  3. FFN
- **Decoder-only** (GPT, Llama) — **2 sub-blocks**: masked self-attention + FFN. No cross-attention because there's no encoder.

Each sub-block is still wrapped in residual + LayerNorm.

### FFN (per-position)
```
FFN(x) = max(0, x·W_1+b_1)·W_2 + b_2     (or GELU instead of ReLU)
hidden_dim = 4·d_model       Independent at each position.
```

### Param Counts
| Component | Params (≈, no bias) |
|---|---|
| MHA (Q,K,V,O) | **4·d²** |
| FFN (d_ff=4d) | **8·d²** |
| **Per encoder block** | **12·d²** |
| Embedding | V·d |
BERT-base: d=768, h=12, N=12, V≈30k → **~110M**.

### Complexity & PE
```
Time:    O(n²·d)         Memory:  O(n²)        ← quadratic in seq len

PE(pos, 2i)   = sin(pos / 10000^(2i/d))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d))
PE(pos+k) is a fixed linear function of PE(pos) → relative pos recoverable.
```

### Training (Vaswani 2017)
```
Adam(β₁=0.9, β₂=0.98, ε=1e-9)
lr = d_model⁻⁰·⁵ · min(step⁻⁰·⁵, step·warmup⁻¹·⁵)
warmup_steps = 4000;  Dropout=0.1;  Label smoothing=0.1
Modern default: AdamW
```

### Code (must recognize)
```python
def scaled_dot_product_attention(Q, K, V, mask=None):
    d_k = Q.size(-1)
    scores = (Q @ K.transpose(-2,-1)) / math.sqrt(d_k)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))  # 0=block
    weights = torch.softmax(scores, dim=-1)
    return weights @ V, weights

# Causal mask (decoder)
mask = torch.tril(torch.ones(n, n))   # lower-tri = allowed; 0 = block
# WARNING: nn.MultiheadAttention's attn_mask uses True=block (opposite!)
```

### Worked example (Transformer) — 3-token self-attention
**Setup:** `X = [[1,0], [0,1], [1,1]]` (3 tokens, d_model = 2). `W_Q = W_K = W_V = I` (identity, for clarity), so `Q = K = V = X`. `d_k = 2`.

**Step 1 — scores `Q·Kᵀ`** (3×3):
`[[1,0,1], [0,1,1], [1,1,2]]`  (entry `[i][j]` = dot product of token i's query with token j's key)

**Step 2 — scale by `√d_k = √2 ≈ 1.414`:**
`[[0.71, 0, 0.71], [0, 0.71, 0.71], [0.71, 0.71, 1.41]]`

**Step 3 — softmax row-wise** (each row → probability dist):
`[[0.40, 0.20, 0.40], [0.20, 0.40, 0.40], [0.25, 0.25, 0.50]]`

**Step 4 — weighted sum `weights · V`** (output shape = input shape):
`[[0.80, 0.60], [0.60, 0.80], [0.75, 0.75]]`

### Worked example (Transformer) — reverse-derive d_model from MHA params
MHA total = 1,048,576 → `4·d² = 1,048,576` → `d² = 262,144` → **d = 512**.
BERT-base block: `12·768² ≈ 7.08M` per block × 12 blocks = **85M** + embedding (`30k·768 ≈ 23M`) ≈ **108M total**.

### Concept terms (Transformers)
- **Token / vocabulary** — discrete units, fixed set.
- **Embedding** (3 senses): token vector / lookup matrix / contextual output.
- **Q / K / V** — learned projections of input; "ask / offer / deliver."
- **Attention** — each pos looks at every other pos with weighted mix.
- **Self-attention** — Q,K,V from same sequence.
- **Cross-attention** — Q from decoder, K,V from encoder.
- **Causal mask** — triangular, blocks future positions.
- **Permutation-invariant** — why PE is needed.
- **Residual + LayerNorm** — wrap each sub-block.
- **Autoregressive** — generate one token at a time.
- **Why beat RNN** — parallel training, direct long-range, scaling.

---

## QUICK NUMERICAL EXAMPLES

**LSTM forward step (one cell):** `f=0.7, i=0.8, o=0.6, C̃=0.5, C_{t-1}=0.4`
- `C_t = f·C_{t-1} + i·C̃ = 0.7·0.4 + 0.8·0.5 = 0.28 + 0.40 = 0.68`
- `h_t = o·tanh(C_t) = 0.6·tanh(0.68) ≈ 0.6·0.591 ≈ 0.355`
- **Trap:** cell update is `f·C + i·C̃` (additive). Order matters: forget old, then write new.

**Cross-entropy:** predicted `p = [0.1, 0.7, 0.2]`, true class = 1 (middle).
- `L = -log(p_true) = -log(0.7) ≈ 0.357`. Wrong class would be `-log(0.1) ≈ 2.303`.

**Softmax by hand:** input `z = [1, 2, 3]`.
- `e^1=2.72, e^2=7.39, e^3=20.09`. Sum = 30.20.
- Output = `[2.72/30.20, 7.39/30.20, 20.09/30.20] ≈ [0.090, 0.245, 0.665]` (sums to 1).

**Single Conv2d output value:** input patch `[[1,0,1],[0,1,0],[1,0,1]]`, filter `[[1,0,1],[0,1,0],[1,0,1]]`.
- Output cell = `Σ (input ⊙ filter) = 1+0+1+0+1+0+1+0+1 = 5`. Slide filter for next position.

**Embedding params:** vocab `V=30,000`, embedding dim `d=512` → `V·d = 30,000·512 = 15.36M` params.

---

## QUICK-REFERENCE FORMULA BANK
*Each formula labeled with what it computes and where it applies.*

```
─── PARAMETER COUNTS (with bias) ───
Linear / FC layer:          (in + 1) · out
Conv2d / ConvTranspose2d:   (k·k·C_in + 1) · C_out
BatchNorm:                  2·C            (γ scale + β shift per channel)
LayerNorm:                  2·D            (γ + β per feature)
Embedding (lookup table):   V · D          (vocab × embed dim)
Vanilla RNN total:          2HC + H²       (when input dim D = vocab C)
LSTM (4 gates) total:       4H(H+D) + 4H
Multi-Head Attention:       4·d²           (Q, K, V, O — each d×d)
FFN (with d_ff = 4·d):      8·d²
Transformer encoder block:  12·d²          (MHA + FFN, no bias)

─── OUTPUT SHAPES (Conv vs ConvTranspose — DIFFERENT!) ───
Conv2d output side:         floor((N_in + 2p - f) / s) + 1     (shrinks)
ConvTranspose2d side:       (N_in − 1)·s − 2p + f              (grows)
Same padding (no shrink):   p = (f − 1) / 2                    (3×3→1, 5×5→2)
Pool 2×2, stride 2:         halves spatial; 0 params
Receptive field of stack:   RF_out = RF_in + (f−1)·jump_in

─── ACTIVATIONS (function : range : where used) ───
Sigmoid σ(z) = 1/(1+e⁻ᶻ)              ∈(0,1)      LSTM gates, binary out
Tanh = (eᶻ−e⁻ᶻ)/(eᶻ+e⁻ᶻ)             ∈(-1,+1)   RNN h_t, LSTM C̃, GAN G out
ReLU = max(0,z)                       ∈[0,∞)     CNN, U-Net, FFN, DCGAN G
LeakyReLU(α): z if z>0 else α·z       ∈ℝ         DCGAN D, SimpleGAN
Softmax: e^z_i / Σ e^z_j              prob dist  multi-class out, attn weights

─── LOSSES (formula : when to use) ───
BCE:        -y·log p - (1-y)·log(1-p)            binary classify, GAN
Cross-Ent:  -log(p_true)                         multi-class (RNN out, U-Net pixel)
Dice:       1 - 2|P∩T| / (|P|+|T|)               imbalanced segmentation (U-Net)
GAN D loss: -log D(real) - log(1-D(fake))        SUM of real + fake (not mean)
GAN G loss: -log D(G(z))    (non-saturating)     used in code; strong gradients

─── ATTENTION & TRANSFORMERS ───
Self-attention output:  softmax(Q·Kᵀ / √d_k) · V      (n × d_v)
Why √d_k scaling:       q·k variance grows with d_k; rescale to keep ≈ 1
Self-attn complexity:   O(n²·d)  time, O(n²) memory  (seq len n)
Positional encoding:    PE(pos,2i) = sin(pos/10000^(2i/d)); cos for 2i+1

─── RECURRENT UPDATES ───
Vanilla RNN:    h_t = tanh(W_xh·x_t + W_hh·h_{t-1} + b_h)
LSTM cell:      C_t = f_t ⊙ C_{t-1} + i_t ⊙ C̃_t      (additive — fixes vanishing)
LSTM hidden:    h_t = o_t ⊙ tanh(C_t)

─── TRAINING UTILITIES ───
Gradient clip:  if ‖g‖ > θ:  g ← g · θ/‖g‖           (typ θ = 5)
Adam default:   β₁=0.9, β₂=0.999, ε=1e-8
Adam DCGAN:     lr=2e-4, β₁=0.5         (lower β₁ stabilizes adversarial)
Adam Vaswani:   β₁=0.9, β₂=0.98, ε=1e-9 + warmup + inv-sqrt LR
```

## DECISION TABLE
| Question contains... | Type | Reference section |
|---|---|---|
| "How many parameters?" | Param counting | Page 1 + topic |
| "Output shape?" | Shape arithmetic | Page 1 |
| "Find k, p, H, D" | Reverse derivation | Page 1 |
| Concrete W matrices | Forward pass | Topic page |
| Probabilities given | Loss compute | Page 1 + topic |
| "Why" / "Explain" | Concept | Topic page |
| Code snippet | Code trace | Topic code blocks |
