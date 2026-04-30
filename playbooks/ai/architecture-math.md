# Architecture Math — Worked Examples

Numerical examples with step-by-step solutions, organized by problem type. Each one includes the formula, the substitution, the arithmetic, and any common gotchas. Covers parameter counting, shape arithmetic, forward-pass numerics, and loss computation across GAN, RNN/LSTM, U-Net, ResNet, and Transformer architectures.

Companion to the conceptual reference in [Math for AI](math-for-ai.md) (derivatives, chain rule, gradient descent) and the [Architecture Glossary](architecture-glossary.md).

---

## Type 1: Parameter Counting

### Formulas

```
Linear (with bias):       (in + 1) · out
Conv2d (with bias):       (k_h · k_w · C_in + 1) · C_out
ConvTranspose2d:          same formula as Conv2d
BatchNorm2d:              2 · C       (γ scale + β shift per channel)
LayerNorm:                2 · D       (γ + β per feature)
Embedding:                V · D       (vocab × embed dim)
Vanilla RNN:              HD + H² + CH + H + C        (D=C: ≈ 2HC + H²)
LSTM:                     4H(H+D) + 4H
MHA (Q,K,V,O):            4 · d_model²
FFN (d_ff = 4d):          8 · d_model²
Per transformer block:    12 · d_model²
```

### Example 1.1: SimpleGAN Generator

Architecture: `100 → 32 → 64 → 128 → 784` (4 Linear layers, all with bias).

```
Layer 1 (100 → 32):    (100 + 1) · 32  =  101 · 32   =  3,232
Layer 2 (32 → 64):     (32 + 1) · 64   =   33 · 64   =  2,112
Layer 3 (64 → 128):    (64 + 1) · 128  =   65 · 128  =  8,320
Layer 4 (128 → 784):   (128 + 1) · 784 =  129 · 784  = 101,136
TOTAL                                              =  114,800
```

**Trap**: Forgetting `+1` for the bias. If problem says "no bias," drop it.

### Example 1.2: SimpleGAN Discriminator

Architecture: `784 → 128 → 64 → 32 → 1`.

```
Layer 1 (784 → 128):   (784 + 1) · 128 =  785 · 128  = 100,480
Layer 2 (128 → 64):    (128 + 1) · 64  =  129 · 64   =   8,256
Layer 3 (64 → 32):     (64 + 1) · 32   =   65 · 32   =   2,080
Layer 4 (32 → 1):      (32 + 1) · 1    =   33 · 1    =      33
TOTAL                                              =  110,849
```

**Memorize**: G = 114,800; D = 110,849. They're roughly balanced (mirror pyramids).

### Example 1.3: Conv2d Layer

Conv2d with input (8 channels, 28×28), 16 filters, kernel 3×3, with bias.

```
Params = (3 · 3 · 8 + 1) · 16
       = (72 + 1) · 16
       = 73 · 16
       = 1,168
```

### Example 1.4: Conv2d (no bias, with BatchNorm)

Same Conv2d as above but `bias=False`, followed by BatchNorm2d(16).

```
Conv params = (3 · 3 · 8) · 16 = 72 · 16 = 1,152
BN params   = 2 · 16            = 32
TOTAL       = 1,184
```

**Note**: BN absorbs the bias function via β, so we save one bias per filter.

### Example 1.5: U-Net DoubleConv(128, 64) with bias

`Conv 3×3 (128 in, 64 out) → BN → ReLU → Conv 3×3 (64 in, 64 out) → BN → ReLU`

```
Conv1 = (3 · 3 · 128 + 1) · 64  =  (1152 + 1) · 64  =  1153 · 64  =  73,792
BN1   = 2 · 64                                                       =     128
Conv2 = (3 · 3 · 64 + 1) · 64   =  (576 + 1) · 64   =  577 · 64   =  36,928
BN2   = 2 · 64                                                       =     128
TOTAL                                                                 = 110,976
```

(With `bias=False` on the convs: subtract 64 + 64 = 128, total = 110,848.)

### Example 1.6: Vanilla RNN

`H = 100, D = 256, C = 256` (input dim = vocab size).

Full formula: `HD + H² + CH + H + C`

```
W_xh:  H · D = 100 · 256 = 25,600
W_hh:  H · H = 100 · 100 = 10,000
W_hy:  C · H = 256 · 100 = 25,600
b_h:   H     = 100
b_y:   C     = 256
TOTAL              =  61,556
```

Quick form (D=C, no biases): `2HC + H² = 2·100·256 + 100² = 51,200 + 10,000 = 61,200`. Adding biases (`H + C = 356`): 61,556 ✓

### Example 1.7: LSTM Layer

`H = 100, D = 256` (single LSTM layer, no output projection).

Formula: `4H(H+D) + 4H`

```
4 · 100 · (100 + 256) + 4 · 100
= 400 · 356 + 400
= 142,400 + 400
= 142,800
```

### Example 1.8: Transformer Encoder Block

`d_model = 512, h = 8, d_ff = 2048` (= 4 · d_model).

```
MHA:  4 · d_model² = 4 · 512² = 4 · 262,144 = 1,048,576
FFN:  8 · d_model² = 8 · 262,144            = 2,097,152
LN×2: 2 · 2 · d_model = 2,048               (negligible)
TOTAL ≈                                       3,148,000
```

Quick formula: per-block ≈ `12 · d_model² = 12 · 262,144 = 3,145,728`. ✓

For BERT-base (N=12 blocks): `12 · 3.15M = 37.7M` (encoder layers only). Plus embedding `30k · 768 = 23M`. Total ≈ 110M.

### Example 1.9: ResNet basic block (no bias, with BN), C in/out = 64

`Conv 3×3 → BN → ReLU → Conv 3×3 → BN → (+ shortcut) → ReLU`

```
Conv1 (3×3, 64→64, no bias):    9 · 64 · 64 = 36,864
BN1:                            2 · 64       =     128
Conv2 (3×3, 64→64, no bias):    9 · 64 · 64 = 36,864
BN2:                            2 · 64       =     128
Identity shortcut:                            =       0  ← parameter-free
TOTAL                                         =  73,984
```

**Trap:** the identity shortcut adds **zero parameters**. Only addition. (Projection shortcut would add `64·64 = 4,096` for a 1×1 conv if shapes mismatch.)

### Example 1.10: ResNet basic block with projection shortcut

Same block but channel count changes 64 → 128 with stride 2 (downsampling):

```
Conv1 (3×3, 64→128, s=2, no bias):   9 · 64 · 128  = 73,728
BN1:                                  2 · 128       =    256
Conv2 (3×3, 128→128, no bias):       9 · 128 · 128 =147,456
BN2:                                  2 · 128       =    256
Projection shortcut (1×1, 64→128, s=2):  64 · 128  =  8,192   ← extra params!
BN_shortcut:                          2 · 128       =    256
TOTAL                                              = 230,144
```

**Notice:** the projection shortcut adds 8,448 params (1×1 conv + BN). The shortcut is no longer parameter-free when shapes change.

### Example 1.11: ResNet bottleneck block, C=256

`1×1 (256→64) → BN → ReLU → 3×3 (64→64) → BN → ReLU → 1×1 (64→256) → BN → (+ shortcut) → ReLU`

```
Conv 1×1 (256→64):   1·1·256·64  = 16,384
BN:                  2 · 64        =    128
Conv 3×3 (64→64):    9 · 64 · 64  = 36,864
BN:                  2 · 64        =    128
Conv 1×1 (64→256):   1·1·64·256  = 16,384
BN:                  2 · 256       =    512
Identity shortcut:                  =     0
TOTAL                              = 70,400
```

**Compare:** at the *same* C=256 channel count, a *basic block* would be `2 · (9·256²) + 2·(2·256) = 1,180,672 + 1,024 = 1,181,696` params. **Bottleneck saves ~17×** by squeezing channels through the 1×1 convs. That's why ResNet-50/101/152 use bottleneck blocks.

---

## Type 2: Shape Arithmetic

### Formulas

```
Conv2d:         N_out = floor((N_in + 2p - f) / s) + 1
ConvTranspose:  N_out = (N_in - 1) · s - 2p + f
Pool 2×2 s=2:   halves spatial
Same padding:   p = (f - 1) / 2  (3×3 → p=1, 5×5 → p=2, 7×7 → p=3)
```

### Example 2.1: Conv2d, kernel 5, no padding

Input 32×32, k=5, s=1, p=0.

```
N_out = floor((32 + 0 - 5) / 1) + 1
      = floor(27) + 1
      = 28
```
Output: 28×28.

### Example 2.2: Conv2d, kernel 3, same padding

Input 28×28, k=3, s=1, p=1.

```
N_out = floor((28 + 2 - 3) / 1) + 1
      = floor(27) + 1
      = 28
```
Output: 28×28 (spatial unchanged — that's why p=1 with k=3 is called "same padding").

### Example 2.3: Conv2d with stride 2 (downsampling)

Input 28×28, k=3, s=2, p=1.

```
N_out = floor((28 + 2 - 3) / 2) + 1
      = floor(27/2) + 1
      = 13 + 1
      = 14
```
Output: 14×14. (Stride 2 roughly halves spatial.)

### Example 2.4: ConvTranspose2d (upsampling)

Input 8×8, k=4, s=2, p=1.

```
N_out = (8 - 1) · 2 - 2·1 + 4
      = 14 - 2 + 4
      = 16
```
Output: 16×16. (Stride 2 doubles spatial.)

### Example 2.5: ConvTranspose2d with no padding

Input 16×16, k=2, s=2, p=0.

```
N_out = (16 - 1) · 2 - 0 + 2
      = 30 + 2
      = 32
```
Output: 32×32. **The k=2, s=2, p=0 ConvT exactly doubles** any input spatial dim.

### Example 2.6: Full pipeline trace (CNN encoder)

Input `(1, 28, 28)`. Apply Conv2d(8, k=3, p=0, s=1) → MaxPool(2) → Conv2d(16, k=3, p=0, s=1) → MaxPool(2).

```
Input:           (1,  28, 28)
Conv1(8 filters, 3×3, p=0, s=1):
  N = (28+0-3)/1 + 1 = 26
  → (8, 26, 26)
Pool1 (2×2, s=2):
  → (8, 13, 13)
Conv2(16 filters, 3×3, p=0, s=1):
  N = (13+0-3)/1 + 1 = 11
  → (16, 11, 11)
Pool2 (2×2, s=2):
  → (16, 5, 5)        ← floor(11/2) = 5
```

### Example 2.7: U-Net encoder trace

Input `(1, 256, 256)`, 4 levels of `DoubleConv (3×3, p=1) → MaxPool(2)`.

```
Input:            (1,    256, 256)
Level 1 DC → 64:  (64,   256, 256)   (p=1 keeps spatial)
Pool:             (64,   128, 128)
Level 2 DC → 128: (128,  128, 128)
Pool:             (128,   64,  64)
Level 3 DC → 256: (256,   64,  64)
Pool:             (256,   32,  32)
Level 4 DC → 512: (512,   32,  32)
Pool:             (512,   16,  16)
Bottom DC → 1024: (1024,  16,  16)
```

**Constraint**: input must be divisible by `2^4 = 16` for 4 levels.

### Example 2.8: ResNet basic block — shape preservation

A basic ResNet block with same-padding convs (k=3, p=1, s=1) preserves spatial shape. Input `(64, 32, 32)`:

```
Conv1 3×3, p=1, s=1:  N_out = floor((32+2-3)/1)+1 = 32   (unchanged)
Conv2 3×3, p=1, s=1:  N_out = 32                          (unchanged)
+ identity shortcut: input (64, 32, 32) added directly    ← shape must match
Output:               (64, 32, 32)
```

**Why this matters:** for the identity shortcut to work, input and output must have the same shape. Same-padding convs (`p = (f-1)/2`) ensure this for stride-1 blocks.

### Example 2.9: ResNet downsampling block (stride 2, channel double)

Input `(64, 32, 32)`. First conv has stride 2 and doubles channels. Second conv keeps shape.

```
Conv1 3×3, s=2, p=1, 64→128:  N_out = floor((32+2-3)/2)+1 = 16
Conv2 3×3, s=1, p=1, 128→128: N_out = 16  (unchanged)
After convs:                  (128, 16, 16)
Input shape:                  (64, 32, 32)  ← MISMATCH!
Need projection shortcut:     1×1 conv, 64→128, stride 2 → output (128, 16, 16) ✓
Output:                       (128, 16, 16)
```

**Trap:** the shortcut must apply the same downsampling so its output matches `F(x)`. Use `Conv2d(64, 128, kernel=1, stride=2)` for the projection.

---

## Type 3: Reverse Derivation

### Example 3.1: LSTM dimensions from W_x shape

LSTM combined gate matrix `W_x` has shape `(400, 356)`. Find H and D.

```
LSTM stacks 4 gates into one matrix of shape (4H, H+D).
Row dim:  4H = 400  →  H = 100
Col dim:  H + D = 356  →  D = 356 - 100 = 256
```

**Answer**: H = 100, D = 256.

### Example 3.2: RNN vocab from total params

Vanilla RNN with `H = 100` has 1.6M parameters total. Find vocab size C (assume D = C).

```
Params (no biases) = 2HC + H² = 1,600,000
2 · 100 · C + 100² = 1,600,000
200·C + 10,000 = 1,600,000
200·C = 1,590,000
C = 7,950
```

(With biases: `2HC + H² + H + C = 1,600,000` → `200C + 10100 + C = 1,600,000` → `201C = 1,589,900` → `C ≈ 7,910`. Close to 7,950 either way.)

### Example 3.3: Find Conv padding from output shape

Conv2d outputs 14×14 from 28×28 input. Stride 2, kernel 4. Find p.

```
14 = floor((28 + 2p - 4) / 2) + 1
13 = floor((24 + 2p) / 2)
13 = (24 + 2p) / 2          (assuming 24+2p is even)
26 = 24 + 2p
p = 1
```

### Example 3.4: Find ConvT kernel from output shape

ConvTranspose2d takes 1×1 input to 4×4 output, stride 1, padding 0. Find kernel size f.

```
N_out = (N_in - 1) · s - 2p + f
4 = (1 - 1) · 1 - 0 + f
4 = 0 + f
f = 4
```

### Example 3.5: Transformer d_model from MHA params

Multi-head attention has 1,048,576 parameters. Find d_model.

```
MHA params = 4 · d_model² = 1,048,576
d_model² = 262,144
d_model = 512
```

### Example 3.6: BERT-base block param check

A transformer encoder block has 7M params. Estimate d_model.

```
12 · d² = 7,000,000
d² = 583,333
d ≈ 764
```

(Actual BERT-base d_model = 768, giving `12 · 768² = 7.08M` per block ✓)

---

## Type 4: Forward-Pass Arithmetic

### Example 4.1: Vanilla RNN one step

Given:
- `W_xh = [[1, 0], [0, 1]]`  (shape 2×2)
- `W_hh = [[1, 1], [0, 1]]`  (shape 2×2)
- `b_h = [0, 0]`
- `x_t = [1, 0]`
- `h_{t-1} = [0, 1]`

Compute `h_t = tanh(W_xh · x_t + W_hh · h_{t-1} + b_h)`.

```
W_xh · x_t:
  [1·1 + 0·0,  0·1 + 1·0] = [1, 0]

W_hh · h_{t-1}:
  [1·0 + 1·1,  0·0 + 1·1] = [1, 1]

Sum + bias:  [1+1+0, 0+1+0] = [2, 1]

h_t = tanh([2, 1]) = [tanh(2), tanh(1)]
                   ≈ [0.964, 0.762]
```

### Example 4.2: LSTM one step (simplified)

Given (using sigmoid `σ` and tanh values):
- `f_t = σ(W_f·z_t) = 0.7`  (forget gate output for one cell)
- `i_t = σ(W_i·z_t) = 0.8`
- `o_t = σ(W_o·z_t) = 0.6`
- `C̃_t = tanh(W_c·z_t) = 0.5`
- `C_{t-1} = 0.4`

Compute `C_t` and `h_t`.

```
C_t = f_t · C_{t-1} + i_t · C̃_t
    = 0.7 · 0.4 + 0.8 · 0.5
    = 0.28 + 0.40
    = 0.68

tanh(C_t) = tanh(0.68) ≈ 0.591

h_t = o_t · tanh(C_t)
    = 0.6 · 0.591
    ≈ 0.355
```

**Trap**: cell update is **additive** (`+`), not multiplicative. And the order is `f·C_{t-1} + i·C̃_t`, not the reverse.

### Example 4.3: Self-attention (one head)

Given `Q = K = V = [[1, 0], [0, 1], [1, 1]]` (3 tokens, d_k = 2).

(Full walkthrough — see "self-attention example" doc; final result):
```
scores = Q · Kᵀ = [[1, 0, 1],
                   [0, 1, 1],
                   [1, 1, 2]]

scaled = scores / √2 ≈ [[0.707, 0,     0.707],
                        [0,     0.707, 0.707],
                        [0.707, 0.707, 1.414]]

weights = softmax(scaled, dim=-1) ≈ [[0.401, 0.198, 0.401],
                                     [0.198, 0.401, 0.401],
                                     [0.248, 0.248, 0.504]]

output = weights · V ≈ [[0.802, 0.599],
                        [0.599, 0.802],
                        [0.752, 0.752]]
```

### Example 4.4: Softmax computation by hand

Given `z = [1, 2, 3]`. Compute softmax.

```
e^1 = 2.718
e^2 = 7.389
e^3 = 20.086

sum = 2.718 + 7.389 + 20.086 = 30.193

softmax = [2.718/30.193,  7.389/30.193,  20.086/30.193]
        ≈ [0.090,         0.245,          0.665]
```

Sum check: `0.090 + 0.245 + 0.665 = 1.000` ✓

### Example 4.5: ResNet forward pass through one block

Input vector (1D for clarity) `x = [1, 2]`. Block computes `F(x)` via two linear layers (no activations for clarity); call the result `[0.5, -0.3]`.

```
F(x) = [0.5, -0.3]
y    = F(x) + x
     = [0.5, -0.3] + [1, 2]
     = [1.5, 1.7]
```

If `F(x) = [0, 0]` (block "does nothing"):
```
y = [0, 0] + [1, 2] = [1, 2] = x       ← exactly the input
```
That's the **identity behavior** — adding more blocks can never hurt because each block has an easy "do nothing" default.

### Example 4.6: ResNet gradient flow through residual block

Suppose loss gradient at output is `∂L/∂y = [0.1, -0.2]`. The block's internal Jacobian `∂F/∂x = 0.05·I` (small, near zero — the block is "saturating").

Without the residual shortcut:
```
∂L/∂x_plain = ∂L/∂y · ∂F/∂x = [0.1, -0.2] · 0.05 = [0.005, -0.010]   ← shrunk 20×
```

With the residual shortcut (`y = F(x) + x`):
```
∂L/∂x_res = ∂L/∂y · (∂F/∂x + I)
          = [0.1, -0.2] · 0.05 + [0.1, -0.2] · 1
          = [0.005, -0.010] + [0.1, -0.2]
          = [0.105, -0.210]    ← gradient mostly flows untouched via the +I path
```

Even when `∂F/∂x → 0`, the gradient still gets through because of the `+I` from the shortcut. **This is why ResNet trains 152 layers without vanishing gradients.**

---

## Type 5: Loss Computation

### Example 5.1: Cross-Entropy

Predicted probabilities `[0.1, 0.7, 0.2]`, true class index = 1 (middle).

```
L = -log(p_true) = -log(0.7) = 0.357
```

If true class index = 0: `L = -log(0.1) = 2.303` (much higher loss for being wrong).

### Example 5.2: GAN BCE — discriminator loss

`D(real) = 0.9, D(fake) = 0.2`.

```
L_D = -log(D(real)) - log(1 - D(fake))
    = -log(0.9) - log(1 - 0.2)
    = -log(0.9) - log(0.8)
    = 0.105 + 0.223
    = 0.328
```

Low loss → D is doing well.

### Example 5.3: GAN BCE — generator loss (non-saturating)

Same `D(fake) = 0.2`.

```
L_G^nonsat = -log(D(fake)) = -log(0.2) = 1.609
```

High loss → G is failing (D detects fakes easily).

If G improves so D(fake) = 0.75:
```
L_G^nonsat = -log(0.75) = 0.288
```
Loss dropped — G is improving.

### Example 5.4: GAN BCE — saturating G loss (the bad version)

Same `D(fake) = 0.2`.

```
L_G^sat = log(1 - D(fake)) = log(1 - 0.2) = log(0.8) = -0.223
```

To minimize, G needs `1 - D(fake)` close to 0, i.e., `D(fake) → 1`. Same goal as non-saturating, but the gradient when `D(fake) ≈ 0` is tiny — that's why we use the non-saturating form.

### Example 5.5: Dice loss

Predicted mask has 100 pixels active. True mask has 80. Intersection (pixels active in both) = 60.

```
Dice = 2 · |P ∩ T| / (|P| + |T|)
     = 2 · 60 / (100 + 80)
     = 120 / 180
     = 0.667

L_Dice = 1 - Dice = 1 - 0.667 = 0.333
```

If predicted mask was all empty (P = 0): intersection = 0, Dice = 0, loss = 1 (max).
If perfect prediction (P = T = 80, intersection = 80): Dice = 160/160 = 1, loss = 0.

### Example 5.6: BCE for one prediction

True label `y = 1`, predicted probability `p = 0.9`.

```
L_BCE = -y·log(p) - (1-y)·log(1-p)
      = -1·log(0.9) - 0·log(0.1)
      = -log(0.9)
      = 0.105
```

If `y = 0, p = 0.1`:
```
L_BCE = -0·log(0.1) - 1·log(0.9)
      = -log(0.9) = 0.105
```

If `y = 0, p = 0.9` (wrong):
```
L_BCE = -0·log(0.9) - 1·log(0.1)
      = -log(0.1) = 2.303 (high!)
```

---

## Type 6: Concept Reasoning (with brief math when needed)

### Example 6.1: Why Tanh at GAN G output?

Real images are normalized to `[-1, +1]` via `(x · 2) - 1`. Tanh squashes outputs to `[-1, +1]`, matching the real-image range. If G output and real images had different ranges, D would learn "anything outside this range is fake" as a free shortcut, ignoring image content.

### Example 6.2: Why √d_k scaling in attention?

Without scaling, dot products `Q · Kᵀ` grow proportionally to `d_k`. For `d_k = 64`, dot products of unit-variance Q, K have variance ≈ 64 (std ≈ 8) — softmax of these saturates (one entry near 1, others near 0), and gradients become tiny. Dividing by `√d_k` rescales so the variance stays ≈ 1.

### Example 6.3: Why concat (not add) in U-Net skip connections?

Concatenation (`dim=1`) preserves both signals — encoder feature map *and* decoder feature map are kept as separate channels. Addition would mix them, losing information. The decoder's subsequent DoubleConv learns how to weight them.

### Example 6.4: Why does LSTM solve vanishing gradient?

The cell-state update is **additive**: `C_t = f_t ⊙ C_{t-1} + i_t ⊙ C̃_t`. The gradient `∂C_t / ∂C_{t-1} = f_t` (element-wise). When the forget gate `f_t ≈ 1`, this derivative is ≈ 1, so the gradient flows backward through time without geometric decay. Vanilla RNN's `∂h_t / ∂h_{t-1}` is `W_hh^T · diag(tanh'(...))` — repeatedly multiplying by `W_hh^T` makes gradients shrink (or explode) geometrically.

### Example 6.5: Why is the GAN minimax loss "moving target"?

In standard supervised learning, the loss landscape is fixed once the dataset and loss function are chosen. In GANs, D's loss landscape depends on G's outputs (which keep changing as G learns), and G's gradient signal depends on D's current decision boundary. Both networks reshape the other's loss surface every step → no stable optimum is approached monotonically.

### Example 6.6: Why doesn't BatchNorm work in transformers?

BatchNorm normalizes across the batch dimension, which assumes a fixed per-feature distribution per position. Transformer inputs have variable sequence lengths and the "feature" axis is `d_model`, but each token's position in the sequence is meaningful — averaging across batch positions destroys per-token information. **LayerNorm** normalizes across the feature dimension *within each token*, independent of batch size or other tokens.

### Example 6.7: Why does ResNet enable training of very deep networks?

The **degradation problem** in plain deep CNNs: even on training data, accuracy gets worse as depth increases beyond ~20 layers. Two causes: (1) vanishing gradients through the deep stack, and (2) optimization difficulty — even when capacity exists, the network struggles to learn the identity function (the trivial "do nothing" mapping). ResNet's residual block `y = F(x) + x` solves both:
1. **Gradient flow**: the identity shortcut means `∂L/∂x = ∂L/∂y · (1 + ∂F/∂x)`. The `+1` gives the gradient a direct path back to `x`, regardless of how small `∂F/∂x` becomes.
2. **Easy default**: `F = 0` is the trivial identity behavior. The network can always recover the input as a fallback. Adding more blocks can never hurt.

ResNet won ImageNet 2015 with 152 layers (3.57% top-5 error) — the first time models broke past ~22 layers.

### Example 6.8: ResNet skip connection vs U-Net skip connection

Both add wiring that bypasses some layers, but the operation and purpose differ.

| | ResNet | U-Net |
|---|---|---|
| Operation | **Addition**: `F(x) + x` | **Concatenation**: `torch.cat([up, skip], dim=1)` |
| Where | Within a block (same shape) | Cross-block (encoder → decoder) |
| Purpose | Preserve gradient flow through depth | Re-inject high-resolution spatial detail lost in pooling |
| Param impact | Identity = 0 params; projection = small 1×1 | Channels stack (no extra params for concat) |

**Common exam question:** "Why does U-Net use concat instead of add?" Concat preserves both signals as separate channels (the decoder's DoubleConv learns how to weight them). Add would mix them and lose the encoder's spatial information that the decoder specifically needs.

### Example 6.9: ResNet vs LSTM — same idea on different axes

Both architectures solve the gradient-flow problem with **additive shortcuts**, but along different axes:
- **ResNet**: `y = F(x) + x` — additive shortcut through *depth*. Lets gradient flow back through 50–152 stacked layers.
- **LSTM**: `C_t = f_t ⊙ C_{t-1} + i_t ⊙ C̃_t` — additive update through *time*. Lets gradient flow back through long sequences.

In both cases, when the "controlled" part (`F` or `f_t`) is near zero, the other path keeps the signal alive. **Transformer's residual stream** (`x + Sublayer(x)`) is the same idea, applied around every encoder/decoder sub-block.

---

## Type 7: Code Trace

### Example 7.1: GAN training loop — what does `with torch.no_grad():` do?

```python
# D-step
fake_images = G(z).detach()           # OR equivalently:
with torch.no_grad():
    fake_images = G(z)
d_loss_fake = bce(D(fake_images), zeros)
d_loss.backward()
```

**Answer**: `torch.no_grad()` (or `.detach()`) tells PyTorch not to track gradients through G during this part of the forward pass. So when `backward()` is called, the gradient computation stops at `fake_images` and never reaches G's weights. This prevents G from being accidentally updated during the D-step — only D's optimizer is called.

### Example 7.2: GAN G-step — why `target=1` for fake images?

```python
# G-step
fake_images = G(z)                    # NOT detached
g_loss = bce(D(fake_images), ones)    # target=1
g_loss.backward()
```

**Answer**: The target encodes G's *goal*, not the truth. G wants D to classify its fakes as real. Using `target=1` makes the loss high when D outputs 0 (fake) and low when D outputs 1 (real), so minimizing it pushes G toward outputs that fool D.

### Example 7.3: BPTT gradient accumulation

```python
for t in reversed(range(T)):
    dW += dh_raw @ h[t-1].T          # note: += not =
```

**Answer**: `+=` accumulates the gradient contribution from each time step. Because the same weight matrix W is reused at every time step (weight sharing across time), the total `dL/dW` is the sum of contributions across all time steps. Using `=` instead would overwrite, keeping only the last step's contribution.

### Example 7.4: U-Net skip connection

```python
x = self.up(x)                        # ConvTranspose, doubles spatial
x = torch.cat([x, skip], dim=1)       # ← skip connection
x = self.double_conv(x)
```

**Answer**: The `torch.cat([x, skip], dim=1)` concatenates the upsampled decoder features `x` with the saved encoder features `skip` along the **channel dimension** (`dim=1` = channels in PyTorch's `(B, C, H, W)` layout). Channel counts add (e.g., 256 + 256 → 512); spatial dims must already match.

### Example 7.5: Self-attention mask polarity

```python
# Manual implementation
mask = torch.tril(torch.ones(n, n))   # lower-triangular = allowed
scores = scores.masked_fill(mask == 0, float('-inf'))

# PyTorch built-in
nn.MultiheadAttention(attn_mask=upper_triangular_True_mask)
```

**Answer**: In the manual code, `1` means "allowed" and `0` means "blocked." We fill `-inf` where `mask == 0`. In `nn.MultiheadAttention`, the convention is **opposite**: `True` (or non-zero in additive masks) means **blocked**. This is a common silent bug when porting between manual and built-in implementations.

### Example 7.6: Identifying a BPTT bug

```python
# Buggy code
for t in range(T):                     # ← BUG: should be reversed!
    dh = ...
    dW = dh_raw @ h[t-1].T              # ← BUG: = instead of +=
```

Two bugs:
1. The loop should iterate `reversed(range(T))` — gradients flow backward in time.
2. `dW = ...` overwrites; should be `dW += ...` to accumulate.

---

## Quick-recognition Table

| Question phrase | Type | Where to look |
|---|---|---|
| "Total parameters in this layer/network" | Type 1 | Reference Card Page 1 (param formulas) |
| "Output shape after this Conv/Pool/ConvT" | Type 2 | Reference Card Page 1 (shape formulas) |
| "Find k, p, H, D, C given..." | Type 3 | Reverse derivation — work backward from formula |
| Concrete W matrices given, "compute h_t / C_t" | Type 4 | LSTM equations (reference card Page 3) |
| "Compute the loss given D(real), D(fake)" | Type 5 | BCE / CE / Dice formulas |
| "Why does X use Y instead of Z?" | Type 6 | Reference Card topic page (concepts) |
| Code snippet with "what does this line do?" | Type 7 | Reference Card code blocks |

---

## Practice strategy (last hour)

1. Pick 2 examples from each Type. Solve without looking at the answer. Then check.
2. If you got Type 1 (param counting) and Type 2 (shape arithmetic) right, you have ~50 points of the exam locked.
3. Type 4 (forward pass) is mostly mechanical once you recognize the equations.
4. Types 5–7 are quick if you know what's being asked.

If you can do these 7 patterns mechanically, the architectures become straightforward. The reference card has every formula you need; this doc shows you how to use them.
