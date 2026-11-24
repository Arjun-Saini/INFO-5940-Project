# CoursePilot

*An AI agent that helps Cornell students plan their semester, then stay on top of it.*

A student can ask for “two ML electives and no Friday classes,” compare schedules, and say “swap out the 8am.” Once they choose a schedule, they can upload a syllabus for one of its courses and ask “When is Homework 3 due?” CoursePilot will show the supporting passage and let the student add the confirmed deadline to their calendar.

## Team

| Full name | NetID | Main area |
|---|---|---|
| [Full name] | [netid] | Course data and schedule solver |
| [Full name] | [netid] | Planner agent, router, and memory |
| [Full name] | [netid] | Course materials RAG and deadlines |
| [Full name] | [netid] | UI, tracing, and evaluation |

| Project detail | Plan |
|---|---|
| Coding agent | [Team’s selected coding agent] |
| Methodology | AI-Driven Development Lifecycle (AI-DLC) v1.0.1 |
| LLM provider | Ollama, running locally (agent: `qwen3:8b`; router and query rewriting: `llama3.2:3b`) |
| Final deadline | December 16 |
| Status | Project proposal. Features, commands, and metrics below describe the planned implementation. |

## Problem and why we are using an LLM

Cornell students make semester plans from information spread across Class Roster, degree checklists, and prerequisite descriptions. After choosing their courses, they face a similar problem across syllabi and assignment specs. A question as simple as “What’s due this week?” can require looking through several documents. Our first version focuses on Cornell undergraduate CS students, one upcoming term, and a set of courses and requirements the team can verify.

An LLM helps interpret requests such as “nothing too heavy alongside my project,” resolve follow-up references such as “the 8am,” and find passages that express a policy in different words. The student can inspect how CoursePilot interpreted a preference and change it. Python tools check meeting times, credits, encoded prerequisites, and requirement progress. For questions about uploaded materials, the agent cites the passage supporting its answer or tells the student which information it could not find.

## Planned LLM integration

CoursePilot will use a router and two specialist agents. Both run on free, open models served locally by Ollama, so the product needs no API key and no student data leaves the laptop. A small, fast model (`llama3.2:3b`) will route messages and rewrite search queries. A larger model with native tool calling (`qwen3:8b`) will interpret planning requests, select tools, and compose answers. Model names are configuration, not code, so a teammate with less memory can switch to `qwen3:4b`; we will confirm the final choices on development examples. Because these models are small, every tool argument and structured output is validated with Pydantic, and all hard rules (conflicts, credits, prerequisites, citation validity) are enforced in Python rather than trusted to the model.

The **planner agent** will turn goals into constraints, call course search and eligibility tools, and ask a solver for candidate schedules. The **materials agent** will search uploaded documents, answer questions with citations, and prepare deadlines for review. The two agents will share structured session state. When a student keeps a schedule, its course IDs become available to the materials agent. This is how “What’s due this week in my courses?” will refer to the courses the student actually selected.

## Project scope and feasibility

By December 16, we plan to demonstrate three capabilities in one local Streamlit app:

1. **Plan and refine a schedule.** For one term and a documented set of CS courses, return up to three schedules that satisfy the encoded hard constraints. Explain which preferences each schedule meets and flag any eligibility conditions that still require confirmation.
2. **Ask questions about course materials.** Upload text-based syllabus and assignment PDFs, ask questions across turns, and open the source passage for each factual answer.
3. **Review and export deadlines.** Show extracted deadlines alongside their sources, let the student correct or confirm them, and export approved events as an `.ics` file for import into Google Calendar.

The four work areas share one agent-loop implementation, one database, and one evaluation runner. We will cache one term of Class Roster data and hand-review the prerequisite rules and CS requirement categories included in the demo. The README and UI will identify that coverage so students know which checks were performed.

## Planned techniques

The values below, including retrieval depth, chunk size, routing threshold, and tool-call budget, are starting settings. We will tune them on development examples and freeze them before the final evaluation.

### 1. Multi-agent design

The router will send each request to a specialist with its own prompt and tool list. The planner will have four course and scheduling tools. The materials agent will have three document and calendar tools. Both will use the same loop: the model requests a tool, the app validates and executes the call, and the model receives the result. The loop ends with a final answer or after eight tool calls for that request. At the limit, the agent will report any verified partial result and explain what remains unresolved.

A topic switch will trigger routing again. If a student follows a planning request with “What’s the late policy for CS 4780?”, the materials agent will receive the selected course and relevant conversation context. It can answer without loading the planner’s entire solver output.

### 2. Intent identification

The router will receive the new message, the last three turns, and relevant state such as the active course or chosen schedule. The model will return a Pydantic-validated record with an intent label, extracted details, a confidence estimate, and an optional clarifying question.

Supported intents are `plan_schedule`, `refine_schedule`, `ask_course_info`, `ask_deadline`, `export_calendar`, and `out_of_scope`. For example, “Move my 8am lecture and keep Fridays free” should produce `refine_schedule`, identify the referenced lecture, and preserve the Friday constraint.

Clarification will depend on both the model output and the session state. If two lectures match “my 8am,” the app will ask which lecture the student means even if the intent itself is clear. We will start with `0.6` as a candidate confidence threshold and use development examples to decide whether it improves routing. Final evaluation will report intent accuracy and a confusion matrix.

### 3. Tool calling

Each tool will be a Python function with Pydantic input and output models. The app will validate model-generated arguments before execution. Tools will return structured results that distinguish invalid arguments, missing information, and valid searches with no matches. This gives the agent a concrete next step, such as asking for a course ID or explaining that no schedule satisfies the current constraints.

| Tool | Main inputs | Result |
|---|---|---|
| `search_courses` | Query, term, optional level, day, and credit filters | Up to 10 matching courses with course IDs, descriptions, and available sections |
| `check_prereqs` | Course ID and completed courses | `met`, `unmet`, or `needs_confirmation`, with the rule and reason |
| `check_requirements` | Completed courses and proposed courses | Progress and remaining gaps for each supported CS requirement category |
| `solve_schedule` | Candidate sections, eligibility results, hard constraints, and preferences | Up to three distinct schedules with scores and outstanding conditions, or an infeasible result |
| `search_materials` | Search queries and course or document filters | Ranked passages with stable chunk IDs, document names, sections, and pages |
| `extract_deadlines` | Document ID and course calendar context | Candidate events with original date text, interpreted date if available, and source passage |
| `export_ics` | Confirmed event IDs from the current session | An `.ics` file containing the approved events |

### 4. Retrieval-augmented generation over course materials

**Ingestion.** PyMuPDF will extract text and page numbers from uploaded PDFs. We will identify sections using headings, formatting, and common syllabus labels such as “Grading” and “Late Policy.” Long sections will be split into chunks of about 500 tokens. Each chunk will retain its course, document, heading, page or page range, and stable ID. If heading detection fails, the page reference will still give the student a way to inspect the source.

**Retrieval.** PostgreSQL full-text search will find exact terms such as `HW3` or `P2`. A local `bge-small-en-v1.5` embedding model and pgvector will support semantic matches such as “handing something in late” and “late submission policy.” After applying course filters, we will retrieve up to 20 results from each method, merge the rankings with reciprocal rank fusion, and send the top five passages to the agent. We will also reuse the embedding model for course-description search.

**Evaluation.** For answerable questions, we will check whether at least one accepted supporting section appears in the top five. We will inspect failures separately when the source text was lost during extraction, the wrong course was searched, or the correct passage ranked too low.

### 5. Query rewriting

Before document retrieval, the model will receive the question, recent context, selected courses, and a short glossary. It will return search queries, course filters, and a date range when relevant.

- **Resolve references.** After a discussion of CS 4780 Homework 3, “When is it due?” becomes “CS 4780 Homework 3 due date.” If the conversation mentions two assignments, the agent will ask which one the student means.
- **Expand vocabulary.** A query about a “prelim” can retain the original word and add “midterm exam.” Course codes and assignment identifiers will be preserved.
- **Handle multiple courses.** For “What’s due this week?”, the app will use the chosen schedule to identify relevant courses and query confirmed events for the requested dates. It can search uploaded materials for missing information and label newly found dates as awaiting review.

We will run the same questions with rewriting enabled and disabled, keeping the retrieval index, filters, and top-five cutoff fixed. We will report the change in retrieval success alongside the extra latency and token cost.

### 6. Grounded citations

The materials agent will receive passages labeled with IDs such as `[C1]` and `[C2]`. It will associate each factual claim with its supporting passage. In Streamlit, a student can open a citation to see the document, section, page, and extracted text.

We will check two distinct properties:

| Check | How we will verify it |
|---|---|
| **Citation validity** | Before display, verify that the cited ID belongs to evidence retrieved for this request. For a quoted date or number, check the relevant passage while accounting for equivalent formats such as “December 5” and “Dec 5.” |
| **Answer support** | During evaluation, compare the answer with the source and the expected fact. A response that says “48 hours” fails if the cited late policy grants 24 hours, even when it links to the right page. |

If a citation fails validation, the agent may retry once using the retrieved evidence. If support is still missing, it will report that it could not verify the answer from the uploaded materials. Five unanswerable questions will test this behavior.

### 7. Deterministic rules and schedule solving

**Course data and prerequisites.** PostgreSQL will store courses, enrollment components, meeting times, credits, prerequisite rules, and supported CS requirement categories. An LLM may draft a logical expression from prerequisite prose, and a teammate will review each rule in the supported course set. For “CS 2110 or equivalent, and MATH 1920,” the rule must preserve the equivalency branch. A student who completed MATH 1920 and reports an unverified equivalent to CS 2110 will receive `needs_confirmation`. The UI will show the condition alongside any affected schedule.

**Hard constraints and preferences.** OR-Tools CP-SAT will select compatible sections and enforce meeting-time conflicts, required component combinations, the student’s credit range, verified eligibility restrictions, and explicit exclusions. “I cannot attend on Fridays” excludes Friday sections. “I’d prefer Fridays free” ranks schedules with that property more highly. For “nothing too heavy,” the planner may propose a lower credit target and show that interpretation to the student. Requirement checks will show progress toward supported categories. If the student explicitly asks to satisfy one this term, the planner can enforce it.

**Results.** The solver will return up to three distinct schedules when available. A separate checker will validate every displayed schedule against the encoded hard constraints. If none is feasible, the planner will explain the conflict it can establish and ask which constraint the student is willing to change. We will measure both rule violations and how often the planner returns a schedule on cases known to be feasible.

### 8. Session memory

Postgres will store messages, structured state, and a running summary under a session ID. Structured state will include the term, completed courses, hard constraints, preferences, candidate schedules, selected section IDs, and current course or assignment. Once a conversation exceeds roughly 20 turns, the router model will summarize older dialogue. Each specialist will receive the summary, relevant state, and up to six recent turns.

A refinement changes only what the student requested. **“Swap out the 8am”** will replace the referenced section and preserve other choices where feasible. **“No classes before 9am”** will add a schedule-wide constraint. We will use five scripted conversations to check the expected state after each turn, including a handoff from planning to course materials.

### 9. Human review and calendar export

A student will explicitly select which proposed schedule becomes active. CoursePilot will use that choice to create the course list available to the materials agent. Enrollment remains the student’s responsibility.

For deadlines, a review table will show the assignment, original wording, interpreted date and time, source passage, and confirmation status. A phrase such as “Friday of Week 6” requires a course-specific week-one anchor. If the syllabus and term calendar do not establish that anchor, or two documents give conflicting dates, the student can resolve the ambiguity before export. A `TBD` item will remain unresolved until a date is supplied.

The Python `icalendar` library will generate the `.ics` file from confirmed events. Each event will include its source in the description. We will test the exported file through an actual calendar import, checking both date-only deadlines and deadlines with explicit times.

### 10. Observability

Each request will receive a `trace_id`. The trace will record its intent, selected agent, tool names, relevant arguments, durations, errors, model names, token use, and total latency. For a wrong deadline answer, the team should be able to tell whether the router selected the wrong agent, the search targeted the wrong assignment, or the answer misstated a correctly retrieved date.

A basic trace viewer will be available in Streamlit. If time permits, a developer page will summarize median and 95th-percentile latency, tokens per request, and tool error rate. The evaluation runner will also save these measurements so the team can compare configurations before the dashboard is finished.

### 11. Evaluation

The planned command `python -m eval.run` will run versioned cases from `eval/` and write a report. We will use separate development examples for prompt changes, thresholds, and retrieval settings. The following figures are final evaluation targets.

| Capability | Planned cases | Measures |
|---|---|---|
| Planning | 15 hand-checked requests covering feasible, infeasible, and unresolved-eligibility cases | Zero encoded hard-rule violations. Report how many known-feasible requests produce a schedule and whether unresolved prerequisites are flagged. |
| Routing | 50 labeled messages, including follow-ups | Target at least 90% intent accuracy. Report the confusion matrix and inspect unnecessary or missed clarifying questions. |
| Retrieval and cited Q&A | 30 questions, including 25 answerable and 5 unanswerable | On answerable cases, target at least 85% top-five source hits and 90% correct source citations. Check answer support separately. Report correct abstentions out of five. |
| Deadline extraction | Hand-labeled deadline lists for three syllabi | Target at least 90% precision and recall **before student corrections**. Check that export contains exactly the confirmed events. |
| Session memory | Five scripted multi-turn conversations | Compare expected and actual state after each turn, including preserved constraints and selected course IDs. |

For open-ended answers, a different model will score correctness, groundedness, and helpfulness on a 1–5 scale using the expected answer and source passages. We will manually review about 20% of its judgments. Reports will include counts alongside percentages so readers can interpret results from the small test sets.

### 12. Local hosting

Ollama will run natively on each laptop (so it can use the GPU or Apple Silicon) and serve the models at `localhost:11434`. Docker Compose will run two services. `db` will run PostgreSQL with pgvector and a persistent volume. `app` will run Streamlit, the agents, the Python tools, and the local embedding model, and will reach Ollama on the host at `host.docker.internal:11434`. The embedding model will be downloaded during the image build. A first-start seed script will load the cached roster and reviewed rules and can be rerun without duplicating records.

The planned local setup is:

```bash
# Install Ollama from https://ollama.com/download, then:
ollama pull qwen3:8b
ollama pull llama3.2:3b

cp .env.example .env   # local settings only; no API keys
docker compose up --build
```

The app will be available at `http://localhost:8501`. The `.env` file will be excluded from version control. Internet is needed only for setup (installing Ollama, pulling models, building the image); the demo itself runs offline. The 8B model needs roughly 8 GB of free memory.

## Timeline

| Dates | Milestone | Completion check |
|---|---|---|
| Nov 22–24 | AI-DLC Inception, user stories, scope, tool schemas, Docker skeleton | Agree on the supported course set, hard-rule coverage, shared-state format, and one demo conversation. Each owner has sample inputs and outputs. |
| Nov 25–Dec 1 | Build the first integrated workflow around a small set of real courses and one syllabus. Plan around Thanksgiving availability. | From one Streamlit session, generate a schedule, keep it, upload a syllabus for a selected course, and open a citation from an answer. |
| Dec 2–7 | Expand reviewed course data, add refinements, deadline review and export, and evaluation cases | Complete a follow-up planning request and import a confirmed deadline into a calendar. If the earlier workflow is delayed, prioritize it ahead of dashboard work. |
| Dec 8–12 | Run the full evaluation, compare retrieval with and without rewriting, and fix the largest failures | Produce a report with case counts, quality metrics, token use, latency, and representative failures. |
| Dec 13–15 | Buffer, final documents, and demo rehearsal | A teammate can start the app from the documented setup and complete the demo without manual database edits. |
| Dec 16 | Submit | Final repository, README, AI-DLC artifacts, and evaluation report. |

## Collaboration plan

We will meet one-three times a week. The main in-person meeting covers AI-DLC review gates, design decisions, and integration. We will regularly sync progress and share Thanksgiving and finals availability in advance.

All code and documents will live in one GitHub repository, with tasks tracked in GitHub Projects. During Inception, one teammate will drive the coding agent while the team works through questions and approvals together. During Construction, each teammate will own the area in the Team table and contribute its evaluation examples. Each pull request will receive one teammate review. We will merge changes at least every two days and run the shared demo as soon as the first components connect.

## Future work

Rubric-based feedback on drafts, long-term student profiles, MCP servers for the course and materials tools, broader degree-rule coverage, support for additional majors, a Next.js frontend, Canvas integration, and direct Google Calendar sync.

## Planned repository layout

```text
README.md                    # Project proposal and planned local setup
docker-compose.yml           # Streamlit app and PostgreSQL services
.env.example                 # Required environment variables
app/                         # UI, router, agents, tools, session state, tracing
data/                        # Cached roster, reviewed rules, coverage notes
eval/                        # Development examples, final cases, runner, reports
vision.md                    # Project vision
technical-environment.md     # Stack and environment decisions
.aidlc-rule-details/         # AI-DLC rules; location depends on coding agent
aidlc-docs/
  inception/                 # Inception artifacts
  audit.md                   # Agent-generated audit trail
  aidlc-state.md             # Workflow state
```
