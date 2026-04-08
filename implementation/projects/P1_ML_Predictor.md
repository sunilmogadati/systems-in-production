# Project P1: ML Predictor

**Week:** 2 (Machine Learning Fundamentals)
**Time estimate:** 6-8 hours
**Prerequisites:** Python for AI notebook, Math for AI notebook, Step 1 (Linear Regression), Step 2 (Logistic Regression), ML Fundamentals notebook

---

## Objective

Build a **binary classifier** that predicts whether a bank customer will churn (leave the bank). You will use real-world data, apply the full ML lifecycle from the ML Fundamentals notebook, and produce an evaluation report that demonstrates you can train, evaluate, and explain a model.

This is your first portfolio project. By the end, you will have a GitHub repo with working code, evaluation metrics, and an explanation of your model's decisions.

---

## The Dataset

**Bank Customer Churn** — available on [Kaggle](https://www.kaggle.com/datasets/shantanudhakadd/bank-churn-modelling) or included in the project starter.

| Feature | Type | Description |
|:---|:---|:---|
| CreditScore | Numeric | Customer's credit score |
| Geography | Categorical | Country (France, Germany, Spain) |
| Gender | Categorical | Male / Female |
| Age | Numeric | Customer age |
| Tenure | Numeric | Years as a bank customer |
| Balance | Numeric | Account balance |
| NumOfProducts | Numeric | Number of bank products used |
| HasCrCard | Binary | Has a credit card (1/0) |
| IsActiveMember | Binary | Is an active member (1/0) |
| EstimatedSalary | Numeric | Estimated salary |
| **Exited** | **Binary (Target)** | **Did the customer leave? (1 = yes, 0 = no)** |

> **Why this dataset?** It has a mix of numeric and categorical features, a binary target, and class imbalance (~20% churn). These are real-world conditions — not a clean textbook dataset.

---

## Requirements

Your project must include ALL of the following:

### 1. Data Exploration and Preparation
- [ ] Load the dataset and display basic statistics (shape, dtypes, null counts)
- [ ] Visualize the target distribution (is it balanced? what is the churn rate?)
- [ ] Handle categorical features (one-hot encoding or label encoding — explain your choice)
- [ ] Split into train and test sets (80/20 or 70/30 — explain your choice)
- [ ] Apply any feature scaling if needed (explain why or why not)

### 2. Model Training
- [ ] Train at least **2 different models** (e.g., Logistic Regression + Random Forest, or Random Forest + XGBoost)
- [ ] Use **cross-validation** (k-fold, minimum k=5) — not just a single train/test split
- [ ] Document which hyperparameters you tried and why

### 3. Evaluation (minimum 3 metrics)
- [ ] **Accuracy** — overall correctness
- [ ] **Precision** — of the customers we predicted would churn, how many actually did?
- [ ] **Recall** — of the customers who actually churned, how many did we catch?
- [ ] **F1 Score** — harmonic mean of precision and recall
- [ ] **Confusion matrix** — visualized as a heatmap
- [ ] **ROC-AUC (Receiver Operating Characteristic — Area Under Curve)** — how well does the model separate churners from non-churners?
- [ ] Compare all metrics across your 2+ models in a single table
- [ ] Explain which metric matters most for this business problem and why

> **Hint:** For churn prediction, the bank cares more about **recall** (catching churners) than precision (being right when you say "churn"). Missing a churner costs the bank a customer. A false alarm just triggers a retention call. Think about this when choosing your best model.

### 4. Explainability (SHAP)
- [ ] Run **SHAP (SHapley Additive exPlanations)** on your best model
- [ ] Generate a SHAP summary plot showing which features drive churn
- [ ] Interpret the top 3 features in plain English: "Customers with [X] are [Y]% more likely to churn because [Z]"
- [ ] Generate at least 1 SHAP force plot for an individual prediction and explain it

### 5. Code Quality
- [ ] All code in a Jupyter notebook or Python scripts
- [ ] Functions for reusable logic (do not repeat yourself)
- [ ] Comments explaining WHY, not just WHAT (25%+ comment density)
- [ ] A `requirements.txt` listing all dependencies
- [ ] A `.gitignore` (exclude `__pycache__/`, `.venv/`, data files if large)

### 6. README
- [ ] Problem statement (1 paragraph)
- [ ] Dataset description (source, size, features)
- [ ] Approach (what models you tried, why)
- [ ] Results (best model, key metrics, SHAP summary)
- [ ] How to run (setup, install, execute)

### 7. Reflection Journal
After completing the project, answer these questions in a separate `REFLECTION.md`:

1. What was the hardest part of this project?
2. If you saw the reference solution, what would you do differently?
3. What surprised you in the data or results?
4. If you had to deploy this model tomorrow, what would you need to add?

---

## Step-by-Step Guide

### Step 1: Set up your project
```bash
mkdir p1-ml-predictor && cd p1-ml-predictor
python -m venv .venv
source .venv/bin/activate
pip install pandas scikit-learn matplotlib seaborn shap xgboost jupyter
pip freeze > requirements.txt
```

### Step 2: Explore the data
Load the CSV. Look at it. Plot it. Understand it before touching any model. What is the churn rate? Which features look predictive? Any missing values?

### Step 3: Prepare the data
Encode categoricals. Scale numerics if needed. Split train/test. This is the boring part that determines 80% of your model's performance.

### Step 4: Train models
Start with Logistic Regression (the baseline — fast, interpretable). Then try Random Forest or XGBoost. Use `cross_val_score` with `cv=5` for each.

### Step 5: Evaluate
Build a comparison table. Plot the confusion matrix. Calculate ROC-AUC. Decide which model wins — and explain why.

### Step 6: Explain with SHAP
Run SHAP on the winning model. Generate the summary plot. Pick 3 individual predictions and explain them.

### Step 7: Write the README and reflection
### Step 8: Push to GitHub

```bash
git init && git add . && git commit -m "P1: Bank churn predictor with SHAP explainability"
gh repo create p1-ml-predictor --public --push
```

---

## Grading Rubric

| Category | Points | What We Look For |
|:---|:---|:---|
| Data exploration and preparation | 15 | Clean, handled nulls/categoricals, visualized target distribution |
| Model training (2+ models, cross-validation) | 20 | Correct use of cross-validation, multiple models compared |
| Evaluation (3+ metrics, confusion matrix, ROC-AUC) | 20 | Correct metrics, comparison table, business interpretation |
| SHAP explainability | 15 | Summary plot, force plot, plain-English interpretation of top features |
| Code quality (comments, structure, reproducibility) | 15 | WHY comments, functions, requirements.txt, runs on someone else's machine |
| README + Reflection journal | 15 | Clear, concise, honest reflection |
| **Total** | **100** | |

---

## The Interview Story

> "I built a customer churn predictor for a bank dataset — 10,000 customers, 11 features, 20% churn rate. The key challenge was class imbalance — the model kept predicting 'no churn' because that was right 80% of the time. I compared Logistic Regression versus Random Forest using 5-fold cross-validation and focused on recall because missing a churner costs the bank a customer, while a false alarm just triggers a retention call. Random Forest won with 82% recall and 0.87 ROC-AUC. SHAP analysis showed the top churn drivers were age (older customers churn more), number of products (customers with 1 product churn most), and geography (Germany had 2x the churn rate of France). I can show you the SHAP plots."

---

## Reference: External Projects to Study First

Before building your own, study how others approached similar problems:

| Repo | What to Study |
|:---|:---|
| [Kaggle kernels for this dataset](https://www.kaggle.com/datasets/shantanudhakadd/bank-churn-modelling/code) | See how other data scientists approached the same problem. Compare their feature engineering and model choices to yours. |
| [ML Fundamentals notebook (Section 1-7)](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/ML_Fundamentals.ipynb) | Your primary reference for the ML lifecycle, cross-validation, SHAP, and MLflow. |

---

## Deliverables

Push to GitHub with this structure:

```
p1-ml-predictor/
  README.md
  REFLECTION.md
  requirements.txt
  .gitignore
  data/                    # Dataset (or instructions to download)
  notebooks/
    exploration.ipynb      # Data exploration and visualization
    modeling.ipynb         # Training, evaluation, SHAP
  src/                     # (optional) Python modules if you refactor
```
