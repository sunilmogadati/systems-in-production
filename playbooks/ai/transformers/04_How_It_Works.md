# Transformers — How It Works

**Layer norm placement (Pre-LN vs Post-LN), warmup schedules, gradient flow through deep stacks, attention patterns, training stability.**

---

## Why Transformer Training Is Different

A transformer's stability tradeoffs are different from RNN's or CNN's:

- **Vanishing/exploding gradients** are mostly fixed by residual connections + careful normalization
- **Stability problems** show up as NaN losses, oscillating training, or sudden divergence
- **Attention pattern collapse** — heads degenerate to attending to nothing useful — is a transformer-specific failure
- **Training instabilities** at scale (large models, long sequences) are an active area of research

The training loop is straightforward. The hyperparameters and architecture choices are where production transformers differ from textbook examples.

---

## Pre-LN vs Post-LN — The Most Important Architectural Choice

### Post-LN (the original 2017 paper)

```
y = LayerNorm(x + Sublayer(x))
```

Layer norm AFTER the residual addition. Used by BERT, original transformer, T5.

**Problem**: gradients through deep stacks (>10 layers) are unstable. Training requires carefully tuned learning-rate warmup. Without warmup, loss diverges.

### Pre-LN (the modern default)

```
y = x + Sublayer(LayerNorm(x))
```

Layer norm BEFORE the sublayer. Used by GPT-2/3/4, Llama, Mistral, Claude (almost certainly), and most production transformers since ~2019.

**Why it works better**: gradients flow through the residual connection unchanged (since LN comes before Sublayer, the `+ x` adds the unmodified residual). Deep stacks are stable. Warmup is less critical.

**Recommendation for new models: always Pre-LN.** Post-LN appears only in legacy code. The performance difference is small at typical depths (12-24 blocks); the stability difference is large.

---

## Learning Rate Warmup

Original transformer training used a **warmup schedule**: linearly increase LR from 0 to peak over a warmup period, then decay. This was essential for Post-LN; with Pre-LN, it is less critical but still common practice.

```python
def lr_lambda(step):
    if step < warmup_steps:
        return step / warmup_steps
    return max(0.0, 1 - (step - warmup_steps) / (total_steps - warmup_steps))

scheduler = torch.optim.lr_scheduler.LambdaLR(optimizer, lr_lambda)
```

**Typical schedules:**

| Schedule | Use Case |
|---|---|
| Linear warmup + cosine decay | Modern default for fine-tuning |
| Linear warmup + linear decay | Slightly simpler, also common |
| Inverse-sqrt decay (original paper) | Pretraining from scratch |
| Constant after warmup | Sometimes for short fine-tuning runs |

**Warmup duration**: usually 1-5% of total training steps. For very large models, more.

**Peak learning rate**:
- Pretraining a transformer from scratch: 1e-4 to 6e-4
- Fine-tuning a pretrained model: 1e-5 to 5e-5
- LoRA / adapter fine-tuning: 1e-4 to 1e-3

These are starting points. Tune for your specific task.

---

## Reading Attention Patterns

Trained transformers develop **specialized attention patterns** in different heads. Visualizing them is a key debugging tool.

### Common Patterns

| Pattern | What Heads Attend To |
|---|---|
| **Diagonal** | Each token attends mostly to itself |
| **Vertical bars** | All tokens attend to a few "anchor" positions (like start/end tokens, BOS/EOS) |
| **Off-diagonal** | Each token attends to its immediate neighbors (positional patterns) |
| **Long-range** | A few heads find specific syntactic or semantic patterns far apart |
| **Uniform / collapsed** | The head has not learned anything; equal weight everywhere |

Visualizing the `Attention(Q, K) = softmax(QKᵀ/√d_k)` matrix per head per layer reveals what the model has learned. Tools like BertViz make this interactive.

### When Attention Patterns Look Wrong

| Symptom | Likely Cause |
|---|---|
| All heads look the same | Too few heads or model collapsed; reduce d_model or num_heads |
| Many heads are uniform | Heads are dead; model has more capacity than needed |
| Diagonal-only patterns even in deep layers | Model is degenerate; check residual connections, normalization |
| All attention concentrated on a few tokens (sink tokens) | Known phenomenon; not necessarily wrong, but worth understanding |

---

## Gradient Flow Through Deep Stacks

A modern transformer has 12-120 blocks. Each block adds:

```
x_{n+1} = x_n + Attn(LN(x_n)) + FFN(LN(x_n + Attn(LN(x_n))))
```

The residual stream `x` flows through the entire stack additively. Gradient flows back the same way:

```
∂L/∂x_n = ∂L/∂x_{n+1} · (1 + ∂(Attn + FFN)/∂x_n)
```

The `+1` term is the residual shortcut — gradient passes through every block unchanged. **This is what makes 100-layer transformers trainable.** Same principle as ResNet's residual connections (covered in [Computer Vision → ResNet](../computer-vision/architectures/resnet.md)).

---

## Common Failure Modes

### 1. NaN Loss After A Few Steps

**Cause**: numerical instability, often from softmax saturation or exploding gradients.

**Fixes**:
- Lower learning rate
- Add gradient clipping (`max_norm = 1.0`)
- Switch to Pre-LN if using Post-LN
- Check for NaN/Inf in inputs (especially after preprocessing)
- Use mixed-precision (BF16/FP16) carefully — FP16 needs loss scaling

### 2. Loss Plateaus Early

**Cause**: learning rate wrong, model under-capacity, or attention heads not specializing.

**Fixes**:
- Try a larger model (more layers, more heads, larger d_model)
- Adjust learning rate up or down by 3x
- Check the data quality
- Verify the loss is correctly masked for padded positions

### 3. Loss Decreases But Generation Quality Is Poor

**Cause**: model has memorized but not generalized; or bad sampling at inference.

**Fixes**:
- More data, more diverse data
- Try different sampling: temperature, top-k, top-p
- Add dropout if overfitting
- Check tokenization — are tokens consistent across train and inference?

### 4. Attention Heads All Look the Same

**Cause**: model has too many heads for d_model (each head gets too few dimensions).

**Fix**: reduce `num_heads` or increase `d_model`. Rule of thumb: `d_k = d_model / num_heads ≥ 32`.

---

## Mixed Precision Training

Modern transformers train in mixed precision (FP16 or BF16) to fit larger models in GPU memory and run faster.

| Format | Range | When to Use |
|---|---|---|
| **FP32** | Full precision | Reference; rarely used end-to-end |
| **FP16** | Half precision | Common pre-2022; needs loss scaling |
| **BF16** | Brain Float | Modern default — same range as FP32, half the precision; no loss scaling needed |

```python
# PyTorch automatic mixed precision
scaler = torch.cuda.amp.GradScaler()

for batch in loader:
    with torch.cuda.amp.autocast(dtype=torch.bfloat16):
        logits = model(batch)
        loss = loss_fn(logits, targets)

    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

For a transformer with `d_model = 1024` and 24 layers, BF16 training uses ~half the GPU memory of FP32 with no loss scaling complications.

---

## A Transformer Debugging Workflow

When training is misbehaving:

| # | Check | How |
|---|---|---|
| 1 | **Is gradient clipping in place?** | `clip_grad_norm_(max_norm=1.0)` between backward and step |
| 2 | **Pre-LN, not Post-LN?** | Check the block's forward order |
| 3 | **Loss is masked for padded positions?** | If using padded sequences, ensure padded positions are excluded from loss |
| 4 | **Causal mask is correct?** | For decoder-only: `triu(ones, diagonal=1)` |
| 5 | **Learning rate sensible?** | 1e-4 for pretraining, 1e-5 for fine-tuning |
| 6 | **Warmup applied?** | Especially important for Post-LN, large models |
| 7 | **Mixed precision configured?** | BF16 if available; FP16 with loss scaling otherwise |
| 8 | **Attention patterns reasonable?** | Visualize a few heads; should not all look uniform |

---

## When to Switch Architectures

Switch to a different sequence-modeling architecture if:

| Symptom | Switch To |
|---|---|
| Sequences are very long (10K+) and full attention is too expensive | **Long-context Transformer** (FlashAttention, RingAttention, sparse attention) |
| Streaming inference required, constant memory | **RNN/LSTM/GRU** (see [Sequence Models](../sequence-models/)) |
| Sequences are very long AND streaming | **State Space Models** (Mamba, S4) — emerging alternative |
| Tiny edge device, parameters are the constraint | **Small distilled transformer** or RNN-T (recurrent transformer) |

---

**Next:** [05 — Building It](05_Building_It.md) — Encoder vs decoder vs encoder-decoder choice. Pretraining vs fine-tuning. LoRA, QLoRA, parameter-efficient fine-tuning.
