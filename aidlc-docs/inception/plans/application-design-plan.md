# Application Design Plan

**Stage**: INCEPTION - Application Design
**Inputs**: `requirements.md` (79 FRs, DEC-01 to DEC-25), `stories.md` (25 stories, Tier 1 demo script), `technical-environment.md` (code layout in §1, `handle_message` and `app/llm.py` examples in §6)
**Outputs** (in `aidlc-docs/inception/application-design/`): `components.md`, `component-methods.md`, `services.md`, `component-dependency.md`, `application-design.md`

This stage defines **components, interfaces and how they're orchestrated**. Detailed business rules (the CP-SAT model, the trust gate checks, RRF scoring, date arithmetic) are left to Functional Design for each unit. Several questions below matter because four owners will build against these interfaces in parallel.

---

## 1. Fixed Starting Points

These come from earlier stages and are not re-asked:
- **Code layout:** technical-environment.md §1, which covers `app/ui`, `service.py`, `llm.py`, `router.py`, `agents/`, `tools/`, `ingest/`, `db/`, `state.py` and `tracing.py`.
- **Model calls:** `app/llm.py` is the only module that calls Ollama (TC-03).
- **Validation:** every tool input and output, router decision and state object is a Pydantic model (TC-02).
- **Tools:** 7, split per agent (FR-AGT-04). The shared agent loop has an 8-call budget (FR-AGT-01 to 03).
- **SQL** lives only in `app/db/`, using `psycopg` 3 with parameterized SQL (TC-04).
- **Checker:** a separate checker re-validates schedules (FR-ELG-08).
- **Security:** security-critical logic lives in dedicated modules (NFR-SEC-11).

---

## 2. Execution Checklist

- [x] **Step 1: Components.** Write `components.md`: every component's purpose, responsibilities, interface summary, owner area (DATA, PLAN, MAT or UX), and the requirements and stories it serves.
- [x] **Step 2: Methods.** Write `component-methods.md`: signatures with Pydantic input and output types for every public method. This includes the 7 tools, the router, the agent loop, the service facade, repositories, ingestion, the checker and the validators. Business rules are left out.
- [x] **Step 3: Services.** Write `services.md`: service definitions and orchestration flows for the Tier 1 demo steps (plan, keep, upload, ask, review, export) and for ingestion, following the answers to Q1–Q3.
- [x] **Step 4: Dependencies.** Write `component-dependency.md`: a dependency matrix, the communication patterns, and data-flow diagrams (Mermaid with text alternatives), with no dependency cycles.
- [x] **Step 5: Misuse cases.** Document the misuse cases in the design (NFR-SEC-11): MS-01 to MS-03 plus any new ones found while designing, each mapped to the component that defends against it.
- [x] **Step 6: Consolidate.** Write `application-design.md`, which consolidates the four documents and records the decisions from §3.
- [x] **Step 7: Validate.** Check that every Tier 1 story step in the demo script is served by a defined flow, that every FR maps to a component, that the dependency graph has no cycles, that the Mermaid syntax is valid, and that security and PBT compliance hold.

---

## 3. Questions

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe your answer after the tag.

### Question 1: Who runs the solver after the student confirms the interpretation panel?
US-02 and US-03 split planning into two parts: **interpret**, after which the student edits the panel, then **solve** when the student clicks "Find schedules". After a button click, the edited constraints are already structured, so a model isn't needed to choose which tools to call.

A) **Two-phase, with deterministic solving (recommended).** For phase 1, the planner agent runs its loop and returns a structured, editable `Interpretation`. For phase 2, the "Find schedules" button calls a service method that runs eligibility checks, the solver and the checker **in Python without the model**. The model is then called once only to write the explanation of the results. Pinning and excluding (DEC-22) re-run phase 2 directly

B) **Everything in the agent loop.** The edited interpretation is sent back to the planner agent as a message, and the model decides when to call `check_prereqs` and `solve_schedule`

C) **Hybrid.** Phase 2 is deterministic for button actions, but chat refinements ("swap out the 8am") go through the agent loop, which ends by calling the same phase 2 service

X) Other (please describe after [Answer]: tag below)

[Answer]: C) Hybrid. Button actions ("Find schedules", pin, exclude) call the deterministic phase 2 service directly: eligibility checks, solver and checker in Python, then one model call to explain the results. Chat refinements ("swap out the 8am", "no classes before 9am") go through the planner's agent loop, which resolves the reference into structured constraint changes and then calls the same phase 2 service. This keeps one solving path for both entry points, while still letting the model interpret free-text refinements that A doesn't cover.

### Question 2: What does the UI call besides `handle_message`?
technical-environment.md §6.1 makes `handle_message` the single entry point for **chat messages**. Several actions are button clicks, not chat messages: keep a schedule, pin or exclude a course, upload a PDF, confirm or edit a deadline row, set an anchor override, download the `.ics` file, and change completed courses in the sidebar.

A) **A service facade (recommended).** `app/service.py` exposes `handle_message` plus explicitly named action functions (`keep_schedule`, `solve_confirmed`, `upload_document`, `update_event`, `export_calendar`, `set_completed_courses`, …). Each one is validated, traced, and runs through the same top-level error handler. The UI never imports agents, tools or `app/db/` directly

B) **Everything through `handle_message`.** Button clicks are turned into synthetic chat messages ("keep schedule A"), so every action goes through routing

C) **The UI calls tools and repositories directly** for button actions, and only chat goes through `service.py`

X) Other (please describe after [Answer]: tag below)

[Answer]: A) A service facade in `app/service.py`: `handle_message` plus named action functions, each validated, traced and run through the same top-level error handler. The UI never imports agents, tools or `app/db/` directly.

### Question 3: How is session state persisted?
`SessionState` (FR-MEM-01) holds the completed courses, constraints, interpretation, candidate schedules, active schedule, current course and assignment, and the running summary.

A) **One validated JSON document per session (recommended).** `SessionState` is stored as a `jsonb` row and loaded and saved whole once per request, with a schema version field. Messages, events and their audit records, documents and traces stay in their own tables, because they are queried on their own

B) **Fully normalized tables** for constraints, schedules and so on

C) **Streamlit `st.session_state` only**, with nothing persisted to Postgres

X) Other (please describe after [Answer]: tag below)

[Answer]: A) One validated JSON document per session (`jsonb`, with a schema version), loaded and saved whole once per request. Messages, events and their audit records, documents and traces stay in their own tables.

### Question 4: How independent is the separate checker from the solver?
FR-ELG-08 requires a separate checker, independent of the solver. A bug shared by both (for example, wrong meeting-overlap logic in a shared helper) would let a violating schedule through both.

A) **Independent logic (recommended).** `app/tools/checker.py` shares only the Pydantic data models with the solver. It has its own overlap, component, credit, inclusion and exclusion logic written directly in Python, and it never imports from `app/tools/solver.py` or from the solver's helpers. A lint or test rule enforces this

B) **Separate module, shared helpers.** The checker and the solver both use a common `meetings.py` time-overlap utility

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Independent logic. The checker shares only the Pydantic data models with the solver and never imports solver code or helpers, enforced by a test or lint rule, so one bug can't get past both.

### Question 5: Should the ingestion cache be committed to git?
Ingestion needs internet access and Ollama, and may take a long time on a laptop. Its output is deterministic for the same input, model and prompt version (FR-ING-10).

A) **Commit it (recommended).** One teammate runs ingestion. The raw export and the derived cache in `data/` (JSON keyed by input hash) are committed, so every teammate and the demo use identical derived rules, and the seed script loads them offline. Re-running ingestion is only needed for a new term, a model change or a prompt change

B) **Don't commit.** Every teammate runs ingestion locally, and `data/` is in `.gitignore`

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Commit it. The raw export and the derived cache are generated automatically (no hand edits), so committing them keeps every teammate and the demo on identical rules, and the seed script works offline. Re-run ingestion only for a new term, model or prompt version.

### Question 6: How should the shared agent loop be structured?
Both agents share one loop (FR-AGT-01). They differ in prompt, model, tool registry and how they finish.

A) **Data-driven (recommended).** One `run_agent(spec: AgentSpec, …)` function, where `AgentSpec` is a Pydantic model holding the system prompt, model name, an allowlisted tool registry (name mapped to input model, output model and function), and a finalizer. `planner.py` and `materials.py` each define a spec plus their own pre- and post-processing

B) **Inheritance.** A `BaseAgent` class with `PlannerAgent` and `MaterialsAgent` subclasses that override hooks

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Data-driven: one `run_agent(spec: AgentSpec, …)` with an allowlisted tool registry per spec. The allowlist in the spec is also what enforces MS-02 (an agent can't call another agent's tools).

### Question 7: How should slow work run inside Streamlit?
Some steps take seconds: a full agent answer (about 30 s), embedding an uploaded PDF, and the solver.

A) **Synchronous, with progress feedback (recommended).** Service calls block the Streamlit script run, and the UI shows `st.status` or a spinner with step updates (routing, searching, solving, checking). This is simple for a single local user

B) **Background workers.** Uploads and solving run in a thread pool or task queue, and the UI polls for results

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Synchronous with progress feedback (`st.status` or a spinner with step updates). Simplest for a single local user, and the solver's time limit (NFR-PERF-03) bounds the wait.

### Question 8: How should database access be organized?
All SQL lives in `app/db/` (TC-04).

A) **One repository module per aggregate (recommended):** `courses_repo` (courses, sections, meetings), `derived_repo` (rules, tags, workload, categories, cache index), `sessions_repo` (state, messages, summary), `documents_repo` (documents, chunks, embeddings, hybrid search queries), `events_repo` (events and their audit records), and `traces_repo`. Each exposes typed functions that return Pydantic models. Migrations are plain SQL files run in order by a small runner

B) **A single `db/queries.py`** holding every query

X) Other (please describe after [Answer]: tag below)

[Answer]: A) One repository module per aggregate, with typed functions returning Pydantic models and plain-SQL migrations run in order.

---

## 4. Follow-up Question (from the Q1 answer)

### Follow-up 1: How does a chat refinement reach the phase 2 solve service?
Your Q1:C answer says chat refinements go through the planner's agent loop, which "resolves the reference into structured constraint changes and then calls the same phase 2 service". There are two ways to build that, and they interact with two approved requirements:
- **FR-AGT-04** lists `solve_schedule` as one of the planner's **model-callable** tools.
- **FR-PLN-05** says the planner MUST use `search_courses`, `check_prereqs`, `check_requirements` and `solve_schedule`.

With Q1:C, initial planning ends at the editable interpretation panel, and the button path calls phase 2 without the model. So the only place the model could still call `solve_schedule` is in refinements.

A) **`solve_schedule` stays a model-callable tool (recommended).** Its input is a validated `ConstraintChanges` (for example: replace section X of a course, add a hard constraint, remove a constraint, pin a course, exclude a course), and its implementation calls the phase 2 service (eligibility checks, then the solver, then the checker). The button path calls the same phase 2 service directly. Both requirements stay as written. The refinement's solve shows up in the trace as a tool call and counts toward the 8-call budget. The model can't pass a raw solver model, only constraint changes, so MS-02 still holds

B) **The loop ends with a structured `ConstraintChanges` output, and no solve tool is called.** The service applies the changes and calls phase 2 itself. `solve_schedule` leaves the model's tool registry, and FR-AGT-04 and FR-PLN-05 are amended to say the service, not the model, invokes the solver

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Keep `solve_schedule` as a model-callable tool whose input is a validated list of constraint changes, implemented by calling the same solve service the button uses. This is what we intended with Q1:C: one solving path, FR-AGT-04 and FR-PLN-05 unchanged, refinements visible in traces within the 8-call budget, and no raw solver inputs from the model, so MS-02 still holds.
