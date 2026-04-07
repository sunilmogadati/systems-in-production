# The System Shift

**Presented by Sunil Mogadati | Emergtech Business Solutions**

---

## SLIDE 1: Title

### The System Shift
**Why Every IT Role Is Changing, and What to Do About It**

Sunil Mogadati
Emergtech Business Solutions | Future of AI

> Speaker notes: Keep this up while people settle in. Don't read your resume. Just say: "I'm Sunil. I've spent 25+ years building and fixing complex software systems. Full-stack, integrations, databases (relational, NoSQL, MongoDB), message-driven systems, cloud, data pipelines, and now AI. Today I want to share what's actually changing and what it means for your career."

---

## SLIDE 2: My Lens

### "I fix systems that don't respond to more tools or more people."

25+ years building and operating complex software systems.
Now extending that into AI and data systems in production.

Ground truth leadership: from the codebase to the boardroom.

When delivery stalls, I go to where the truth lives (the code, the data, the system), and bring clarity to where decisions are made.

> Speaker notes: "I've worked across Java, Spring Boot, Node, Python, C/C++, Angular, React on the application side. Relational databases, NoSQL, MongoDB. Message-driven systems, Kafka. Microservices, Kubernetes, CI/CD. Unix/Linux modernizations and upgrades. AWS, GCP. Integrations across every kind of system you can imagine. And now AI and ML systems in production. But the real work has always been the same: understanding how these systems behave, where they break, and bringing that ground truth to the people making decisions. That's what I mean by ground truth leadership."

---

## SLIDE 3: The Old World

### How IT Used to Work

**Separate roles. Separate systems. Separate teams.**

| Role | Focus |
|---|---|
| Developer | Write code |
| QA | Test functionality |
| DevOps | Deploy + monitor |
| Data Engineer | Build pipelines |
| Business Analyst | Write requirements |
| Product Manager | Define what to build |
| Project Manager | Keep it on track |
| Networking/Infra | Connectivity + servers |

Each role had clear boundaries.
You could build an entire career inside one lane.

> Speaker notes: "Raise your hand if you identify with one of these roles. Now, how many of you have been asked to do something outside your lane in the last 6 months?" Pause. Let them feel it.

---

## SLIDE 4: The New World

### Everything Is Becoming One Connected System

**Before:**
Dev, QA, DevOps, Data, Business, PM (separate silos)

**Now:**
Code + Data + AI + Infra + Business Logic = One System (end-to-end)

AI is not replacing roles.
AI is collapsing the boundaries between them.

> Speaker notes: "This is the key shift. It's not that developers will disappear. It's that a developer who doesn't understand data, or a QA engineer who can't validate AI outputs, or a BA who can't think in system flows, or a PM who can't evaluate AI tradeoffs, they become less relevant. Not because they're bad at their job. Because the job itself changed."

---

## SLIDE 5: The Biggest Opportunity of Our Lifetime

### AI is not a threat. It is a lever.

What AI actually enables:
- Solve problems that were previously too expensive or too complex
- Move up the value chain: from repetitive execution to judgment and design
- Access resources and capabilities that used to be gated behind corporate budgets
- A farmer, a teacher, a nurse can now have the same analytical power as a Fortune 500 team

This is not about survival. This is about what becomes possible.

AI is a technology of freedom. It removes gatekeeping. It removes exploitation of access to privileged resources. It puts capability in the hands of people who could never afford it before.

**The question is not "will AI take my job?" The question is "what can I now do that I could not do before?"**

> Speaker notes: "I want to be clear about my position. I am 100% pro-AI. Not because it's trendy, but because of what it unlocks. Think about a small clinic in rural India that can now do diagnostic imaging. Think about a solo entrepreneur who can now build a data pipeline that used to require a team of 10. This is real. And the people in this room, whatever role you're in, you are in the best position to take advantage of this. Because you understand systems. Most people don't."

---

## SLIDE 6: New Roles Are Emerging

### Roles That Did Not Exist 3 Years Ago

| Role | What They Do |
|---|---|
| **AI Engineer** | Build AI-powered applications: RAG, agents, ML pipelines, AI integrations |
| **Forward-Deployed Engineer (FDE)** | Embed with clients to diagnose, build, and fix complex systems on-site |
| **ML Platform Engineer** | Build the infrastructure ML teams use: feature stores, GPU clusters, experiment tracking |
| **Prompt Engineer** | Design AI interactions, evaluation, prompt chains for production quality |
| **LLMOps Engineer** | Production lifecycle for LLM apps: cost, latency, prompt versioning, evaluation, guardrails |

**Real use cases being built right now:**
- Customer support systems that understand context (not keyword matching)
- Document processing that extracts structure from unstructured data
- Predictive systems for operations (call volume, demand, failure prediction)
- Internal tools that surface insights from company data
- Workflow automation that replaces repetitive manual processes

**How to get there:**
You don't start from scratch. You extend from where you are.
Dev + ML fundamentals + system design thinking = AI Engineer.
Strong system operator + diagnostic ability + client-facing = FDE.

> Speaker notes: "These roles didn't exist 3 years ago. Now they're some of the fastest growing and highest paying in tech. And here's what's interesting: none of them require a PhD. They're for people who know how to build production software and can learn the AI/ML layer. The FDE role is particularly interesting. It's modeled after what companies like Palantir pioneered: you embed with the client, you understand their system, you diagnose what's broken, and you build the fix. That's not a junior role. That's the kind of thing people in this room, with your experience, are uniquely positioned for."

---

## SLIDE 7: What Actually Matters Now

### The Skills That Let You Thrive

It's not about which tools you know.
It's about how you think about systems.

**Most people know tools.**
What creates real value:
- Understanding how systems evolve under pressure
- Knowing where integrations break
- Seeing how production actually behaves (vs how it was designed)
- Recognizing when the problem is the process, not the technology

**Your job title matters less.
Your ability to understand the full system matters more.**

> Speaker notes: "I've worked across Java, Spring Boot, Node, Python, databases of every kind, relational, NoSQL, MongoDB, message-driven architectures, Kafka, AWS, GCP, data pipelines. But the real work has always been understanding how these systems behave and fail in production. That's what's transferable. That's what AI can't replace. And that's what will let you thrive in this new world."

---

## SLIDE 8: A Quick Tour

### One System, Six Components

Imagine a production diagnostic system that:
- Collects data from databases, logs, documents, and APIs
- Monitors for anomalies before they become outages
- Diagnoses root causes by correlating signals across systems
- Creates tickets with findings and recommended actions

Each demo you're about to see is a component of that system.
All run on Google Colab (no setup, no cost). All on GitHub.

> Speaker notes: "I'm going to show you a system I'm building. It collects data from databases, application logs, deployment records, runbooks, and APIs. It monitors, diagnoses, and surfaces findings automatically. Each piece I'm about to show you is a component of that system. This is not 6 separate tutorials. This is one system, viewed through 6 lenses."

---

## SLIDE 9: Demo 1 - System Design (Star Schema)

### How Data Systems Are Actually Designed

**What you're seeing:** Architecture thinking rendered as documentation.

- Mermaid diagrams that render directly on GitHub
- Fact tables, dimension tables, surrogate keys
- Bronze (raw) to Silver (clean) to Gold (analytics-ready) pipeline

**Role in the diagnostic system:** This is how you model incidents, deployments, metrics, and services so they can be queried together. Without this, every query hits a mess.

> Speaker notes: "Before you can diagnose anything, your data has to be structured. This models incidents, deployments, metrics, and services into clean relationships. Without this, you're running queries against flat tables with 300 columns." 2-3 min max.

**Demo link:** [Star Schema Design on GitHub](https://github.com/sunilmogadati/systems-in-production/tree/main/playbooks/data/star-schema-design)

---

## SLIDE 10: Demo 2 - Machine Learning Pipeline

### When AI Makes Decisions, How Do You Know It's Right?

**What you're seeing:** A full ML pipeline: data prep, feature engineering, model training, evaluation.

Key moments:
- SHAP explainability: "WHY did the model decide this?"
- Cross-validation: "Can we trust these numbers?"
- Architect Decision Checklist: "What would you choose and why?"

**Role in the diagnostic system:** Can we predict which P3 incident will escalate to P1? Can we classify root causes automatically? The system learns from past incidents to flag the next one before it becomes an outage.

> Speaker notes: "This ML pipeline trains 3 models and compares them side by side. In our diagnostic system, this predicts which incidents will escalate. The SHAP chart shows WHY the model flagged something. That's not a black box, that's a transparent recommendation." 2-3 min.

**Demo link:** [ML Fundamentals on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_ML_Fundamentals.ipynb)

---

## SLIDE 11: Demo 3 - RAG (AI Answering From Your Data)

### Everyone Talks About Chatbots. This Is What's Underneath.

**What you're seeing:** Retrieval-Augmented Generation: making AI answer questions from YOUR documents.

Steps:
1. Ingest documents
2. Create embeddings (semantic search)
3. Retrieve relevant chunks
4. Generate answer using LLM (Large Language Model)

**Role in the diagnostic system:** An engineer debugging at 2 AM needs answers from runbooks, post-mortems, and docs. RAG makes that searchable by meaning, not keywords. Ask "what's the fix for connection pool exhaustion" and it finds the answer across all your documentation.

> Speaker notes: "In our diagnostic system, RAG is the knowledge layer. It ingests runbooks, post-mortems, architecture docs. When the system detects an anomaly, RAG retrieves relevant documentation and past fixes. No more searching Confluence at 2 AM." 2-3 min.

**Demo link:** [RAG from Scratch on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_RAG_from_Scratch.ipynb)

---

## SLIDE 12: Demo 4 - Cloud Data Pipeline (GCP)

### How Data Actually Moves, From Source to Insight

**What you're seeing:** A real data pipeline on Google Cloud Platform.

Source to GCS (Storage) to BigQuery (Warehouse) to Transformation to Analytics

- Bronze: raw data, untouched
- Silver: cleaned, deduped, timezone-fixed
- Gold: star schema, ready for queries and dashboards

**Role in the diagnostic system:** Logs, metrics, incidents, deployments arrive in different formats from different systems. This pipeline moves them from raw to clean to analysis-ready. This is the plumbing underneath everything else.

> Speaker notes: "In our diagnostic system, this is how data gets from source to usable. Logs come in as JSON, metrics as CSV, incidents from ticketing APIs. The pipeline cleans, deduplicates, fixes timezone issues, and structures it for analysis. Without this, nothing else works." 2-3 min.

**Demo link:** [GCP Pipeline on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/M08_Cloud_Data_Pipeline_Build.ipynb)

---

## SLIDE 13: Demo 5 - Deep Learning (Neural Networks)

### When AI Learns, What Does That Actually Look Like?

**What you're seeing:** A Convolutional Neural Network learning to recognize images.

- Training loop: epochs, loss curves, accuracy
- What the model "sees" at each layer
- Training diagnostics: overfitting, underfitting, learning rate

**Role in the diagnostic system:** Memory usage climbing over 5 days. Normal growth or a leak? A neural network trained on historical metrics detects anomalies that dashboards miss. Pattern recognition on time-series data.

> Speaker notes: "In our diagnostic system, deep learning watches the metrics. CPU, memory, latency, error rates. It learns what normal looks like for each service. When something drifts outside normal, it flags it. Remember that search-service memory leak in our dataset? It built up over 5 days before crashing. A trained model catches that on day 2." 2 min.

**Demo link:** [Deep Learning on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Deep_Learning_PyTorch.ipynb)

---

## SLIDE 14: Demo 6 - AI Agents (AI That Takes Actions)

### Beyond Answering Questions: AI That Does Things

**What you're seeing:** An AI agent that can:
- Use tools (search, calculate, look up data)
- Make multi-step decisions
- Chain actions together

**Role in the diagnostic system:** This is the orchestrator. The agent receives an alert. It queries the database for recent deployments. It checks logs for error patterns. It searches the runbook for known fixes. Then it creates a ticket: here's what happened, here's what's related, here's the likely root cause, here's the recommended action.

> Speaker notes: "This is where it all comes together. The agent doesn't just answer questions. It takes action. It receives an alert, queries the DB, reads the logs, checks the runbook via RAG, correlates with recent deployments, and creates a ticket with findings. That's not a chatbot. That's a diagnostic system. And it needed every skill in this room to build: data engineering, software development, AI/ML, DevOps, QA to validate it, PM to scope it." 2 min.

**Demo link:** [AI Agents on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Agents.ipynb)

---

## SLIDE 15: The Common Thread

### One System. Six Components. Every Role Needed.

What you just saw:

Pipeline ingests logs, metrics, incidents, deployments (DE + DevOps)
Star schema structures it for analysis (Data Modeling + BA)
ML predicts which incidents will escalate (AI/ML Engineer)
Deep learning detects anomalies in metrics (AI/ML Engineer)
RAG retrieves answers from runbooks and docs (AI Engineer)
Agent orchestrates all of it and creates tickets (AI Engineer + Dev)

Every role in this room contributed to one system.

**That's what "the boundaries are collapsing" actually looks like in practice.**

> Speaker notes: "Data pipeline, data model, ML, deep learning, RAG, agents. Six components, one system. And it took every skill in this room. Data engineering to build the pipeline. A developer to build the agent. QA to validate the outputs. A PM to define what 'good' looks like. DevOps to keep it running. This is not about any one role. It's about understanding how they connect."

---

## SLIDE 16: How to Learn Any of This

### A Methodology That Works

When approaching any new technology or system, follow this flow:

1. **Start with WHY** - What human problem does this solve? What becomes possible?
2. **Concept** - Understand it in plain language, with real-world analogies
3. **Hello World** - See it work in 10 lines of code. Get a result first.
4. **How it works** - Now understand what happened under the hood
5. **Decisions** - What are the tradeoffs? What would you choose and why?
6. **Real world** - How do companies actually use this in production?
7. **Build something real** - Production grade, real users, real problems
8. **Deploy** - From laptop to cloud
9. **Monitor and maintain** - Observability, troubleshooting, continuous improvement

**Every notebook and concept guide in the GitHub repo follows this structure.**

Don't just learn tools. Build mental models.

> Speaker notes: "This is the methodology I use for everything I build and teach. Start with why it matters for real people, not just the technology. Get a result fast (hello world). Then go deep. Every piece of content in the GitHub repo follows this flow. You can use it yourself for anything you're learning."

---

## SLIDE 17: How to Design Any AI System

### The 10-Step AI System Design Framework

Use this for interviews, architecture reviews, or any new AI project:

```
 1. Requirements    - Latency, throughput, accuracy, cost, compliance
 2. Data pipeline   - Sources, preprocessing, storage, quality validation
 3. Model selection - Pre-trained vs fine-tuned vs from scratch
 4. Training        - Hardware, frequency, experiment tracking
 5. Serving         - Real-time vs batch vs streaming, API design
 6. Monitoring      - Accuracy, latency, drift, fairness, business metrics
 7. Scaling         - Auto-scaling, GPU management, cost model
 8. Security        - Input validation, auth, adversarial defense
 9. Governance      - Versioning, bias audit, regulatory, model cards
10. Iteration       - Retraining triggers, A/B testing, improvement
```

**Example: "Design a customer support AI system"**
Step 1: Must respond in <2 seconds, handle 1000 concurrent users
Step 2: Ingest support docs + ticket history, refresh weekly
Step 3: RAG with pre-trained embeddings + hosted LLM
Step 5: Real-time serving via API
Step 6: Track answer quality, escalation rate, cost per query
...and so on through all 10 steps.

> Speaker notes: "Memorize these 10 steps. In an interview, write them on the whiteboard first, then work through each one. Even if you run out of time, the interviewer sees that you know the full picture. But this isn't just for interviews. Use it for any real system you're building. The full framework with checklists is in the GitHub repo."

---

## SLIDE 18: What This Means for Your Career

### You Don't Need to Switch Roles. But You Cannot Stay Narrow.

"If your current role becomes less valuable, expand sideways."

| Your Role | Expand Into |
|---|---|
| Developer | Data flows + system design + AI integration |
| QA | System behavior validation + AI output testing |
| DevOps | System reliability across data + AI + services |
| Data Engineer | Features to AI to decisions (end-to-end) |
| Business Analyst | Business to system behavior (not just requirements) |
| Product Manager | AI capability evaluation + system tradeoffs |
| Project Manager | Cross-system delivery + AI timeline realities |
| Networking/Infra | Latency, scale, distributed AI workloads |

**You don't need to become an AI expert.
But you need to understand how AI fits into the system you're working on.**

> Speaker notes: "I'm not telling you to switch careers. I'm telling you the walls between these roles are dissolving. The people who thrive are the ones who can see across the boundary, not just inside their own box. And the good news: you're already closer than you think. Every skill you have transfers. You just need to extend it."

---

## SLIDE 19: What to Do Next

### Practical Steps You Can Take This Week

1. **Pick one AI concept and build a hello world** - even if it's just 10 lines in a Colab notebook
2. **Read one system design** - understand how data flows through a real system, not just your part of it
3. **Ask "what problem could AI solve in my current work?"** - not "what AI tool should I learn"
4. **Learn the 10-step framework** - use it to evaluate any AI system or interview question
5. **Build something real** - not just a portfolio piece. A tool that solves a problem for real users, at your current job

All the notebooks, concept guides, and frameworks are here:

**GitHub:** github.com/sunilmogadati/systems-in-production
- Fully executable on Google Colab (no setup, no cost)
- ML, Deep Learning, RAG, Agents, Data Pipelines, System Design
- 10-step framework, interview prep, architect decision checklists

> Speaker notes: "Everything I showed today is available, fully executable, right now. The goal isn't to consume content. The goal is to build. Start with one hello world. Then ask what problem at your current job could benefit from this. Build that. That's how you move up the value chain. Not by switching careers, but by expanding what you can do in your current role."

---

## SLIDE 20: Closing

### The Future Is Not Dev vs QA vs DevOps.

### The Future Is People Who Understand Systems, and People Who Don't.

"AI is new. Systems are not. The opportunity is at the intersection."

**Sunil Mogadati**
Emergtech Business Solutions
github.com/sunilmogadati/systems-in-production

> Speaker notes: End here. Don't add more. Let it land. If people want to talk, they'll come to you after. If someone asks about cohort or community, THEN mention Skool. Don't push it. Just be available.

---

## DEMO LINKS (for your browser tabs)

Open these before the presentation:

1. **Star Schema Design:** https://github.com/sunilmogadati/systems-in-production/tree/main/playbooks/data/star-schema-design
2. **ML Fundamentals:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_ML_Fundamentals.ipynb
3. **RAG from Scratch:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_RAG_from_Scratch.ipynb
4. **GCP Pipeline:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/M08_Cloud_Data_Pipeline_Build.ipynb
5. **Deep Learning:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Deep_Learning_PyTorch.ipynb
6. **Agents:** https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Agents.ipynb

---

## GAMMA INSTRUCTIONS

When pasting into Gamma:

1. **Theme:** Dark background (charcoal #333333 or similar)
2. **Accent color:** Emergtech orange #E85D26
3. **Font:** Clean sans-serif (Helvetica, Inter, or similar)
4. **Logo:** Add Emergtech logo to title slide and footer
5. **Layout:** Minimal. Let text breathe. No stock photos.
6. **Tables:** Use the orange accent for headers

Tell Gamma:
- "Professional tech presentation"
- "Dark theme with orange accent"
- "Minimal design, no stock images"
- "Corporate but modern"
