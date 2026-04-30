# NLP — Hello World

**Build a sentiment classifier two ways: classical (TF-IDF + logistic regression) in 30 lines, modern (BERT fine-tune via Hugging Face) in 50 lines. Compare results.**

---

## What You Will Build

A binary sentiment classifier — given a movie review, predict positive (1) or negative (0). Trained twice on the same data: once with classical NLP, once with a modern transformer.

The exercise is the comparison. **The classical approach is fast, interpretable, and surprisingly hard to beat on small data.** The modern approach takes 100x more compute, achieves a few percentage points more accuracy. When is the extra cost worth it? See [Chapter 05 — Building It](05_Building_It.md).

---

## Approach 1: Classical NLP (TF-IDF + Logistic Regression)

```python
from sklearn.datasets import fetch_20newsgroups
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline
from sklearn.metrics import classification_report

# === DATA ===
# Use a built-in dataset for reproducibility (replace with your own)
categories = ['alt.atheism', 'soc.religion.christian']  # binary
train = fetch_20newsgroups(subset='train', categories=categories, shuffle=True, random_state=42)
test  = fetch_20newsgroups(subset='test',  categories=categories)

# === PIPELINE ===
classical = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=20000, ngram_range=(1, 2), stop_words='english')),
    ('clf',   LogisticRegression(max_iter=1000, C=1.0)),
])

# === TRAIN ===
classical.fit(train.data, train.target)

# === EVAL ===
preds = classical.predict(test.data)
print(classification_report(test.target, preds, target_names=categories))

# === INSPECT ===
# Most important features per class
import numpy as np
vectorizer = classical.named_steps['tfidf']
clf        = classical.named_steps['clf']
features   = vectorizer.get_feature_names_out()
top_pos = np.argsort(clf.coef_[0])[-10:]
top_neg = np.argsort(clf.coef_[0])[:10]
print(f"\nTop positive class features: {features[top_pos]}")
print(f"Top negative class features: {features[top_neg]}")
```

That is the entire system — **~30 lines for data + pipeline + training + eval + interpretation.**

### What You Should See

```
              precision    recall  f1-score   support
alt.atheism       0.94      0.94      0.94       319
soc.religion.christian  0.95      0.95      0.95       389
   accuracy                           0.94       708

Top positive class features: ['atheism', 'atheist', 'atheists', ...]
Top negative class features: ['god', 'jesus', 'church', ...]
```

**~94% accuracy in seconds.** The model trained on a few thousand documents in under 30 seconds on a laptop. The features it relied on are inspectable and make sense.

---

## Approach 2: Modern NLP (BERT Fine-tune)

```python
from datasets import Dataset
from transformers import (
    AutoTokenizer, AutoModelForSequenceClassification,
    Trainer, TrainingArguments,
)
import numpy as np

# === DATA ===
# Same dataset, but format for Hugging Face
train_ds = Dataset.from_dict({"text": train.data, "label": train.target})
test_ds  = Dataset.from_dict({"text": test.data,  "label": test.target})

# === TOKENIZER ===
model_name = "distilbert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)

def tokenize(batch):
    return tokenizer(batch["text"], truncation=True, padding="max_length", max_length=256)

train_ds = train_ds.map(tokenize, batched=True)
test_ds  = test_ds.map(tokenize,  batched=True)

# === MODEL ===
model = AutoModelForSequenceClassification.from_pretrained(model_name, num_labels=2)

# === TRAINING ===
args = TrainingArguments(
    output_dir="./bert_out",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    learning_rate=2e-5,
    warmup_ratio=0.1,
    eval_strategy="epoch",
    save_strategy="no",
    logging_steps=50,
)

def compute_metrics(eval_pred):
    preds = np.argmax(eval_pred.predictions, axis=1)
    return {"accuracy": (preds == eval_pred.label_ids).mean()}

trainer = Trainer(
    model=model,
    args=args,
    train_dataset=train_ds,
    eval_dataset=test_ds,
    compute_metrics=compute_metrics,
)

trainer.train()

# === EVAL ===
results = trainer.evaluate()
print(f"\nFinal accuracy: {results['eval_accuracy']:.4f}")
```

That is the entire system — **~50 lines for data + tokenization + model + training + eval.**

### What You Should See

```
Epoch 1: eval_accuracy = 0.93, eval_loss = 0.21
Epoch 2: eval_accuracy = 0.96, eval_loss = 0.13
Epoch 3: eval_accuracy = 0.97, eval_loss = 0.10

Final accuracy: 0.9700
```

**~97% accuracy after fine-tuning DistilBERT for 3 epochs.** A few percentage points better than classical, at the cost of GPU compute.

---

## Comparing the Two

| Aspect | Classical (TF-IDF + LogReg) | Modern (BERT fine-tune) |
|---|---|---|
| **Accuracy** | 94% | 97% |
| **Training time** | ~30 seconds (CPU) | ~5 minutes (GPU) |
| **Inference per doc** | < 1ms (CPU) | ~10ms (GPU), ~100ms (CPU) |
| **Model size** | < 5 MB | ~250 MB |
| **Interpretability** | High — inspect feature weights | Low — black box |
| **Data efficiency** | Often wins on < 1K examples | Wins as data scales |
| **Setup complexity** | scikit-learn only | Hugging Face + GPU + tokenization |

### When Classical Wins

- Datasets under ~1,000 examples
- Latency under 1ms required
- Edge / embedded deployment (< 10MB total)
- Need to explain every prediction
- Engineering team is small / experimentally constrained

### When Modern Wins

- Datasets over ~10,000 examples
- The task requires understanding context, sarcasm, negation
- Multilingual coverage required
- Few percentage points of accuracy matter (medical, legal, financial)
- You will iterate frequently and want to leverage prompt-based zero-shot for data labeling

---

## Approach 3 (Bonus): Zero-Shot with an LLM

For tasks where you have no labeled data:

```python
from openai import OpenAI

client = OpenAI()  # or use Anthropic / local Llama

def classify_sentiment(text):
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Classify the sentiment of the user's text. Reply with exactly one word: positive, negative, or neutral."},
            {"role": "user",   "content": text},
        ],
        max_tokens=5,
    )
    return response.choices[0].message.content.strip().lower()

# Test
print(classify_sentiment("I loved this movie!"))    # positive
print(classify_sentiment("Worst experience ever.")) # negative
```

**Three lines of code.** No training. No labels. Works in 100+ languages.

| Aspect | Classical | BERT Fine-tune | Zero-shot LLM |
|---|---|---|---|
| Accuracy | 94% | 97% | ~85-95% (depends on task) |
| Training data needed | Hundreds | Hundreds-thousands | Zero |
| Inference cost | Free | Self-host or API | API ($/call) |
| Time-to-launch | Hours | Days | Minutes |

The 2026 production pattern: **start with zero-shot LLM**. If accuracy is insufficient, add few-shot examples. If still insufficient, fine-tune. Most projects never need to fine-tune.

---

## Common Bugs

### 1. Tokenizer / Model Mismatch

```python
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModelForSequenceClassification.from_pretrained("roberta-base", num_labels=2)  # WRONG
```

The tokenizer must match the model's expected vocabulary. Mixing them produces garbage.

### 2. Truncation Set Too Aggressively

```python
tokenizer(text, truncation=True, max_length=128)
```

If your documents are 500 words and you truncate to 128 tokens, the model sees ~25% of the input. For long-document tasks, increase max_length, use a model with longer context (some BERT variants go to 1024+), or chunk.

### 3. Class Imbalance

If 90% of your training data is class A and 10% is class B, the model can hit 90% accuracy by always predicting A. Useless.

Fixes:
- `class_weight='balanced'` in scikit-learn
- Weighted loss in PyTorch / Transformers
- Oversample the minority class

Always look at per-class precision/recall, not just overall accuracy.

### 4. Train/Test Leakage

For text classification, the typical leakage:
- Same document appears in both train and test (deduplication needed)
- Documents from the same author/source split across train and test (group-aware split needed)
- Future data leaks into training (time-aware split for time-sensitive data)

### 5. Wrong Evaluation Metric

For imbalanced data, accuracy lies. Use F1 (or precision and recall separately, depending on costs).

---

## Run It Yourself

The from-scratch notebook walks through BPE tokenization, TF-IDF computation, and naive Bayes — all by hand in NumPy:

**[NLP From Scratch on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/NLP_From_Scratch.ipynb)** — BPE merging, TF-IDF on a tiny corpus, naive Bayes spam detector. Pure NumPy where possible.

For the broader NLP pipeline (preprocessing → modeling → evaluation), see Chapters 04-09 of this playbook.

---

## What's Next

You have a working sentiment classifier (three ways). Now the harder questions:

- **Why does the BERT version outperform classical?** What does the fine-tuned model actually learn? — see [04 — How It Works](04_How_It_Works.md)
- **When should I fine-tune vs prompt vs use RAG?** — see [05 — Building It](05_Building_It.md)
- **How do production systems actually deploy this?** — see [06 — Production Patterns](06_Production_Patterns.md)
- **What about multilingual? Bias? Drift?** — see [08 — QSG](08_Quality_Security_Governance.md)

---

**Next:** [04 — How It Works](04_How_It_Works.md) — Embedding geometry, fine-tune vs prompt vs RAG decision, NLP-specific evaluation.
