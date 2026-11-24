# Components

Components follow the code layout in technical-environment.md §1. Owner areas: **DATA** course data and solver; **PLAN** planner, router and memory; **MAT** course materials and deadlines; **UX** UI, tracing and evaluation; **SHARED** is the Foundation, built first. Units Generation finalizes which unit each component belongs to.

Components marked 🔒 hold security-critical logic, which NFR-SEC-11 requires to live in dedicated modules.

---

## 1. Presentation Layer (`app/ui/`)

| ID | Component | Module | Owner | Purpose and responsibilities | Serves |
|---|---|---|---|---|---|
| UI-1 | Chat page | `app/ui/chat.py` | UX | The main page: the chat input and transcript, the **completed-courses sidebar** (multiselect plus free text), the **interpretation panel** (edit constraints and topic sets, the "default" credit-range label, pin or exclude solver-chosen courses), the **schedule comparison** (A, B and C, weekly view, conditions, "Keep this schedule"), the **citation viewer**, upload, and progress feedback (`st.status`). Calls only `app/service.py` | US-01–04, 07, 09, 11–13 |
| UI-2 | Deadline review page | `app/ui/deadlines.py` | UX | The per-document review table (wording, interpreted date, source, status), editing and confirming rows, the per-course anchor override, and the `.ics` download button. Calls only `app/service.py` | US-14, 15 |
| UI-3 | Trace viewer page | `app/ui/traces.py` | UX | Lists and filters traces and shows a trace's details. Calls only `app/service.py` | US-21 |

---

## 2. Service Layer

| ID | Component | Module | Owner | Purpose and responsibilities | Serves |
|---|---|---|---|---|---|
| SV-1 | Service facade 🔒 | `app/service.py` | SHARED (lead PLAN) | The **only** entry point for the UI (Q2:A). `handle_message` for chat, plus named action functions for button clicks. Each call creates a trace, loads and saves `SessionState`, validates its inputs, and runs inside the top-level error handler (SECURITY-15) | All stories |
| SV-2 | Planning service | `app/tools/planning.py` | DATA | **Phase 2 solve**, the single solving path used by both the button and the `solve_schedule` tool (Q1:C, Follow-up 1:A). It applies `ConstraintChanges`, gathers candidates, runs `check_prereqs` for each candidate, then the solver, then the **checker**, and drops any schedule that fails. Model-free | US-03, 05, 07, 08 |
| SV-3 | Document service | `app/tools/documents.py` | MAT | Upload pipeline: validation (through UP-1), text extraction, section detection, chunking, embedding and storage. Model-free | US-11 |

---

## 3. Routing and Agents (`app/router.py`, `app/agents/`)

| ID | Component | Module | Owner | Purpose and responsibilities | Serves |
|---|---|---|---|---|---|
| AG-1 | Router | `app/router.py` | PLAN | Classifies intent from the message, the last 3 turns and state, using the router model and returning a validated `RouteDecision`. Decides whether to ask a clarifying question (confidence threshold, ambiguous references) | US-02, 12, 13 |
| AG-2 | Agent loop 🔒 | `app/agents/loop.py` | SHARED (lead PLAN) | `run_agent(spec, …)` (Q6:A). The model asks for a tool, the loop checks it against the spec's **allowlist**, validates the arguments, runs the tool with the server-built `AgentContext`, and returns the result or a structured error. Enforces the 8-call budget, records each call in the trace, and runs the spec's finalizer. **The session and every ownership ID come from `AgentContext`, never from model arguments (AD-12)** | US-03, 12, MS-02 |
| AG-3 | Planner agent | `app/agents/planner.py` | PLAN | Two `AgentSpec`s (AD-11). `INTERPRET_SPEC` allows **only** `search_courses`, so phase 1 **cannot solve**. `REFINE_SPEC` allows `search_courses`, `check_prereqs`, `check_requirements` and `solve_schedule`. **Phase 1** (`INTERPRET_SPEC`): the request becomes an `Interpretation`. **Refinement** (`REFINE_SPEC`): the reference becomes `ConstraintChanges`, then the agent calls `solve_schedule`. **Explanation:** one model call that explains phase 2 results. Also maps topic phrases to `TOPIC_TAGS` | US-02, 03, 05, 07, 08 |
| AG-4 | Materials agent | `app/agents/materials.py` | MAT | Its `AgentSpec` (prompt telling the model to ignore instructions inside documents, `AGENT_MODEL`, tools `search_materials`, `extract_deadlines` and `export_ics`). Rewrites the query with the router model first (FR-MAT-05, Tier 2, with an on/off switch). Labels passages `[C1]…`, then sends the answer to CV-1 for validation, retrying once | US-12, 13, 15, 16, MS-01 |

---

## 4. Tools (`app/tools/`)

**Model-callable tools** (registered in an `AgentSpec`):

| ID | Tool | Module | Owner | Agent | Purpose | Serves |
|---|---|---|---|---|---|---|
| T-1 | `search_courses` | `courses.py` | DATA | Planner | Query plus filters, returning up to 10 courses with sections and tags. Hybrid search: full-text plus pgvector over descriptions | US-02, 03 |
| T-2 | `check_prereqs` | `prereqs.py` | DATA | Planner | Maps the none, trusted, uncertain and missing statuses to a result, evaluating trusted trees with three-valued logic (technical-environment.md §6.2) | US-04 |
| T-3 | `check_requirements` | `requirements.py` | DATA | Planner | Progress and gaps for each category. Uncertain categories return `needs_confirmation`. **Tier 2** | US-06 |
| T-4 | `solve_schedule` | `solve_tool.py` (tool wrapper) | DATA | Planner | Input is `ConstraintChanges` only, and the implementation calls **SV-2**. The model never passes raw solver inputs | US-07, MS-02 |
| T-5 | `search_materials` | `materials.py` | MAT | Materials | Hybrid retrieval: full-text plus pgvector, course and document filters, 20 + 20 results fused with RRF, top 5 returned | US-12 |
| T-6 | `extract_deadlines` | `deadlines.py` | MAT | Materials | Candidate events from a document. The `document_id` must pass `ownership.require_document`. The model extracts wording, and dates are resolved through DT-1 | US-14, 16 |
| T-7 | `export_ics` | `export.py` | MAT | Materials | Builds an `.ics` file from **confirmed** events only, using `icalendar`. Takes **no model arguments**: the session comes from `AgentContext` | US-15 |

Every model-callable tool receives `(args, ctx: AgentContext)`. Its input model never contains a session ID or student data such as completed courses; those come from `ctx` (AD-12).

**Deterministic support** (not model-callable):

| ID | Component | Module | Owner | Purpose | Serves |
|---|---|---|---|---|---|
| SO-1 | CP-SAT solver | `solver.py` | DATA | Builds and solves the CP-SAT model: hard constraints (FR-ELG-03, including inclusions and fill), the penalty for `needs_confirmation` courses, soft-preference scoring, distinctness and the time limit | US-03 |
| CK-1 | Schedule checker 🔒 | `checker.py` | DATA | **Independent** re-validation (Q4:A). Imports only the shared models, never the solver or its helpers. A test enforces this | US-03, NFR-COR-01 |
| CV-1 | Citation validator 🔒 | `citations.py` | MAT | Checks that cited IDs are in the evidence retrieved for this request, that quoted dates and numbers appear in the passage (allowing equivalent formats), and the DEC-24 relative-date labeling rule | US-12, MS-01 |
| DT-1 | Date resolver | `dates.py` | MAT | Anchor precedence (per-course override, then `TERM_START`, then the derived anchor), DEC-25 Monday–Sunday weeks, `TODAY_OVERRIDE`, and America/New_York | US-14, 16 |
| UP-1 | Upload validator 🔒 | `uploads.py` | MAT | Size limit (20 MB) and PDF signature checks before parsing, plus rejection of PDFs with no extractable text | US-11, MS-03 |
| OW-1 | Ownership checks 🔒 | `ownership.py` | SHARED (lead MAT) | **Server-side ownership checks (AD-12, a SECURITY-05/11 control).**<br/>- `require_document`: the document belongs to `ctx.session_id`, and its course is in the active schedule.<br/>- `require_event`: the event belongs to `ctx.session_id`.<br/>- `require_active_course`: the course is in the active schedule. Used for materials filters, uploads and anchor overrides.<br/>- `require_catalog_course` and `require_catalog_section`: the ID exists in the configured term's ingested catalog. Used for planner tool IDs, because catalog courses are shared, not owned by a session.<br/>Each failure returns `ToolError(kind="forbidden")` or `ToolError(kind="unknown_id")`. Called by every tool **and** every service action that receives such an ID | MS-02 |

---

## 5. Ingestion (`app/ingest/`)

| ID | Component | Module | Owner | Purpose | Serves |
|---|---|---|---|---|---|
| IN-1 | Course source adapter | `sources/base.py` (protocol), `sources/class_roster.py` | DATA | `CourseSource` protocol. The Class Roster adapter fetches with `httpx` and validates every API response with Pydantic | US-17 |
| IN-2 | Prerequisite status classifier 🔒 | `prereq_status.py` | DATA | Classifies each prerequisite field as none, text or missing (FR-ING-02), in Python only, using `NO_PREREQ_PATTERNS` | US-17, 18 |
| IN-3 | Deriver | `derive.py` | DATA | Model calls for rule trees, topic tags, workload cues and requirement categories. Untrusted text is wrapped in delimiters, and outputs are validated with Pydantic | US-18, 19, MS-01 |
| IN-4 | Trust gate 🔒 | `trust_gate.py` | DATA | Deterministic checks (a)–(e) from FR-ING-07, with the `UNEXPRESSIBLE_PHRASES` constant. Pure functions, so they can be **re-run by the seed script** (see the misuse case "edited committed cache") | US-18, 19 |
| IN-5 | Cache | `cache.py` | DATA | Stores derived records in `data/` (committed, Q5:A), keyed by a hash of (input text, model, prompt version). Makes runs resumable | US-18, 20 |
| IN-6 | Anchor deriver | `anchor.py` | DATA | Finds the most common lecture start date, with `TERM_START` overriding it | US-17 |
| IN-7 | Ingestion CLI | `__main__.py` | DATA | `python -m app.ingest`: progress display and end-of-run summary | US-17, 20 |
| IN-8 | Seed loader | `seed.py` | DATA | Idempotent load of `data/` into Postgres at first start. Re-validates each record's schema and re-runs the deterministic trust gate checks | US-17 |

---

## 6. Foundation (shared, built first)

| ID | Component | Module | Owner | Purpose | Serves |
|---|---|---|---|---|---|
| FN-1 | Settings 🔒 | `app/config.py` | SHARED | Pydantic settings loaded from the environment (models, `num_ctx`, `TERM`, `SUBJECTS`, `TOPIC_TAGS`, thresholds, `DEFAULT_CREDIT_RANGE`, `TODAY_OVERRIDE`, `TERM_START`, `REQUIREMENTS_DOC`, `DATABASE_URL`). **Refuses to start while `POSTGRES_PASSWORD` is the placeholder** (NFR-SEC-09) | All |
| FN-2 | LLM wrapper | `app/llm.py` | SHARED | `chat` and `chat_structured` (technical-environment.md §6.4). Sets `num_ctx`, temperature 0, and whether thinking is on, and reports token counts. The only module that imports `ollama` | All model use |
| FN-3 | Embeddings | `app/embeddings.py` | SHARED | Loads the `bge-small-en-v1.5` model once per process and provides `embed(texts)` | T-1, T-5, SV-3 |
| FN-4 | Database | `app/db/connection.py`, `migrations/*.sql`, `migrate.py` | SHARED | Connection pool and an ordered plain-SQL migration runner (Q8:A) | All repositories |
| FN-5 | Repositories | `app/db/{courses,derived,sessions,documents,events,traces}_repo.py` | Each area owns its repo | Typed functions that return Pydantic models. All SQL is parameterized | All persistence |
| FN-6 | Session state | `app/state.py` | PLAN | The `SessionState` model (with a schema version), `load_state` and `save_state` as one `jsonb` document (Q3:A), and the running summary trigger (Tier 2) | US-01, 09, 13 |
| FN-7 | Tracing and logging | `app/tracing.py` | UX | Creates `Trace`, records spans for tools and model calls, writes `record_trace`, and configures the structured logger (SECURITY-03). Never stores document text | US-21 |
| FN-8 | Errors 🔒 | `app/errors.py` | SHARED | `UserFacingError`, `safe_call` and the top-level handler: log with the `trace_id`, return a generic message, fail closed (SECURITY-15) | All |
| FN-9 | Shared models | `app/models/{courses,derived,planning,materials,events}.py` | SHARED (each area owns its file) | Pydantic models shared across components. This is the neutral module that lets the solver and the checker share **only data models** (Q4:A). It holds no logic beyond validators. Listed in technical-environment.md §1 (AD-09) | All |

---

## 7. Evaluation (`eval/`)

| ID | Component | Module | Owner | Purpose | Serves |
|---|---|---|---|---|---|
| EV-1 | Evaluation runner | `eval/run.py`, `eval/suites/*.py` | UX (suites from every owner) | Runs the 8 suites against `app.service` and the tools. Records configuration, latency and tokens, and writes the report. `app/` never imports `eval/` (FR-EVAL-05) | US-22 |

---

## Requirement-area Coverage

| FR area | Components |
|---|---|
| FR-ING | IN-1–IN-8, FN-3 |
| FR-PLN | AG-3, SV-2, T-1, UI-1 |
| FR-ELG | T-2, T-3, T-4, SV-2, SO-1, CK-1 |
| FR-REF | AG-3, T-4, SV-2 |
| FR-SEL | SV-1, FN-6 |
| FR-MAT | SV-3, UP-1, T-5, AG-4, CV-1, UI-1 |
| FR-DL | T-6, T-7, DT-1, UI-2, FN-5 (`events_repo`) |
| FR-RTE | AG-1, SV-1 |
| FR-AGT | AG-2, AG-3, AG-4 |
| FR-MEM | FN-6, AG-1 |
| FR-TRC | FN-7, UI-3 |
| FR-EVAL | EV-1 |
| FR-UI | UI-1, UI-2, UI-3 |
