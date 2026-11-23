# CoursePilot: Technical Environment

This document records the technical choices for CoursePilot, an AI agent that helps Cornell CS students plan a semester schedule and manage course deadlines. AI-DLC should treat these decisions as constraints. The product runs entirely locally on team laptops for the demo, including the LLM, which runs in Ollama. Nothing is deployed or hosted online, and no paid API or API key is needed.

## 1. Language, Package Manager, and Framework

| Area | Choice | Notes |
|---|---|---|
| Language | Python 3.12 | One language for the UI, agents, tools, solver, and evaluation runner. Type hints are required on all function signatures. |
| Package manager | `uv` | Dependencies are declared in `pyproject.toml`, and `uv.lock` is committed so every teammate gets identical versions. Run `uv sync` to install and `uv run <cmd>` to execute. |
| UI framework | Streamlit (>= 1.40) | Chat interface, schedule weekly view, citation viewer, deadline review table, and trace viewer. |
| Data validation | Pydantic v2 | Every tool input and output, router decision, and session state object is a Pydantic model. |
| LLM runtime | Ollama (free, runs locally) | Serves open models on the student's own laptop at `http://localhost:11434`. No API key and no per-token cost. |
| LLM access | `ollama` Python library | All model calls go through one wrapper module, `app/llm.py`, which uses the library's chat API with native tool calling and JSON-schema structured output. |
| Models | Agent model: `qwen3:8b`. Router and query-rewriting model: `llama3.2:3b`. | Both support tool calling in Ollama. Model names are set in `.env` (`AGENT_MODEL`, `ROUTER_MODEL`), not hard-coded, so teammates with less memory can switch to a smaller model such as `qwen3:4b`. The team will confirm these choices on development examples and record the final names in the README. |
| Database | PostgreSQL 16 with `pgvector` | Stores courses, sections, prerequisite rules, requirement categories, sessions, messages, document chunks, embeddings, confirmed events, and traces. Accessed with `psycopg` 3 and plain parameterized SQL. |
| Constraint solver | Google OR-Tools CP-SAT | Selects compatible sections and enforces hard constraints. |
| PDF parsing | PyMuPDF (`pymupdf`) | Text-based PDFs only. OCR is out of scope. |
| Embeddings | `bge-small-en-v1.5` via `sentence-transformers`, running locally | Used for document-chunk and course-description search. Downloaded at image build time. |
| Calendar export | `icalendar` | Generates `.ics` files from confirmed events only. |
| HTTP client | `httpx` | For any other outbound HTTP, such as the Class Roster API if we use it instead of a cached export. |
| Lint and type check | `ruff` (lint and format), `mypy` | Run before every pull request. |

Because the models are small and local, the app compensates in code rather than trusting the model:

- Every model-generated tool call and structured output is validated with Pydantic. On a validation failure the wrapper retries once with the error message, then returns a structured failure.
- Context size is set explicitly on every call (`num_ctx`, default 8192 from `OLLAMA_NUM_CTX`), because Ollama's default context window can be too small for tool results and retrieved passages.
- Temperature is 0 for routing, rewriting, and tool selection.
- All hard rules (conflicts, credits, prerequisites, citation validity) are enforced by Python, never by the model.

Layout of the application code (matches the README):

```text
app/
  ui/           # Streamlit pages: chat, schedule view, citations, deadline review, traces
  service.py    # handle_message(): single entry point the UI calls
  llm.py        # the only module that talks to Ollama
  router.py     # intent classification and clarification logic
  agents/       # planner.py, materials.py, shared agent loop
  tools/        # course search, prereqs, requirements, solver, materials, deadlines, export
  db/           # connection, migrations, repositories (SQL lives here only)
  state.py      # SessionState models and persistence
  tracing.py    # trace_id creation and trace recording
data/           # cached roster export, reviewed prerequisite rules, coverage notes
eval/           # development examples, final cases, runner, reports
tests/          # unit and integration tests
```

## 2. Cloud Provider and Deployment Model

**Cloud provider: none.** CoursePilot runs entirely on team laptops and is not deployed or hosted online.

**Deployment model: a local web app that calls a model running in Ollama.**

- **Ollama** is installed natively on each teammate's laptop and runs as a local service on `localhost:11434`. It runs on the host, not in Docker, so it can use the laptop's GPU or Apple Silicon acceleration. Models are downloaded once with `ollama pull`.
- **Docker Compose** runs two services:
  - `db`: PostgreSQL 16 with the `pgvector` extension and a named volume so data persists between runs.
  - `app`: the Streamlit app, the agents, the Python tools, the OR-Tools solver, and the local embedding model, in one container. It is served at `http://localhost:8501`, bound to `localhost` only.
- The `app` container reaches Ollama on the host at `http://host.docker.internal:11434` (set as `OLLAMA_HOST`). Linux teammates add `extra_hosts: ["host.docker.internal:host-gateway"]` to the `app` service, and may need Ollama configured to accept connections from the Docker network.
- A seed script runs on first start. It loads the cached Class Roster data and the team-reviewed prerequisite and requirement rules. It is idempotent, so rerunning it never duplicates rows.
- Class Roster data is cached in `data/` for one term. The app does not call Class Roster at runtime during the demo.
- **Internet is needed only for setup** (installing Ollama, pulling models, building the Docker image). Once those are done, the demo needs no internet connection and sends no data to any third party.
- **Hardware:** the 8B agent model needs roughly 8 GB of free memory. Teammates with less can use the 4B alternative.

Setup:

```bash
# 1. Install Ollama (https://ollama.com/download), then pull the models
ollama pull qwen3:8b
ollama pull llama3.2:3b

# 2. Configure and start the app
cp .env.example .env
# .env sets OLLAMA_HOST, AGENT_MODEL, ROUTER_MODEL, OLLAMA_NUM_CTX, DATABASE_URL
docker compose up --build
```

`.env.example` contains the following variables, and none of them is a secret:

```text
OLLAMA_HOST=http://host.docker.internal:11434
AGENT_MODEL=qwen3:8b
ROUTER_MODEL=llama3.2:3b
OLLAMA_NUM_CTX=8192
DATABASE_URL=postgresql://coursepilot:coursepilot@db:5432/coursepilot
```

## 3. Test Framework

| Purpose | Tool |
|---|---|
| Unit and integration tests | `pytest` |
| Coverage | `pytest-cov` (target: 80% on `app/tools/`, `app/state.py`, and the schedule checker) |
| Property-based tests for the solver and checker | `hypothesis` |
| Evaluation of LLM behavior | `python -m eval.run` (custom runner over versioned cases in `eval/`) |

Testing rules:

- **Deterministic code is tested without any LLM.** Prerequisite logic, the solver, the schedule checker, the `.ics` exporter, and citation validation must have unit tests that run offline and quickly.
- **Unit tests mock `app/llm.py`.** No unit test may call a real model. Tests that need a running Ollama are marked `@pytest.mark.live` and are excluded by default.
- **Database tests** run against a throwaway Postgres schema created by a fixture, never the demo database.
- **Hard-rule invariant:** a property-based test asserts that no schedule returned by `solve_schedule` ever has overlapping meetings, a missing required component, or credits outside the requested range.
- **Evaluation** uses separate development examples and frozen final cases, as described in the README. Reports include raw counts alongside percentages, and record the model names and `num_ctx` used so results are reproducible. Because the models run locally and cost nothing per call, token counts and latency are the cost measures, and estimated dollar cost is recorded as zero.

Commands:

```bash
uv run pytest                  # fast offline tests
uv run pytest -m live          # tests that call the real model (needs Ollama running)
uv run python -m eval.run      # evaluation runner
uv run ruff check . && uv run mypy app
```

## 4. Prohibited Libraries

| Library or practice | Reason | Use instead |
|---|---|---|
| LangChain, LlamaIndex, CrewAI, AutoGen | They hide the tool-calling loop, which is the core of this project, and make traces, the eight-call budget, and the course learning goals harder to demonstrate. | The `ollama` library with our own agent loop and Pydantic-validated tools. |
| SQLAlchemy and other ORMs | An extra abstraction over a small, fixed schema, with pgvector and full-text queries that are clearer in SQL. | `psycopg` 3 with parameterized SQL in `app/db/`. |
| Chroma, FAISS, Pinecone, Weaviate | A second datastore would split state from Postgres and complicate hybrid retrieval. | PostgreSQL full-text search plus `pgvector`. |
| OpenAI SDK, Anthropic SDK, Google GenAI SDK, and other hosted-LLM SDKs | The product must stay free and run locally with no API keys or data leaving the laptop, and one client keeps traces and token counts comparable. | The `ollama` Python library, called only from `app/llm.py`. |
| LiteLLM and other multi-provider LLM gateways | Add a layer between the app and the model that we do not need with a single local runtime. | The `ollama` Python library. |
| PyPDF2 | Unmaintained, and weaker page-level extraction. | PyMuPDF. |
| `ics` (library) and hand-built `.ics` strings | Hand-built calendar text easily breaks on escaping and time zones. | `icalendar`. |
| `python-constraint`, `pulp`, or a hand-written search loop for scheduling | Less reliable for guaranteeing conflict-free schedules, and hard to test against. | OR-Tools CP-SAT. |
| `requests` | We want a single HTTP client. | `httpx`. |
| `pickle`, `eval()`, `exec()` | Unsafe on untrusted input, including model output and uploaded files. | JSON with Pydantic models. |
| Selenium, Playwright, or scrapers against Class Roster or Canvas | Fragile, may violate site terms, and Canvas is out of scope. | The cached roster export or the official Class Roster API. |
| `unittest` and `nose` | Inconsistent test style across the team. | `pytest`. |
| Any new dependency not in `pyproject.toml` | Prevents unreviewed packages and supply-chain surprises. | Propose it in a pull request with a reason. |

## 5. Security Basics

**Secrets and configuration**
- No API keys are needed. `.env` holds only local configuration (Ollama address, model names, database URL). It is still listed in `.gitignore`, and `.env.example` holds placeholder values.
- If a teammate later adds any real credential (for example a Class Roster key), it goes only in `.env`. A secret-scanning pre-commit hook (`gitleaks`) runs on every commit, and any key that is ever committed must be rotated immediately.

**Local model safety**
- Ollama listens on `127.0.0.1` by default and must not be exposed to the network. Teammates should not set `OLLAMA_HOST=0.0.0.0` on a shared or public network.
- Models are pulled only from the official Ollama library and referenced by exact tag. The model names used are recorded in the README and evaluation reports.
- No student data leaves the laptop at runtime, because the model, database, embeddings, and solver all run locally.

**Untrusted input**
- Uploaded PDFs and model output are untrusted. Uploads are limited to PDF files under 20 MB, checked by file signature, not just extension.
- Text from uploaded documents is passed to the model as clearly delimited evidence, never as instructions. The materials agent's system prompt tells it to ignore instructions found inside documents.
- All model-generated tool arguments are validated with Pydantic before execution. Invalid calls return a structured error to the agent and are not run.
- Tools run only the fixed set of Python functions registered for that agent. The model cannot run arbitrary code, shell commands, or arbitrary SQL.

**Data**
- All SQL is parameterized. No string-built queries.
- Student data is session-scoped and limited to what the product needs (completed courses, preferences, uploaded syllabi). We do not collect NetIDs, names, or grades. Long-term profiles are out of scope.
- Traces and logs record tool names, arguments, and metrics. Uploaded document text is stored in the database, not in logs.
- Test and demo syllabi should be public or team-owned documents.

**Correctness as a safety property** (especially important with small local models)
- Every displayed schedule is re-validated by a separate checker against the encoded hard constraints.
- Citations are verified against the evidence retrieved for the current request before display. Unverifiable answers are reported as unverified.
- Prerequisite uncertainty is surfaced as `needs_confirmation`, never silently treated as met.

**Environment**
- The app binds to `localhost` only and has no user accounts.
- Containers run as a non-root user. Dependencies are pinned through `uv.lock`, and `uv lock --upgrade` changes are reviewed in pull requests.
- At runtime the app makes no outbound internet calls. The only network traffic is between the app container, the database container, and Ollama on the same laptop.

## 6. Code Examples

### 6.1 Typical endpoint

CoursePilot is a Streamlit app, so it does not expose a REST API. The "endpoint" is `handle_message` in `app/service.py`: the one function the UI calls for every student message. It loads session state, routes the message, runs the chosen agent, records the trace, and returns a structured response.

```python
# app/service.py
from __future__ import annotations

import time
import uuid

from pydantic import BaseModel, Field

from app.agents import materials, planner
from app.router import RouteDecision, route_message
from app.state import SessionState, load_state, save_state
from app.tracing import Trace, record_trace


class Citation(BaseModel):
    chunk_id: str
    document: str
    section: str | None = None
    pages: str
    text: str


class ChatResponse(BaseModel):
    trace_id: str
    intent: str
    reply: str
    clarifying_question: str | None = None
    schedules: list[dict] = Field(default_factory=list)  # up to 3, already validated
    citations: list[Citation] = Field(default_factory=list)


def handle_message(session_id: str, text: str) -> ChatResponse:
    """Single entry point called by the Streamlit chat UI for each student message."""
    if not text.strip():
        raise ValueError("Message must not be empty.")

    trace = Trace(trace_id=str(uuid.uuid4()), session_id=session_id)
    started = time.perf_counter()
    state: SessionState = load_state(session_id)

    decision: RouteDecision = route_message(text, state, trace=trace)
    if decision.clarifying_question:
        response = ChatResponse(
            trace_id=trace.trace_id,
            intent=decision.intent,
            reply=decision.clarifying_question,
            clarifying_question=decision.clarifying_question,
        )
    elif decision.intent in {"plan_schedule", "refine_schedule"}:
        response = planner.run(text, decision, state, trace=trace)
    elif decision.intent in {"ask_course_info", "ask_deadline", "export_calendar"}:
        response = materials.run(text, decision, state, trace=trace)
    else:  # out_of_scope
        response = ChatResponse(
            trace_id=trace.trace_id,
            intent="out_of_scope",
            reply="I can help with Cornell CS schedule planning and your course materials.",
        )

    state.append_turn(user=text, assistant=response.reply)
    save_state(session_id, state)
    trace.total_latency_ms = int((time.perf_counter() - started) * 1000)
    record_trace(trace)
    return response
```

Streamlit calls it like this:

```python
# app/ui/chat.py
import streamlit as st
from app.service import handle_message

if prompt := st.chat_input("Ask about schedules or your course materials"):
    result = handle_message(st.session_state["session_id"], prompt)
    st.chat_message("assistant").write(result.reply)
```

### 6.2 Typical function (deterministic tool)

`check_prereqs` in `app/tools/prereqs.py` evaluates a reviewed prerequisite rule tree. It uses three-valued logic so that an unverified equivalent produces `needs_confirmation` instead of a guess.

```python
# app/tools/prereqs.py
from __future__ import annotations

from enum import Enum
from typing import Literal

from pydantic import BaseModel, Field


class PrereqStatus(str, Enum):
    MET = "met"
    UNMET = "unmet"
    NEEDS_CONFIRMATION = "needs_confirmation"


class PrereqRule(BaseModel):
    """A reviewed prerequisite expression. `course_or_equiv` preserves 'or equivalent' branches."""
    op: Literal["course", "course_or_equiv", "and", "or"]
    course: str | None = None
    children: list["PrereqRule"] = Field(default_factory=list)


class CheckPrereqsInput(BaseModel):
    course_id: str
    completed_courses: list[str]
    reported_equivalents: list[str] = Field(
        default_factory=list,
        description="Courses the student says they have an unverified equivalent for.",
    )


class CheckPrereqsOutput(BaseModel):
    course_id: str
    status: PrereqStatus
    rule_text: str
    reason: str


def _evaluate(rule: PrereqRule, done: set[str], equiv: set[str]) -> tuple[PrereqStatus, str]:
    if rule.op == "course":
        ok = rule.course in done
        return (PrereqStatus.MET if ok else PrereqStatus.UNMET,
                f"{rule.course} {'completed' if ok else 'not completed'}")
    if rule.op == "course_or_equiv":
        if rule.course in done:
            return PrereqStatus.MET, f"{rule.course} completed"
        if rule.course in equiv:
            return PrereqStatus.NEEDS_CONFIRMATION, f"unverified equivalent to {rule.course}"
        return PrereqStatus.UNMET, f"{rule.course} or equivalent not completed"

    results = [_evaluate(child, done, equiv) for child in rule.children]
    statuses = [status for status, _ in results]
    reasons = "; ".join(reason for _, reason in results)
    if rule.op == "and":
        if PrereqStatus.UNMET in statuses:
            return PrereqStatus.UNMET, reasons
        if PrereqStatus.NEEDS_CONFIRMATION in statuses:
            return PrereqStatus.NEEDS_CONFIRMATION, reasons
        return PrereqStatus.MET, reasons
    # op == "or"
    if PrereqStatus.MET in statuses:
        return PrereqStatus.MET, reasons
    if PrereqStatus.NEEDS_CONFIRMATION in statuses:
        return PrereqStatus.NEEDS_CONFIRMATION, reasons
    return PrereqStatus.UNMET, reasons


def check_prereqs(args: CheckPrereqsInput, rule: PrereqRule, rule_text: str) -> CheckPrereqsOutput:
    """Check one course's prerequisites against the student's completed courses."""
    status, reason = _evaluate(rule, set(args.completed_courses), set(args.reported_equivalents))
    return CheckPrereqsOutput(
        course_id=args.course_id, status=status, rule_text=rule_text, reason=reason
    )
```

### 6.3 Typical test

This test covers the "CS 2110 or equivalent, and MATH 1920" example from the README, with no LLM and no database.

```python
# tests/tools/test_prereqs.py
import pytest

from app.tools.prereqs import (
    CheckPrereqsInput, PrereqRule, PrereqStatus, check_prereqs,
)

RULE = PrereqRule(
    op="and",
    children=[
        PrereqRule(op="course_or_equiv", course="CS 2110"),
        PrereqRule(op="course", course="MATH 1920"),
    ],
)
RULE_TEXT = "CS 2110 or equivalent, and MATH 1920"


def run(completed, equivalents=()):
    args = CheckPrereqsInput(
        course_id="CS 4780",
        completed_courses=list(completed),
        reported_equivalents=list(equivalents),
    )
    return check_prereqs(args, RULE, RULE_TEXT)


def test_all_prereqs_completed_is_met():
    assert run(["CS 2110", "MATH 1920"]).status == PrereqStatus.MET


def test_unverified_equivalent_needs_confirmation():
    result = run(["MATH 1920"], equivalents=["CS 2110"])
    assert result.status == PrereqStatus.NEEDS_CONFIRMATION
    assert "unverified equivalent to CS 2110" in result.reason


@pytest.mark.parametrize("completed", [["CS 2110"], [], ["MATH 1910"]])
def test_missing_required_course_is_unmet(completed):
    assert run(completed, equivalents=["CS 2110"]).status == PrereqStatus.UNMET
```

### 6.4 LLM client wrapper (the only module that calls Ollama)

All model calls go through `app/llm.py`. It applies the settings every call needs (explicit context size, temperature 0), returns token counts for the trace, and retries once when structured output fails validation. Unit tests replace these functions with fakes.

```python
# app/llm.py
from __future__ import annotations

import os
from typing import TypeVar

from ollama import Client
from pydantic import BaseModel, ValidationError

_client = Client(host=os.environ.get("OLLAMA_HOST", "http://localhost:11434"))
_OPTIONS = {"num_ctx": int(os.environ.get("OLLAMA_NUM_CTX", "8192")), "temperature": 0}

T = TypeVar("T", bound=BaseModel)


class ToolCall(BaseModel):
    name: str
    arguments: dict


class LLMResult(BaseModel):
    content: str
    tool_calls: list[ToolCall]
    prompt_tokens: int
    completion_tokens: int


def chat(model: str, messages: list[dict], tools: list | None = None) -> LLMResult:
    """One model call with optional tools. Tool arguments are validated by the caller."""
    resp = _client.chat(model=model, messages=messages, tools=tools, options=_OPTIONS)
    calls = [
        ToolCall(name=c.function.name, arguments=dict(c.function.arguments))
        for c in (resp.message.tool_calls or [])
    ]
    return LLMResult(
        content=resp.message.content or "",
        tool_calls=calls,
        prompt_tokens=resp.prompt_eval_count or 0,
        completion_tokens=resp.eval_count or 0,
    )


def chat_structured(model: str, messages: list[dict], schema: type[T]) -> T:
    """Ask for JSON matching a Pydantic schema; retry once with the validation error."""
    attempt_messages = list(messages)
    for attempt in range(2):
        resp = _client.chat(
            model=model,
            messages=attempt_messages,
            format=schema.model_json_schema(),
            options=_OPTIONS,
        )
        try:
            return schema.model_validate_json(resp.message.content or "")
        except ValidationError as err:
            if attempt == 1:
                raise
            attempt_messages = attempt_messages + [
                {"role": "user", "content": f"Your last reply was invalid: {err}. Reply again with valid JSON."}
            ]
    raise RuntimeError("unreachable")
```
