# Generative Models — How It Works

**Training instability, mode collapse, vanishing gradients, evaluation metrics. The diagnostic skills generative training demands that classifier training does not.**

---

## Why Generative Training Is Different

A classifier has a clean training signal. The label is unambiguous. Loss curves go down. You stop when test accuracy plateaus. The diagnostic from [Deep Learning → 04](../deep-learning/04_How_It_Works.md) (healthy / overfitting / underfitting / diverging) tells you what to do.

Generative training has **none of this clarity**:

- GAN losses **oscillate** — they do not monotonically decrease.
- VAE losses combine two terms with a weighting hyperparameter; tuning it is part of the model.
- Diffusion losses are surprisingly stable, but loss values are **uninterpretable** — you cannot tell from the loss alone if generated samples look good.
- "Test accuracy" doesn't apply — there is no label. You evaluate by looking at outputs.

Reading generative training requires different diagnostics. This chapter is them.

---

## GAN Failure Modes — A Visual Taxonomy

GAN training has four characteristic failure patterns. Recognizing them instantly is the skill.

### Pattern 1: Healthy Adversarial Equilibrium

```
D loss:  ▆▇▇▆▆▇▇▆▆▇▇▆▆▇    (oscillates around 0.7-1.0)
G loss:  ▅▆▆▅▅▆▆▅▅▆▆▅▅▆    (oscillates around 0.7-1.0)
Outputs: gradually improving
```

Both losses oscillate around similar values. **Neither network is winning.** Generated samples slowly improve over hundreds of thousands of steps. This is what success looks like.

**Action:** None. Keep training. Sample outputs every N epochs to track progress visually.

### Pattern 2: Discriminator Wins (Vanishing Gradients)

```
D loss:  █▇▆▅▄▃▂▁          (drops to near-zero)
G loss:  ▁▂▃▄▅▆▇█          (climbs)
Outputs: stuck — no improvement
```

D becomes too good too fast. It identifies fakes with near-certainty. This means the gradient flowing back to G is tiny — there is no useful signal for G to follow. **G stops learning.**

**Diagnosis.** The discriminator overpowered the generator.

**Fixes:**
- Reduce D's capacity (fewer layers, fewer neurons)
- Use a lower learning rate for D than for G
- Add label smoothing — set D's "real" target to 0.9 instead of 1.0 (prevents D from being too confident)
- Train G multiple steps per D step (e.g., 5:1)
- Switch to Wasserstein loss (WGAN-GP) which avoids gradient saturation

### Pattern 3: Mode Collapse

```
D loss:  ▅▅▅▅▅▅▅▅▅▅▅▅      (stable)
G loss:  ▅▅▅▅▅▅▅▅▅▅▅▅      (stable)
Outputs: same image repeated for different z values
```

The losses look fine. But generated samples lack diversity — the generator produces the same one or few outputs for different noise inputs.

**Diagnosis.** Generator found a "safe" output that fools D and stopped exploring.

**Fixes:**
- **Mini-batch discrimination** — let D see batches of fake samples and detect when they are all similar
- **Wasserstein loss with gradient penalty** (WGAN-GP) — smoother loss landscape, harder to mode-collapse
- **Diversity loss** — explicitly penalize G for low pairwise distances between outputs from different z
- **Spectral normalization** — constrains D's Lipschitz constant, stabilizes training

### Pattern 4: Diverging / Catastrophic Forgetting

```
D loss:  ▁▆▂█▁▇▃█▁          (chaotic, large swings)
G loss:  █▂▇▁▆▂█▁           (chaotic, large swings)
Outputs: oscillate between recognizable and pure noise
```

Training goes unstable. Both networks oscillate wildly. Sample quality is non-monotonic — improves, then collapses, then improves again, then collapses.

**Diagnosis.** Either learning rate too high, or training is in an unstable regime that the architecture cannot escape.

**Fixes:**
- Reduce learning rate by 5-10x
- Lower the Adam beta1 from 0.9 to 0.5 (less momentum, more reactive — DCGAN's recommendation)
- Use spectral normalization on D
- Switch to a more stable loss (Wasserstein with gradient penalty)

---

## The Two-Sample Diagnostic — Always Run This

Every N epochs, generate samples and **save them to disk with a fixed noise vector** so you can track the same z over time:

```python
fixed_z = torch.randn(16, Z_DIM, device=device)   # frozen at start of training

# At every save checkpoint:
G.eval()
with torch.no_grad():
    samples = G(fixed_z)
    save_image(samples, f'samples_epoch_{epoch}.png', nrow=4)
G.train()
```

Watching how the same `fixed_z` evolves over epochs tells you:

- **Improving** — the same noise vector progressively produces a clearer image. Healthy.
- **Stuck** — output never changes. Generator is not learning.
- **Oscillating** — output changes randomly, sometimes better, sometimes worse. Unstable.
- **Collapsing** — different `z` values produce identical outputs. Mode collapse.

**This single visualization is more useful than any loss curve.** Loss curves can lie. Pixels do not.

---

## VAE Diagnostics

VAE training is far more stable than GAN. The two failure modes:

### Posterior Collapse

The encoder learns to ignore the input — every output's latent collapses to the standard normal `N(0, 1)`. The decoder is then a fixed function of pure noise, producing only the average of the training set.

**Symptom.** Reconstruction loss is high; KL term is near 0; outputs all look identical.

**Fix.** Reduce the KL term weight (the `β` in `β·KL`). Start with `β = 0` (pure autoencoder), gradually increase to `β = 1`. This is **KL annealing**.

### Blurry Outputs

VAE outputs are famously blurry compared to GAN. This is **not a bug** — it is the cost of the probabilistic objective. The reconstruction loss (typically pixel MSE) is minimized by the average over plausible reconstructions, which is mathematically a blurred image.

**Mitigations:**
- Use a perceptual loss instead of pixel MSE (compute loss in a CNN feature space, not pixel space)
- Use a VQ-VAE with a discrete latent and a separate prior model (this is the Stable Diffusion approach)
- Accept blurriness as a known property and use VAE for tasks where it does not matter (compression, anomaly detection, latent manipulation)

---

## Diffusion Diagnostics

Diffusion training is the most stable of the three. Standard diagnostic tools largely apply.

### Loss Going Down — But Quality Plateaus

Diffusion loss is a noisy objective. Loss can decrease smoothly while sample quality plateaus or even worsens. **Visual sample evaluation is required** — look at generated samples every N epochs, not just the loss.

### Sampling Speed

Default DDPM uses T = 1000 sampling steps. **This is too slow for most production.** Two solutions:

| Solution | What It Does | Cost |
|---|---|---|
| **DDIM** | Skip steps using a deterministic sampler. 50-100 steps work. | Slight quality loss |
| **Distillation** | Train a student model to do 1-4 steps directly | Significant up-front compute, then very fast inference |

For real-time generation (user types a prompt, image appears in <1 second), distilled diffusion is the only practical option.

### Conditioning Failure

The generator ignores the text prompt. Outputs look fine but unrelated to the prompt.

**Symptoms.** Prompts of "cat" and "dog" both produce generic animals.

**Fix.** Use **classifier-free guidance**. Train both with conditioning and without (drop the conditioning randomly). At inference, scale up the difference between conditional and unconditional outputs:

```
sample = unconditional + guidance_scale · (conditional − unconditional)
```

`guidance_scale` of 5-15 is typical. Higher = more prompt adherence, less diversity. Tune empirically.

---

## Evaluation Metrics — How to Score Generative Quality

Unlike classifiers, you cannot just compute accuracy. Generative quality is partly subjective. Three metrics dominate.

### FID (Fréchet Inception Distance)

The standard metric. Compares the distribution of generated samples to the distribution of real samples in a pre-trained Inception-v3 feature space.

```
FID = ||μ_real − μ_gen||² + Tr(Σ_real + Σ_gen − 2·sqrt(Σ_real · Σ_gen))
```

Where `μ` and `Σ` are the mean and covariance of feature vectors over a sample of images.

| FID Score | Quality |
|---|---|
| < 5 | Hard to distinguish from real (cherry-picked sets only) |
| 5-15 | Excellent (modern Stable Diffusion territory) |
| 15-30 | Good (typical published results 2023-2024) |
| 30-50 | Recognizable but artifacts |
| 50+ | Visibly bad |

**To compute:** generate 10,000-50,000 samples; compute features for both real and generated; compute FID. The library `pytorch-fid` makes this one line of Python.

**Critical caveat:** FID is biased. It assumes Inception features capture all relevant quality dimensions. For non-natural images (medical, satellite), FID may not correlate with actual quality.

### IS (Inception Score)

Older metric, less reliable. Measures whether each individual sample is **classifiable** (an Inception classifier is confident about it) AND whether the distribution of classes is **diverse**.

```
IS = exp(E[KL(p(y|x) || p(y))])
```

| IS Score | Quality |
|---|---|
| 1-3 | Poor |
| 3-7 | OK |
| 7-12 | Good |
| 12+ | Excellent |

**Use IS only for legacy comparisons.** Use FID for modern work. IS does not detect mode collapse and is gameable.

### CLIP Score (for Text-to-Image)

For text-conditioned models, CLIP score measures how well generated images match their prompts.

```
CLIPScore(image, prompt) = cosine_similarity(CLIP_image_embedding, CLIP_text_embedding)
```

| CLIPScore | Match Quality |
|---|---|
| > 0.30 | Strong match |
| 0.20 - 0.30 | Moderate match |
| < 0.20 | Weak match |

CLIP score is the standard for text-to-image evaluation. Combined with FID (which measures realism), you get a 2D metric: CLIPScore × FID.

### Human Evaluation — Still the Gold Standard

For consumer products, **human preference** is the only metric that matters. Pose pairs (your model's output vs a baseline) and ask raters to pick. Standard approaches:

- **Side-by-side** — show both, ask which is better
- **MOS (Mean Opinion Score)** — rate on a 1-5 scale
- **A/B testing in production** — track which model variant produces images users keep / share / iterate on

Industry teams (Midjourney, OpenAI) primarily use human eval. FID is a guide; the final call is human preference.

---

## Common Bugs in Generative Training

### Bug 1: Wrong Output Activation

The most common SimpleGAN bug. The generator's last layer must produce values in the **same range as the data**.

| Data Range | Last Layer Activation |
|---|---|
| [0, 1] (raw pixels / 255) | `Sigmoid` |
| [-1, 1] (normalized) | `Tanh` |
| Unbounded (e.g., audio, regression) | None (linear) |

Mismatch and the loss is broken. Symptom: outputs are always either 0 or 1, never the gradient signal.

### Bug 2: Forgetting `.detach()`

When training D on fakes, the line is `D(fake.detach())`. Without `.detach()`, gradients flow into G during D's update — confusing the math.

### Bug 3: Not Using Batch Normalization

Batch normalization is critical for GAN stability. The original DCGAN paper requires it on most layers (not the generator's final layer or the discriminator's first/last layer). Skip BN and training is harder to converge.

### Bug 4: Wrong Optimizer Settings

GAN-specific Adam settings (from the DCGAN paper):

```
optimizer = Adam(lr=2e-4, betas=(0.5, 0.999))
```

`beta1 = 0.5` instead of the default `0.9` makes Adam more reactive. Standard ML defaults often fail for GAN.

### Bug 5: Saving Models in Train Mode

`G.eval()` before sampling. `G.train()` before next training step. Forgetting the toggle keeps batch normalization in training mode (uses batch statistics), giving inconsistent samples.

---

## A Generative Debugging Workflow

When generative training is misbehaving:

| Step | Check | How |
|---|---|---|
| 1 | **Are sample outputs actually being saved?** | `save_image(...)` in the training loop |
| 2 | **Are losses oscillating reasonably (GAN) or decreasing (VAE/Diffusion)?** | Plot D loss and G loss over epochs |
| 3 | **Are samples improving over epochs?** | Visual inspection of saved samples |
| 4 | **Is mode collapse occurring?** | Generate from many different z, check diversity |
| 5 | **Did `.detach()` and BN go in the right places?** | Code review against canonical reference |
| 6 | **Is data normalized to match the generator's output range?** | `data.min()`, `data.max()` |
| 7 | **Is the architecture standard for the task?** | DCGAN for small images, U-Net for diffusion |
| 8 | **Have you checked FID after enough samples?** | 1000+ samples for stable FID |

> **The 100-sample test.** Generate 100 images. Look at them. Real engineers look. If samples look bad to your eye, no metric will save you. Trust your eyes first; metrics second.

---

**Next:** [05 — Building It](05_Building_It.md) — GAN vs VAE vs Diffusion decisions. Architecture choices. Training recipes for each family.
