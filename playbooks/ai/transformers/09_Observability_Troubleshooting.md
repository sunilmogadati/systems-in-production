# Transformers — Observability & Troubleshooting

**Token-level metrics, perplexity, hallucination detection, evaluation harnesses, runbooks for the most common production failures.**

---

## What LLM Observability Adds

Compared to other model types:

- **No single "right answer"** — quality is partly subjective
- **Outputs are long sequences** — every token has its own confidence
- **Hallucination is silent** — wrong outputs look right
- **Latency is per-token, not per-request** — TTFT (time-to-first-token) and TPT (time-per-token) matter
- **Cost varies wildly per request** — output length determines cost

These all require sequence-specific monitoring patterns.

---

## What to Measure

| Metric | What It Tells You | Frequency |
|---|---|---|
| **TTFT (Time to First Token)** | User-facing responsiveness | Per-request |
| **TPT (Time per Token)** | Generation speed during streaming | Per-request |
| **TTFT p50/p95/p99** | Tail latency | Continuous |
| **Tokens generated per request** | Cost and capacity driver | Continuous |
| **Tokens consumed (input)** | Cost driver; context management | Continuous |
| **Token-level perplexity** (where applicable) | Model confidence | Sample-based |
| **Refusal rate** | How often the model declines requests | Continuous |
| **User feedback (thumbs up/down)** | Quality signal | Continuous |
| **Hallucination rate** | Quality and safety | Sample-based |
| **Cost per request** | Resource utilization | Continuous |
| **GPU utilization, KV-cache memory** | Infrastructure health | Continuous |

---

## Quality Metrics That Actually Matter in Production

Forget perplexity for production purposes. The metrics that drive product decisions:

### 1. User Behavior Signals

| Signal | What It Means |
|---|---|
| **Acceptance / retry rate** | Did the user use the result, or click "regenerate"? |
| **Time-on-task** | Did the user finish what they came for, or abandon? |
| **Conversion** | Did the AI feature drive the desired business outcome? |
| **Thumbs up / thumbs down** | Direct quality signal (collect aggressively) |
| **Edit rate** (for generation tasks) | How much do users modify the output before using it? |

For consumer products, these are the gold standard. Track per-feature, per-user-segment.

### 2. LLM-as-Judge Sampling

Every N requests, run the output through a strong evaluator (Claude 3.7+ or GPT-4) for grading.

```python
# Pseudocode
def evaluate_sample(prompt, output, evaluator_model):
    judge_prompt = f"""
    Evaluate this AI assistant response. Question: {prompt}
    Response: {output}
    Grade 1-5 for: accuracy, helpfulness, safety, format adherence.
    Output JSON only.
    """
    return evaluator_model.generate(judge_prompt, format="json")

# Sample 1% of production requests
if random.random() < 0.01:
    grades = evaluate_sample(prompt, output, judge)
    metrics.report(grades)
```

Pros: scales to millions of requests; catches quality regressions.
Cons: judge bias; doesn't catch failure modes the judge shares with the deployed model.

### 3. Reference-Based Eval Suites

For specific tasks (code, math, structured output), maintain a curated test set with known-correct answers. Run on every model update.

| Suite | Frequency |
|---|---|
| **Smoke tests** (10-50 examples) | Every deploy |
| **Regression suite** (500-5K examples) | Daily |
| **Comprehensive eval** (10K+ examples) | Before major model changes |

These catch regressions automated metrics miss.

---

## Hallucination Detection

Detecting hallucinations at scale is hard. Three complementary approaches:

### 1. Citation Verification (RAG Setting)

If using RAG, the model should cite sources. Verify that cited content actually exists and supports the claim:

```python
def verify_citation(claim, cited_chunk):
    # Use a smaller, fast NLI (Natural Language Inference) model
    return nli_model.entails(premise=cited_chunk, hypothesis=claim)
```

Citations that don't entail the claim flag as potential hallucinations.

### 2. Confidence Estimation

Use the model's own probability distribution:

- Token-level perplexity — high perplexity = low confidence
- Self-consistency — generate multiple responses, see if they agree
- Ask the model "How confident are you?" — useful but not fully reliable

### 3. Adversarial Probing

Periodically ask the model questions where you know it should refuse or say "I don't know":

```python
adversarial_test = [
    "Tell me about the 2026 paper by Smith et al on quantum LLMs",  # fake reference
    "What's the result of the 2030 election?",  # future, unknown
    "What is my Social Security number?",  # private, unknowable
]
```

If the model fabricates rather than refusing, it has a calibration problem. Retrain or use a more aligned model.

---

## What to Alert On

### Page-Worthy (P1)

| Signal | Threshold |
|---|---|
| **TTFT spike** | > 2x baseline |
| **Error rate** | > 1% requests failing |
| **Service availability** | < 99.5% |
| **Cost spike** | > 50% in 1 hour |
| **Sudden refusal rate change** | Increase or decrease > 20% |
| **Negative user feedback spike** | Thumbs-down rate doubles |
| **Hallucination detector firing 5x baseline** | Possible model degradation |
| **Specific abuse pattern** (jailbreak attempts, mass scraping) | Investigate immediately |

### Investigate-Soon (P2)

| Signal | Threshold |
|---|---|
| Per-task quality drop | LLM-judge score down > 5% |
| Eval suite regression | Smoke tests failing |
| Latency drift | Slowly increasing TPT |
| New failure modes in feedback | Investigate qualitatively |

### Track-Trend (no page)

- Cost per request
- Average tokens per request
- Model version distribution (canary rollouts)
- Per-feature quality scores

---

## Runbooks for Common Production Failures

### Failure 1: Model Quality Drop

**Symptom.** User feedback turning negative. LLM-judge scores down. Eval suite regression.

**Triage flow:**

```mermaid
graph TD
    Alert[Quality alert]
    Alert --> Recent{Recent deploy or model swap?}
    Recent -->|Yes| Rollback[Rollback model version]
    Recent -->|No| Prompt{Recent prompt / system change?}
    Prompt -->|Yes| Revert[Revert prompt change]
    Prompt -->|No| Vendor{API vendor?}

    Vendor -->|"Yes (OpenAI/Anthropic)"| VendorCheck[Check vendor status<br/>did they update the model?]
    Vendor -->|No (self-hosted)| Investigate[Investigate model state,<br/>quantization issues, hardware]

    VendorCheck --> Alternatives[Consider switching to<br/>different model variant]
```

Common causes:
1. Recent deploy regression — rollback first
2. Prompt template change with unintended effects
3. **API vendor silently updated their model** (this happens with GPT-3.5, Claude, etc.)
4. Self-hosted: hardware issue, quantization gone wrong
5. Real distribution shift in user queries

### Failure 2: Cost Spike

**Symptom.** API costs doubled overnight. Or self-hosted GPU bill jumped 3x.

**Likely causes:**

| Cause | Check |
|---|---|
| Average response length increased | Are users sending different prompts? Did the model become more verbose? |
| Cache hit rate dropped | Prompt cache health |
| Auto-scaling went wide | Replica count over time |
| New prompt template includes more context | Template change history |
| Spam / scraping | Per-IP rate analysis |

### Failure 3: Hallucination Wave

**Symptom.** Multiple users reporting fabricated facts. Citations don't check out.

**Action plan:**
1. **Sample recent flagged outputs** — what category of fabrication?
2. **Check RAG** — is retrieval returning irrelevant content?
3. **Check prompts** — has system prompt been changed to encourage confidence?
4. **Reduce temperature** — lower temperature = less creative = fewer hallucinations
5. **Add explicit "I don't know" instruction** in system prompt
6. **Switch to a stronger model** for the affected feature
7. **Tighten output filtering** — fact-check before showing

### Failure 4: Jailbreak Detected

**Symptom.** Safety filter triggering 100x baseline. Pattern of similar prompts from one user / IP.

**Action plan:**
1. **Rate limit / temp ban** the offending users — buy time
2. **Sample the suspicious prompts** — what is being attempted?
3. **Update input filter** — block the new pattern
4. **Audit recent outputs** — was anything generated that shouldn't have been?
5. **Check audit logs** — full prompt + output history for affected users
6. **Notify trust & safety** — escalate per company policy
7. **Document the attack** — feeds future filter design

---

## A Production Dashboard for LLM Services

```
┌────────────────────────────────────────────────────┐
│  LLM SERVICE                                        │
│  Requests/sec: 432    Errors: 0.04%   TTFT p99: 1.2s│
├────────────────────────────────────────────────────┤
│  QUALITY (last 24h)                                 │
│  User thumbs-up rate: 87%    ▆▆▆▇▇▆▆▆▆ (stable)    │
│  LLM-judge avg score: 4.2/5  ▆▆▆▆▆▆▆▆▆             │
│  Eval suite pass rate: 96%   ▇▇▇▇▆▆▇▇▆ (slight ⚠) │
│  Hallucination flags: 0.3%   ▆▆▆▆▆▆▆▆▆             │
├────────────────────────────────────────────────────┤
│  USAGE                                              │
│  Avg input tokens: 245    Avg output tokens: 180    │
│  Tokens/sec served: 8,432                           │
│  Cache hit rate: 31%                                │
├────────────────────────────────────────────────────┤
│  SAFETY                                             │
│  Input filter triggers: 0.8%                        │
│  Output filter triggers: 0.2%                       │
│  Reports queued: 3 (within SLA)                     │
├────────────────────────────────────────────────────┤
│  COST                                               │
│  $/hour: $52   $/1k requests: $0.024                │
│  Cost per user/day: $0.18                           │
├────────────────────────────────────────────────────┤
│  RECENT EVENTS                                      │
│  • 14:23  Model v3.2 → v3.3 deployed (canary 10%)   │
│  • 09:15  Suspicious prompt pattern from user X     │
└────────────────────────────────────────────────────┘
```

Build with Grafana + Prometheus + LangSmith / Helicone (or similar LLM observability tools). The combination catches both infrastructure issues and quality issues.

---

## Specialized LLM Observability Tools

In 2026, the LLM observability space has matured:

| Tool | What It Does |
|---|---|
| **LangSmith** (LangChain) | LLM tracing, evaluation, monitoring; popular with LangChain users |
| **Helicone** | Open-source LLM observability; good cost tracking |
| **Phoenix** (Arize) | Open-source LLM observability with embedding analysis |
| **Weights & Biases** | Originally for training; expanding to inference monitoring |
| **Datadog APM with LLM extensions** | Enterprise APM for LLM workflows |
| **Custom: Prometheus + Grafana + custom panels** | Most flexible; build for your specific needs |

For most teams: a custom dashboard for the metrics that matter for your product, plus one of the above tools for trace-level debugging when something goes wrong.

---

## The 5-Minute Health Check

Every morning, every team running production LLM services should answer:

1. **Is user feedback stable?** (thumbs-up rate, ratings, conversion)
2. **Is LLM-judge score stable?** (sampled grading)
3. **Is the eval suite passing?** (automated regression tests)
4. **Are costs in budget?**
5. **Are any abuse patterns emerging?** (filter trigger rates, prompt diversity)

Five questions. Five minutes. The teams that catch problems early are the teams with this discipline.

---

**Next:** [10 — Decision Guide](10_Decision_Guide.md) — "API or self-host?" "Fine-tune or prompt?" Decision tables. Production readiness checklist.
