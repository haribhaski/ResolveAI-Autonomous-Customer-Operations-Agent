# ResolveAI — Autonomous Customer Operations Agent

> **Project status: Planned / in development.** This README defines the intended architecture and development roadmap. Features, integrations, evaluation results, and deployments described below should not be treated as implemented until checked off and verified.

ResolveAI is a production-oriented AI customer-operations assistant designed to **understand support requests, retrieve relevant policies, investigate account activity, and take authorized actions through tools**. Unlike a question-answering chatbot, ResolveAI aims to resolve an issue end to end while leaving a clear audit trail.

**Example request:** “I was charged twice last month. Can you check and help me get a refund?”

The intended workflow is to identify the account and issue, query transaction records, retrieve the relevant refund policy, determine eligibility, ask for any required confirmation, submit a refund *request* via an API, and explain the result. A refund is **never** issued merely because an LLM suggested it.

## Why build this?

Traditional support bots can answer FAQs but often cannot safely act on real customer issues. Agentic systems can call APIs, but real-world use also requires **authorization, bounded tools, persistent state, observability, and measurable reliability**.

ResolveAI explores how to build an agent that is useful *and* controlled: it can investigate and propose actions, but high-impact operations pass deterministic checks and explicit approval gates.

## Core capabilities (target)

- **Conversational intake:** Understand user requests and ask clarifying questions when needed.
- **Hybrid retrieval:** Search policy documents with lexical and vector retrieval and return grounded references.
- **Tool-using workflow:** Query mock orders, subscriptions, payments, and tickets via typed APIs.
- **Action controls:** Apply deterministic business rules, authorization checks, and human-in-the-loop approval before sensitive actions.
- **Durable execution:** Persist workflows across interruptions and use idempotency keys for side-effecting API calls.
- **Recovery:** Handle timeouts, failed tools, and incomplete information without silently claiming success.
- **Observability:** Trace agent steps, tool calls, response times, errors, and estimated LLM cost.
- **Evaluation:** Measure completion, tool selection, safety, latency, and cost on a reproducible synthetic benchmark.
- **Optional phase 2:** Voice-based interactions with speech-to-text and text-to-speech.

## Architecture

```mermaid
flowchart TD
    U[Customer: web chat / optional voice] --> FE[Next.js frontend]
    FE --> API[FastAPI backend]
    API --> AUTH[Authentication & authorization]
    AUTH --> LG[LangGraph workflow]
    LG --> TRIAGE[Intent & context classification]
    TRIAGE --> RETRIEVE[Policy retrieval]
    TRIAGE --> LOOKUP[Customer / order / billing tools]
    RETRIEVE --> VDB[(PostgreSQL + pgvector)]
    LOOKUP --> DB[(PostgreSQL mock business data)]
    RETRIEVE --> PLAN[Action proposal]
    LOOKUP --> PLAN
    PLAN --> GUARD[Deterministic policy and permission checks]
    GUARD -->|Needs approval| APPROVE[Human approval]
    GUARD -->|Read-only / permitted| EXEC[Tool execution]
    APPROVE -->|Approved| EXEC
    APPROVE -->|Rejected| RESPONSE[Explain outcome]
    EXEC --> EXT[Mock ticket / refund-request APIs]
    EXT --> RESPONSE
    RESPONSE --> FE
    LG -. traces .-> OBS[LangSmith or Langfuse]
```

### Why LangGraph?

A stateful graph is better suited than a free-form multi-agent chat for workflows that must **pause for approval, resume after failure, and enforce predictable transitions**. LangChain can provide model and tool abstractions; LangGraph coordinates their execution. Specialized agents/nodes should be introduced only where they add clarity, not simply to increase the number of agents.

## Proposed tech stack

| Layer | Technology | Purpose |
| --- | --- | --- |
| Frontend | Next.js, TypeScript | Chat UI, approvals, ticket status |
| API | Python, FastAPI, Pydantic | Validated endpoints and tool interfaces |
| Agent workflow | LangGraph + LangChain | Stateful orchestration, structured tool calls |
| LLM | OpenAI or Groq API | Intent analysis and grounded responses |
| Database | PostgreSQL | Accounts, orders, transactions, tickets, audit events |
| Retrieval | pgvector + PostgreSQL full-text search | Semantic and lexical document search |
| Background work (optional) | Redis + Celery | Longer-running tasks and retries |
| Evaluation | pytest, custom scenario runner | Functional, safety, and regression tests |
| Tracing | LangSmith or Langfuse | Step-level debugging and cost/latency analysis |
| Deployment | Docker Compose | Reproducible local environment |
| Voice (optional) | LiveKit Agents, STT/TTS provider | Spoken customer interactions |

## Example workflow: duplicate charge

1. **Understand:** Detect a billing-dispute request, without assuming the charge is actually duplicated.
2. **Authenticate:** Confirm the user is authorized to view the relevant account.
3. **Investigate:** Retrieve eligible transactions and look for potential duplicates.
4. **Ground:** Retrieve applicable billing/refund policy sections with citations.
5. **Decide:** Propose the next appropriate step based on evidence and deterministic policy checks.
6. **Approve:** Request explicit confirmation before opening a refund request or any other consequential action.
7. **Execute:** Call an idempotent mock `create_refund_request` API; record the response.
8. **Report:** Return the ticket/reference ID and distinguish **request submitted** from **refund issued**.

If a tool fails or the evidence is insufficient, the workflow should state the uncertainty and route to a human rather than invent a resolution.

## Repository layout (proposed)

```text
resolveai/
├── README.md
├── .env.example
├── docker-compose.yml
├── backend/
│   ├── app/
│   │   ├── main.py                 # FastAPI entry point
│   │   ├── api/                    # Chat, approval, ticket endpoints
│   │   ├── agents/                 # LangGraph nodes and graph
│   │   ├── tools/                  # Read-only and action APIs
│   │   ├── retrieval/              # Indexing and hybrid search
│   │   ├── policies/               # Deterministic action checks
│   │   ├── models/                 # Request/response schemas
│   │   ├── db/                     # Database access and migrations
│   │   └── observability/          # Traces, logs, metrics
│   ├── tests/
│   └── requirements.txt
├── frontend/                       # Next.js UI
├── data/
│   ├── synthetic/                  # Fake customer and payment data
│   └── policies/                   # Example policy documents
├── evals/
│   ├── scenarios.jsonl
│   └── run_evals.py
└── docs/
    ├── architecture.md
    └── demo.md
```

> This is the **target** layout. Paths and commands below are templates until the corresponding files are implemented.

## Getting started (after implementation)

### Requirements

- Python 3.11+
- Node.js 20+
- Docker and Docker Compose
- API credentials for a supported LLM provider

### Configuration

Create a `.env` file from `.env.example` and supply the necessary variables. Example placeholders:

```dotenv
DATABASE_URL=postgresql://resolveai:local_password@localhost:5432/resolveai
LLM_PROVIDER=openai
OPENAI_API_KEY=replace_me
# Or configure a Groq provider with GROQ_API_KEY=replace_me
LANGSMITH_TRACING=false
LANGSMITH_API_KEY=
```

**Never commit `.env` files or API keys.** The eventual app should validate configuration at startup.

### Planned local start

```bash
# Once docker-compose.yml and the service images exist:
docker compose up --build
```

The planned development endpoints are a Next.js frontend at `http://localhost:3000` and a FastAPI service at `http://localhost:8000/docs`. **These addresses are not live demo links.**

## API design (proposal)

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/api/chat` | Create or continue a workflow |
| `GET` | `/api/workflows/{id}` | Inspect state and permitted next actions |
| `POST` | `/api/workflows/{id}/approve` | Approve or reject a specific pending action |
| `GET` | `/api/tickets/{id}` | Read ticket status |
| `GET` | `/health` | Check service health |

Mutating tools will require account ownership validation, scoped permissions, explicit approval where necessary, and idempotency safeguards.

## Evaluation plan

Build a benchmark of **100–200 synthetic scenarios**, including:

- Duplicate-charge and refund-request cases
- Cancellation and subscription updates
- Policy questions that do not require tools
- Missing records and ambiguous user requests
- Tool timeouts, retries, and partial API failures
- Unauthorized requests and attempts to bypass approval
- Prompt injection embedded in retrieved policy documents

| Metric | How to measure |
| --- | --- |
| End-to-end task success | Scenarios completed with the correct verified outcome / total |
| Tool-selection accuracy | Correct tools selected against annotated expected tool calls |
| Unauthorized-action rate | Disallowed side effects / adversarial or unauthorized scenarios; **target: 0** |
| Groundedness | Responses supported by retrieved policy or tool evidence |
| Latency | Median and p95 end-to-end request duration |
| Cost | Estimated LLM cost per completed workflow |
| Recovery success | Failed-tool scenarios that recover or escalate correctly |

**Results: Not measured yet.** Add a results table, test date, model version, scenario counts, and reproducible commands only after running the benchmark. Do not claim production readiness or performance gains without evidence.

## Development roadmap

- [ ] **Phase 1 — Core MVP:** database + synthetic seed data, FastAPI, one LangGraph workflow, read-only tools, policy retrieval, text chat.
- [ ] **Phase 2 — Safe actions:** typed action tools, user confirmation, permission checks, idempotency, ticket creation, persistent workflow state.
- [ ] **Phase 3 — Quality:** automated tests, evaluation suite, failure injection, observability, Docker setup, demo and screenshots.
- [ ] **Phase 4 — Stretch:** voice interface, background jobs, load testing, deployment.

## Security and responsible design

- Use **synthetic customer and transaction data** in demos and tests; do not upload real private billing or health data.
- Treat retrieved documents and user prompts as untrusted inputs, never as authority to override system rules.
- Validate permissions and action policies in backend code rather than solely relying on LLM output.
- Require confirmation for consequential actions and log who authorized them.
- Make side-effecting operations idempotent, auditable, and reversible where possible.
- Limit PII in traces and redact sensitive fields from logs.

## Demo

*Coming after the MVP is implemented.* Planned demo: a synthetic duplicate-charge case showing evidence retrieval, proposed action, user approval, ticket creation, and the audit trail.

## Project goals

ResolveAI is intended to demonstrate the engineering skills involved in building **reliable tool-using AI applications**: orchestration, API design, retrieval, persistence, authorization, testing, observability, and deployment. The goal is not to maximize agent count; it is to resolve tasks **correctly, safely, and measurably**.

## License

A license has not been selected yet. Add a `LICENSE` file before releasing the repository for public reuse.
