# Units of Work

**Stage**: INCEPTION - Units Generation
**Plan**: `aidlc-docs/inception/plans/unit-of-work-plan.md` (Q1–Q8 answered and approved)
**Deployment model**: one application, deployed as one `app` container plus `db` (technical-environment.md §2). Units are **modules** of one Python package, grouped for development and for the per-unit Construction loop. None is a separately deployed service.

Owner areas: **DATA** course data and solver; **PLAN** planner, router and memory; **MAT** course materials and deadlines; **UX** UI, tracing and evaluation. Component IDs refer to `components.md`.

---

## Summary

| Unit | Name | Primary owner | Secondary | Tiers | Components |
|---|---|---|---|---|---|
| **U0** | Foundation | All four, one integrator (Q2:A) | — | 1 | FN-1–FN-9, AG-2, OW-1, SV-1 skeleton, Docker and tooling, test fakes |
| **U1** | Course Ingestion | DATA | UX (first week, Q1:A) | 1, 2 | IN-1–IN-8, `courses_repo` and `derived_repo` write side |
| **U2** | Eligibility and Solver | DATA | — | 1, 2 | T-2, T-3, T-4 (`solve_tool`), SV-2, SO-1, CK-1 |
| **U3** | Planner, Router and Memory | PLAN | — | 1, 2, 3 | AG-1, AG-3, T-1, FN-6 memory behavior, the planning service actions |
| **U4** | Materials and Deadlines | MAT | — | 1, 2 | UP-1, SV-3, T-5, CV-1, DT-1, T-6, T-7, AG-4, `documents_repo`, `events_repo`, the materials service actions |
| **U5** | UI and Evaluation | UX | All (evaluation suites, Q8:A) | 1, 2 | UI-1–UI-3, EV-1, the trace service actions |

---

## U0: Foundation

**Purpose.** Provide the shared contracts and infrastructure that let U1–U4 build in parallel from day 2 (Q3:A).

**Who builds it (Q2:A).** All four owners build it together in about 1–2 days, each writing the pieces closest to their area, and one teammate integrates and reviews. Suggested split:

| Owner | U0 pieces |
|---|---|
| PLAN | `agents/loop.py` (`AgentSpec`, `AgentContext`, `run_agent`, `ToolError`), `state.py`, `sessions_repo` |
| UX | `tracing.py`, `traces_repo`, `errors.py`, Docker and Compose, `pyproject.toml` and `uv.lock`, the `ruff`, `mypy`, `pytest` and `gitleaks` setup |
| DATA | `models/courses.py`, `models/derived.py`, `models/planning.py`, the read side of `courses_repo` and `derived_repo`, the **fixture catalog** |
| MAT | `models/materials.py`, `models/events.py`, `tools/ownership.py` 🔒, the `documents` and `events` tables |
| Integrator (any) | `config.py`, `llm.py`, `embeddings.py`, `db/connection.py`, `db/migrate.py`, the `service.py` skeleton (`handle_message` plus `errors.top_level` and tracing, with placeholder routing) |

**Responsibilities.**
- **Contracts:** settings, the LLM wrapper, embeddings, the connection and migration runner, and the **v1 schema for every table known from the design**. That covers courses, sections, meetings, derived records, sessions, messages, documents, chunks, events, event audit records and traces, so every unit starts with tables in place. Also the shared models, the agent loop with allowlists and `AgentContext`, the ownership checks, session state, tracing and logging, the error handling, and the service facade skeleton.
- **Test fakes and stand-ins (Q3:A):**
  - `tests/fakes/llm.py`: a scripted fake for `chat` and `chat_structured`.
  - `tests/fixtures/catalog/`: about 20 courses in the **ingestion output format**, covering trusted, uncertain, none and missing prerequisite statuses, `ml` tags (trusted and uncertain), a Friday-only section and an 8 AM section. Loadable by a fixture seed.
  - `tests/fakes/planning.py`: a stub `solve_with_checks` that returns a canned `SolveResult`.
  - `tests/fakes/session.py`: a builder for `SessionState` and `AgentContext`, including an active schedule.
- **Import-boundary tests** for rules R1–R12.

**Done when:**
- `docker compose up` starts the app, with Postgres on an unpublished port, a non-root container and pinned images.
- The app refuses to start while the password is still the placeholder.
- Migrations apply cleanly, and the fixture catalog loads.
- `handle_message` returns a traced placeholder reply.
- `run_agent` runs a fake two-tool spec with allowlist, validation and budget tests.
- The ownership checks have tests that use another session's IDs.
- The import-boundary tests pass, and `ruff`, `mypy` and `pytest` are green.

---

## U1: Course Ingestion

**Purpose.** Build the term's course data and every derived record automatically, then cache and seed them.

**Owner.** DATA, with UX as secondary owner during the first week (Q1:A).

**Tier 1 pass:**
- IN-1 Class Roster adapter, run with `SUBJECTS=CS`.
- IN-2 none/text/missing classification.
- IN-3 prerequisite rules and topic tags.
- IN-4 trust gate for rules.
- IN-5 cache.
- IN-6 anchor.
- IN-7 CLI.
- IN-8 seed with gate re-check.
- The `courses_repo` and `derived_repo` write side.
- The committed `data/` cache (Q5:A from Application Design).

**Tier 2 pass:**
- Workload signals and requirement categories (IN-3 and IN-4 for categories).
- Resumable runs with a summary.
- Adding MATH to `SUBJECTS`.
- The prerequisite-parsing and topic-tagging **evaluation suites**, with their cases (Q8:A).

**Provides to others.** Real derived data in the same format as the U0 fixture catalog (integration point IP-1).

**Done when (Tier 1):**
- `uv run python -m app.ingest --term FA26 --subjects CS` completes, and `data/` is committed.
- The seed loads it idempotently.
- The US-17 and US-18 criteria pass, including the trust gate property tests (NFR-TEST-05(d)).

---

## U2: Eligibility and Solver

**Purpose.** Deterministic eligibility, solving and independent checking: the single solving path.

**Owner.** DATA.

**Tier 1 pass:**
- T-2 `check_prereqs`, a tool wrapper that takes completed courses from `ctx`.
- SV-2 `solve_with_checks`, including inclusions and fill (DEC-22, 23).
- SO-1 CP-SAT with distinctness, the `needs_confirmation` penalty and the time limit.
- CK-1 the independent checker.
- T-4 `solve_tool`.

**Tier 2 pass:**
- T-3 `check_requirements`.
- Requirement-gap ranking for fill courses.
- Workload penalty support.
- Infeasibility explanation data (FR-ELG-09).
- The **planning evaluation suite** (15 cases).

**Provides to others.** The real `solve_with_checks`, which replaces U0's stub for U3 (IP-2).

**Done when (Tier 1):**
- The US-03 and US-04 criteria pass against the fixture catalog, and then against U1's data.
- The NFR-TEST-05(a), (b) and (c) property tests pass.
- The checker passes its import-boundary test (R4).

---

## U3: Planner, Router and Memory

**Purpose.** Turn chat into interpretations and refinements. Route every message. Own the planning-side service actions and the session-memory behavior.

**Owner.** PLAN.

**Tier 1 pass:**
- AG-1 router, with `RouteDecision` and the clarification checks.
- AG-3 planner: `INTERPRET_SPEC` with `interpret`, and `explain`.
- T-1 `search_courses`.
- Service actions: `handle_message` routing, `set_completed_courses`, `solve_confirmed`, `keep_schedule`.
- The active-schedule handoff in `SessionState` (FR-SEL, FR-MEM-01 and 03).

**Tier 2 pass:**
- `REFINE_SPEC` with `refine` and `apply_changes` (US-07).
- "Nothing too heavy" interpretation (US-05).
- Topic-switch re-routing, `out_of_scope` replies and the running summary (FR-RTE-03 to 05, FR-MEM-02).
- The **routing** (50 messages) and **session-memory** (5 conversations) evaluation suites.

**Tier 3.** Completed-course entry in chat (US-10), only if Tier 2 is done.

**Provides to others.** The active schedule's course IDs in state (IP-3), and the planning service actions to U5 (IP-4).

**Done when (Tier 1):** the US-01, 02 and 09 criteria pass with the fake LLM, and live tests (`-m live`) cover the demo messages in steps 2 and 8.

---

## U4: Materials and Deadlines

**Purpose.** Upload, cited Q&A, deadline extraction and review, and `.ics` export.

**Owner.** MAT.

**Tier 1 pass:**
- UP-1 upload validation and SV-3 the document pipeline.
- T-5 `search_materials` (FTS + vector + RRF).
- AG-4 materials agent.
- CV-1 citation validation (DEC-24).
- DT-1 dates (anchor precedence, DEC-25 weeks).
- T-6 `extract_deadlines`.
- T-7 `export_ics`.
- `documents_repo` and `events_repo` (session-filtered).
- Service actions: `upload_document`, `open_deadline_review`, `update_event`, `set_anchor_override`, `export_calendar`.
- MS-01 and MS-03.

**Tier 2 pass:**
- Query rewriting with its on/off switch (US-13).
- "What's due this week?" (US-16).
- The **retrieval and cited Q&A**, **query-rewriting** and **deadline-extraction** evaluation suites.

**Done when (Tier 1):**
- The US-11, 12, 14 and 15 criteria and the MS-01 and MS-03 criteria pass.
- The `.ics` round-trip property test (NFR-TEST-05(e)) passes.
- A manual import into Google Calendar is recorded.

---

## U5: UI and Evaluation

**Purpose.** Streamlit pages over the service facade, plus the evaluation runner and the trace viewer.

**Owner.** UX. Feature owners own their evaluation suites (Q8:A).

**Tier 1 pass:**
- UI-1 chat page: sidebar, interpretation panel, schedule comparison, keep, upload, citation viewer.
- UI-2 deadline review page with download.
- Built against each unit's service responses as they land (Q7:A).

**Tier 2 pass:**
- UI-3 trace viewer, with `list_traces` and `get_trace`.
- Progress indicators (FR-UI-02).
- The requirement-category display (FR-UI-04).
- EV-1 `eval/run.py`: the shared harness and report format, plus the coverage report.

**UI contract (Q7:A).** A feature unit is done when its service action returns everything its stories' UI checklists need. UX reviews any PR that changes a service response model.

**Done when (Tier 1):** the 14-step demo script runs through the UI against the real units.

---

## Code Organization (greenfield)

The AI-DLC default for a multi-unit monolith is `src/{unit-name}/`. **This project instead uses the layered `app/` layout that technical-environment.md §1 makes binding (TC-01, NFR-PORT-04).** Units map onto that layout by path, not by top-level folders.

```text
<workspace root>/
  pyproject.toml, uv.lock, Dockerfile, docker-compose.yml, .env.example   # U0
  app/
    config.py, errors.py, llm.py, embeddings.py, tracing.py, state.py      # U0
    service.py            # U0 skeleton; action functions added by U3, U4, U5
    models/               # U0; each area owns its file (courses/derived/planning: DATA, materials/events: MAT)
    db/connection.py, migrate.py, migrations/0001_*.sql                    # U0 (v1 schema)
    db/migrations/01xx_*.sql   # U1    02xx: U2    03xx: U3    04xx: U4    05xx: U5
    db/sessions_repo.py, traces_repo.py                                    # U0
    db/courses_repo.py, derived_repo.py                                    # U0 read side, U1 write side
    db/documents_repo.py, events_repo.py                                   # U4
    agents/loop.py                                                         # U0
    agents/planner.py                                                      # U3
    agents/materials.py                                                    # U4
    router.py                                                              # U3
    tools/ownership.py                                                     # U0
    tools/courses.py                                                       # U3 (search_courses)
    tools/prereqs.py, requirements.py, planning.py, solver.py,
          checker.py, solve_tool.py                                        # U2
    tools/uploads.py, documents.py, materials.py, citations.py,
          dates.py, deadlines.py, export.py                                # U4
    ingest/                                                                # U1
    ui/chat.py, deadlines.py, traces.py                                    # U5
  data/                    # U1 output (committed)
  eval/run.py, eval/harness/                                               # U5
  eval/suites/{prereq_parsing,tagging}.py        # U1
  eval/suites/planning.py                        # U2
  eval/suites/{routing,session_memory}.py        # U3
  eval/suites/{retrieval,rewriting,deadlines}.py # U4
  eval/cases/{dev,final}/<suite>/                # same owners as the suites
  tests/
    fakes/, fixtures/catalog/                    # U0
    foundation/                                  # U0, including the import-boundary tests
    ingest/                                      # U1
    eligibility/                                 # U2, with property tests in test_*_props.py
    planner/                                     # U3
    materials/                                   # U4
    ui/                                          # U5 (smoke tests)
```

**Conventions:**
- **Migration numbering.** Each unit has its own number range, so parallel branches never collide.
- **Property tests** live in files named `test_*_props.py`, kept separate from example-based tests (PBT-10, advisory).
- **Branches (Q4:A).** Short-lived branches named `u<n>/<slice>`, for example `u2/solver-checker`, merged within about 2 days after one review and green `pytest`, `ruff` and `mypy`.
- **`aidlc-docs/` changes** are merged one at a time, and `audit.md` stays append-only.
- **Each coding-agent session starts by reading `aidlc-state.md` (Q5:A).** Approvals are recorded as team decisions, made at the weekly Zoom meeting or in the group chat.
