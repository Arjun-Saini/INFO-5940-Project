# Unit of Work Plan

**Stage**: INCEPTION - Units Generation (Part 1: Planning)
**Inputs**: `application-design.md` (components, AD-01 to AD-12, import rules R1–R12), `stories.md` (25 stories with owner and tier tags, demo script), `requirements.md` §10 (tiers), `execution-plan.md` (provisional unit table)
**Outputs** (in `aidlc-docs/inception/application-design/`): `unit-of-work.md`, `unit-of-work-dependency.md`, `unit-of-work-story-map.md`

CoursePilot is **one deployable application** (a single `app` container plus `db`). Units of work here are **modules** of one Python package, grouped for development and for the per-unit Construction loop, not separately deployed services.

---

## 1. Fixed Starting Points
- **Deployment:** one repository, one `app/` package, one Docker image (technical-environment.md §1–2). No microservices.
- **Owner areas** (README Team table): DATA, PLAN, MAT and UX. Each story already has a primary and secondary owner.
- **Foundation first:** the execution plan builds a shared Foundation unit before the others.
- **Tiers:** Tier 1 (the demo critical path) is built before Tier 2 (requirements §10, DEC-20).

---

## 2. Execution Checklist (Part 2: Generation)

- [ ] **Step 1: Define units.** Write `unit-of-work.md`. For each unit, give its purpose, its components (IDs from `components.md`), owner(s), tiers, public interfaces it provides, and its "done" definition.
- [ ] **Step 2: Code organization (greenfield).** Document in `unit-of-work.md` the directory layout, how each unit maps to `app/` paths, the `tests/` layout, the `eval/` suites per unit, and the branching and integration convention chosen in Q4.
- [ ] **Step 3: Dependencies.** Write `unit-of-work-dependency.md` with a unit dependency matrix, the build order, the integration contracts between units (models, functions, fakes), and the points where units integrate. Check that there are no cycles and that the dependencies agree with rules R1–R12.
- [ ] **Step 4: Story map.** Write `unit-of-work-story-map.md`, assigning every story (US-01–22, MS-01–03) to a primary unit, with any contributing units and the tier.
- [ ] **Step 5: Construction sequencing.** Record the order of the Construction loop chosen in Q5 and Q6, and which per-unit stages each unit runs (from the execution plan's per-unit table, adjusted to the final units).
- [ ] **Step 6: Validate.** Check that every component belongs to exactly one unit, every story to at least one unit, and every Tier 1 demo step to units built in the Tier 1 pass. Check that owner workloads are reasonably balanced, and that the unit boundaries respect the import rules.

---

## 3. Questions

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe your answer after the tag.

### Question 1: How should the system be split into units? *(Story grouping, business domain)*

A) **Six units (recommended):**
- **U0 Foundation** (shared): config, errors, LLM wrapper, embeddings, shared models, database and migrations, session state, tracing, agent loop, ownership checks, service facade skeleton, Docker.
- **U1 Course Ingestion** (DATA): `app/ingest/`.
- **U2 Eligibility and Solver** (DATA): prerequisites, requirements, planning service, solver, checker, `solve_tool`.
- **U3 Planner, Router and Memory** (PLAN): router, planner agent, `search_courses`, session-memory behavior.
- **U4 Materials and Deadlines** (MAT): uploads, documents, `search_materials`, citations, dates, deadlines, export, materials agent.
- **U5 UI and Evaluation** (UX): the Streamlit pages, trace viewer, evaluation runner.

B) **Five units:** the same as A, but U1 and U2 merge into one DATA unit, "Course Data and Solver", because one owner holds both

C) **Four units, one per owner,** with Foundation components divided among them rather than forming a separate unit

X) Other (please describe after [Answer]: tag below)

[Answer]: 

### Question 2: Who builds the Foundation unit, and how? *(Team alignment)*
Every other unit depends on U0's models, agent loop, database layer and service skeleton.

A) **Together, in about 1–2 days (recommended).** Each owner writes the U0 pieces closest to their area, the same split as the `app/models/` files and repositories. For example, PLAN writes the agent loop and state, UX writes tracing and Docker, DATA writes the course models and repository, and MAT writes the document and event models and ownership checks. One teammate integrates and reviews

B) **One teammate builds all of U0** while the others start Functional Design for their own units against the U0 interfaces in `component-methods.md`

X) Other (please describe after [Answer]: tag below)

[Answer]: 

### Question 3: How do units progress in parallel before their dependencies are finished? *(Dependencies)*
For example, U3 (planner) needs U2's planning service, U4 needs U0's ownership checks, and U5 needs every service action.

A) **Contract-first with fakes (recommended).** U0 publishes the shared models, function signatures, and test fakes:
- a fake `llm` with scripted replies
- a small **fixture catalog** of about 20 courses as JSON, the same format as the ingestion output
- a stub planning service that returns a canned `SolveResult`

Each unit builds and tests against the contracts and swaps in the real dependency when it lands. This also matches the unit-test rule that `app/llm.py` is mocked

B) **Strict sequence.** A unit starts only after the units it depends on are finished

X) Other (please describe after [Answer]: tag below)

[Answer]: 

### Question 4: What branching and integration convention should units use? *(Code organization)*
The README says each PR gets one teammate review, and merges happen at least every two days.

A) **Short-lived branches per unit slice (recommended).** For example `u2/solver-checker` or `u4/upload`. Each one is merged to `main` within about 2 days, after one review and green `pytest`, `ruff` and `mypy`. The shared `aidlc-docs/` is updated only on the branch for the stage being approved, to avoid conflicts in `audit.md` and `aidlc-state.md`

B) **One long-lived branch per unit,** merged when the unit is complete

X) Other (please describe after [Answer]: tag below)

[Answer]: 

### Question 5: How should the AI-DLC Construction loop run across units? *(Team alignment, process)*
The AI-DLC rules complete each unit's design and code before moving to the next unit. With four people, units can run in parallel, but each one still passes its own approval gates.

A) **Foundation first, then the other units in parallel (recommended).** After U0 completes, U1–U4 run their own Construction loops at the same time. Each owner drives the coding agent in their own session and branch, and approval gates happen at the weekly meeting or in the group chat. U5's UI work follows the service actions as they land. `aidlc-state.md` tracks each unit's stage separately

B) **Strictly sequential:** U0, U1, U2, U3, U4, U5, one unit at a time, through a single coding-agent session

X) Other (please describe after [Answer]: tag below)

[Answer]: 

### Question 6: How do tiers interact with the per-unit loop? *(Technical considerations)*
Several units hold both Tier 1 and Tier 2 work. For example, U2 has the Tier 1 solver and checker but the Tier 2 requirement categories, and U4 has Tier 1 cited Q&A but the Tier 2 query rewriting.

A) **Two passes (recommended).** Each unit's design and code stages first cover **only its Tier 1 scope**, which completes the end-to-end demo early. A second, lighter pass through the same unit covers its Tier 2 scope: Functional Design is updated, then Code Generation runs again. The audit and state files record both passes

B) **One pass per unit** covering all of its tiers, with units ordered so that Tier 1-heavy units finish first

X) Other (please describe after [Answer]: tag below)

[Answer]: 

### Question 7: Who builds the UI for each feature? *(Team alignment)*
The UI pages (UI-1 to UI-3) sit in U5, but each story's UI checklist is closely tied to a specific owner's feature, for example the interpretation panel (PLAN) or the review table (MAT).

A) **UX owns the pages, and feature owners provide the data (recommended).** U5 builds every Streamlit page against the service actions. Each feature owner's unit is "done" when its service action returns everything its stories' UI checklists need. UX reviews any PR that changes a service response model

B) **Each feature owner builds the UI for their own stories** inside `app/ui/`, and U5 is reduced to the trace viewer, shared layout and evaluation

X) Other (please describe after [Answer]: tag below)

[Answer]: 

### Question 8: Who owns the evaluation suites? *(Team alignment)*
The README says each owner contributes evaluation examples for their area.

A) **Runner in U5, suites with the feature owners (recommended).** U5 owns `eval/run.py`, the report format and the shared harness. Each unit owns its suite and its development and final cases: U1 prerequisite parsing and tagging; U2 planning; U3 routing and session memory; U4 retrieval and citations, query rewriting, and deadline extraction. Each suite is part of that unit's Tier 2 pass

B) **U5 owns the runner and every suite and its cases,** and other owners only review the cases

X) Other (please describe after [Answer]: tag below)

[Answer]: 
