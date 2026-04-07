# Deep Learning — Security and Governance

**Adversarial attacks, data privacy, bias, compliance, and building AI systems that are safe to deploy.**

---

## Why This Chapter Exists

A model that is 99% accurate but can be tricked by adding invisible noise to an image is not production-ready. A model trained on biased data that denies loans disproportionately to one demographic is a lawsuit. A model deployed in healthcare without regulatory approval is illegal.

Security and governance are not afterthoughts. They are architectural requirements — as fundamental as latency and accuracy. A CTO evaluating a system asks about accuracy. A CISO (Chief Information Security Officer, pronounced "SEE-so") asks about attack surface. A compliance officer asks about audit trail. The system must satisfy all three.

---

## Adversarial Attacks — When Models Are Fooled

### What Are Adversarial Examples?

Small, carefully crafted changes to the input that cause the model to make a confident wrong prediction. To a human, the modified input looks identical to the original. To the model, it looks like a completely different class.

**The classic example:** A stop sign with tiny stickers placed in specific locations. To a human driver, it is obviously a stop sign. To a CNN image classifier, it becomes a speed limit sign — with 95% confidence. In an autonomous vehicle, this is life-threatening.

**Why this happens:** Neural networks learn to rely on features that are statistically useful but not semantically meaningful. A model might classify a panda based on a subtle texture pattern rather than the shape of the animal. An adversary exploits those fragile features by nudging them — imperceptibly to humans, decisively to the model.

### Types of Adversarial Attacks

| Attack | What It Does | Risk Level |
|:---|:---|:---|
| **Evasion** | Modify input at inference time to get a wrong prediction | High — anyone with access to the API can try |
| **Poisoning** | Inject malicious data into the training set to corrupt the model | Critical — subtle, hard to detect, affects all future predictions |
| **Model extraction** | Query the model thousands of times to reverse-engineer its behavior | Medium — enables the attacker to find vulnerabilities offline |
| **Membership inference** | Determine whether a specific data point was in the training set | Medium — privacy violation (was this patient's X-ray used for training?) |

### Defenses

| Defense | What It Does | Tradeoff |
|:---|:---|:---|
| **Adversarial training** | Include adversarial examples in the training data so the model learns to handle them | Slower training, slight accuracy drop on clean inputs |
| **Input validation** | Check inputs for anomalies before passing to the model (unusual pixel distributions, out-of-range values) | May reject legitimate edge cases |
| **Ensemble models** | Use multiple models with different architectures. If they disagree, flag the input for review. | Higher compute cost, more complexity |
| **Gradient masking** | Make it harder for attackers to compute gradients (which they need to craft adversarial inputs) | Arms race — sophisticated attackers can work around this |
| **Certified defenses** | Mathematical guarantees that small input perturbations cannot change the prediction | Limited to small perturbation budgets; does not scale to large modifications |

> **The reality:** No defense is perfect. Adversarial robustness is an active research area. For production, the pragmatic approach is: (1) adversarial training for known attack types, (2) input validation as a first line of defense, (3) monitoring for anomalous prediction patterns, (4) human-in-the-loop for high-stakes decisions.

---

## Data Privacy

### The Problem

Deep learning models are trained on data. That data often contains personal information — medical records, faces, financial transactions, browsing history, voice recordings. The model may inadvertently memorize specific training examples, making it possible to extract private information from the model itself.

### Regulatory Landscape

| Regulation | Pronounced | Region | Key Requirements |
|:---|:---|:---|:---|
| **GDPR** | "G-D-P-R" (General Data Protection Regulation) | European Union | Right to be forgotten, consent for data use, data minimization, explainability of automated decisions |
| **HIPAA** | "HIP-uh" (Health Insurance Portability and Accountability Act) | United States (healthcare) | Protected Health Information (PHI) must be encrypted, access-controlled, and auditable |
| **CCPA** | "C-C-P-A" (California Consumer Privacy Act) | California, United States | Right to know what data is collected, right to delete, opt-out of data sales |
| **EU AI Act** | — | European Union | Risk-based classification of AI systems. High-risk systems (medical, hiring, law enforcement) face strict requirements: conformity assessments, transparency, human oversight. |

### Techniques for Privacy-Preserving ML

| Technique | What It Does | When to Use |
|:---|:---|:---|
| **Data anonymization** | Remove or mask personally identifiable information (PII — Personally Identifiable Information) before training | Always. Minimum baseline. |
| **Differential privacy** | Add calibrated noise to the training process so the model cannot memorize individual examples | When training on sensitive data (medical, financial). Reduces accuracy slightly. |
| **Federated learning** | Train the model on distributed devices (phones, hospitals) without centralizing the data. Only gradients (not data) are shared. | When data cannot leave the device or institution (healthcare, banking). |
| **Secure enclaves** | Process sensitive data inside hardware-encrypted environments (TEE — Trusted Execution Environment) | When regulatory requirements demand that data is never exposed, even during processing. |

---

## Bias and Fairness

### The Problem

A model trained on biased data produces biased predictions. If historical hiring data shows that men were hired more often (because of past bias, not qualification), a model trained on that data will learn to prefer male candidates — perpetuating and automating the bias at scale.

This is not a hypothetical. Amazon built a resume screening model that systematically penalized resumes containing the word "women's" (as in "women's chess club captain"). The model was scrapped before deployment, but it took years of work to discover the bias.

### Types of Bias

| Type | What It Is | Example |
|:---|:---|:---|
| **Training data bias** | The data reflects historical or societal bias | Loan approval data that reflects decades of discriminatory lending practices |
| **Representation bias** | Some groups are underrepresented in the training data | A facial recognition model trained mostly on lighter-skinned faces that fails on darker-skinned faces |
| **Measurement bias** | The features used as proxies correlate with protected attributes | Using zip code as a feature — it correlates with race due to residential segregation |
| **Aggregation bias** | Treating all subgroups as one population when they have different patterns | A medical model trained on data from young adults that performs poorly on elderly patients |

### Mitigation Strategies

| Strategy | When to Apply | What It Does |
|:---|:---|:---|
| **Balanced training data** | Before training | Ensure proportional representation of all relevant subgroups |
| **Fairness metrics** | During evaluation | Measure accuracy, false positive rate, and false negative rate PER subgroup. Overall accuracy can hide subgroup disparities. |
| **Bias auditing** | Before deployment | Independent review of model predictions across demographics. Required by EU AI Act for high-risk systems. |
| **Counterfactual testing** | Before deployment | Change the protected attribute (gender, race) in the input and check if the prediction changes. If it does, the model is using that attribute (directly or indirectly). |
| **Model cards** | At deployment | A standardized document describing the model's intended use, training data, evaluation results, known limitations, and fairness assessments. Proposed by Google in 2018, increasingly standard. |

---

## Governance Framework — NIST AI RMF

**NIST AI RMF (National Institute of Standards and Technology, AI Risk Management Framework, pronounced "nist A-I R-M-F")** is the US government's framework for managing AI risk. It is not a law — it is a voluntary framework. But it is increasingly referenced in procurement requirements, audits, and regulatory guidance.

### The Four Functions

| Function | What It Covers | Plain English |
|:---|:---|:---|
| **GOVERN** | Policies, roles, culture of responsible AI | "Who is responsible for AI risk? What are the rules?" |
| **MAP** | Identify and categorize AI risks in context | "What could go wrong with THIS specific system in THIS specific use case?" |
| **MEASURE** | Quantify and track risks with metrics | "How do we know if the risk is getting better or worse?" |
| **MANAGE** | Prioritize, respond to, and communicate risks | "What do we do about the risks we found?" |

### Practical Application

For a deep learning deployment, NIST AI RMF translates to:

| NIST Function | What the Engineer Does |
|:---|:---|
| GOVERN | Document the model's intended use, stakeholders, and risk tolerance. Get sign-off from leadership. |
| MAP | Identify risks: adversarial attacks, bias, data drift, privacy, regulatory. Assess likelihood and impact for each. |
| MEASURE | Define metrics for each risk. Example: "Measure false negative rate per demographic subgroup every month." |
| MANAGE | Define mitigation for each risk. Example: "If subgroup FNR exceeds 5%, trigger model audit and potential retraining." |

This framework is covered hands-on in the Enterprise Integration project (P5), where a NIST AI RMF assessment is a deliverable.

---

## Security Checklist for Deployment

| Category | Check | Priority |
|:---|:---|:---|
| **Input validation** | Reject inputs outside expected ranges (image dimensions, file types, text length) | Critical |
| **Rate limiting** | Limit API requests per user to prevent model extraction and denial-of-service | Critical |
| **Authentication + Authorization** | Only authorized users/services can call the model API. RBAC (Role-Based Access Control, pronounced "R-back"). | Critical |
| **Audit trail** | Log every prediction: input, output, model version, timestamp, user | High |
| **Data encryption** | Encrypt data at rest (storage) and in transit (network) | High |
| **Model versioning** | Every deployed model traceable to its training data, code, and evaluation results | High |
| **Adversarial testing** | Test the model against known attack types before deployment | Medium |
| **Bias audit** | Evaluate fairness metrics per demographic subgroup | Medium (High for hiring, lending, healthcare) |
| **PII handling** | Ensure no PII in training data, or proper anonymization/consent | High (Critical for healthcare/finance) |
| **Content safety** | For generative models: filter inputs and outputs for harmful content | High |

---

**Next:** [09 — Observability](09_Observability_Troubleshooting.md) — Monitoring deployed models, detecting accuracy drift, latency dashboards, and the drill-down debugging method for production ML systems.
