# CoursePilot: Vision

## Summary

CoursePilot is an AI agent designed to help Cornell students plan and manage their semester. Students can make requests like "2 ML electives and no Friday classes", compare up to 3 valid schedules, and refine them through chat ("swap out the 8am"). Once a student finalizes their schedule, they can upload their syllabi and ask questions like "when is Homework 3 due?". CoursePilot answers with a citation the student can open to confirm the deadline themselves, then export it to their calendar.

The agent's LLM interprets "fuzzy" goals, follow-up references, and differently worded policies. CoursePilot also includes deterministic Python tools for hard rules such as meeting-time conflicts, credits, prerequisites, and requirement progress. Nothing in the product is curated by hand. CoursePilot derives prerequisite rules, topic tags (such as "ML"), workload signals, and requirement categories automatically from course names, descriptions, and an official requirements document, so it works for a new term, or another institution, by changing configuration and re-running ingestion. Anything it cannot derive confidently is shown as needing confirmation, never treated as satisfied. The first version runs locally as a Streamlit application with open models served by Ollama, and covers one term of Cornell CS courses (Fall 2026 as the demo stand-in).

## Problem Statement

To plan a semester, Cornell students need information from several separate sources, including Class Roster, degree checklists, and prerequisite descriptions. To accomplish a goal such as "2 ML electives, no Friday classes, nothing too heavy alongside a project", students must read every class description, manually check meeting times for conflicts with other classes, and research the course load themselves. This is time-consuming, and students can still miss vital information. Existing tools search courses and detect conflicts, but none understands preferences and delivers a fitting schedule.

After choosing courses and starting the semester, students must keep up with multiple syllabi, deadlines, and individual course policies. A simple question like "what is due this week?" means checking each syllabus and course schedule for deadlines and slip-day policies.

General chatbots don't solve this, because they don't have access to Cornell's current course data or the student's actual course documents. They cannot guarantee that a schedule is free of conflicts or meets prerequisite requirements.

## Target Users

- **Cornell undergraduate CS students planning their next semester.** Example: a junior who has finished their core requirements and needs electives that fit their projects and commitments outside of class.
- **Cornell undergraduate CS students managing their current semester.** They are working through 4–6 courses and keeping track of each one's deadlines and policies.

## What We're Building

A local Streamlit app with a PostgreSQL database, a router, and two specialized agents. The agents run on open models served locally by Ollama and share structured session data and state.

- **Router:** classifies each message into an intent such as `plan_schedule`, `refine_schedule`, or `ask_course_info`. It uses the last 3 turns and session state, and asks a clarifying question when a request is ambiguous.
- **Planner agent:** turns requests and goals into hard constraints and soft preferences, calls course search tools, and asks a constraint solver (OR-Tools CP-SAT) for up to 3 distinct schedules. For each schedule, it highlights which preferences are met and whether any constraints still need more information.
- **Materials agent:** searches uploaded course documents such as syllabi using hybrid retrieval, backs up answers with citations the student can open, and extracts deadlines for the student to review.

## Key User Journeys

### Journey 1: Plan and refine a schedule

1. The student enters the courses they have completed and their request for the upcoming semester: "I need 2 ML electives, no Friday classes, and nothing too heavy alongside my project."
2. CoursePilot treats "no Friday classes" and "2 ML electives" as hard constraints, and a lighter credit target as a soft preference.
3. The planner agent searches courses, checks prerequisites and requirement progress, and runs the solver.
4. The student sees up to 3 schedules, each with a weekly view, the preferences it satisfies, and any condition that still needs information.
5. The student asks to "swap out the 8am". Only that section is replaced; every other choice and constraint stays intact.
6. If no schedule satisfies the request, CoursePilot explains the conflict and asks which constraints the student can relax.

**Outcome:** a valid schedule the student understands. The student then handles enrollment themselves.

### Journey 2: Ask about course materials

1. The student marks a schedule as active, so its courses become available to the materials agent.
2. The student uploads a syllabus and homework instructions.
3. The student asks "what's the late policy?" and then "when is Homework 3 due?". Follow-ups like "when is it due?" are resolved from conversation context.
4. The agent backs factual claims with citations. Opening one shows the document, section, page, and extracted text behind the claim.
5. If a question cannot be answered from the materials, CoursePilot says which information it couldn't find instead of guessing.

**Outcome:** a fast answer the student can verify against the source.

### Journey 3: Review and export deadlines

1. The student asks "what's due this week?" or opens the deadline review for an uploaded syllabus.
2. A review table lists each assignment with the original wording, the interpreted date and time, the source passage, and a status.
3. The student corrects ambiguous or conflicting dates, fills in TBD items, and confirms correct events.
4. The student exports an `.ics` file containing only confirmed events, each with its source in the description, and imports it into Google Calendar.

**Outcome:** a calendar of deadlines the student has checked, each one traceable to the syllabus.

## MVP Features (In Scope)

1. **Automatic course ingestion:** pull the configured term and subjects from the Class Roster API through a data-source adapter, and use the LLM to derive prerequisite rules, topic tags, workload signals, and requirement categories from course text and the official requirements document. Every output is validated, cached with its source text, and given a confidence; low-confidence output is flagged, never trusted. Changing the term requires only configuration and a re-run.
2. **Natural-language schedule planning:** parse goals into hard constraints and soft preferences, display that interpretation so the student can edit it, and return up to 3 distinct schedules.
3. **Deterministic eligibility and constraint checks:** prerequisite checks return `met`, `unmet`, or `needs_confirmation` with the derived rule, its source text, and the reason. Requirement progress is shown for the extracted CS categories. Topic constraints such as "2 ML electives" use the derived tags, and the student can adjust which courses count before solving. The solver enforces meeting-time conflicts, required component combinations (lecture, discussion, lab), credit range, verified eligibility, and explicit exclusions, and a separate checker re-validates every displayed schedule.
4. **Chat refinement and infeasibility handling:** follow-ups change only what was requested ("swap out the 8am" changes only that section). When no schedule exists, the planner explains the conflict and asks which constraint to change.
5. **Schedule selection and handoff:** the student keeps one schedule, and its course IDs become the course list for materials questions and multi-course queries.
6. **Course material upload and cited Q&A:** upload text-based syllabus and assignment PDFs, ask questions across courses, resolve references like "it" from chat history, and get answers with citations that open to the document, section, page, and text. The agent says explicitly when it cannot verify an answer from the uploaded materials.
7. **Multi-course deadline questions:** "What's due this week?" collects confirmed events for the selected courses and requested dates, and labels newly found dates as awaiting review.
8. **Deadline review and `.ics` export:** an editable review table with original wording, interpreted date and time, source passage, and confirmation status. Handles relative dates, conflicting dates, and TBD items before export, and exports only confirmed events.
9. **Intent routing with clarification:** messages route to the correct specialist agent, topic switches re-route mid-conversation, and the app asks clarifying questions when a request is ambiguous.
10. **Session memory:** messages, structured state (term, completed courses, constraints, preferences, candidate and selected schedules, current course or assignment), and a running summary persist for the session, so long conversations keep their context.
11. **Request tracing for the team:** every request has a trace (intent, agent, tools, arguments, durations, errors, model, tokens, latency) viewable in a Streamlit trace viewer, so the team can diagnose wrong answers.

## Out of Scope

- Rubric-based feedback on draft submissions.
- Long-term student profiles: state lasts for a session only.
- Majors other than CS (the design is portable, but only the CS requirements document is ingested and evaluated).
- Catalog adapters for institutions other than Cornell.
- Any hand-curated course data, tags, or rules.
- More than one term, and live seat availability or waitlists.
- Enrollment, add/drop, or Student Center integration.
- Canvas integration.
- Direct Google Calendar sync.
- Scanned or image-only PDFs (OCR).
- MCP servers for the course and materials tools.
- A Next.js frontend, user accounts and multi-user support, cloud hosting, and mobile apps.
- Hosted LLM APIs (the product runs on local open models only).
- The metrics dashboard (p50/p95 latency, tokens per request, tool error rate).

## Success Metrics

Measured with the evaluation runner (`python -m eval.run`) on final test cases kept separate from development examples:

- **Planning:** 15 hand-checked requests covering feasible, infeasible, and unresolved-eligibility cases. Zero hard-rule violations in any displayed schedule. Report how many known-feasible requests return a schedule, and whether unresolved prerequisites are flagged.
- **Prerequisite parsing:** hand-labeled rule trees for 30 courses. At least 85% exactly correct, and every error fails safe (it may produce `needs_confirmation`, never a false `met`).
- **Topic tagging:** hand-labeled tags for 40 courses. Precision and recall per tag, with at least 85% precision for `ml`.
- **Routing:** 50 labeled messages, including follow-ups. At least 90% intent accuracy, with a confusion matrix and a review of unnecessary or missed clarifying questions.
- **Retrieval and cited Q&A:** 30 questions (25 answerable, 5 unanswerable). At least 85% top-5 source hits and at least 90% correct source citations on answerable questions, with answer support checked separately. Report correct abstentions out of 5.
- **Query rewriting:** report the change in retrieval success with rewriting on versus off, along with the added latency and tokens.
- **Deadline extraction:** hand-labeled lists for 3 syllabi. At least 90% precision and recall before student corrections. The exported `.ics` contains only confirmed events and imports into a working calendar.
- **Session memory:** 5 scripted multi-turn conversations. Expected and actual state match after every turn, including preserved constraints, selected course IDs, and the planning-to-materials handoff.

Reports include raw counts alongside percentages because the test sets are small.

## Open Questions

1. **Term and subjects:** which term and which subjects (CS only, or also MATH and other prerequisite subjects) do we ingest for the demo?
2. **Requirement categories:** which official CS checklist version do we ingest, and which of its categories (core, electives, practicum, technical electives, etc.) must the extraction handle for the demo?
3. **Class Roster access:** does the Class Roster API return everything ingestion needs (meeting times, components, credits, descriptions, prerequisite text) for the chosen term?
4. **"Too heavy":** how should the automatically derived workload signal be combined with the credit target, and how much should it weigh?
5. **Week-one anchors:** where does the anchor date for relative deadlines (e.g., "Friday of Week 6") come from when the syllabus doesn't state it? The official term calendar by default, with a student override?
6. **Completed-course entry:** without long-term profiles, how do students quickly enter completed courses each session?
7. **Model selection:** are `qwen3:8b` (agent) and `llama3.2:3b` (router and rewriting) accurate enough on our development examples, and fast enough on every teammate's laptop? Does the planned 0.6 router confidence threshold improve routing with a small local model?
8. **Demo scope:** what is the one end-to-end demo conversation (plan, refine, keep, upload, ask, export) we commit to having working first?
