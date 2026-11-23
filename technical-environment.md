# CoursePilot: Technical Environment

This document records the technical choices for CoursePilot, an AI agent that helps Cornell CS students plan a semester schedule and manage course deadlines. AI-DLC should treat these decisions as constraints. The product runs locally on team laptops for the demo, and nothing is deployed or hosted online.

## 1. Language, Package Manager, and Framework

| Area | Choice | Notes |
|---|---|---|
| Language | Python 3.12 | One language for the UI, agents, tools, solver, and evaluation runner. Type hints are required on all function signatures. |
| Package manager | `uv` | Dependencies are declared in `pyproject.toml`, and `uv.lock` is committed so every teammate gets identical versions. Run `uv sync` to install and `uv run <cmd>` to execute. |
| UI framework | Streamlit (>= 1.40) | Chat interface, schedule weekly view, citation viewer, deadline review table, and trace viewer. |
| Data validation | Pydantic v2 | Every tool input and output, router decision, and session state object is a Pydantic model. |
| LLM access | Anthropic Python SDK (Claude API) | Router and query rewriting use a Haiku-class model. Planning and answer composition use a Sonnet-class model. Model IDs are set in `.env` (`ROUTER_MODEL`, `AGENT_MODEL`), not hard-coded. |
| Database | PostgreSQL 16 with `pgvector` | Stores courses, sections, prerequisite rules, requirement categories, sessions, messages, document chunks, embeddings, confirmed events, and traces. Accessed with `psycopg` 3 and plain parameterized SQL. |
| Constraint solver | Google OR-Tools CP-SAT | Selects compatible sections and enforces hard constraints. |
| PDF parsing | PyMuPDF (`pymupdf`) | Text-based PDFs only. OCR is out of scope. |
| Embeddings | `bge-small-en-v1.5` via `sentence-transformers`, running locally | Used for document-chunk and course-description search. Downloaded at image build time. |
| Calendar export | `icalendar` | Generates `.ics` files from confirmed events only. |
| HTTP client | `httpx` | For any outbound HTTP, such as the Class Roster API if we use it instead of a cached export. |
| Lint and format | `ruff` (lint and format), `mypy` (type check) | Run before every pull request. |

Layout of the application code (matches the README):

```text
app/
  ui/           # Streamlit pages: chat, schedule view, citations, deadline review, traces
  service.py    # handle_message(): single entry point the UI calls
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

**Deployment model: local web app run with Docker Compose.**

- `db` service: PostgreSQL 16 with the `pgvector` extension and a named volume so data persists between runs.
- `app` service: the Streamlit app, the agents, the Python tools, the OR-Tools solver, and the local embedding model, all in one container. The app is served at `http://localhost:8501`, bound to `localhost` only.
- A seed script runs on first start. It loads the cached Class Roster data and the team-reviewed prerequisite and requirement rules. It is idempotent, so rerunning it never duplicates rows.
- **LLM provider:** the hosted Claude API, called over the internet using `ANTHROPIC_API_KEY`. This is the only external service. Embeddings, retrieval, and solving all run locally.
- Class Roster data is cached in `data/` for one term. The app does not call Class Roster at runtime during the demo.

Setup:

```bash
cp .env.example .env
# Set ANTHROPIC_API_KEY and DATABASE_URL in .env
docker compose up --build
```

Each teammate uses their own API key. Keys are never shared in chat or committed.

## 3. Test Framework

| Purpose | Tool |
|---|---|
| Unit and integration tests | `pytest` |
| Coverage | `pytest-cov` (target: 80% on `app/tools/`, `app/state.py`, and the schedule checker) |
| Property-based tests for the solver and checker | `hypothesis` |
| Evaluation of LLM behavior | `python -m eval.run` (custom runner over versioned cases in `eval/`) |

Testing rules:

- **Deterministic code is tested without any LLM.** Prerequisite logic, the solver, the schedule checker, the `.ics` exporter, and citation validation must have unit tests that run offline and quickly.
- **Unit tests mock the Anthropic client.** No unit test may call the real Claude API. Tests that do are marked `@pytest.mark.live` and are excluded by default.
- **Database tests** run against a throwaway Postgres schema created by a fixture, never the demo database.
- **Hard-rule invariant:** a property-based test asserts that no schedule returned by `solve_schedule` ever has overlapping meetings, a missing required component, or credits outside the requested range.
- **Evaluation** uses separate development examples and frozen final cases, as described in the README. Reports include raw counts alongside percentages.

Commands:

```bash
uv run pytest                  # fast offline tests
uv run pytest -m live          # tests that call the real Claude API (needs a key)
uv run python -m eval.run      # evaluation runner
uv run ruff check . && uv run mypy app
```

## 4. Prohibited Libraries

| Library or practice | Reason | Use instead |
|---|---|---|
| LangChain, LlamaIndex, CrewAI, AutoGen | They hide the tool-calling loop, which is the core of this project, and make traces, the eight-call budget, and the course learning goals harder to demonstrate. | The Anthropic SDK with our own agent loop and Pydantic-validated tools. |
| SQLAlchemy and other ORMs | An extra abstraction over a small, fixed schema, with pgvector and full-text queries that are clearer in SQL. | `psycopg` 3 with parameterized SQL in `app/db/`. |
| Chroma, FAISS, Pinecone, Weaviate | A second datastore would split state from Postgres and complicate hybrid retrieval. | PostgreSQL full-text search plus `pgvector`. |
| OpenAI SDK and other LLM provider SDKs | The project standardizes on the Claude API so cost, tokens, and traces are comparable. | The Anthropic Python SDK. |
| PyPDF2 | Unmaintained, and weaker page-level extraction. | PyMuPDF. |
| `ics` (library) and hand-built `.ics` strings | Hand-built calendar text easily breaks on escaping and time zones. | `icalendar`. |
| `python-constraint`, `pulp`, or a hand-written search loop for scheduling | Less reliable for guaranteeing conflict-free schedules, and hard to test against. | OR-Tools CP-SAT. |
| `requests` | We want a single HTTP client. | `httpx`. |
| `pickle`, `eval()`, `exec()` | Unsafe on untrusted input, including model output and uploaded files. | JSON with Pydantic models. |
| Selenium, Playwright, or scrapers against Class Roster or Canvas | Fragile, may violate site terms, and Canvas is out of scope. | The cached roster export or the official Class Roster API. |
| `unittest` and `nose` | Inconsistent test style across the team. | `pytest`. |
| Any new dependency not in `pyproject.toml` | Prevents unreviewed packages and supply-chain surprises. | Propose it in a pull request with a reason. |

## 5. Security Basics

**Secrets**
- `ANTHROPIC_API_KEY` and `DATABASE_URL` live only in `.env`, which is listed in `.gitignore`. `.env.example` holds placeholder names with no real values.
- A secret-scanning pre-commit hook (`gitleaks`) runs on every commit. Any key that is ever committed must be rotated immediately.

**Untrusted input**
- Uploaded PDFs and model output are untrusted. Uploads are limited to PDF files under 20 MB, checked by file signature, not just extension.
- Text from uploaded documents is passed to the model as clearly delimited evidence, never as instructions. The materials agent's system prompt tells it to ignore instructions found inside documents.
- All model-generated tool arguments are validated with Pydantic before execution. Invalid calls return a structured error to the agent and are not run.
- Tools run only the fixed set of Python functions registered for that agent. The model cannot run arbitrary code, shell commands, or arbitrary SQL.

**Data**
- All SQL is parameterized. No string-built queries.
- Student data is session-scoped and limited to what the product needs (completed courses, preferences, uploaded syllabi). We do not collect NetIDs, names, or grades. Long-term profiles are out of scope.
- Traces and logs record tool names, arguments, and metrics, but never API keys. Uploaded document text is stored in the database, not in logs.
- Test and demo syllabi should be public or team-owned documents.

**Correctness as a safety property**
- Every displayed schedule is re-validated by a separate checker against the encoded hard constraints.
- Citations are verified against the evidence retrieved for the current request before display. Unverifiable answers are reported as unverified.
- Prerequisite uncertainty is surfaced as `needs_confirmation`, never silently treated as met.

**Environment**
- The app binds to `localhost` only and has no user accounts.
- Containers run as a non-root user. Dependencies are pinned through `uv.lock`, and `uv lock --upgrade` changes are reviewed in pull requests.
- The only outbound traffic is to the Claude API (and the Class Roster API if used).

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
