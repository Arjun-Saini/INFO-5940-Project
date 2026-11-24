# CoursePilot: Requirements

**Stage**: INCEPTION - Requirements Analysis
**Depth**: Comprehensive
**Date**: 2026-11-24
**Sources**: `vision.md`, `technical-environment.md` and `README.md` (as of commit `da6c501`), `requirement-verification-questions.md` (Q1–Q14), `requirement-clarification-questions.md` (C1–C4), `requirement-direction-change-questions.md` (D1–D5), and the requirements review of 2026-11-24

---

## 1. Intent Analysis

| Item | Assessment |
|---|---|
| **User request** | "Using AI-DLC, I want to start a new project. Please read vision.md and technical-environment.md, then begin the AI-DLC workflow." |
| **Request type** | New project (greenfield) |
| **Clarity** | Clear. A detailed vision, technical environment and proposal are given, and 23 clarifying answers fill the remaining gaps |
| **Scope** | System-wide: ingestion pipeline, router, two agents, 7 tools, RAG, CP-SAT solver, session memory, tracing, evaluation runner and Streamlit UI |
| **Complexity** | Complex. The system has several components, LLM-derived data under hard correctness guarantees, a four-person team, a fixed deadline (December 16) and measurable evaluation targets |

**Summary.** CoursePilot is a local Streamlit app for Cornell undergraduate CS students. It plans a semester schedule from natural-language goals, answers cited questions about uploaded syllabi, and exports confirmed deadlines to `.ics`. Models run in Ollama, and Python enforces every hard rule. **Nothing is curated by hand.** Course data, prerequisite rules, topic tags, workload signals and requirement categories are all derived automatically from the course catalog and an official requirements document. Every derived item passes deterministic checks before the product trusts it.

---

## 2. Decision Record

This table lists the decisions behind these requirements. "Superseded" marks an answer replaced by the no-manual-curation direction change (C2).

| ID | Topic | Decision | Source |
|---|---|---|---|
| DEC-01 | Term | Fall 2026 (`TERM=FA26`) is a stand-in for the next term to plan. The term is a configuration value | Q1 |
| DEC-02 | Course set | Every course ingested for the configured `TERM` and `SUBJECTS` (default `CS,MATH`). There is no hand-reviewed subset | C2 (supersedes Q2) |
| DEC-03 | Requirement categories | Extracted from `REQUIREMENTS_DOC` (the current CS major checklist). CS core and CS electives must be correct. Other categories are extracted on a best-effort basis and shown as `needs_confirmation`, never counted as satisfied | Q4, C2, D4 (supersedes Q3) |
| DEC-04 | Course data source | Automatic pull from the Class Roster API through a `CourseSource` adapter, cached in `data/`. No hand-built fallback | C2 (supersedes Q5) |
| DEC-05 | Workload | Soft credit target plus an automatically derived workload signal, used only as a soft penalty and shown to the student | C2 (supersedes Q6) |
| DEC-06 | Week-one anchor | Derived at ingestion as the most common start date among the term's lecture sections. An optional `TERM_START` overrides it, and the student can override it per course in the review table | Q7, D5 |
| DEC-07 | Completed courses | A sidebar multiselect of ingested courses plus every course referenced by derived rules and categories, plus free text. Free-text courses can only produce `needs_confirmation` | Q8, C3 |
| DEC-08 | Distinct schedules | Prefer differences in courses. Fall back to differences in sections only when not enough course combinations are feasible | Q9 |
| DEC-09 | Time zone and events | America/New_York. Deadlines with a stated time become timed events, and date-only deadlines become all-day events | Q10 |
| DEC-10 | Latency | Soft target: routing under about 3 s and a full answer under about 30 s (Apple Silicon, 16 GB). Reported in evaluation, with a progress indicator in the UI | Q11 |
| DEC-11 | "Today" | `TODAY_OVERRIDE` in `.env`. When it is unset, the real date in America/New_York is used. Tests and evaluation always set it | C1 |
| DEC-12 | Topic tags | Model-assigned from course name and description, limited to a fixed vocabulary set by `TOPIC_TAGS` and enforced by a Pydantic enum. The student can edit which courses count before solving | C2, D3 |
| DEC-13 | Trusting derived rules | Deterministic cross-checks **and** a model score above the threshold. A failure is stored as uncertain, which gives `needs_confirmation`. No self-consistency pass | C2, D1 |
| DEC-14 | `needs_confirmation` in the solver | Eligible, with a soft penalty. Outstanding conditions are listed on each schedule, and only `unmet` courses are excluded | D2 |
| DEC-15 | Extensions | Security Baseline: enabled, with the N/A, adapted and fully enforced split from C4:A. Resiliency: disabled. PBT: Partial (PBT-02, 03, 07, 08 and 09 are blocking) | Q12–Q14, C4 |
| DEC-16 | Topic set defaults | The interpretation panel pre-selects courses with a trusted tag. It lists uncertain-tag courses unselected and labeled, and the student can add them | Review (confirmed A-01) |
| DEC-17 | Topic vs. category | "N ML electives" counts N courses from the confirmed topic set. It does not also require them to count toward CS electives unless the student asks | Review (confirmed A-02) |
| DEC-18 | Trust-gate scope | The trust gate applies to prerequisite rules and requirement categories. Topic tags and workload signals are trusted on the model score plus Pydantic validation (vocabulary enum), because they have no course codes to cross-check and only affect student-confirmed constraints or soft scoring | Review (confirmed A-03) |
| DEC-19 | No prerequisites vs. missing data | A prerequisite field that the catalog returned but is empty, or that says "none", means no prerequisites (`met`). Only a field the catalog didn't return is missing (`needs_confirmation`) | Review |
| DEC-20 | Delivery priority | Tier 1: critical path for the end-to-end demo. Tier 2: every other MUST. Tier 3: SHOULD items (§10) | Review |
| DEC-21 | Default credit range | `DEFAULT_CREDIT_RANGE` (default 12–18) applies when the student gives no range. It is shown as an editable item in the interpretation panel | Stories review (OP-01) |
| DEC-22 | Filling the credit range | When the named constraints don't reach the credit range, the solver fills the remaining credits from eligible ingested courses only (never `unmet`, never excluded). They are ranked by soft preferences and, once requirement categories exist (Tier 2), by remaining requirement gaps. The panel labels them solver-chosen, and the student can pin or exclude each one | Stories review (OP-02) |
| DEC-23 | Explicit inclusions | Courses the student names ("I also want to take CS 4820") or pins are a hard constraint, the counterpart of exclusions. Both the solver and the separate checker enforce them | Stories review (OP-05) |
| DEC-24 | Citations for relative dates | The citation covers the original wording. A computed date is labeled as an interpretation from the week-one anchor and is exempt from the quoted-date check | Stories review (OP-03) |
| DEC-25 | Week numbering | Weeks run Monday to Sunday. Week 1 is the week containing the anchor date. "This week" is the Monday–Sunday week containing today | Stories review (OP-04) |

---

## 3. System Context and Constraints

### 3.1 Users
- **Planning student**: a Cornell undergraduate CS student planning their next semester (for example, a junior who has finished core courses).
- **Managing student**: a Cornell undergraduate CS student taking 4–6 courses who tracks deadlines and policies.
- **Team developer**: one of the four teammates. They run ingestion and evaluation and read traces to diagnose wrong answers.

### 3.2 Fixed technical constraints
`technical-environment.md` is a binding constraint (TC). In summary:
- **TC-01**: Python 3.12, managed with `uv` (`uv.lock` committed). Every function signature has type hints.
- **TC-02**: Streamlit ≥ 1.40 for the UI. Pydantic v2 for every tool input and output, router decision, state object and derived record.
- **TC-03**: Ollama only, through the `ollama` library, called only from `app/llm.py`. The agent model is `AGENT_MODEL` (default `qwen3:8b`), and the router and rewrite model is `ROUTER_MODEL` (default `llama3.2:3b`). `num_ctx` is set explicitly on every call. Temperature is 0 for routing, rewriting, tool selection and ingestion derivation.
- **TC-04**: PostgreSQL 16 + pgvector, accessed through `psycopg` 3 with parameterized SQL in `app/db/` only.
- **TC-05**: OR-Tools CP-SAT, PyMuPDF, `bge-small-en-v1.5` (sentence-transformers), `icalendar`, and `httpx` (ingestion only).
- **TC-06**: The prohibited libraries in technical-environment.md §4 are banned. Any new dependency must be proposed in a PR.
- **TC-07**: The app runs locally only. Docker Compose runs `db` and `app`, and Ollama runs on the host. At runtime the app makes no outbound internet calls.
- **TC-08**: Testing uses `pytest`, `pytest-cov`, `hypothesis` and `python -m eval.run`. Unit tests never call a real model.

### 3.3 Configuration
Every portability lever is configuration, not code or curated data.

| Variable | Purpose | Status |
|---|---|---|
| `OLLAMA_HOST`, `AGENT_MODEL`, `ROUTER_MODEL`, `OLLAMA_NUM_CTX` | Model runtime | In technical-environment.md |
| `TERM`, `SUBJECTS` | What ingestion pulls | In technical-environment.md |
| `REQUIREMENTS_DOC` | Official requirements document to extract categories from | In technical-environment.md |
| `TODAY_OVERRIDE` | Reference date for relative questions | In technical-environment.md |
| `POSTGRES_PASSWORD`, `DATABASE_URL` | Local database password (placeholder `change-me` in `.env.example`) and connection string | In technical-environment.md (NFR-SEC-09) |
| `TOPIC_TAGS` | Fixed topic vocabulary (DEC-12). Default: `ml,ai,systems,theory,security,graphics,hci,pl,data` | In technical-environment.md |
| `TERM_START` | Optional override for the derived week-one anchor (DEC-06) | In technical-environment.md |
| `DERIVE_CONFIDENCE_THRESHOLD` | Minimum model score for a derived item (DEC-13). Default: `0.8` | In technical-environment.md |
| `DEFAULT_CREDIT_RANGE` | Credit range used when the student gives none (DEC-21). Default: `12-18` | **New**: not yet in technical-environment.md `.env.example` |

---

## 4. Functional Requirements

Each requirement has an ID for traceability. **MUST** marks MVP scope, and **SHOULD** marks a stretch goal.

### 4.1 Automatic Course Ingestion (FR-ING)

| ID | Requirement |
|---|---|
| FR-ING-01 | The system MUST provide an ingestion command (`uv run python -m app.ingest`). It pulls the configured `TERM` and `SUBJECTS` from the Class Roster API through a `CourseSource` adapter interface and saves the raw export in `data/`. The adapter interface MUST let another institution's catalog be added as a second adapter without changes outside `app/ingest/sources/`. |
| FR-ING-02 | For each course, ingestion MUST capture the course ID, cross-listings, title, description, credits, enrollment components (lecture, discussion, lab and so on), sections, meeting days and times, section start and end dates, and prerequisite text. A field the source does not return MUST be stored as missing, never filled by hand. **Prerequisite status MUST distinguish three cases (DEC-19):** (a) **none**: the source returned the prerequisite field and it is empty after normalizing whitespace, or its whole text matches an entry in the `NO_PREREQ_PATTERNS` constant (for example "none" or "no prerequisites"). The course has no prerequisites, and no model call is made. (b) **text**: the source returned non-empty text that is not a "none" match. The text goes to derivation (FR-ING-03) and the trust gate (FR-ING-07). (c) **missing**: the source did not return the field at all. A "none" match is decided by Python alone, and text containing any course code is never a "none" match. |
| FR-ING-03 | The agent model MUST convert each course's prerequisite text into a rule tree using the grammar `course`, `course_or_equiv`, `and`, `or`, with a model confidence score. "X or equivalent" MUST become a `course_or_equiv` branch. |
| FR-ING-04 | The agent model MUST assign topic tags to each course from its name and description, with a confidence score and a one-line justification. It may only use tags from the `TOPIC_TAGS` vocabulary, and the vocabulary MUST be enforced by a Pydantic enum built from configuration. |
| FR-ING-05 | Ingestion MUST derive a workload signal for each course from credits, components (for example, labs or projects) and model-extracted workload cues in the description (for example, "project-intensive" or "weekly problem sets"). Each signal MUST store its contributing cues for display. |
| FR-ING-06 | Ingestion MUST extract requirement categories from `REQUIREMENTS_DOC`. The category rule format MUST express explicit course lists, subject and course-number ranges with exclusions (for example, "any CS 4000+ except CS 4999"), and course-count or credit thresholds. CS core and CS electives MUST be extracted and evaluated correctly. Any other category, or any rule the format cannot express, MUST be stored as uncertain and shown as `needs_confirmation`, never counted as satisfied. |
| FR-ING-07 | **Trust gate (DEC-13).** The gate applies to every derived requirement category and to every prerequisite rule in status (b) **text** from FR-ING-02. Status (a) **none** and status (c) **missing** are decided without the model, so they never reach the gate. A derived prerequisite rule or requirement category is *trusted* only if all of these hold: (a) after normalizing cross-listings, every course code in the source text appears in the derived rule; (b) every course in the rule appears in the source text; (c) the source text contains no wording the grammar can't express (at least "permission of instructor", "corequisite", "concurrent enrollment", "recommended", and "equivalent" without a matching `course_or_equiv` branch); (d) the model score is at least `DERIVE_CONFIDENCE_THRESHOLD`; and (e) the output passes Pydantic validation. Anything else MUST be stored as *uncertain*. The list of unexpressible phrases MUST be one module-level constant so it can be extended. |
| FR-ING-08 | `check_prereqs` MUST map prerequisite status as follows. **None** (FR-ING-02a) returns `met` for every student, with the reason "no prerequisites listed" and the source text, if any. **Trusted** rule trees are evaluated with three-valued logic (FR-ELG-01). An **uncertain** rule (it failed the trust gate) returns `needs_confirmation` for every student, never `met` and never `unmet`, and the reason names the failed check. **Missing** (FR-ING-02c) returns `needs_confirmation` for every student, with the reason "prerequisite data not provided by the catalog". |
| FR-ING-09 | A topic tag scored below the threshold MUST be stored and labeled as uncertain in the UI. |
| FR-ING-10 | Every derived item MUST be stored with its source text, the model name, a timestamp and the check results. It MUST be cached by a hash of (input text, model name, prompt version), so re-running ingestion on unchanged input returns identical output without calling the model. |
| FR-ING-11 | Ingestion MUST derive the term's default week-one anchor as the most common start date among its lecture sections and store it with the term. If `TERM_START` is set, it MUST override the derived value. |
| FR-ING-12 | Ingestion MUST be resumable. An interrupted run that is restarted MUST process only items that are not yet cached, and MUST report progress and a summary at the end (courses ingested, rules trusted vs. uncertain, tags uncertain, failures). |
| FR-ING-13 | A seed script MUST load the cached export and the ingestion outputs into Postgres on first start. It MUST be idempotent, so re-running it never duplicates rows. |
| FR-ING-14 | Course text and requirements-document text MUST be passed to the model as clearly delimited data, never as instructions (NFR-SEC-11). |

### 4.2 Natural-Language Schedule Planning (FR-PLN)

| ID | Requirement |
|---|---|
| FR-PLN-01 | The planner agent MUST turn a student request into **hard constraints** (for example, "no Friday classes", "2 ML electives", explicit exclusions, explicit inclusions such as "I also want to take CS 4820", credit range) and **soft preferences** (for example, "prefer Fridays free", "nothing too heavy"). If the student gives no credit range, the planner MUST use `DEFAULT_CREDIT_RANGE` (default 12–18) as a hard constraint (DEC-21). |
| FR-PLN-02 | Before solving, the UI MUST show this interpretation in an editable **interpretation panel**: each constraint and preference with its type (hard or soft); the credit range, labeled "default" when it came from `DEFAULT_CREDIT_RANGE`, and editable; the credit target; and, for each topic constraint, the list of courses that will count. After solving, the panel MUST label every course the solver added to fill the credit range as **solver-chosen**. The student MUST be able to **pin** a solver-chosen course, which turns it into an explicit inclusion, or **exclude** it, which turns it into an explicit exclusion, and then re-solve (DEC-22). |
| FR-PLN-03 | For a topic constraint ("N ML electives"), the planner MUST map the student's phrase ("ML", "machine learning", "AI") onto the `TOPIC_TAGS` vocabulary. The panel lists the matching courses. The student can add or remove courses, and the solver enforces "N courses from the confirmed set". |
| FR-PLN-04 | "Nothing too heavy" MUST be interpreted as a lower soft credit target plus a soft penalty on courses with a high workload signal. Both MUST be shown in the panel and be editable. The penalty weight MUST be configurable, and its default is tuned on development examples. |
| FR-PLN-05 | The planner MUST use `search_courses` (query, term, optional level, day and credit filters, returning up to 10 courses with IDs, descriptions and sections), `check_prereqs`, `check_requirements` and `solve_schedule`. Tool arguments are validated as described in FR-AGT. |
| FR-PLN-06 | The planner MUST return up to 3 distinct schedules (DEC-08). Each schedule shows a weekly view, the preferences it meets, its total credits and workload, its outstanding conditions (FR-ELG-05), and its requirement progress. |
| FR-PLN-07 | Completed courses MUST be entered through a sidebar multiselect listing every ingested course plus every course referenced by derived prerequisite rules or requirement categories. A free-text field accepts other courses (DEC-07). Free-text courses are stored and shown, but act only as reported equivalents and can produce only `needs_confirmation`. |
| FR-PLN-08 | **Stretch (SHOULD):** entering completed courses in chat, parsed and shown back for confirmation. |

### 4.3 Eligibility, Requirements and Solving (FR-ELG)

| ID | Requirement |
|---|---|
| FR-ELG-01 | `check_prereqs` MUST evaluate a rule tree with three-valued logic (`met`, `unmet`, `needs_confirmation`) as in technical-environment.md §6.2. It returns the status, the rule as text, the **source text** and a reason. |
| FR-ELG-02 | `check_requirements` MUST report progress and remaining gaps for each extracted category, given completed and proposed courses. Uncertain categories MUST be shown as `needs_confirmation` and never counted as satisfied. The UI MUST say which categories are fully checked (CS core and CS electives). |
| FR-ELG-03 | `solve_schedule` (CP-SAT) MUST enforce as hard constraints: no overlapping meetings; a valid combination of required components for every chosen course; total credits within the requested range; exclusion of `unmet` courses; explicit exclusions (for example, no Friday sections, or an excluded course); **explicit inclusions**: every course the student names or pins MUST appear in the schedule (DEC-23); and topic constraints over the confirmed course set. **Filling the credit range (DEC-22):** when the named constraints don't reach the credit range, the solver MUST fill the remaining credits only from eligible ingested courses. It MUST never use an `unmet` course or an explicitly excluded course. `needs_confirmation` courses are allowed with the FR-ELG-04 penalty. If the student explicitly asks to satisfy a requirement category this term, that category is also enforced. |
| FR-ELG-04 | `needs_confirmation` courses MUST be eligible but carry a soft penalty, so the solver prefers schedules with fewer of them (DEC-14). |
| FR-ELG-05 | Each schedule MUST list its outstanding conditions: every `needs_confirmation` course with its rule, source text and reason. |
| FR-ELG-06 | Soft preferences (for example, preferring Fridays free, a lower credit target, low workload) MUST affect only scoring and ranking, never feasibility. Solver-chosen fill courses MUST be ranked by soft preferences and, once requirement categories exist (FR-ING-06 and FR-ELG-02, Tier 2), by how much they close remaining requirement gaps (DEC-22). |
| FR-ELG-07 | **Distinctness (DEC-08).** The solver MUST first look for schedules whose course sets differ pairwise. Only if fewer than 3 exist may it add schedules that differ only in sections. |
| FR-ELG-08 | A **separate checker**, independent of the solver, MUST re-validate every schedule against the hard constraints in FR-ELG-03 before it is displayed, **including** explicit inclusions (every named or pinned course is present) and the rule that solver-chosen courses are never `unmet` or excluded. A schedule that fails MUST NOT be displayed, and the failure MUST be traced. |
| FR-ELG-09 | When no schedule is feasible, the solver MUST return a structured infeasible result that names the conflicting constraints it can identify. The planner MUST explain the conflict and ask which constraint the student will relax. |
| FR-ELG-10 | Tools MUST return structured results that distinguish invalid arguments, missing information, and a valid search with no matches. |

### 4.4 Chat Refinement (FR-REF)

| ID | Requirement |
|---|---|
| FR-REF-01 | A refinement MUST change only what the student asked for. "Swap out the 8am" replaces only the referenced section and keeps all other choices and constraints where feasible. |
| FR-REF-02 | A schedule-wide addition ("no classes before 9am") MUST add a hard constraint and re-solve, keeping the earlier constraints. |
| FR-REF-03 | If a reference matches more than one item (two 8am sections, for example), the system MUST ask which one the student means, even when the intent is clear. |

### 4.5 Schedule Selection and Handoff (FR-SEL)

| ID | Requirement |
|---|---|
| FR-SEL-01 | The student MUST explicitly mark one schedule as active. Its course IDs and section IDs are saved in session state. |
| FR-SEL-02 | The active schedule's course IDs MUST become the course list for materials questions and multi-course deadline queries. |
| FR-SEL-03 | The system MUST NOT enroll the student or contact any registration system. |

### 4.6 Course Materials and Cited Q&A (FR-MAT)

| ID | Requirement |
|---|---|
| FR-MAT-01 | The student MUST be able to upload text-based PDF syllabi and assignment documents for courses in the active schedule. Uploads MUST be under 20 MB and MUST be checked by file signature, not extension alone. |
| FR-MAT-02 | Ingestion of uploads MUST extract text with page numbers (PyMuPDF) and detect sections from headings and common syllabus labels ("Grading", "Late Policy"). It MUST split long sections into chunks of about 500 tokens. Each chunk MUST keep its course, document, heading, page or page range, and a stable ID. If heading detection fails, the page reference MUST still be kept. |
| FR-MAT-03 | A PDF with no extractable text MUST be rejected with a message that scanned or image-only PDFs are not supported. |
| FR-MAT-04 | `search_materials` MUST run hybrid retrieval: Postgres full-text search plus pgvector similarity (bge-small), with course and document filters, up to 20 results from each method, reciprocal rank fusion, and the top 5 passages sent to the agent. These numbers are starting settings, tuned on development examples and then frozen. |
| FR-MAT-05 | **Query rewriting** (router model) MUST resolve references from context ("when is it due?" becomes "CS 4780 Homework 3 due date"), expand vocabulary while keeping the original words, course codes and assignment IDs, and output search queries, course filters and an optional date range. If the reference is ambiguous (two assignments, for example), the agent MUST ask. Rewriting MUST be switchable on and off for evaluation. |
| FR-MAT-06 | The materials agent MUST receive passages labeled `[C1]`, `[C2]` and so on, and attach each factual claim to its supporting passage. |
| FR-MAT-07 | **Citation validity.** Before display, every cited ID MUST belong to evidence retrieved for the current request. A quoted date or number MUST appear in the cited passage, allowing for equivalent formats ("December 5" and "Dec 5"). **Relative dates (DEC-24):** the citation covers the **original wording** ("Friday of Week 6"), which MUST appear in the passage. A date computed from it ("Friday, October 2, 2026") MUST be visibly labeled as an interpretation from the week-one anchor, not as cited text, and is exempt from the quoted-date check. If this fails, the agent may retry once. If it fails again, the answer MUST be reported as unverified. |
| FR-MAT-08 | Opening a citation MUST show the document, section, page and extracted text. |
| FR-MAT-09 | When the uploaded materials can't answer a question, the system MUST say which information it could not find instead of guessing. |

### 4.7 Deadlines and Calendar Export (FR-DL)

| ID | Requirement |
|---|---|
| FR-DL-01 | `extract_deadlines` MUST produce candidate events from a document. Each event has the original date wording, the interpreted date and time if one can be determined, the source passage, and a status (`awaiting_review`, `confirmed`, `conflict`, `tbd`). |
| FR-DL-02 | Relative dates ("Friday of Week 6") MUST be resolved against the course's week-one anchor: the student's per-course override, else `TERM_START`, else the derived anchor (DEC-06). **Week numbering (DEC-25):** weeks run Monday to Sunday, and Week 1 is the week that contains the anchor date. Every date resolved from the default anchor MUST be marked for review. |
| FR-DL-03 | "Today" for relative questions ("this week", "next Friday") MUST be `TODAY_OVERRIDE` when it is set, and otherwise the current date in America/New_York (DEC-11). "This week" MUST be the Monday–Sunday week that contains today (DEC-25). |
| FR-DL-04 | The **review table** MUST show each event's assignment, original wording, interpreted date and time, source passage and status. The student MUST be able to edit dates and times, fill in TBD items, resolve conflicts between documents, and confirm events. A TBD item MUST stay unresolved until the student supplies a date. |
| FR-DL-05 | "What's due this week?" MUST collect **confirmed** events for the active schedule's courses in the requested date range. It MAY search materials for missing information, and any newly found dates MUST be labeled as awaiting review. |
| FR-DL-06 | `export_ics` MUST generate an `.ics` file with `icalendar` containing **only confirmed events**. Each event includes its source (document, section, page) in the description. All times are in America/New_York (with VTIMEZONE). Deadlines with a stated time become timed events, and date-only deadlines become all-day events (DEC-09). |
| FR-DL-07 | Each change to an event's date, time or status MUST be recorded with a timestamp and before and after values (NFR-SEC-13). |

### 4.8 Intent Routing (FR-RTE)

| ID | Requirement |
|---|---|
| FR-RTE-01 | The router (router model) MUST classify each message into one of `plan_schedule`, `refine_schedule`, `ask_course_info`, `ask_deadline`, `export_calendar` or `out_of_scope`. It uses the new message, the last 3 turns and relevant session state. |
| FR-RTE-02 | The router MUST return a Pydantic-validated record with the intent, extracted details, a confidence and an optional clarifying question. |
| FR-RTE-03 | The system MUST ask a clarifying question when the request is ambiguous, either because the confidence is below the router threshold (starting value 0.6, kept only if development examples show it helps) or because session state makes the reference ambiguous. |
| FR-RTE-04 | A topic switch in mid-conversation MUST re-route the message. The receiving agent gets the selected courses and relevant context, not the other agent's full output. |
| FR-RTE-05 | `out_of_scope` messages MUST get a fixed reply that describes what CoursePilot can help with. |

### 4.9 Shared Agent Loop (FR-AGT)

| ID | Requirement |
|---|---|
| FR-AGT-01 | Both agents MUST share one loop. The model requests a tool, the app validates the arguments with Pydantic, runs only a tool registered for that agent, and returns the result to the model. |
| FR-AGT-02 | A validation failure MUST be returned to the model as a tool result it can correct, and the tool MUST NOT run. |
| FR-AGT-03 | The loop MUST stop at a final answer or after **8 tool calls** per request. At the limit, the agent reports any verified partial result and what is still unresolved. |
| FR-AGT-04 | The planner's tools are `search_courses`, `check_prereqs`, `check_requirements` and `solve_schedule`. The materials agent's tools are `search_materials`, `extract_deadlines` and `export_ics`. |
| FR-AGT-05 | `chat_structured` MUST retry once with the validation error, then return a structured failure. |

### 4.10 Session Memory (FR-MEM)

| ID | Requirement |
|---|---|
| FR-MEM-01 | Postgres MUST store, per session ID: messages, structured state (term, completed courses, reported equivalents, hard constraints, preferences, confirmed topic course sets, candidate schedules, active schedule with course and section IDs, current course and assignment) and a running summary. |
| FR-MEM-02 | Once a conversation passes about 20 turns, the router model MUST summarize the older dialogue. Each specialist receives the summary, relevant state and up to 6 recent turns. |
| FR-MEM-03 | State MUST last for the session only. There are no long-term profiles. |

### 4.11 Tracing (FR-TRC)

| ID | Requirement |
|---|---|
| FR-TRC-01 | Every request MUST get a `trace_id` and a trace that records the intent, agent, tool names, relevant arguments, durations, errors, model names, whether thinking was on, token counts and total latency. |
| FR-TRC-02 | A Streamlit **trace viewer** MUST list traces and show their details, so the team can tell routing errors from retrieval errors and answer errors. |
| FR-TRC-03 | Traces MUST NOT contain uploaded document text (NFR-SEC-03). |

### 4.12 Evaluation Runner (FR-EVAL)

| ID | Requirement |
|---|---|
| FR-EVAL-01 | `uv run python -m eval.run` MUST run versioned cases from `eval/` and write a report. Development examples and frozen final cases MUST be kept separate. |
| FR-EVAL-02 | The runner MUST cover the suites in §7 and report raw counts next to percentages. |
| FR-EVAL-03 | Reports MUST record the model names, `num_ctx`, `TODAY_OVERRIDE`, the configuration (for example, rewriting on or off), latency and tokens. Estimated dollar cost is recorded as zero. |
| FR-EVAL-04 | Open-ended answers MUST be scored 1–5 for correctness, groundedness and helpfulness by a different model, and about 20% of those judgments are reviewed by hand. |
| FR-EVAL-05 | Hand-labeled evaluation data (prerequisite trees, tags, deadlines, routing labels) is **test data, not product curation**. It MUST live only in `eval/` and MUST NOT be read by the product at runtime. |

### 4.13 UI (FR-UI)

| ID | Requirement |
|---|---|
| FR-UI-01 | The Streamlit pages MUST cover chat, the interpretation panel, the schedule comparison with a weekly view, the citation viewer, the deadline review table, and the trace viewer. |
| FR-UI-02 | A progress indicator MUST appear while a request runs (DEC-10). |
| FR-UI-03 | Every derived rule, tag, workload signal and requirement category shown to the student MUST be able to show its source text. Uncertain items MUST be labeled visibly. |
| FR-UI-04 | The UI MUST show the configured term and which requirement categories are fully checked. |

---

## 5. Non-Functional Requirements

### 5.1 Performance (NFR-PERF)
| ID | Requirement |
|---|---|
| NFR-PERF-01 | Soft target: routing under about **3 s** and a full agent answer (including tool calls) under about **30 s**, measured with the default models on an Apple Silicon laptop with 16 GB of memory. These are reported in evaluation and are not pass/fail gates. |
| NFR-PERF-02 | Ingestion is an offline batch job with no latency target. It MUST be resumable (FR-ING-12) and show progress. |
| NFR-PERF-03 | The solver MUST time out within a configurable limit and return the best verified results found, or a timeout result, rather than hang. |
| NFR-PERF-04 | Thinking MUST be turned off for routing and tool selection. |

### 5.2 Correctness as a Safety Property (NFR-COR)
| ID | Requirement |
|---|---|
| NFR-COR-01 | **Zero hard-rule violations** in any displayed schedule (enforced by FR-ELG-08). |
| NFR-COR-02 | **Fail-safe derivation.** `met` may come only from a **none** status found by Python (FR-ING-02a) or from a rule tree that passed every trust-gate check (FR-ING-07). A missing field or an uncertain rule never gives `met`. Evaluation counts every false `met` (§7). |
| NFR-COR-03 | Every hard rule (conflicts, credits, prerequisites, topic constraints, citation validity) MUST be enforced in Python, never by the model. |
| NFR-COR-04 | Running ingestion again on unchanged input with the same model and prompt version MUST give identical stored output (FR-ING-10). |

### 5.3 Security (NFR-SEC), per the Security Baseline split chosen in C4:A
| ID | Rule | Requirement |
|---|---|---|
| NFR-SEC-01 | SECURITY-01 (adapted) | The Postgres port MUST NOT be published to the host network, so only the `app` container reaches it on the Compose network. Data at rest relies on full-disk encryption on the laptop (FileVault or BitLocker), and the README MUST say so. No TLS between the app and database containers. |
| NFR-SEC-03 | SECURITY-03 (adapted) | Structured logs (timestamp, `trace_id`, level, message) go to stdout and the traces table through one configured logger. They contain no document text, no secrets and no PII. `print` is not used for logging. |
| NFR-SEC-04 | SECURITY-04 (adapted) | Streamlit's XSRF protection MUST be on. Streamlit can't set custom response headers, and the README MUST record this limitation. The app is bound to `localhost`. |
| NFR-SEC-05 | SECURITY-05 | Every input is validated with Pydantic: chat messages (with a maximum length), tool arguments, router output, uploads (≤ 20 MB, PDF signature), Class Roster API responses and every derived record. Every string has an explicit maximum length. All SQL is parameterized. Model output and document text are escaped before display, so Streamlit never renders them as raw HTML. |
| NFR-SEC-09 | SECURITY-09 | **No default credentials.** The database password MUST NOT be a fixed default that is shared or committed. `.env.example` MUST use a placeholder, Compose MUST read the value from `.env`, and the app MUST refuse to start if the placeholder is still set. Error messages shown to users MUST be generic, with details only in logs and traces. Images MUST use current, supported versions. This matches technical-environment.md §2: `POSTGRES_PASSWORD=change-me` is a placeholder, and `DATABASE_URL` takes the password from `.env`. |
| NFR-SEC-10 | SECURITY-10 | `uv.lock` is committed. Every Docker image (base image, `pgvector`) MUST be pinned to an exact tag, never `latest`. A dependency vulnerability scanner MUST be documented in the build instructions and run before a PR; it is chosen in NFR Requirements and added through a PR per TC-06. Unused dependencies are removed. Models are pulled only from the official Ollama library by exact tag. Packages come only from PyPI. An SBOM is generated as part of the build instructions. |
| NFR-SEC-11 | SECURITY-11 | Security-critical logic (upload validation, tool-argument validation, citation validation, the trust gate) MUST live in dedicated modules. Validation, a tool allowlist per agent and the separate checker give defense in depth. Misuse cases MUST be documented in the design, including at least: prompt injection inside uploaded syllabi or course descriptions; a crafted PDF (oversized, wrong type, or no text); model-generated tool calls with wrong or malicious arguments; and attempts to make the model run arbitrary SQL or code. Rate limiting does not apply because there is no public endpoint (the app is bound to `localhost`). |
| NFR-SEC-13 | SECURITY-13 | No `pickle`, `eval()` or `exec()`. JSON plus Pydantic only. Changes to confirmed events are auditable (FR-DL-07). No scripts are loaded from an external CDN. |
| NFR-SEC-15 | SECURITY-15 | Every external call (database, Ollama, file I/O, `httpx` during ingestion) MUST have explicit error handling and release its resources in `finally` or a context manager. `handle_message` and the Streamlit entry point MUST have a top-level error handler that logs the error and returns a generic message. On any error, validation and checks MUST fail closed: no unvalidated schedule, citation or derived rule may be shown as verified. |
| NFR-SEC-16 | technical-environment.md §5 | `.env` is in `.gitignore`. A `gitleaks` pre-commit hook runs on every commit. Ollama is never exposed beyond the host (no `OLLAMA_HOST=0.0.0.0` on shared networks). Containers run as a non-root user. The app collects no NetIDs, names or grades. |

The Security Baseline rules SECURITY-02, 06, 07, 08, 12 and 14 are **N/A** for this deployment. See §9 for the reasons.

### 5.4 Privacy (NFR-PRIV)
| ID | Requirement |
|---|---|
| NFR-PRIV-01 | No student data leaves the laptop at runtime. The model, database, embeddings and solver all run locally. |
| NFR-PRIV-02 | Data is scoped to the session and limited to what the product needs: completed courses, preferences and uploaded documents. |
| NFR-PRIV-03 | Test and demo syllabi MUST be public or owned by the team. |

### 5.5 Portability and Maintainability (NFR-PORT)
| ID | Requirement |
|---|---|
| NFR-PORT-01 | Switching to another term MUST need only configuration changes (`TERM`, optionally `TERM_START`) and a re-run of ingestion. No code or data edits. |
| NFR-PORT-02 | Supporting another institution MUST need only a new `CourseSource` adapter, a new `REQUIREMENTS_DOC` and configuration (building the adapter itself is out of scope). |
| NFR-PORT-03 | Model names, thresholds, the topic vocabulary and the retrieval settings MUST be configuration, not hard-coded. |
| NFR-PORT-04 | The code MUST follow the layout in technical-environment.md §1, including `app/ingest/`. `ruff` and `mypy` MUST pass before every PR. |

### 5.6 Testability (NFR-TEST)
| ID | Requirement |
|---|---|
| NFR-TEST-01 | Deterministic code MUST have fast unit tests that run offline and need no model: prerequisite evaluation, the trust-gate cross-checks, the solver, the checker, requirement evaluation, anchor derivation, date resolution, the `.ics` exporter and citation validation. |
| NFR-TEST-02 | Unit tests MUST mock `app/llm.py`. Tests that need Ollama are marked `@pytest.mark.live` and excluded by default. |
| NFR-TEST-03 | Database tests MUST use a throwaway schema created by a fixture. |
| NFR-TEST-04 | Coverage target: 80% on `app/tools/`, `app/state.py`, the schedule checker, and the trust gate in `app/ingest/`. |
| NFR-TEST-05 | **Property-based tests (PBT Partial: PBT-02, 03, 07, 08, 09)** with `hypothesis` MUST cover at least: |
| | (a) *Solver invariants (PBT-03):* no schedule from `solve_schedule` has overlapping meetings, a missing required component, credits outside the range, an `unmet` course, a missing explicit inclusion, a solver-chosen course that is `unmet` or excluded, or a violated exclusion or topic constraint. |
| | (b) *Checker agreement (PBT-03):* every solver output passes the checker, and schedules generated with a planted violation fail it. |
| | (c) *Prerequisite evaluation (PBT-03):* for any student: adding completed courses never changes `met` to something else; a **none** status always gives `met`; an **uncertain** rule always gives `needs_confirmation`; and a **missing** status always gives `needs_confirmation`. |
| | (d) *Trust gate and status classification (PBT-03):* a rule tree that is missing any course code from its source text is never trusted. Prerequisite text that contains any course code is never classified as **none**. A field the source didn't return is always classified as **missing**, never as **none**. |
| | (e) *`.ics` round-trip (PBT-02):* export confirmed events, parse the file with `icalendar`, and get the same set of events, times, all-day flags and descriptions. Unconfirmed events never appear. |
| | (f) *Pydantic round-trips (PBT-02):* derived records and `SessionState` survive JSON serialization and deserialization unchanged. |
| | (g) *Generators (PBT-07):* shared domain strategies for sections, meetings, rule trees and events that respect domain constraints. |
| | (h) *Reproducibility (PBT-08):* shrinking stays on, and seeds are logged on failure or fixed in the test run. |
| NFR-TEST-06 | Business-critical paths MUST also have example-based tests (advisory under Partial PBT, but required by the technical environment), for example the "CS 2110 or equivalent, and MATH 1920" case. |

### 5.7 Usability and Setup (NFR-USE)
| ID | Requirement |
|---|---|
| NFR-USE-01 | A teammate MUST be able to run the demo from the documented setup (pull the models, `cp .env.example .env`, run ingestion, `docker compose up --build`) without editing the database by hand. |
| NFR-USE-02 | Internet is needed only for setup and ingestion. The demo itself runs offline. |
| NFR-USE-03 | Teammates with less than about 8 GB of free memory MUST be able to switch to `qwen3:4b` by changing configuration alone. |

---

## 6. Key User Journeys (acceptance view)

1. **Plan and refine.** The student enters completed courses in the sidebar and types "2 ML electives, no Friday classes, nothing too heavy". The interpretation panel shows 2 hard constraints, a lower credit target, a workload penalty and the list of `ml` courses, and the student edits it. The student sees up to 3 checked schedules with outstanding conditions. "Swap out the 8am" replaces only that section. If nothing is feasible, CoursePilot explains why and asks which constraint to relax.
2. **Ask about materials.** The student marks a schedule active, uploads a syllabus, and asks "What's the late policy?" and then "When is Homework 3 due?". The answers carry citations that open to the document, section, page and text, or CoursePilot says what it could not find.
3. **Review and export deadlines.** The student asks "What's due this week?" (relative to `TODAY_OVERRIDE`), reviews the extracted events, fixes the anchor or ambiguous dates, confirms events, and exports an `.ics` file that imports into Google Calendar.

---

## 7. Success Metrics

The final cases are kept separate from development examples. Every result is reported as raw counts and percentages.

| Area | Cases | Target |
|---|---|---|
| Planning | 15 hand-checked requests (feasible, infeasible, unresolved eligibility) | 0 hard-rule violations. Report how many feasible requests return a schedule and how many unresolved prerequisites are flagged |
| Prerequisite parsing | Hand-labeled trees for 30 courses (including "or equivalent", corequisite and permission-only wording) | ≥ 85% exactly correct. **0 false `met`** (every error fails safe) |
| Topic tagging | Hand-labeled tags for 40 courses | Precision and recall per tag. Precision ≥ 85% for `ml` |
| Routing | 50 labeled messages, including follow-ups | ≥ 90% intent accuracy, with a confusion matrix and a review of clarifying questions |
| Retrieval and cited Q&A | 30 questions (25 answerable, 5 unanswerable) | ≥ 85% top-5 source hits and ≥ 90% correct citations on answerable questions. Answer support checked separately. Correct abstentions out of 5 |
| Query rewriting | Same questions with rewriting on and off | Change in retrieval success, plus the added latency and tokens |
| Deadline extraction | Hand-labeled lists for 3 syllabi | ≥ 90% precision and recall before student corrections. The `.ics` contains only confirmed events and imports into a working calendar |
| Session memory | 5 scripted multi-turn conversations | Expected and actual state match after every turn, including the handoff from planning to materials |
| Latency | All of the above | Reported against NFR-PERF-01 |

---

## 8. Out of Scope

- Any hand-curated course data, tags, flags or rules in the product.
- Catalog adapters for institutions other than Cornell. Majors other than CS (the design is portable, but only the CS requirements document is ingested and evaluated).
- More than one term, live seat availability, and waitlists.
- Enrollment, add/drop, Student Center integration, Canvas integration, and direct Google Calendar sync.
- Scanned or image-only PDFs (OCR).
- Rubric-based feedback on drafts, and long-term student profiles.
- MCP servers, a Next.js frontend, user accounts and multi-user support, cloud hosting, and mobile apps.
- Hosted LLM APIs.
- The metrics dashboard (p50/p95 latency, tokens per request, tool error rate). The evaluation runner records these numbers instead.
- Self-consistency (double derivation) during ingestion (D1). It may be revisited if the prerequisite-parsing evaluation shows it is needed.

---

## 9. Extension Compliance at This Stage

### Security Baseline (enabled)
| Rule | Status | Rationale |
|---|---|---|
| SECURITY-01 | Compliant (adapted) | NFR-SEC-01: the Postgres port is not published, and the laptop has full-disk encryption. TLS is waived for a single-host Compose network (C4:A) |
| SECURITY-02 | N/A | No load balancers, API gateways or CDNs |
| SECURITY-03 | Compliant (adapted) | NFR-SEC-03: structured logs go to stdout and the traces table, with no document text or PII |
| SECURITY-04 | Compliant (adapted) | NFR-SEC-04: XSRF protection, `localhost` binding, and the header limitation documented |
| SECURITY-05 | Compliant | NFR-SEC-05 |
| SECURITY-06 | N/A | No cloud IAM |
| SECURITY-07 | N/A | No cloud network. The `localhost` binding and the unpublished database port are covered by NFR-SEC-01 and NFR-SEC-04 |
| SECURITY-08 | N/A | Single user on `localhost` with no accounts or authentication. Session data is scoped by session ID |
| SECURITY-09 | Compliant | NFR-SEC-09 requires no default database credentials. technical-environment.md §2 uses a placeholder password and refuses to start while it is set |
| SECURITY-10 | Compliant | NFR-SEC-10. The scanner tool is chosen in NFR Requirements |
| SECURITY-11 | Compliant | NFR-SEC-11. Misuse cases are listed and must be carried into Application Design |
| SECURITY-12 | N/A | No authentication. No credentials are hard-coded (NFR-SEC-09 and NFR-SEC-16) |
| SECURITY-13 | Compliant | NFR-SEC-13 |
| SECURITY-14 | N/A | No alerting infrastructure for a local demo (C4:A). Traces give diagnostics |
| SECURITY-15 | Compliant | NFR-SEC-15 |

### Property-Based Testing (Partial)
| Rule | Status | Rationale |
|---|---|---|
| PBT-02 | Compliant | NFR-TEST-05(e) and (f) list the required round-trips |
| PBT-03 | Compliant | NFR-TEST-05(a)–(d) list the required invariants |
| PBT-07 | Compliant | NFR-TEST-05(g) |
| PBT-08 | Compliant | NFR-TEST-05(h) |
| PBT-09 | Compliant | `hypothesis` is in technical-environment.md §3 |
| PBT-01, 04, 05, 06, 10 | Advisory | Partial mode. PBT-01 will still be applied in Functional Design |

### Resiliency Baseline
Disabled (Q13:B). Not loaded or enforced.

---

## 10. Delivery Priority for December 16

MUST and SHOULD say **whether** a requirement is in MVP scope. Tiers say **in what order** the work is done (DEC-20). Every MUST is still MVP scope. A Tier 2 item is not optional, but if time runs out, Tier 2 is cut before any Tier 1 work. Moving any MUST out of scope before December 16 is a scope change and MUST be logged in `audit.md`.

### Tier 1: Critical path for the end-to-end demo
The demo is: ingest CS courses → plan with the solver and checker → keep a schedule → upload a syllabus → get a cited answer → review deadlines → export `.ics`.

| Area | Requirements |
|---|---|
| Ingestion of CS prerequisites and tags | FR-ING-01, 02, 03, 04, 07, 08, 09, 10, 11, 13, 14 (run with `SUBJECTS=CS`) |
| Planning with the solver and checker | FR-PLN-01, 02, 03, 05, 06, 07; FR-ELG-01, 03, 04, 05, 06, 07, 08, 10; FR-SEL-01, 02, 03 |
| Cited Q&A | FR-MAT-01, 02, 03, 04, 06, 07, 08, 09 |
| Deadline review and export | FR-DL-01, 02, 04, 06, 07 |
| Enabling dependencies (the four areas above can't run end to end without these) | FR-AGT-01–05 (shared agent loop); FR-RTE-01, 02 (routing to the correct agent in one chat); FR-MEM-01, 03 (session state that carries the active schedule to the materials agent); FR-UI-01 (every page except the trace viewer), FR-UI-03 |

The Tier 1 demo request is "2 ML electives, no Friday classes". "Nothing too heavy" depends on Tier 2 items (FR-ING-05, FR-PLN-04).

### Tier 2: All other MUST requirements
| Area | Requirements |
|---|---|
| Ingestion | FR-ING-05 (workload signal), FR-ING-06 (requirement categories), FR-ING-12 (resumable runs); adding `MATH` to `SUBJECTS` |
| Planning | FR-PLN-04 ("nothing too heavy"); FR-ELG-02 (requirement progress); FR-ELG-09 (infeasibility explanation) |
| Refinement | FR-REF-01, 02, 03 |
| Materials | FR-MAT-05 (query rewriting, with an on/off switch) |
| Deadlines | FR-DL-03 (`TODAY_OVERRIDE`), FR-DL-05 ("What's due this week?") |
| Routing and memory | FR-RTE-03, 04, 05; FR-MEM-02 |
| Tracing | FR-TRC-01, 02, 03; FR-UI-01 (trace viewer) |
| UI | FR-UI-02, FR-UI-04 |
| Evaluation | FR-EVAL-01–05 (the README timeline needs these by December 8) |
| Non-functional | NFR-PERF-01, 02 (latency reporting, ingestion progress); NFR-TEST-04 (coverage target); NFR-USE-03 (switch to `qwen3:4b`) |

### Tier 3: SHOULD
| Area | Requirements |
|---|---|
| Planning | FR-PLN-08 (completed-course entry in chat) |

Tier 3 starts only after Tier 2 is complete, and it must never displace Tier 1 or Tier 2 work.

### Constraints that apply in every tier
These requirements constrain **whatever code is built**, in any tier, so they are never deferred: NFR-COR-01–04, NFR-SEC-01–16 (Security Baseline, blocking), NFR-PRIV-01–03, NFR-PORT-01–04, NFR-PERF-03, 04, NFR-TEST-01–03, 05, 06 (PBT Partial, blocking for the code each item covers), and NFR-USE-01, 02. For example, a Tier 1 solver ships with its NFR-TEST-05(a)/(b) property tests. Requirement categories (Tier 2) ship with the trust gate when they are built.

### Mapping to the README timeline
| README milestone | Tier work |
|---|---|
| Nov 25–Dec 1 | Tier 1 ingestion, planning, selection and cited Q&A (README check: schedule → keep → upload → open a citation) |
| Dec 2–7 | Tier 1 deadline review and export, then Tier 2 (README check: a follow-up planning request, plus a confirmed deadline imported into a calendar) |
| Dec 8–12 | Tier 2 evaluation runs and fixes |
| Dec 13–15 | Buffer. Tier 3 only if Tier 2 is complete |

---

## 11. Traceability

| Vision MVP feature | Requirements | Metric |
|---|---|---|
| 1. Automatic course ingestion | FR-ING-01–14, NFR-COR-02/04, NFR-PORT-01/02 | Prerequisite parsing, topic tagging |
| 2. Natural-language planning | FR-PLN-01–08, FR-UI-01 | Planning |
| 3. Deterministic checks | FR-ELG-01–10, NFR-COR-01/03, NFR-TEST-05 | Planning (0 violations) |
| 4. Refinement and infeasibility | FR-REF-01–03, FR-ELG-09 | Planning, session memory |
| 5. Selection and handoff | FR-SEL-01–03, FR-MEM-01 | Session memory |
| 6. Upload and cited Q&A | FR-MAT-01–09 | Retrieval and cited Q&A, query rewriting |
| 7. Multi-course deadline questions | FR-DL-03, FR-DL-05 | Deadline extraction, session memory |
| 8. Deadline review and `.ics` export | FR-DL-01–07 | Deadline extraction |
| 9. Intent routing | FR-RTE-01–05 | Routing |
| 10. Session memory | FR-MEM-01–03 | Session memory |
| 11. Request tracing | FR-TRC-01–03 | (used for diagnosis) |
