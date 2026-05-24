# Human-Centric System Design: The monday.com Agent Team
### How specialized AI agents and engineering teams co-own work on monday.com

---

## 1. Executive Summary: The Core Concept

Today's AI features on monday.com (like Sidekick, Magic, or Vibe) operate on a **request-and-response** model: *a user asks the AI to summarize a thread or write a formula, and the AI immediately replies.*

This system design introduces **Hybrid Human-Agent Collaboration**. Instead of a single chatbot, we introduce a **team of specialized AI agents** that run in the background. They monitor your monday.com boards, coordinate tasks with each other, talk to external tools (like GitHub and CI/CD pipelines), and proactively nudge human team members at defined "approval gates."

The monday.com workspace is no longer just a tracking tool for humans—it becomes a **shared, living environment** where humans and specialized AI agents co-own the engineering lifecycle.

---

## 2. The Concrete Technology Stack

To turn this design into a reliable production system, we select a modern, enterprise-grade AI and engineering stack. We ground our architecture in these specific technologies:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        THE TECHNOLOGY STACK                            │
├───────────────────┬────────────────────────────────────────────────────┤
│ Language & Core   │ Python 3.11 (optimized for asynchronous execution) │
├───────────────────┼────────────────────────────────────────────────────┤
│ Agent Framework   │ LangGraph (by LangChain)                           │
│                   │ - Perfect for stateful, cyclic agent workflows     │
├───────────────────┼────────────────────────────────────────────────────┤
│ Ingestion API     │ FastAPI (Asynchronous Python microservices)        │
├───────────────────┼────────────────────────────────────────────────────┤
│ Core Brain (LLMs) │ - Claude 3.5 Sonnet (for complex reasoning)        │
│                   │ - GPT-4o-mini (for fast, low-cost triage/summaries)│
│                   │ - Cohere Command R+ (for native multilingual/RTL)  │
├───────────────────┼────────────────────────────────────────────────────┤
│ Router            │ DeBERTa-v3-base (Local small classifier)           │
├───────────────────┼────────────────────────────────────────────────────┤
│ Caching & State   │ Redis Enterprise (Fast, shared memory/locks)       │
├───────────────────┼────────────────────────────────────────────────────┤
│ Database          │ PostgreSQL with the pgvector extension              │
│                   │ - Stores relational board data + embeddings        │
├───────────────────┼────────────────────────────────────────────────────┤
│ Message Queue     │ RabbitMQ (Handles rate limits & backpressure)      │
├───────────────────┼────────────────────────────────────────────────────┤
│ Infrastructure    │ AWS ECS (Fargate) + AWS KMS (Secret management)    │
└───────────────────┴────────────────────────────────────────────────────┘
```

* **Python 3.11 & FastAPI:** Powers the asynchronous webhook receiver that swallows events from monday.com and external developer tools.
* **LangGraph:** Our core framework for constructing the agents. Unlike simple linear pipelines, LangGraph supports cyclic loops and complex multi-agent state definitions, letting agents hand off tasks and double-check each other's work.
* **The LLM Strategy:** We do not use one expensive model for everything.
  * **Claude 3.5 Sonnet** is our heavy-lifter, used for high-risk reasoning like evaluating code readiness, comparing acceptance criteria, and drafting changelogs.
  * **GPT-4o-mini** handles low-level, high-frequency, sub-cent tasks like summarizing pull requests, parsing titles, and reading simple update logs.
  * **Cohere Command R+** is utilized for native multilingual and Right-to-Left (RTL) localization, significantly cutting down on token costs for non-English workspaces.
* **DeBERTa-v3-base:** A tiny, highly specialized natural language processing (NLP) encoder running locally inside Docker containers. It classifies and routes incoming events in milliseconds for a fraction of a cent.
* **PostgreSQL + pgvector:** Our database of record. It stores workspace configuration tables, and the `pgvector` extension allows us to perform semantic vector searches for things like retrieving contextually similar past tasks or tracking user preferences.
* **Redis Enterprise:** Provides sub-millisecond in-memory caching. It tracks active sprints, user lists, and locks item records while an agent is modifying them to prevent write conflicts.
* **RabbitMQ:** Acts as the traffic cop. It queues incoming webhooks and controls the rate of outbound mutations to respect monday.com API limits.

---

## 3. Meet the Agent Team (Relatable Roles)

To build an efficient team, we partition responsibilities among **five specialized agents**, mirroring a mature software development organization:

```
             ┌─────────────────────────┐
             │ Incoming Workspace Event│
             └────────────┬────────────┘
                          │
                 ┌────────┴────────┐
                 │ DeBERTa Router  │
                 └────────┬────────┘
                          │
         ┌────────────────┼────────────────┬────────────────┐
         ▼                ▼                ▼                ▼
   ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
   │  Triage   │    │  Sprint   │    │Dev Liaison│    │  QA Gate  │
   │  Agent    │    │  Agent    │    │   Agent   │    │   Agent   │
   └─────┬─────┘    └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
         │                │                │                │
         └────────────────┼────────────────┴────────────────┘
                          ▼
                    ┌───────────┐
                    │  Release  │
                    │  Agent    │
                    └───────────┘
```

### 3.1 The Triage Agent (`triage`)
* **Metaphor:** The Intake Coordinator.
* **Job:** Sits at the front desk. When a new bug, feature request, or ticket lands in the Backlog/Inbox board, this agent reads it, classifies its priority, estimates its impact, and routes it to the correct board and engineer.
* **Safety Rail:** If it detects a major system outage (P0/P1), it halts autonomous routing, bypasses regular boards, and immediately pages the on-call engineer.

### 3.2 The Sprint Agent (`sprint`)
* **Metaphor:** The Project Manager.
* **Job:** Monitors the overall health of the current sprint. It watches team capacity, flags tickets that are dragging on too long (stale items), detects when a team member is Out-of-Office, and proposes rebalancing tasks.
* **Safety Rail:** It can flag items on the board, but it is **never** allowed to move tasks across sprint boundaries without a manager's direct click.

### 3.3 The Dev Liaison Agent (`dev_liaison`)
* **Metaphor:** The Tech Lead Bridge.
* **Job:** Syncs the monday.com board with actual code activity. It connects via secure MCP (Model Context Protocol) to GitHub or GitLab. When a developer opens a Pull Request or pushes code, it automatically updates the monday.com item status, links the commit, and summarizes the code changes in plain English.
* **Safety Rail:** It operates purely as a visual synchronizer; it cannot merge code or close issues autonomously.

### 3.4 The QA Gate Agent (`qa_gate`)
* **Metaphor:** The Rigorous Tester.
* **Job:** Enforces quality gates. When an engineer moves a task to "Ready for QA," this agent checks the linked CI/CD system (like GitHub Actions or CircleCI) to ensure automated test suites passed. It then reads the task's original "Acceptance Criteria" from monday.com and verifies that the pull request contains matching logic.
* **Safety Rail:** If the task touches critical areas (like `security`, `payments`, or `PII`), it blocks autonomous sign-off and demands a senior human QA lead's signature.

### 3.5 The Release Agent (`release`)
* **Metaphor:** The Release Manager.
* **Job:** Coordinates the final push. It aggregates all items marked "QA Approved," checks release readiness, generates clean, user-friendly markdown release notes/changelogs using monday's Vibe Doc capability, and prepares announcements.
* **Safety Rail:** It drafts the changelog and prepares the Slack announcement, but a human must click "Approve and Release" before any external communication is sent.

---

## 4. The Lifecycle of a Task (A Human Story)

To understand how this system actually works, let’s follow a single bug report through its lifecycle:

```
[1] User reports bug in Inbox
      │
      ▼
[2] Triage Agent routes & assigns to Engineer (GPT-4o-mini)
      │
      ▼
[3] Engineer opens GitHub PR ──► Dev Liaison Agent syncs status & adds summary (GPT-4o-mini)
      │
      ▼
[4] QA Gate Agent verifies CI tests pass & checks Acceptance Criteria (Claude 3.5 Sonnet)
      │
      ▼
[5] Release Agent aggregates bug-fix & drafts user changelog in monday Docs (Claude 3.5 Sonnet)
      │
      ▼
[6] Product Manager approves ──► Release Agent publishes & archives
```

1. **Intake:** A customer submits a ticket: *"Payment screen crashes when clicking checkout on iOS in Hebrew."*
2. **Triage:** The **Triage Agent** picks up the event. It uses **GPT-4o-mini** to analyze the text. It detects that the bug is localized (Hebrew) and targets the payment flow. Because "payment" is a high-sensitivity category, it labels it as `High Priority`, moves it from the Inbox board to the Active Sprint board, and assigns it to "Sarah," the lead iOS payment engineer.
3. **Development:** Sarah opens a branch in GitHub and begins working. When she opens a Pull Request, the **Dev Liaison Agent** receives a webhook, links the PR directly to the monday.com item, translates the technical git diff into a simple, non-technical status update, and moves the board column to "In Progress."
4. **Testing Gate:** Once Sarah fixes the bug, she changes the status to "Ready for QA." The **QA Gate Agent** steps in. It queries GitHub Actions via MCP and verifies that Sarah's unit tests passed. It uses **Claude 3.5 Sonnet** to read the ticket's Acceptance Criteria and compares it against the PR diff to ensure the Hebrew localization bug was specifically addressed. Since it's a "payment" item, it flags a QA Lead for final sign-off.
5. **Changelog Prep:** Once approved, the ticket is marked "Ready to Release." As the release window approaches, the **Release Agent** scans the board, aggregates Sarah's ticket with other completed work, and drafts an elegant, customer-facing changelog inside a **monday Doc**.
6. **The Hand-off:** The Product Manager receives a notification with a simple preview card showing the draft changelog. She clicks "Approve." The Release Agent archives the completed tasks, posts the release notes, and updates the customer.

---

## 5. Solving the Hard Technical Problems

In an enterprise environment, simple LLM scripts fail quickly. Here is how our technology stack solves the hardest production hurdles:

### 5.1 API Rate Limits (The Token Bucket Guard)
monday.com's GraphQL API strictly limits how many operations can be performed per minute. If five agents start updating dozens of board items simultaneously, they will crash into rate limits, dropping events.
* **Our Solution (RabbitMQ + Token Bucket):** All agent mutations (writes to monday.com) are routed through a centralized **Action Executor** service. This service runs a **Token Bucket algorithm** in Python, tracking available API capacity per tenant account. 
* High-priority updates (like QA blocks or P0 alerts) bypass the queue. Regular status updates and summaries are held in a **RabbitMQ queue** and trickled out smoothly as the token bucket refills, ensuring 100% delivery without ever triggering a `Rate Limit Exceeded` error.

### 5.2 Multi-Language and RTL Token Inflation
Standard LLM tokenizers split text into small sub-word chunks. While English words usually map to a single token, Right-to-Left (RTL) scripts like Hebrew or Arabic are split into many more sub-word fragments. A single sentence in Hebrew can consume **3 to 4 times more tokens** than English, causing API costs to skyrocket and exceeding context window lengths.
* **Our Solution (Cohere Command R+ & Dynamic Context):** For Hebrew and Arabic workspaces, our system swaps out standard models for **Cohere Command R+** (specifically optimized for non-English performance and translation). 
* Additionally, we run a text-preprocessing step using **Unicode NFC normalization**. When querying our **pgvector database** for context, we dynamically reduce the retrieve limit (`top-k` snippets) for RTL locales, keeping prompts tight, fast, and within a predictable budget (scaling up to **$0.15/item** dynamically only when necessary).

### 5.3 Keeping Customer Code and Credentials Safe
Agents need access to systems like GitHub, GitLab, and Jira. In a multi-tenant cloud application, storing customer credentials globally is a security nightmare.
* **Our Solution (monday Secure App Storage):** We never store integration secrets in our own application database. Instead, we leverage monday.com's native **Secure App Storage**. 
* When our MCP adapters need to pull GitHub data for a tenant, they fetch the short-lived, encrypted OAuth token dynamically using the user's active session signature. The secret is held purely in-memory in Python and immediately discarded after the API call completes, ensuring absolute tenant isolation.

### 5.4 How the Agents Learn from Human Rejections (Discard & Learn)
If a Product Manager rejects a release changelog or overrides a triage assignment, the agent must not repeat that mistake.
* **Our Solution (pgvector Feedback Loops):** When a human rejects or modifies an agent's proposal at an approval gate, the system triggers our **Discard & Learn loop**:

```
[1] Human rejects/modifies agent suggestion
      │
      ▼
[2] System strips all PII (names, emails)
      │
      ▼
[3] Failed scenario + correction stored in pgvector database
      │
      ▼
[4] Future tasks query pgvector: "Have I failed at this before?"
      │
      ▼
[5] Inject correction as a negative few-shot prompt: "DO NOT do X; instead do Y"
```

* This memory database is isolated per customer workspace, meaning agents learn the unique nuances, vocabularies, and preferences of individual teams over time without mixing any cross-tenant data.

---

## 6. Business Success & Production Posture

### 6.1 Success KPIs
We measure success through simple, high-impact business metrics rather than just technical ML stats:

* **Cycle Time (P50/P90):** We target a **≥20% reduction** in the time it takes for an item to go from Backlog to Released.
* **Human Attention Cost:** We aim to reduce coordination time to **≤4 minutes per task** (from a baseline of 12-15 minutes).
* **Escape Rate:** Defects that bypass the QA Gate Agent to production must remain **under 3%**.

### 6.2 Pre-User Complaint Alerting
Instead of waiting for customers to complain, our Python microservices stream telemetry directly to dashboards. We watch a primary metric: **The Escape Rate**. 
* If our telemetry detects that users are reverting/undoing agent actions or overriding suggestions more than **5% of the time in a 6-hour window**, our system triggers a high-priority on-call alert and automatically drops the workspace's autonomy level back to `suggestion_only` mode.

---

## 7. Concrete MVP Roadmap

To build trust and validate safety, we roll the agent team out in three logical phases:

```
Phase 1: Observation (Wk 1-10) ──► Phase 2: Assisted (Wk 11-14) ──► Phase 3: Autonomous (Wk 15+)
  - Agents run in dry-run        - Low-risk status updates live   - High-risk gates unlocked
  - Verify triage routes         - Human approves rebalances      - Self-improving feedback loop
  - Calibrate LLM judges         - Rate limiters validated        - Scale to 100% of workspaces
```

1. **Phase 1: Observation (Weeks 1–10):** The Triage, Dev Liaison, and Sprint agents run in "dry-run" shadow mode. They observe the boards and generate proposals in the database, but do not write back to monday.com. We compare their planned actions against actual human actions to verify accuracy.
2. **Phase 2: Assisted Action (Weeks 11–14):** We turn on live integrations for low-risk tasks. The Dev Liaison syncs status changes, while the Sprint and Triage agents suggest rebalances and assignments via **interactive Preview Cards** in the monday UI.
3. **Phase 3: Gated Autonomy (Weeks 15+):** Once the workspace achieves a $70\%$ approval acceptance rate over 14 days, the workspace can opt-in to autonomous writes. Even in full autonomy, high-risk gates (such as signing off on security tasks or publishing external changelogs) remain strictly locked behind human approval.

---

*This reader-friendly system design details a production-ready, highly secure, and cost-controlled agentic architecture, grounded in LangGraph, PostgreSQL/pgvector, and Redis, custom-tailored for the monday.com ecosystem.*
