# Services and Orchestration

## 1. Service Definitions

| Service | Module | Kind | Responsibility |
|---|---|---|---|
| **Service facade** (SV-1) | `app/service.py` | Entry point | The only module the UI calls. Each call opens a `Trace` and runs inside `errors.top_level`. Calls that use session state load it at the start and save it at the end. Each call is synchronous, and the UI shows progress with `st.status` (Q7:A) |
| **Planning service** (SV-2) | `app/tools/planning.py` | Deterministic | Phase 2 solve, the **single solving path**. The "Find schedules", pin and exclude buttons call it through the facade, and chat refinements call it through the `solve_schedule` tool (Q1:C, Follow-up 1:A) |
| **Document service** (SV-3) | `app/tools/documents.py` | Deterministic | Upload pipeline: validate, extract, detect sections, chunk, embed, store |
| **Agent services** | `app/agents/planner.py`, `app/agents/materials.py` | Model-driven | Run `run_agent` with their `AgentSpec`, then apply post-processing (citation validation for materials, the explanation call for the planner) |
| **Ingestion** | `app/ingest/` (CLI) | Offline batch | Pulls the catalog, derives records, gates them and caches them in `data/`. Not part of the running app |
| **Seed** | `app/ingest/seed.py` | Startup | Loads the committed `data/` into Postgres, idempotently, with re-validation |

**Cross-cutting rules for every flow:**
- The model never enforces a hard rule. Python does (NFR-COR-03).
- Every model output is validated with Pydantic before use.
- Every tool call passes the spec's allowlist and input validation before it runs.
- Errors fail closed: nothing unvalidated is shown as verified, and the user sees a generic message.

---

## 2. Orchestration Flows

Each flow below is listed as numbered steps, which is also the text alternative for the diagrams in `component-dependency.md`. A **[model]** tag marks a model call.

### F1: Chat planning request (phase 1), demo step 2
`handle_message` →
1. `load_state`. **[model]** `route_message` returns `plan_schedule`.
2. **[model]** `planner.interpret` runs the agent loop with **`INTERPRET_SPEC`, whose only allowed tool is `search_courses`**. Phase 1 therefore cannot produce schedules before the student confirms the panel (AD-11). It may call `search_courses` to resolve course and topic names. It returns an `Interpretation`, applying `DEFAULT_CREDIT_RANGE` if no range was given and filling `topic_sets` from tags: trusted tags pre-selected, uncertain tags listed but unselected (DEC-16).
3. Save `Interpretation` in state, `save_state`, `record_trace`.
4. Return `ChatResponse(interpretation=…)`. The UI renders the editable panel.

### F2: "Find schedules" button (phase 2), demo steps 3–5
`solve_confirmed(session_id, interpretation)` →
1. Validate the edited `Interpretation`.
2. `planning.solve_with_checks` (see F2a).
3. **[model]** `planner.explain(result)` writes text from the structured results. The schedules shown come from `SolveResult`, never from the generated text.
4. Save the candidate schedules in state and return `PlanResponse`.

### F2a: `planning.solve_with_checks` (SV-2), shared by F2, F3 and F4
1. Apply `ConstraintChanges`, if any, to the current `Interpretation`.
2. Gather candidates from `courses_repo`: courses in topic sets and inclusions, plus fill candidates (eligible ingested courses, minus exclusions) (DEC-22).
3. For each candidate, call `check_prereqs` to build an `EligibilityMap`. `unmet` courses are dropped, and `needs_confirmation` courses are kept with a penalty.
4. `solver.solve(problem, time_limit_s)` returns schedules, an infeasible result, or a timeout result.
5. `checker.check` runs on every schedule. Failing schedules are dropped, and each drop is recorded in the trace.
6. Mark the fill courses as `solver_chosen` and return the `SolveResult`.

### F3: Chat refinement, "swap out the 8am" (Tier 2)
`handle_message` →
1. **[model]** `route_message` returns `refine_schedule`. If the reference is ambiguous (two 8 AM sections), return a clarifying question and stop.
2. **[model]** `planner.refine` runs the agent loop with `REFINE_SPEC`. The model resolves the reference and calls `solve_schedule(ConstraintChanges(replace_sections=[CS 4780-001]))`.
3. The tool validates the arguments and runs F2a. The result goes back to the model.
4. **[model]** The finalizer writes the reply. The loop allows at most 8 calls in total.

### F4: Pin or exclude a solver-chosen course
`apply_changes(session_id, ConstraintChanges(pin=[…]) or (exclude=[…]))` → F2a → **[model]** `explain` → `PlanResponse`.

### F5: Keep a schedule, demo step 6
`keep_schedule(session_id, "A")` → set `active_schedule` (course and section IDs) → `save_state`. No model call, and no call to any external system.

### F6: Upload a syllabus, demo step 7
`upload_document(session_id, course_id, filename, data)` →
1. `ownership.require_active_course`: the course must be in the active schedule, otherwise reject.
2. `uploads.validate`: size, then signature, otherwise `UploadRejected` (MS-03).
3. `documents.ingest`: PyMuPDF text with page numbers, then section detection, then chunks of about 500 tokens, then `embed`.
4. Store everything with `documents_repo` in **one transaction**, so nothing partial is left after a failure.
5. Return `DocumentMeta`.

### F7: Question about materials, demo steps 8–10
`handle_message` →
1. **[model]** `route_message` returns `ask_course_info`.
2. **[model]** `materials.rewrite_query` (Tier 2, switchable). In Tier 1, the raw question plus the active schedule's course filters is used instead.
3. **[model]** `materials.run`. The agent loop calls `search_materials`, and the passages come back labeled `[C1]…[C5]` inside document delimiters.
4. `citations.validate(draft, evidence)`. If it fails, **[model]** retries once. If it fails again, the answer is marked **unverified**.
5. For relative dates, the original wording is cited and the computed date (DT-1) is labeled as an interpretation (DEC-24).
6. Return `ChatResponse(citations=…)`. If nothing relevant was retrieved, the answer says what couldn't be found (FR-MAT-09).

### F8: Deadline review, demo steps 11–12
`open_deadline_review(session_id, document_id)` → `ownership.require_document` →
1. If no events are cached for the document: **[model]** `extract_deadlines` pulls wording and assignment names from the chunks, then `dates.resolve` computes the date using the anchor precedence and Monday–Sunday weeks. Events are stored as `awaiting_review`, `conflict` or `tbd`.
2. Return the rows.

`update_event` → `ownership.require_event` → validate the `EventEdit` → `events_repo.update` plus `events_repo.audit(before, after)` in one transaction.

### F9: Export, demo step 13
Either through chat (`handle_message`, then **[model]** router returns `export_calendar`, then the materials agent calls the `export_ics` tool) or through the button (`export_calendar`):
1. `events_repo.confirmed_for(active_schedule courses)`.
2. `icalendar` builds the file: America/New_York with a VTIMEZONE, timed or all-day events, and a source description for each.
3. Return `IcsFile`, or `NothingToExport` if no events are confirmed.

### F10: "What's due this week?" (Tier 2)
`handle_message` →
1. **[model]** The router returns `ask_deadline`.
2. `dates.today()` and `week_range` give the date range. `events_repo` returns the confirmed events in range for the active courses.
3. Optionally, `search_materials` looks for missing items. Any newly found dates are labeled **awaiting review**.

### F11: Ingestion (offline, P4)
`python -m app.ingest --term FA26 --subjects CS` →
1. `ClassRosterSource.fetch`, validated, and the raw export is saved to `data/`.
2. For each course:
   - `prereq_status.classify`.
   - If the status is `text`: check the cache, and on a miss **[model]** `derive.prereq_rule`, then `trust_gate.check_rule`, then `cache.put`.
   - Topic tags the same way. Workload signals the same way (Tier 2).
3. **[model]** Requirement categories from `REQUIREMENTS_DOC`, then `trust_gate.check_category` (Tier 2).
4. `anchor.derive`.
5. Print a summary: courses ingested, rules trusted vs. uncertain, uncertain tags, failures. The developer then commits `data/` (Q5:A).

### F12: Seed at startup
`seed.load(data/)` →
1. Validate each envelope against its Pydantic schema.
2. **Re-run the deterministic checks (a), (b), (c) and (e) of `trust_gate`** on every trusted rule. A failure downgrades the rule to uncertain and is logged.
3. Idempotent upserts. Return `SeedReport`.

### F13: Evaluation (P4)
`python -m eval.run` → each suite calls `app.service` (end to end) or the tools directly (unit-level suites) with a fixed `TODAY_OVERRIDE` → the report records counts, metrics, configuration, tokens and latency.

---

## 3. Tier 1 Demo Coverage

| Demo step | Flow |
|---|---|
| 1 Completed courses | `set_completed_courses` |
| 2 Planning message | F1 |
| 3 Find schedules | F2 |
| 4 Schedules A, B and C | F2a |
| 5 Condition detail | F2a (the condition comes from `check_prereqs`) |
| 6 Keep a schedule | F5 |
| 7 Upload | F6 |
| 8–10 Questions | F7 |
| 11–12 Review and confirm | F8 |
| 13 Export | F9 (chat) |
| 14 Calendar import | Manual check outside the app |

Every Tier 1 demo step is served by a defined flow.

---

## 4. Misuse Cases (NFR-SEC-11)

| ID | Misuse | Defending components | Design response |
|---|---|---|---|
| MS-01 | Prompt injection in an uploaded syllabus | AG-4, CV-1, FN-2 | Document text goes only inside delimited evidence blocks, and the system prompt says to ignore instructions in documents. Claims must cite retrieved passages (CV-1), so an injected claim is either visibly quoted from the document or rejected as unverified |
| MS-01 | Prompt injection in a course description during ingestion | IN-2, IN-3, IN-4 | Prerequisite status comes from the prerequisite **field** (IN-2), never from the description. Rule trees must pass the deterministic gate. Tags are limited to the enum and only affect topic sets the student confirms (DEC-18) |
| MS-02 | Malicious or malformed tool arguments | AG-2, OW-1, tool input models, FN-5 | Allowlist per `AgentSpec`, and phase 1 can't solve (`INTERPRET_SPEC` allows only `search_courses`, AD-11). Pydantic bounds on every argument. Parameterized SQL only. `solve_schedule` takes `ConstraintChanges`, not raw solver inputs. The 8-call budget applies. **Ownership never comes from the model (AD-12):** the session ID and student data (completed courses, reported equivalents) come only from the server-built `AgentContext`. Every document, event and active-course ID a tool or service action receives is checked by `ownership.require_*` against the current session or active schedule **before** any action, and planner course and section IDs must exist in the ingested catalog. A failed check returns `ToolError(forbidden \| unknown_id)`. Repository queries for documents and events also filter by `session_id` (defense in depth). This is a SECURITY-05/11 control |
| MS-03 | Crafted or oversized PDF | UP-1, SV-3, FN-8 | Size and signature checks happen before parsing. Parser errors become a generic message. The upload is stored in one transaction, so no partial records remain |
| MC-04 *(new)* | **Hand-edited committed cache.** Someone edits a derived record in `data/` to force a rule to "trusted" | IN-8, IN-4 | The seed script re-validates schemas and **re-runs the deterministic trust gate checks** on load, so a rule tree that doesn't match its source text is downgraded to uncertain. Git history also shows who changed the file (SECURITY-13). Check (d), the model score, can't be re-verified offline, so it is documented as a residual risk that code review covers |
| MC-05 *(new)* | **Malformed or hostile catalog API response** | IN-1 | Every response is validated with Pydantic (types and length bounds). Courses that fail validation are skipped and counted in the summary, never loaded partly |
| MC-06 *(new)* | **`.ics` field injection.** Text from a syllabus with line breaks or iCalendar control text in an assignment name | T-7 | Fields are set only through `icalendar` property APIs (never as raw strings), with length bounds on the summary and description |
| MC-07 *(new)* | **Rendering untrusted text as HTML** (XSS-like) in Streamlit | UI-1, UI-2 | Model output and document text are rendered as plain text or Markdown with `unsafe_allow_html` never enabled for them (NFR-SEC-05) |
