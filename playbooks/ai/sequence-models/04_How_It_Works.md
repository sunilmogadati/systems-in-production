# Sequence Models — How It Works

**Vanishing and exploding gradients diagnosed in practice. Gradient clipping. Sequence length tradeoffs. The diagnostic skills recurrent training requires.**

---

## Why Recurrent Training Is Different

Feedforward training has clean diagnostics — loss curves, train/test gap, the four patterns from [Deep Learning → How It Works](../deep-learning/04_How_It_Works.md). Recurrent training adds two failure modes that do not happen in feedforward networks:

| Failure | What Happens |
|---|---|
| **Vanishing gradients** | Loss stops decreasing after a while. Training plateaus. The model cannot learn long-range dependencies. |
| **Exploding gradients** | Loss goes to NaN (Not a Number). Training diverges immediately. |

Both are caused by the **multiplicative chain through time**. Both are diagnosable. Both have standard fixes.

---

## Diagnosing Vanishing Gradients

Symptoms:

- Loss decreases for a while, then plateaus at a clearly suboptimal value
- The model learns short-range patterns but ignores long-range ones
- Predictions become identical regardless of context far in the past
- For a character-level model: predictions match the local 5-10 characters but ignore the broader meaning

How to confirm:

```python
# After loss.backward(), inspect gradient norms by depth in the unrolled network
for name, param in model.named_parameters():
    if param.grad is not None:
        grad_norm = param.grad.norm().item()
        print(f"{name}: grad norm = {grad_norm:.6f}")
```

If the gradients on **early-timestep weights** are orders of magnitude smaller than on late-timestep weights (or near zero), you have vanishing gradients.

For a vanilla RNN with sequence length 50, you typically see:
```
W_xh.grad norm: 1e-3   ← reasonable
W_hh.grad norm: 1e-7   ← effectively zero — vanished
```

### Fixes for Vanishing Gradients

In rough order of effectiveness:

1. **Switch to LSTM or GRU.** This is the primary fix. Vanilla RNNs cannot remember more than ~10 timesteps reliably; LSTMs/GRUs can handle hundreds. **In production, almost no one uses vanilla RNN.**
2. **Use proper initialization.** Orthogonal initialization for `W_hh` keeps the spectral radius near 1, which delays vanishing.
3. **Use shorter sequences during early training.** Curriculum learning — start with short sequences, lengthen as training progresses.
4. **Add residual connections.** A "highway network" or skip connection that wraps the recurrent step lets gradient flow around the recurrence.
5. **Move to attention / Transformers** if the task allows batch processing.

---

## Diagnosing Exploding Gradients

Symptoms:

- Loss goes to NaN within the first few batches
- Loss occasionally spikes to absurd values (1e10) before training crashes
- Weights become NaN or Inf

How to confirm:

```python
# Same gradient inspection as above, but you'll see norms like:
W_hh.grad norm: 1e15   ← exploded
```

### The Universal Fix — Gradient Clipping

Cap the total gradient norm at a maximum:

```python
loss.backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=5.0)
optimizer.step()
```

This single line solves exploding gradients almost universally. Typical `max_norm` values: 1.0 (aggressive), 5.0 (default), 10.0 (loose).

**Always include gradient clipping when training recurrent models.** Even if you do not have an exploding-gradient bug today, you will eventually — when input distribution shifts, when sequence length increases, or when learning rate is bumped. Clipping is cheap insurance.

---

## The Forget-Gate Bias Trick

For LSTMs, **initialize the forget gate's bias to 1.0 instead of 0.0**:

```python
for name, param in model.named_parameters():
    if 'bias_ih' in name or 'bias_hh' in name:
        n = param.shape[0]
        param.data[n//4 : n//2].fill_(1.0)   # forget gate bias to 1
```

Why: at initialization, sigmoid(0) = 0.5. The forget gate is half-open by default — half the cell state passes through, half is forgotten. With `bias = 1.0`, sigmoid(1) ≈ 0.73 — gate is more open. Information flows through earlier in training, gradients reach further back, and the LSTM learns long dependencies faster.

This is a documented trick from the LSTM literature. PyTorch's `nn.LSTM` does NOT do this automatically. If your LSTM is slow to learn long-range structure, try this.

---

## Sequence Length Tradeoffs

| Sequence Length | Memory | Compute | Training Behavior |
|---:|---|---|---|
| **1-10 timesteps** | Tiny | Fast | No gradient issues; either RNN or LSTM works |
| **10-100** | Small | Moderate | LSTM/GRU recommended; vanilla RNN starts to fail |
| **100-1000** | Larger | Slower | LSTM/GRU; consider truncated BPTT |
| **1000+** | Large | Slow | Truncated BPTT essential; consider Transformer if batch processing |

**Truncated BPTT** processes long sequences in chunks, carrying hidden state across chunks but only backpropagating within each chunk:

```python
hidden = None
for chunk_start in range(0, sequence_length, CHUNK_LEN):
    chunk = full_sequence[chunk_start : chunk_start + CHUNK_LEN]
    logits, hidden = model(chunk, hidden)

    # Detach so backprop only goes through this chunk
    hidden = (hidden[0].detach(), hidden[1].detach())

    loss = loss_fn(logits, targets[chunk_start : chunk_start + CHUNK_LEN])
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

Trade-off: truncated BPTT cannot learn dependencies longer than the chunk length. Pick chunk length based on your task's expected dependency range.

---

## The Four Patterns of Recurrent Training

### Pattern 1: Healthy Convergence

```
Train loss:  ████▓▓▓▒▒▒░░░░░  (decreasing steadily)
Test loss:   ████▓▓▓▒▒▒░░░░░  (decreasing, slightly above train)
Gradient norms: roughly stable across timesteps
```

**Action:** Nothing. Keep training.

### Pattern 2: Vanishing Gradients

```
Train loss:  █████████▒▒▒▒▒▒  (decreases briefly, then plateaus high)
Test loss:   █████████▒▒▒▒▒▒
Gradient norms: orders of magnitude smaller for early timesteps
```

**Diagnosis:** Cannot learn long-range dependencies.

**Fix:** LSTM/GRU; proper initialization; shorter sequences during warmup.

### Pattern 3: Exploding Gradients

```
Train loss:  ▆▆█▁██▁▆▁██  (chaotic, occasional NaN)
Gradient norms: spiking to 1e10+
```

**Diagnosis:** Multiplicative chain blowing up.

**Fix:** Gradient clipping (`max_norm=5.0`). Lower learning rate. Shorter sequences.

### Pattern 4: Overfitting

```
Train loss:  ████▓▓▓▒▒▒░░░░░  (keeps decreasing)
Test loss:   ████▓▓▓▒▒▓████  (decreases, then increases)
```

Same as feedforward overfitting. Standard fixes: dropout (between LSTM layers), reduce model size, more data.

---

## A Recurrent Debugging Checklist

When recurrent training is misbehaving, work through these in order:

| # | Check | How |
|---|---|---|
| 1 | **Is gradient clipping in place?** | `clip_grad_norm_` between `backward()` and `step()` |
| 2 | **Are you using LSTM/GRU, not vanilla RNN?** | `nn.LSTM(...)` not `nn.RNN(...)` |
| 3 | **Is the LSTM forget-gate bias initialized to 1?** | Apply the trick above (PyTorch default is 0) |
| 4 | **Are you detaching hidden state across batches?** | `hidden[0].detach(), hidden[1].detach()` for LSTM |
| 5 | **Is sequence length reasonable?** | Try shorter; if better, use truncated BPTT |
| 6 | **Are gradients vanishing or exploding?** | Print `param.grad.norm()` for each param |
| 7 | **Is the learning rate appropriate?** | Adam at 1e-3 is the default. RNNs sometimes need 1e-4. |
| 8 | **Is the data correct?** | Print actual sequences fed to the model. Are tokens/chars in the right order? |

---

## When to Switch to Transformer

If you find yourself:

- Routinely fighting vanishing/exploding gradients
- Needing sequences > 1000 timesteps with full backprop
- Training on enough data to leverage attention's parallelism (>100K sequences)
- Working in NLP where Transformers dominate

…it is time to switch to Transformer. See the [Transformers playbook](../transformers/) for that path.

If your problem fits any of these:

- Streaming inference with constant memory
- Time-series with <10K sequences
- Edge / embedded deployment with parameter constraints
- Sequential decision making with persistent state

…recurrent models are still the right tool. The next chapter covers building them properly.

---

**Next:** [05 — Building It](05_Building_It.md) — RNN vs LSTM vs GRU vs Transformer. Architecture choices, training recipes, when to use which.
