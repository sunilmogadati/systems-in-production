# Machine Learning Fundamentals — Hello World

**Train a model, evaluate it honestly, explain its predictions — in 20 lines.**

> **Python note:** The code below uses `import`, `print`, and standard library calls. If any syntax is unfamiliar, the [Python for AI notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/Python_Basics.ipynb) covers everything needed. But try running the code first.

---

## What This Demonstrates

In the next few minutes:
1. Load a real dataset (California housing prices)
2. Train two models (Linear Regression and Random Forest)
3. Evaluate with the right metrics — not just accuracy
4. Explain a prediction with SHAP (SHapley Additive exPlanations) — "WHY did the model predict this price?"
5. See the bias-variance tradeoff in action — one model underfits, one overfits

This is the entire ML lifecycle in miniature. The notebook expands every step. This is the preview.

---

## The Code — All of It

```python
# === SETUP ===
from sklearn.datasets import fetch_california_housing
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.linear_model import LinearRegression
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_error, r2_score
import numpy as np

# === STEP 1: Load data ===
data = fetch_california_housing()
X, y = data.data, data.target  # X = features (8 columns), y = median house price
feature_names = data.feature_names

print(f"Dataset: {X.shape[0]:,} houses, {X.shape[1]} features")
print(f"Features: {feature_names}")
print(f"Target: Median house price (in $100Ks)")

# === STEP 2: Split into train and test ===
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
print(f"\nTrain: {X_train.shape[0]:,} houses | Test: {X_test.shape[0]:,} houses")

# === STEP 3: Train two models ===
lr = LinearRegression()
rf = RandomForestRegressor(n_estimators=100, random_state=42)

lr.fit(X_train, y_train)
rf.fit(X_train, y_train)

# === STEP 4: Evaluate — not just accuracy ===
for name, model in [("Linear Regression", lr), ("Random Forest", rf)]:
    train_pred = model.predict(X_train)
    test_pred = model.predict(X_test)

    train_r2 = r2_score(y_train, train_pred)
    test_r2 = r2_score(y_test, test_pred)
    test_rmse = np.sqrt(mean_squared_error(y_test, test_pred))

    print(f"\n{name}:")
    print(f"  Train R²: {train_r2:.3f}")
    print(f"  Test R²:  {test_r2:.3f}")
    print(f"  Gap:      {train_r2 - test_r2:.3f}")
    print(f"  Test RMSE: ${test_rmse * 100_000:,.0f}")

# === STEP 5: Cross-validate for reliability ===
print("\n5-Fold Cross-Validation (R²):")
for name, model in [("Linear Regression", lr), ("Random Forest", rf)]:
    scores = cross_val_score(model, X_train, y_train, cv=5, scoring='r2')
    print(f"  {name}: {scores.mean():.3f} ± {scores.std():.3f}")
```

## Expected Output

```
Dataset: 20,640 houses, 8 features
Features: ['MedInc', 'HouseAge', 'AveRooms', 'AveBedrms', 'Population', 'AveOccup', 'Latitude', 'Longitude']
Target: Median house price (in $100Ks)

Train: 16,512 houses | Test: 4,128 houses

Linear Regression:
  Train R²: 0.610
  Test R²:  0.596
  Gap:      0.014
  Test RMSE: $73,280

Random Forest:
  Train R²: 0.974
  Test R²:  0.809
  Gap:      0.165
  Test RMSE: $50,340

5-Fold Cross-Validation (R²):
  Linear Regression: 0.607 ± 0.028
  Random Forest: 0.806 ± 0.023
```

---

## What Just Happened — The Bias-Variance Tradeoff in Two Numbers

| Model | Train R² | Test R² | Gap | Diagnosis |
|:---|:---|:---|:---|:---|
| **Linear Regression** | 0.610 | 0.596 | 0.014 (small) | **Underfitting** — both scores are mediocre. The model is too simple. It assumes a straight-line relationship but house prices are more complex. |
| **Random Forest** | 0.974 | 0.809 | 0.165 (moderate) | **Mild overfitting** — train score is near-perfect but test score drops. The model memorized some training noise. Still performs much better than linear regression. |

**R² (pronounced "R-squared")** = "how much of the variation in house prices does the model explain?" R² of 0.80 means the model explains 80% of the price variation. The remaining 20% is noise or features the model does not have access to (school quality, renovation status, neighborhood vibe — not in this dataset).

**RMSE (Root Mean Squared Error, pronounced "R-M-S-E")** = average prediction error in the same units as the target. RMSE of $50,340 means the model's predictions are off by about $50K on average. For houses worth $100K-$500K, that is reasonable. For a $1M house, it is excellent. Context matters.

**The gap** is the thermometer:
- Linear Regression gap = 0.014 → consistent but weak (bias problem)
- Random Forest gap = 0.165 → stronger but slightly overfit (variance problem)

---

## Adding SHAP — Explaining WHY

```python
# === STEP 6: Explain a prediction ===
# pip install shap
import shap

# SHAP explains the Random Forest's predictions
explainer = shap.TreeExplainer(rf)
shap_values = explainer.shap_values(X_test[:100])  # Explain first 100 test predictions

# Summary plot: which features matter most across all predictions?
shap.summary_plot(shap_values, X_test[:100], feature_names=feature_names)

# Single prediction explanation
print(f"\nHouse #0: Predicted price = ${rf.predict(X_test[:1])[0] * 100_000:,.0f}")
print(f"          Actual price    = ${y_test[0] * 100_000:,.0f}")
shap.force_plot(explainer.expected_value, shap_values[0], X_test[0], feature_names=feature_names)
```

**What SHAP shows:**

The summary plot reveals: **MedInc (Median Income, pronounced "med-ink")** is the strongest predictor — neighborhoods with higher income have higher prices. This is not surprising. But SHAP also shows that **Latitude** and **Longitude** matter — coastal California is more expensive than inland. And **AveOccup (Average Occupancy, pronounced "ave-OCK-yup")** has a non-linear effect — overcrowded housing depresses prices, but so does very low occupancy (vacant buildings).

The force plot for a single house shows: "this house was predicted at $280K because MedInc pushed the price up by $90K (wealthy neighborhood) but AveOccup pushed it down by $30K (overcrowded block)."

That is actionable. The stakeholder can ask: "show me all houses where AveOccup pushed the price down more than $50K" — and investigate those neighborhoods for investment or intervention.

---

## What This Hello World Proved

| ML Lifecycle Step | What We Did | What We Saw |
|:---|:---|:---|
| Data Collection | `fetch_california_housing()` | 20,640 houses, 8 features |
| Data Preparation | `train_test_split()` | 80/20 split |
| Model Training | Trained Linear Regression + Random Forest | Two models, same data |
| Evaluation | R², RMSE, cross-validation | Bias-variance tradeoff visible: LR underfits, RF slightly overfits |
| Explainability | SHAP summary + force plot | MedInc drives prices, Latitude/Longitude capture location premium |
| Experiment comparison | Two models side by side | Random Forest wins on R² but overfits more — the tradeoff is real |

---

## Try It Yourself

1. **Run the code** in the [ML Fundamentals notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/ML_Fundamentals.ipynb) — Section 9 has the full pipeline with visualizations
2. **Change something:**
   - Add `max_depth=5` to the Random Forest: `RandomForestRegressor(n_estimators=100, max_depth=5)`. Does the train-test gap shrink? (Hint: yes — limiting depth is regularization)
   - Try `n_estimators=10` instead of 100. Does performance drop? By how much?
3. **Apply to P1:** The bank churn dataset has 10 features, not 8. It is classification (churn/no-churn), not regression (price). The metrics change (precision/recall instead of R²/RMSE). But the lifecycle is identical — load, split, train, evaluate, explain, track.

---

## What Comes Next

| Question | Where to Find the Answer |
|:---|:---|
| How do I choose precision vs recall vs F1? | [04 — How It Works](04_How_It_Works.md) — evaluation metrics deep dive |
| How do I prevent overfitting systematically? | [05 — Decisions](05_Building_It.md) — regularization, feature selection, hyperparameter tuning |
| How do Tesla, Google, and banks use ML in production? | [06 — Real World](06_Production_Patterns.md) — production case studies |
| How do I deploy a model to serve 10,000 users? | [07 — System Design](07_System_Design.md) — serving, scaling, cloud |
| What if someone games the model? | [08 — Security & Governance](08_Quality_Security_Governance.md) — adversarial inputs, bias, compliance |
| How do I know if my deployed model is still working? | [09 — Observability](09_Observability_Troubleshooting_Troubleshooting.md) — monitoring, drift, debugging |
| Give me the one-page decision reference | [10 — Checklist](10_Decision_Guide.md) |

---

**Hands-on notebook:** [ML Fundamentals Notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/ML_Fundamentals.ipynb) — the full pipeline with visualizations, diagnostics, and experiment tracking.

**Project:** [P1 — ML Predictor](https://github.com/sunilmogadati/systems-in-production/blob/main/implementation/projects/P1_ML_Predictor.md) — Bank churn prediction. Apply everything from this material to a real business problem.
