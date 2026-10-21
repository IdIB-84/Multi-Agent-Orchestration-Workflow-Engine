Multi-Agent Orchestration Engine (backend)
An orchestrator turns a business objective into a validated task DAG, runs worker agents
in parallel as dependencies resolve, retries failures, enforces a token budget, applies a QA
gate with one revision round, and streams every step as events.
```
 client ──POST /runs──▶ FastAPI ──▶ orchestrator.execute()
    ▲                                   │  validate_dag → schedule()
    │ SSE /runs/{id}/events             ▼
    └──────── store.emit ◀── run_task (retry, timeout, budget) ──▶ LLM provider
                                       │
        extractor → regulatory_analyst → risk_scorer → strategy_writer → qa_reviewer
                                                            ▲_______ feedback ______│
```
Run
    pip install -r requirements.txt
    export ANTHROPIC_API_KEY=...   # omit to use FakeLLM
    export API_KEY=change-me
    uvicorn app.main:app --reload
    pytest

    curl -X POST localhost:8000/runs -H 'x-api-key: change-me' -H 'content-type: application/json' \
      -d '{"objective":"Analyze this report and generate a compliance strategy","document":"..."}'
    curl -N localhost:8000/runs/<id>/events -H 'x-api-key: change-me'

Design decisions (good interview material)
DAG, not a prompt chain: independent work runs in parallel and plans are data you can validate.
Plans are untrusted: `validate_dag` rejects unknown agents, dangling deps and cycles, so an LLM planner can propose plans safely.
Provider interface: agents depend on `LLM`, not a vendor SDK; tests use `FakeLLM`.
Failure handling: per-call timeout, 3 attempts, exponential backoff with jitter, malformed JSON counts as a retryable failure.
Cost control: hard per-run token budget, concurrency cap.
Quality gate: reviewer output can send the writer back once with concrete feedback.
Observability: every state change is an event (replayable audit trail + SSE).
Known gaps before real production use
Durability: runs execute in-process (`asyncio.create_task`). Move to a queue/worker (Arq, Celery) or Temporal so runs survive restarts.
Storage: replace `InMemoryStore` with Postgres (runs, tasks, events) and Redis pub/sub for multi-replica SSE.
Auth: replace the static API key with OAuth/JWT, per-tenant isolation, rate limits.
Prompt-injection defence: treat the document as data, add output schema validation (Pydantic) per agent, and human approval before any external action.
Observability: OpenTelemetry traces per task, cost/latency/retry-rate metrics, structured logs.
Evals: golden-document regression tests for each agent's output quality.
Deploy: Dockerfile included; add CI, IaC, secrets manager, autoscaling workers.
Dynamic planning (agent selection)
Agents are not created by the model. They live in a curated registry (`app/agents.py`, each
with a description). For every run the planner (`app/planner.py`) shows the registry to the LLM,
which proposes a task graph as JSON. `check_plan` then treats that JSON as untrusted: known
agents only, 1-8 tasks, safe ids, no cycles or dangling deps, a writer and a reviewer required,
and only the reviewer may depend on the writer. Any failure falls back to `DEFAULT_PLAN` and
the reason is recorded in the `plan` event. To add a capability, add an `Agent` to the registry
(or load it from config or a DB table with an admin approval step); the planner picks it up
automatically. A human-approval gate before execution is a natural next step.
Human approval gate
Send `"require_approval": true` with `POST /runs`. The planner still runs, but the run stops in
`awaiting_approval` with the proposed graph visible via `GET /runs/{id}` (and a `plan` event on the
stream). No worker agent executes until `POST /runs/{id}/approval` with
`{"approved": true|false, "comment": "..."}`. The decision (who, when, why) is stored on the run and
emitted as an `approval` event, giving you an audit trail. Rejection or a one-hour timeout ends the
run as `rejected` with zero agent calls made. Known gap: approvals live in process memory, so for
multiple replicas persist the pending state (Postgres) and signal workers through Redis or Temporal.
