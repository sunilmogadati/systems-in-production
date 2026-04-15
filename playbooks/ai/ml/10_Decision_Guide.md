# Machine Learning Fundamentals — Decision Guide

**Quick-reference cards for every ML decision. Print this. Bookmark this. Use this before every project.**

---

## Decision 1: Do I Need ML?

Not every problem needs ML. Rules are cheaper, faster, more interpretable, and easier to maintain. ML is justified when rules break down.

| Signal | Use Rules | Use ML |
|:---|:---|:---|
| **Pattern complexity** | Simple, can be expressed in 10-20 if/else statements | Complex, non-linear, involves interactions between many variables |
| **Pattern stability** | Pattern does not change over time | Pattern shifts — what predicted escalation last quarter may not predict it this quarter |
| **Maintainability** | Rules are manageable — a human can audit the full ruleset | Rules have grown to hundreds or thousands, nobody fully understands them, they contradict each other |
| **Data availability** | Small dataset (<500 rows) or no labeled data | Thousands of labeled examples available |
| **Decision speed** | Deterministic answer needed (compliance, safety-critical) | Probabilistic answer acceptable (ranking, recommendation, triage) |
| **Explainability** | Must explain exact reasoning step by step | "These features contributed most" is sufficient explanation |

### The Decision Flow

```mermaid
graph TD
    A["New prediction problem"] --> B{"Can 10-20 rules<br/>solve it?"}
    B -->|Yes| C["Use rules.<br/>Done."]
    B -->|No| D{"Do you have<br/>>1,000 labeled<br/>examples?"}
    D -->|No| E["Collect data first.<br/>Use rules in the meantime."]
    D -->|Yes| F{"Does the pattern<br/>change over time?"}
    F -->|No| G["ML is optional.<br/>Rules may still be<br/>simpler and cheaper."]
    F -->|Yes| H["ML is justified.<br/>Start with a baseline."]
```

**Production Diagnostic System example:** Could rules predict incident escalation?

A first attempt: "If error rate > 5% AND deployment in last 2 hours AND off-hours, then flag as high risk." This catches some escalations. But it misses the interaction effects — a rising error TREND is more predictive than a threshold. The number of related alerts matters, but only in combination with service tier. After writing 40+ rules with diminishing returns, ML is justified.

---

## Decision 2: Algorithm Selection Guide

| Algorithm | Best For | Data Size | Interpretability | Speed | When to Choose |
|:---|:---|:---|:---|:---|:---|
| **Linear Regression** | Predicting a continuous number | Any | High — coefficients show feature impact directly | Very fast | Baseline for regression. Use when relationships are roughly linear. |
| **Logistic Regression** | Binary classification | Any | High — coefficients show feature impact directly | Very fast | Baseline for classification. Always start here. |
| **Ridge / Lasso** | Linear models with regularization | Any | High | Very fast | When Logistic/Linear Regression overfits or when feature selection is needed (Lasso) |
| **Random Forest** | Mixed feature types, non-linear relationships | Medium to large | Medium — feature importance available, individual trees are interpretable | Fast (training), fast (inference) | Good default when the baseline is not enough. Robust to outliers and missing values. |
| **GradientBoosting (XGBoost / LightGBM)** | Tabular data — classification or regression | Medium to large | Medium — SHAP provides per-prediction explanations | Fast (inference), moderate (training) | Best tabular performance in most benchmarks. The go-to for production systems on structured data. |
| **Neural Network (MLP)** | Tabular data with very large datasets | Large (>100K rows) | Low — requires SHAP or LIME for explanation | Slower (training), fast (inference) | Only when tree-based models plateau AND you have enough data to justify the complexity |
| **CNN (Convolutional Neural Network)** | Images, spatial data | Large + image data | Low | Slow (training), moderate (inference) | Image classification, object detection, medical imaging |
| **Transformer / LLM (Large Language Model)** | Text, sequence data | Very large | Low | Slow | NLP (Natural Language Processing), text classification, generation, summarization |

### Comprehensive Algorithm Reference

| Algorithm | Type | Best For | Interpretable? | Handles Missing? | Speed | When to Use |
|:---|:---|:---|:---|:---|:---|:---|
| **Linear Regression** | Regression | Continuous target, linear relationships | Yes | No | Fast | Baseline for regression. Simple relationships. Coefficients directly show feature impact. |
| **Logistic Regression** | Classification | Binary or multi-class, linear decision boundaries | Yes | No | Fast | Baseline for classification. Always start here. Interpretability required by stakeholders or regulators. |
| **Ridge / Lasso / ElasticNet** | Regression | Regularized linear models, feature selection (Lasso) | Yes | No | Fast | Many features, multicollinearity. Lasso for automatic feature selection. ElasticNet when both properties are needed. |
| **Decision Tree** | Both | Simple rule-based decisions, visualization | Yes | Yes | Fast | When explainability is the top priority. Small datasets. Generating human-readable rules. |
| **Random Forest** | Both | General-purpose tabular data, mixed feature types | Medium | Yes | Medium | Default choice when the baseline is not enough. Robust to outliers and noise. Low tuning effort. |
| **GradientBoosting / XGBoost** | Both | Best tabular performance, structured data | Low | Yes | Medium | Production systems on structured data. When you need the best accuracy and can invest in tuning. |
| **LightGBM** | Both | Large datasets, fast training | Low | Yes | Fast | Millions of rows. Native categorical feature handling. Fastest boosting library. |
| **CatBoost** | Both | Categorical-heavy data | Low | Yes | Medium | Many categorical features. Less preprocessing required than XGBoost or LightGBM for categoricals. |
| **SVM (Support Vector Machine)** | Classification | Small-to-medium data, clear class margins | Medium | No | Slow on large data | Text classification on small datasets. When you have fewer than 10K rows and clear separation. |
| **Naive Bayes** | Classification | Text, high-dimensional sparse data | Yes | Yes (by design) | Very fast | Spam filters, text classification, document categorization. Extremely fast training and inference. |
| **KNN (K-Nearest Neighbors)** | Both | Similarity-based predictions, no training phase | Yes | No | Slow on large data | Recommendation systems, small datasets. No training needed — predictions are based on nearest neighbors at inference time. |
| **K-Means** | Clustering | Group similar items, known number of groups | Medium | No | Fast | Customer segmentation, incident grouping, any case where you want K distinct groups. |
| **DBSCAN** | Clustering | Irregular-shaped clusters, outlier detection | Medium | No | Medium | Geospatial data, noise-heavy data. When the number of clusters is unknown. |
| **Hierarchical Clustering** | Clustering | Exploring cluster hierarchy, small datasets | Medium | No | Slow on large data | Exploratory analysis. When you want to visualize how groups relate to each other (dendrogram). |
| **PCA** | Dim. Reduction | Reduce features, remove noise, speed up training | Medium | No | Fast | Preprocessing step when you have many features. Visualization of high-dimensional data. |
| **t-SNE / UMAP** | Dim. Reduction | Visualize high-dimensional data in 2D | Low | No | Medium | Visualizing clusters, exploring embedding spaces. Not for model input — only for visualization. |
| **Isolation Forest** | Anomaly Detection | Find outliers without labels | Low | Yes | Fast | Production monitoring, fraud detection. No labeled anomalies needed. Scales to millions of points. |
| **LOF (Local Outlier Factor)** | Anomaly Detection | Local density anomalies | Low | No | Slow on large data | Fraud detection, manufacturing defects. When anomalies are defined by local context, not global statistics. |
| **Autoencoders** | Anomaly Detection | Complex, high-dimensional anomaly patterns | Low | No | Medium | Sensor streams, network traffic, image anomalies. When simpler methods do not capture the pattern. |
| **Neural Network (MLP)** | Both | Tabular data with very large datasets | Low | No | Slow (training) | Only when tree-based models plateau AND you have >100K rows to justify the complexity. |
| **CNN (Convolutional Neural Network)** | Classification | Images, spatial data | Low | No | Slow (training) | Image classification, object detection, medical imaging. |
| **Transformer / LLM** | Both | Text, sequence data | Low | No | Slow | NLP tasks: text classification, generation, summarization, question answering. |

### The Algorithm Decision Flowchart

The first question is always: **what kind of problem do you have?**

```mermaid
graph TD
    A["What kind of<br/>problem?"] --> B{"Predict a<br/>number?"}
    A --> C{"Predict a<br/>category?"}
    A --> D{"Find groups<br/>or patterns?"}
    A --> E{"Find unusual<br/>items?"}
    A --> F{"Too many<br/>features?"}

    B -->|"Regression"| B1{"< 1,000 rows<br/>or need<br/>interpretability?"}
    B1 -->|Yes| B2["Linear Regression<br/>(or Ridge/Lasso)"]
    B1 -->|No| B3{"Need best<br/>performance?"}
    B3 -->|Yes| B4["XGBoost / LightGBM"]
    B3 -->|"Good enough<br/>+ robust"| B5["Random Forest"]

    C -->|"Classification"| C1{"< 1,000 rows<br/>or need<br/>interpretability?"}
    C1 -->|Yes| C2["Logistic Regression"]
    C1 -->|No| C3{"Text or<br/>sparse data?"}
    C3 -->|Yes| C4["Naive Bayes<br/>(fast baseline)<br/>then Transformer"]
    C3 -->|"Tabular"| C5{"Need best<br/>performance?"}
    C5 -->|Yes| C6["XGBoost / LightGBM"]
    C5 -->|"Good enough<br/>+ robust"| C7["Random Forest"]

    D -->|"Clustering"| D1{"Know number<br/>of groups?"}
    D1 -->|Yes| D2["K-Means"]
    D1 -->|No| D3{"Data has noise<br/>or irregular<br/>shapes?"}
    D3 -->|Yes| D4["DBSCAN"]
    D3 -->|No| D5["Hierarchical<br/>Clustering"]

    E -->|"Anomaly Detection"| E1{"Have labeled<br/>anomalies?"}
    E1 -->|Yes| E2["Supervised classification<br/>(Logistic Regression,<br/>Random Forest, XGBoost)"]
    E1 -->|No| E3{"Data is<br/>high-dimensional<br/>or complex?"}
    E3 -->|"Simple numeric"| E4["Isolation Forest"]
    E3 -->|"Complex patterns"| E5["Autoencoder or LOF"]

    F -->|"Dim. Reduction"| F1{"For model<br/>input?"}
    F1 -->|Yes| F2["PCA"]
    F1 -->|"For visualization"| F3["t-SNE / UMAP"]
```

### The Algorithm Selection Principle

**Start simple, add complexity only when justified.** Within each problem type:

1. **Regression:** Linear Regression first. Then Ridge/Lasso if overfitting or many features. Then Random Forest. Then XGBoost/LightGBM.
2. **Classification:** Logistic Regression first. Then Random Forest. Then XGBoost/LightGBM. Naive Bayes for text baselines.
3. **Clustering:** K-Means first (if you know K). DBSCAN if you do not. Hierarchical for small exploratory datasets.
4. **Anomaly Detection:** Isolation Forest first. LOF if local context matters. Autoencoders for complex temporal/spatial patterns.
5. **Dimensionality Reduction:** PCA for preprocessing. t-SNE/UMAP for visualization only.

Each step up adds training time, tuning effort, and interpretability cost. Only move to the next level when the simpler model does not meet the success metric defined in the Architect Decision Checklist.

### The Original Algorithm Decision Tree (By Data Type)

```mermaid
graph TD
    A["What is the data type?"] --> B{"Structured /<br/>tabular?"}
    A --> C{"Images?"}
    A --> D{"Text?"}
    A --> E{"Time series?"}

    B -->|Yes| F{"< 1,000 rows?"}
    F -->|Yes| G["Logistic / Linear<br/>Regression"]
    F -->|No| H{"Need best<br/>performance?"}
    H -->|Yes| I["GradientBoosting<br/>(LightGBM / XGBoost)"]
    H -->|"Need simplicity<br/>+ interpretability"| J["Random Forest<br/>or Logistic Regression"]

    C -->|Yes| K["CNN or<br/>Vision Transformer"]
    D -->|Yes| L["Transformer /<br/>Fine-tuned LLM"]
    E -->|Yes| M["GradientBoosting<br/>with time features<br/>OR LSTM / Transformer"]
```

---

## Decision 3: Metric Selection Guide

| Metric | When to Use | Business Example | Plain English |
|:---|:---|:---|:---|
| **Accuracy** | Balanced classes (close to 50/50) AND both error types cost about the same | Classifying support tickets as "billing" vs "technical" — misclassifying either way has similar cost | "How often is the model right overall?" |
| **Precision** | False positives are expensive or dangerous | Spam filter — sending a real customer email to spam loses business | "Of the items flagged, how many are actually positive?" |
| **Recall** | False negatives are expensive or dangerous | Cancer screening, fraud detection, incident escalation prediction | "Of all the real positives, how many did the model catch?" |
| **F1** | Both false positives and false negatives are roughly equally costly | Hiring model — rejecting a good candidate and accepting a bad one both have consequences | "Balanced score between precision and recall" |
| **ROC-AUC (Receiver Operating Characteristic — Area Under Curve)** | Comparing models across all thresholds; balanced or moderately imbalanced classes | Choosing between Model A and Model B — which one separates classes better? | "Does the model rank positives higher than negatives?" |
| **PR-AUC (Precision-Recall AUC)** | Heavily imbalanced classes (>95% one class) where ROC-AUC can be misleading | Rare event detection — only 1% of incidents escalate | "How well does the model find the rare positive class?" |
| **R-squared (R2)** | Regression — predicting a continuous number | House prices, revenue forecasting, latency prediction | "What percentage of the variation does the model explain?" |
| **RMSE (Root Mean Squared Error)** | Regression — need error in the same units as the target | House price prediction — RMSE of $50K means "off by about $50K on average" | "Average prediction error, sensitive to large mistakes" |
| **MAE (Mean Absolute Error)** | Regression — want robustness to outliers | Delivery time prediction — one extreme outlier should not dominate the metric | "Average prediction error, not sensitive to outliers" |

### The Metric Decision Flow

```mermaid
graph TD
    A["What type of<br/>prediction?"] --> B{"Classification<br/>or regression?"}
    B -->|Classification| C{"Classes<br/>balanced?"}
    B -->|Regression| D["Start with R2 + RMSE.<br/>Add MAE if outliers matter."]
    C -->|"Yes (close to 50/50)"| E["F1 or Accuracy"]
    C -->|"No (imbalanced)"| F{"Which mistake<br/>costs more?"}
    F -->|"Missing a positive<br/>(false negative)"| G["Recall"]
    F -->|"False alarm<br/>(false positive)"| H["Precision"]
    F -->|"Both roughly equal"| I["F1 or PR-AUC"]
```

---

## Decision 4: Feature Engineering Checklist

Run through this checklist for every ML project:

| Category | Check | Done? |
|:---|:---|:---|
| **Temporal** | Extract: hour of day, day of week, is weekend, is holiday, month, quarter | |
| **Aggregations** | Counts, sums, averages over time windows (last hour, last day, last week) | |
| **Trends** | Slopes, rates of change over sliding windows | |
| **Categoricals** | One-hot encode or label encode categorical features | |
| **Missing values** | Strategy decided per feature: impute, flag, drop, or fail | |
| **Scaling** | Numeric features scaled (StandardScaler or MinMaxScaler) if using linear models or distance-based algorithms | |
| **Derived** | Domain-specific combinations: ratios, differences, interactions | |
| **Text** | Length, keyword flags, TF-IDF (Term Frequency-Inverse Document Frequency) or embeddings if text features exist | |
| **Leakage check** | Every feature is available at prediction time. No future information. | |
| **Correlation check** | Identify highly correlated features (>0.95). Remove or combine to reduce redundancy. | |

---

## Decision 5: Production Readiness Checklist

Before deploying any model to production:

| Category | Requirement | Pass/Fail |
|:---|:---|:---|
| **Performance** | Primary metric (recall, precision, F1, R2) meets the threshold defined in the Architect Checklist | |
| **Performance** | Train-test gap is <10% (no severe overfitting) | |
| **Performance** | Cross-validation scores are stable (low standard deviation across folds) | |
| **Performance** | Holdout set results are within 3 points of development results | |
| **Subgroups** | Performance checked per relevant subgroup (service tier, region, time of day). No subgroup below minimum threshold. | |
| **Explainability** | SHAP summary plot reviewed — top features make domain sense | |
| **Explainability** | SHAP reviewed for 10+ individual predictions — explanations are coherent | |
| **Experiment tracking** | Training run logged in MLflow (parameters, metrics, artifacts, code version) | |
| **Model card** | Completed: purpose, training data, performance, limitations, intended use, out-of-scope use | |
| **Serving** | API endpoint tested: correct input/output format, handles edge cases (missing features, out-of-range values) | |
| **Serving** | Latency meets requirement (P99 under threshold) | |
| **Serving** | Fallback behavior defined: what happens when the model API is unavailable? | |
| **Monitoring** | Prediction logging enabled | |
| **Monitoring** | Drift detection configured (feature distribution + prediction distribution) | |
| **Monitoring** | Accuracy tracking configured (with ground truth join when available) | |
| **Monitoring** | Alerts configured: accuracy drop, drift, latency spike, error rate | |
| **Governance** | Model registered in model registry with version tag | |
| **Governance** | Approval workflow completed (automated gate + human review) | |
| **Governance** | Rollback plan documented: how to revert to the previous model version | |

---

## Decision 6: The 10-Step System Design Framework — Quick Reference

For full application, see [07 — System Design](07_System_Design.md).

| Step | Key Question | Common Mistake |
|:---|:---|:---|
| **1. Requirements** | What is the latency, throughput, and accuracy target? What is the cost of being wrong? | Jumping to model selection without clarifying requirements |
| **2. Data Pipeline** | Where does the data come from? How fresh? How reliable? | Assuming data is clean and available. It never is. |
| **3. Feature Engineering** | What features capture the signal? Are they available at prediction time? | Using raw data without transformation. Including future information (leakage). |
| **4. Model Selection** | Which algorithm fits the data type, size, and interpretability needs? | Choosing the most complex model first instead of starting with a baseline |
| **5. Training Infrastructure** | CPU or GPU? Single machine or distributed? How often retrain? | Over-provisioning — using GPUs for a 10,000-row tabular problem |
| **6. Serving Infrastructure** | Real-time API, batch, or embedded? What is the fallback? | No fallback — model API goes down, entire system fails |
| **7. Monitoring** | What metrics to track? What alerts to set? | Monitoring latency but not accuracy. The model serves fast but wrong. |
| **8. Scaling** | What is the bottleneck as volume grows? | Premature optimization — scaling before there is a scaling problem |
| **9. Governance** | Who approves model changes? How to audit? How to roll back? | No model registry — "which model is running?" becomes a mystery |
| **10. Iteration** | How does the system improve over time? What triggers retraining? | Ship and forget — no plan for the model to evolve with the data |

---

## Quick Reference — All Decision Tables on One Page

### Do I Need ML?
Rules first. ML when rules fail, data is plentiful, and patterns shift.

### Which Algorithm?
Tabular data: Logistic Regression (baseline) then GradientBoosting (production). Images: CNN. Text: Transformer. Start simple.

### Which Metric?
Ask: "What is the cost of a false positive vs a false negative?" If FN is 10x worse, optimize recall. If FP is 10x worse, optimize precision. Roughly equal: F1.

### Am I Ready for Production?
Model passes metrics. Subgroups checked. SHAP makes sense. Experiment tracked. Monitoring configured. Rollback planned.

---

## Quick Links

| Chapter | Title |
|:---|:---|
| [01](01_Why.md) | Why This Matters |
| [02](02_Concepts.md) | Concepts and Mental Models |
| [03](03_Hello_World.md) | Hello World |
| [04](04_How_It_Works.md) | How It Works |
| [05](05_Building_It.md) | Building It |
| [06](06_Production_Patterns.md) | Production Patterns |
| [07](07_System_Design.md) | System Design |
| [08](08_Quality_Security_Governance.md) | Quality, Security, Governance |
| [09](09_Observability_Troubleshooting.md) | Observability and Troubleshooting |
| **[10](10_Decision_Guide.md)** | **Decision Guide** (this chapter) |

---

**Hands-on notebook:** [ML Fundamentals on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/ML_Fundamentals.ipynb) — the executable pipeline that these decision guides reference.

**Architecture reference:** [CSI Architecture](../../../systems/continuous-system-intelligence/architecture.md) — the system these decisions were applied to.

**Back to start:** [01 — Why This Matters](01_Why.md) — the story that frames everything.
