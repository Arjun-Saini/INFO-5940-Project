# Component Methods

These are signatures and purposes only. **Business rules are designed per unit in Functional Design.** Every input and output type is a Pydantic v2 model (TC-02), and every string field gets an explicit maximum length, with values set in NFR Requirements (SECURITY-05). Type names are proposals that Functional Design may refine without changing a method's role.

---

## 1. Shared Models (`app/models/`, FN-9)

| File | Main models |
|---|---|
| `courses.py` | `Meeting(days, start, end, start_date, end_date)`, `Section(section_id, component, meetings, credits)`, `Course(course_id, cross_listings, title, description, credits, components, sections, prereq_field: PrereqField)` |
| `derived.py` | `PrereqRule` (technical-environment.md §6.2), `PrereqStatus` (enum: `none`, `trusted`, `uncertain`, `missing`), `PrereqRecord(course_id, status, rule, source_text, model_score, failed_checks)`, `TopicTag(tag: TopicTagEnum, confidence, justification, uncertain)`, `WorkloadSignal(score, cues)`, `RequirementCategory(name, rule: CategoryRule, status, source_text)`, `GateResult(trusted, failed_checks)`, `DerivedRecordEnvelope(input_hash, model, prompt_version, created_at, payload)` |
| `planning.py` | `Interpretation(hard: HardConstraints, soft: SoftPreferences, credit_range: CreditRange(min, max, is_default), topic_sets: list[TopicSet], inclusions, exclusions)`, `ConstraintChanges(replace_sections, add_hard, remove, pin, exclude, set_credit_range)`, `EligibilityMap`, `Schedule(label, sections, credits, workload, preferences_met, conditions, solver_chosen: list[course_id])`, `SolveResult(kind: Literal["schedules", "infeasible", "timeout"], schedules, conflicts)`, `CheckReport(ok, violations)` |
| `materials.py` | `DocumentMeta`, `Chunk(chunk_id, course_id, document_id, heading, pages, text)`, `Passage(label: "C1"…, chunk)`, `Citation(chunk_id, document, section, pages, text)`, `CitationCheck(valid, problems)`, `RewrittenQuery(queries, course_filters, date_range)` |
| `events.py` | `CandidateEvent(event_id, course_id, assignment, original_wording, interpreted: datetime | date | None, all_day, source_chunk_id, status: EventStatus, anchor_source)`, `EventStatus` (enum: `awaiting_review`, `confirmed`, `conflict`, `tbd`), `EventEdit`, `EventAudit(event_id, at, before, after)` |

---

## 2. Service Facade: `app/service.py` (SV-1)

Every function creates a `Trace`, runs inside `errors.top_level(trace)`, validates its inputs, loads and saves `SessionState` when it uses session state, and returns a validated response model.

| Method | Input | Output | Purpose |
|---|---|---|---|
| `handle_message(session_id: str, text: str)` | text ≤ max length | `ChatResponse` | Chat entry point: routes the message, then runs the planner (phase 1 or a refinement), the materials agent, a clarifying question, or the out-of-scope reply |
| `set_completed_courses(session_id, completed: list[str], reported_equivalents: list[str])` | course IDs, free text | `SessionSnapshot` | Sidebar updates (US-01) |
| `solve_confirmed(session_id, interpretation: Interpretation)` | edited panel | `PlanResponse` | "Find schedules": runs phase 2 through SV-2, then the planner's explanation call (Q1:C) |
| `apply_changes(session_id, changes: ConstraintChanges)` | pin, exclude, credit-range edit | `PlanResponse` | Button edits re-run phase 2 directly (DEC-22) |
| `keep_schedule(session_id, label: str)` | `"A"`, `"B"` or `"C"` | `SessionSnapshot` | Sets the active schedule (FR-SEL-01) |
| `upload_document(session_id, course_id: str, filename: str, data: bytes)` | `require_active_course`, ≤ 20 MB | `DocumentMeta` | Runs SV-3 for a course in the active schedule |
| `open_deadline_review(session_id, document_id: str)` | `require_document` | `list[CandidateEvent]` | Runs `extract_deadlines` the first time (cached in `events_repo` after that), then returns the rows |
| `update_event(session_id, event_id: str, edit: EventEdit)` | `require_event`, plus the edited date, time or status | `CandidateEvent` | Edit, resolve a conflict, fill a TBD date, or confirm. Writes an `EventAudit` (FR-DL-07) |
| `set_anchor_override(session_id, course_id: str, anchor: date)` | `require_active_course` | `list[CandidateEvent]` | Per-course week-one override. Re-resolves the course's events |
| `export_calendar(session_id)` | | `IcsFile(filename, content: bytes) \| NothingToExport` | Download button, through `export_ics` |
| `list_traces(filters: TraceFilter)` / `get_trace(trace_id: str)` | | `list[TraceSummary]` / `Trace` | Trace viewer (Tier 2) |

---

## 3. Router: `app/router.py` (AG-1)

| Method | Input | Output | Purpose |
|---|---|---|---|
| `route_message(text: str, state: SessionState, *, trace: Trace)` | | `RouteDecision(intent, details, confidence, clarifying_question)` | Router-model classification with structured output, plus clarification decided from both state and confidence |
| `_needs_clarification(decision: RouteDecision, state: SessionState)` | | `str \| None` | Deterministic checks for ambiguous references (for example two 8 AM sections) and the confidence threshold |

---

## 4. Agent Loop: `app/agents/loop.py` (AG-2)

| Item | Signature | Purpose |
|---|---|---|
| `AgentContext` | `AgentContext(session_id, active_course_ids, completed_courses, reported_equivalents, interpretation)` | **Built by the service from server-side `SessionState`, never from model output (AD-12).** Passed to every tool |
| `ToolSpec` | `ToolSpec(name, description, input_model: type[BaseModel], output_model: type[BaseModel], fn: Callable[[InputModel, AgentContext], OutputModel])` | One allowlisted tool. A test checks that no `input_model` declares a session ID or student-data field |
| `AgentSpec` | `AgentSpec(name, system_prompt, model, tools: dict[str, ToolSpec], max_calls: int = 8, finalizer: Callable[[LoopResult], R])` | Agent definition (Q6:A) |
| `run_agent` | `run_agent(spec: AgentSpec, messages: list[dict], *, context: AgentContext, trace: Trace) -> R` | Runs the loop: allowlist check → validate the arguments → run the tool → return the result or a structured error → repeat until there is a final answer or the budget runs out. Rejected and invalid calls are never run (MS-02) |
| `ToolError` | `ToolError(kind: Literal["invalid_args", "unknown_tool", "missing_info", "no_match", "forbidden", "unknown_id", "failed"], message)` | Structured errors the model can see and correct (FR-ELG-10, FR-AGT-02) |

## 5. Planner Agent: `app/agents/planner.py` (AG-3)

| Method | Input | Output | Purpose |
|---|---|---|---|
| `interpret(text, state, *, trace)` | | `Interpretation` | Phase 1, run with `INTERPRET_SPEC`, so it **cannot call `solve_schedule`**. Applies `DEFAULT_CREDIT_RANGE` when no range is given and maps topic phrases to `TOPIC_TAGS` |
| `refine(text, decision, state, *, trace)` | | `PlanResponse` | Refinement loop, run with `REFINE_SPEC`, that resolves the reference into `ConstraintChanges` and calls the `solve_schedule` tool (Follow-up 1:A) |
| `explain(result: SolveResult, interpretation, *, trace)` | | `str` | The single explanation call after phase 2, with **no tools**. It describes the structured results and never changes them |
| `INTERPRET_SPEC` | | `AgentSpec` | Allowlist: `search_courses` **only** (AD-11) |
| `REFINE_SPEC` | | `AgentSpec` | Allowlist: `search_courses`, `check_prereqs`, `check_requirements`, `solve_schedule` |

## 6. Materials Agent: `app/agents/materials.py` (AG-4)

| Method | Input | Output | Purpose |
|---|---|---|---|
| `run(text, decision, state, *, trace)` | | `ChatResponse` | Answers a question with citations, validated by CV-1 with one retry. Unverified answers are labeled |
| `rewrite_query(text, state, *, trace)` | | `RewrittenQuery` | Router-model rewrite. `REWRITE_ENABLED` switches it off for evaluation (Tier 2) |
| `MATERIALS_SPEC` | | `AgentSpec` | Tools: `search_materials`, `extract_deadlines`, `export_ics` |

---

## 7. Tools: `app/tools/`

| Tool or method | Input model | Output model | Notes |
|---|---|---|---|
| `search_courses(args: SearchCoursesInput, ctx)` | query, level, days, credits (the term comes from configuration) | `SearchCoursesOutput(courses[≤10])` | T-1 |
| `check_prereqs(args: CheckPrereqsToolInput, ctx)` | course_id (`require_catalog_course`) | `CheckPrereqsOutput(status, rule_text, source_text, reason)` | T-2. The tool wrapper fills `completed_courses` and `reported_equivalents` from `ctx`, then calls the pure `check_prereqs(CheckPrereqsInput, …)` from technical-environment.md §6.2. The model can't claim courses the student hasn't entered |
| `check_requirements(args: CheckRequirementsToolInput, ctx)` | proposed course IDs (`require_catalog_course`), with completed courses from `ctx` | `CheckRequirementsOutput(categories: list[CategoryProgress])` | T-3, Tier 2 |
| `solve_schedule(args: SolveScheduleInput, ctx)` | `ConstraintChanges` (every course and section ID passes `require_catalog_*`), with state from `ctx` | `PlanResponse` | T-4, in `solve_tool.py`. Calls `planning.solve_with_checks` |
| `planning.solve_with_checks(interpretation, changes, state, *, trace)` | | `SolveResult` | SV-2: gather candidates → `EligibilityMap` → `solver.solve` → `checker.check` on each → drop failures and trace them |
| `solver.solve(problem: SolveProblem, *, time_limit_s)` | candidates, eligibility, constraints, preferences | `SolveResult` | SO-1 |
| `checker.check(schedule: Schedule, problem: SolveProblem)` | | `CheckReport` | CK-1. Independent of the solver |
| `search_materials(args: SearchMaterialsInput, ctx)` | queries, plus optional course filters (`require_active_course`) and document filters (`require_document`). The default filter is the active schedule's courses | `SearchMaterialsOutput(passages[≤5])` | T-5 |
| `citations.validate(answer: DraftAnswer, evidence: list[Passage])` | | `CitationCheck` | CV-1 |
| `extract_deadlines(args: ExtractDeadlinesInput, ctx)` | document_id (`require_document`). The course calendar context is built server-side from the anchors | `ExtractDeadlinesOutput(events)` | T-6. Resolves dates through DT-1 |
| `dates.resolve(wording_parts: RelativeDate, anchors: AnchorSet)` | | `ResolvedDate(value, all_day, anchor_source)` | DT-1 |
| `dates.week_range(today: date)` | | `tuple[date, date]` | DT-1, Monday–Sunday (DEC-25) |
| `dates.today()` | | `date` | `TODAY_OVERRIDE` or the current date in America/New_York |
| `export_ics(args: ExportIcsInput, ctx)` | **none** (an empty model), with the session from `ctx` | `IcsFile \| NothingToExport` | T-7. Confirmed events of `ctx.session_id` for the active schedule's courses only |
| `ownership.require_document(ctx, document_id)` / `require_event(ctx, event_id)` / `require_active_course(ctx, course_id)` / `require_catalog_course(course_id)` / `require_catalog_section(section_id)` | | the owned record, or `ToolError(forbidden \| unknown_id)` | OW-1 |
| `uploads.validate(filename, data: bytes)` | | `ValidatedPdf` or raises `UploadRejected` | UP-1 |
| `documents.ingest(course_id, pdf: ValidatedPdf)` | | `DocumentMeta` | SV-3 |

---

## 8. Ingestion: `app/ingest/`

| Method | Input | Output | Purpose |
|---|---|---|---|
| `CourseSource.fetch(term: str, subjects: list[str])` (protocol) | | `RawExport` | IN-1 |
| `ClassRosterSource.fetch(...)` | | `RawExport` | `httpx`, with Pydantic validation of every response |
| `prereq_status.classify(field: PrereqField)` | present and text, or absent | `Literal["none", "text", "missing"]` | IN-2, pure |
| `derive.prereq_rule(course: Course)` | | `PrereqRecord` | IN-3 → IN-4 |
| `derive.topic_tags(course: Course, vocabulary: list[str])` | | `list[TopicTag]` | IN-3 |
| `derive.workload(course: Course)` | | `WorkloadSignal` | IN-3, Tier 2 |
| `derive.requirement_categories(doc_text: str)` | | `list[RequirementCategory]` | IN-3 → IN-4, Tier 2 |
| `trust_gate.check_rule(rule: PrereqRule, source_text: str, score: float)` | | `GateResult` | IN-4, checks (a)–(e), pure |
| `trust_gate.check_category(category, source_text, score)` | | `GateResult` | IN-4, pure |
| `cache.get(key: CacheKey)` / `cache.put(key, envelope)` | | `DerivedRecordEnvelope \| None` | IN-5 |
| `anchor.derive(sections: list[Section], override: date \| None)` | | `date` | IN-6 |
| `main(argv)` | `--term`, `--subjects` | exit code | IN-7 |
| `seed.load(data_dir: Path)` | | `SeedReport` | IN-8. Idempotent upserts, re-validation and gate re-check |

---

## 9. Foundation

| Method | Output | Purpose |
|---|---|---|
| `config.get_settings()` | `Settings` | FN-1. Cached. Refuses to start on the placeholder password |
| `llm.chat(model, messages, tools=None, *, think: bool, trace)` | `LLMResult` | FN-2 |
| `llm.chat_structured(model, messages, schema: type[T], *, think: bool, trace)` | `T` | FN-2. Retries once, then raises `StructuredOutputError` |
| `embeddings.embed(texts: list[str])` | `list[list[float]]` | FN-3 |
| `db.connection.get_conn()` (context manager) | `psycopg.Connection` | FN-4. Rolls back on error |
| `db.migrate.run()` | `None` | FN-4 |
| `state.load_state(session_id)` / `state.save_state(session_id, state)` | `SessionState` / `None` | FN-6 |
| `state.SessionState.append_turn(user, assistant)` | `None` | FN-6 |
| `tracing.Trace` / `trace.span(name, **attrs)` (context manager) / `tracing.record_trace(trace)` | | FN-7 |
| `tracing.get_logger(name)` | `logging.Logger` | FN-7. Structured JSON on stdout |
| `errors.top_level(trace)` (context manager) | | FN-8. Catches, logs and turns errors into `UserFacingError` |
| Repositories: `courses_repo.*`, `derived_repo.*`, `sessions_repo.*`, `documents_repo.*` (including `fts_search` and `vector_search`), `events_repo.*` (including `audit`), `traces_repo.*` | Pydantic models | FN-5. All SQL is parameterized |

## 10. Evaluation: `eval/`

| Method | Output | Purpose |
|---|---|---|
| `eval.run.main(argv)` | report files | EV-1. Options: `--suite`, `--final`/`--dev`, `--rewrite on\|off` |
| `eval.suites.<name>.run(config: EvalConfig)` | `SuiteResult(counts, metrics, failures)` | One per suite in requirements §7 |
