# Story Generation Plan

**Stage**: INCEPTION - User Stories (Part 1: Planning)
**Input**: `aidlc-docs/inception/requirements/requirements.md` (approved 2026-11-24)
**Outputs**: `aidlc-docs/inception/user-stories/personas.md` and `aidlc-docs/inception/user-stories/stories.md`

This plan describes how the stories will be written. Answer the questions in §3, and the answers will fill in the methodology in §2 before generation starts. Generation (Part 2) does not begin until you approve this plan.

---

## 1. Breakdown Approaches

| Approach | How stories are grouped | Strengths | Weaknesses for CoursePilot |
|---|---|---|---|
| **User journey-based** | By the 3 journeys (plan and refine, ask about materials, review and export) | Mirrors the demo script and the session-memory evaluation conversations | Ingestion and tracing don't fit a student journey |
| **Feature-based** | By the requirement areas (ingestion, planning, eligibility, materials, deadlines, routing, …) | Maps 1:1 to FR IDs and to code modules | Hides the cross-component handoffs where bugs appear |
| **Persona-based** | By planning student, managing student and team developer | Keeps each user's goal clear | The two student personas share much of the system, so stories get duplicated |
| **Domain-based** | By domain (course catalog, schedule, course materials, calendar) | Fits Application Design boundaries | Further from what a demo viewer experiences |
| **Epic-based** | Epics, each split into small stories | Easy to track in GitHub Projects. Epics can follow the journeys, and stories stay small and testable | Needs a rule for what makes an epic |

**Recommendation**: a **hybrid of epics and journeys**. Epics follow the 3 student journeys, plus one epic for the team developer (ingestion, tracing, evaluation). Each epic holds small INVEST stories, and each story is tagged with its FR IDs, tier and owner. Q2 confirms or changes this.

---

## Methodology Decided by the Answers in §3

| Decision | Choice | Source |
|---|---|---|
| Personas | 4 personas: **P1** a junior choosing electives after finishing core (the demo persona); **P2** a sophomore still finishing core, who exercises prerequisite edge cases (reported equivalents, uncertain rules, missing data, the "none" status); **P3** a managing student; **P4** a team developer | Q1:B |
| Organization | Epics by journey: **E1** Plan and refine, **E2** Ask about materials, **E3** Review and export deadlines, **E4** Developer operations. Every story is tagged with its tier | Q2:A |
| Acceptance-criteria format | Given / When / Then for behavior, plus a short UI checklist where needed | Q3:C |
| Detail level | Concrete data in every story. Course facts are **illustrative** and must be checked against the ingested Fall 2026 data before reuse as test cases. `stories.md` will carry this caveat at the top | Q4:A |
| Failure and misuse | Main failure cases go in each story's acceptance criteria. Separate **misuse stories** cover prompt injection (in a syllabus or a course description), malicious tool arguments and crafted PDFs (NFR-SEC-11) | Q5:A |
| Owners | A primary owner area for each story, plus a secondary owner for stories that cross areas. The areas are: **DATA** course data and solver; **PLAN** planner, router and memory; **MAT** course materials RAG and deadlines; **UX** UI, tracing and evaluation | Q6:A |
| Developer work | Written as P4 user stories in E4, with the same acceptance-criteria format | Q7:A |
| Size | Medium: one story per journey step, about 15–25 stories in total, misuse stories included. The INVEST check (Step 9) splits any story that one primary owner can't finish in about 3–4 days | Q8:B |
| NFRs | NFRs appear as acceptance criteria in the stories they affect. A **Constraints** section lists cross-cutting NFRs that every story inherits | Q9:A |

---

## 2. Execution Checklist (Part 2: Generation)

- [x] **Step 1: Personas.** Generate `personas.md` with the personas chosen in Q1, each with their goals, context, frustrations, technical comfort, and the journeys or epics they use.
- [x] **Step 2: Epic skeleton.** Create the epics from the approach chosen in Q2, each with a goal and its FR range.
- [x] **Step 3: Tier 1 stories.** Write every story on the Tier 1 critical path (requirements §10) first. Check that the stories, read in order, describe the whole end-to-end demo.
- [x] **Step 4: Tier 2 stories.** Write stories for the remaining MUST requirements.
- [x] **Step 5: Tier 3 stories.** Write stories for the SHOULD items (FR-PLN-08).
- [x] **Step 6: Unhappy-path and misuse stories.** Cover failure cases with the scope chosen in Q5.
- [x] **Step 7: Acceptance criteria.** Give every story acceptance criteria in the format and level of detail chosen in Q3 and Q4.
- [x] **Step 8: Metadata.** Tag every story with its FR or NFR IDs, tier, persona and the owner area (as chosen in Q6 and Q7).
- [x] **Step 9: INVEST check.** Check each story for Independent, Negotiable, Valuable, Estimable, Small and Testable. Split any story that fails Small at the size chosen in Q8, and note any dependencies a story must keep.
- [x] **Step 10: Coverage check.** Map every FR (all 79) and every NFR in requirements §10 "Constraints that apply in every tier" to at least one story or acceptance criterion, following Q9. List any requirement left unmapped, with the reason.
- [x] **Step 11: Persona map.** Add a persona-to-story table to `stories.md`.
- [x] **Step 11b: Demo script** (added at plan approval). End `stories.md` with a short "Demo script" section. It lists the Tier 1 stories in order as the end-to-end demo conversation for persona P1, with the exact student messages and expected results.
- [x] **Step 12: Final check.** Check that the Markdown and the tables render, the IDs are consistent, and no story contradicts `requirements.md`.

---

## 3. Questions

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe your answer after the tag.

### Question 1
Which personas should `personas.md` define?

A) **3 personas:** a planning student, a managing student, and a team developer (this matches requirements §3.1)

B) **4 personas:** split the planning student into a junior who has finished core courses and is choosing electives (the demo persona), and a sophomore still finishing core courses, who exercises prerequisite edge cases more. Keep the managing student and the team developer

C) **2 personas:** a single "CS student" who both plans and manages a semester, and the team developer

X) Other (please describe after [Answer]: tag below)

[Answer]: B) 4 personas: a junior choosing electives after finishing core (the demo persona), a sophomore still finishing core who hits more prerequisite edge cases (reported equivalents, uncertain or missing prerequisite data), the managing student, and the team developer. Splitting the planning student gives the prerequisite and `needs_confirmation` behavior its own realistic user instead of hiding it inside the demo persona.

### Question 2
How should stories be organized?

A) **Epics by journey (recommended):** Epic 1 Plan and refine, Epic 2 Ask about materials, Epic 3 Review and export deadlines, and Epic 4 Developer operations (ingestion, tracing, evaluation). The epics contain small stories

B) **Feature-based:** one group per requirement area (ingestion, planning, eligibility, refinement, selection, materials, deadlines, routing, memory, tracing, evaluation)

C) **Epics by tier:** Epic 1 is Tier 1 (the demo critical path), Epic 2 is Tier 2, and Epic 3 is Tier 3

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Epics by journey: Epic 1 Plan and refine, Epic 2 Ask about materials, Epic 3 Review and export deadlines, Epic 4 Developer operations (ingestion, tracing, evaluation), each with stories tagged by tier so the Tier 1 path is still easy to see.

### Question 3
Which format should acceptance criteria use?

A) **Given / When / Then** scenarios. Easy to turn into scripted evaluation conversations and pytest cases

B) **Checklists** of testable statements ("The interpretation panel lists each topic-set course with its tag confidence")

C) **Given / When / Then for behavior, plus a short checklist for UI details** that don't fit scenarios (columns, labels, visible source text)

X) Other (please describe after [Answer]: tag below)

[Answer]: C) Given/When/Then for behavior, plus a short checklist for UI details that don't fit a scenario.

### Question 4
How detailed should the acceptance criteria be?

A) **Concrete examples with real data.** For example, "Given completed courses CS 1110, CS 2110 and MATH 1920, and the reported equivalent…". Each story names at least one concrete course, phrase or date. The criteria can then be reused directly as development examples

B) **Generic behavior.** Criteria describe the rule without specific data ("Given a course whose prerequisite field is missing…"), and concrete data is chosen later in evaluation

C) **Generic criteria, plus one concrete example** for each Tier 1 story only

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Concrete examples with real data, so the criteria can be reused as development examples and session-memory scripts. Course facts in the examples (codes, prerequisites, meeting times) are illustrative and must be checked against the ingested Fall 2026 data before they are reused as test cases.

### Question 5
How much should stories cover unhappy paths and misuse cases (NFR-SEC-11)?

A) **Within each story:** every story's acceptance criteria include its main failure cases (for example, infeasible, ambiguous reference, uncertain rule, unanswerable question, scanned PDF). Security misuse cases (prompt injection in a syllabus, malicious tool arguments, a crafted PDF) become separate "misuse stories" in the relevant epic

B) **Within each story only:** failure cases and misuse cases both go in the stories' acceptance criteria, with no separate misuse stories

C) **Happy paths only.** Failure and misuse behavior is left to Functional Design and NFR Design

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Main failure cases inside each story's acceptance criteria, plus separate misuse stories for the security misuse cases (prompt injection in a syllabus or course description, malicious tool arguments, crafted PDFs), so NFR-SEC-11 has clear, testable coverage.

### Question 6
Should each story be tagged with the owner area from the README Team table?

A) **Yes, a primary owner area for each story:** course data and solver; planner, router and memory; course materials RAG and deadlines; or UI, tracing and evaluation. Stories that cross areas list a secondary owner

B) **No.** Ownership is decided later in Units Generation and GitHub Projects

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Yes: a primary owner area from the README Team table for each story, plus a secondary owner for stories that cross areas. This makes handoffs between owners explicit for Construction.

### Question 7
How should the team developer's work (ingestion, tracing, evaluation) be written?

A) **As user stories for the developer persona** ("As a team developer, I want to re-run ingestion for a new term so that…"), with the same acceptance-criteria format

B) **As technical enablers** (not user stories), listed in a separate section with acceptance criteria

C) **Leave them out of the stories.** They are covered by FR-ING, FR-TRC and FR-EVAL and by later design stages

X) Other (please describe after [Answer]: tag below)

[Answer]: A) As user stories for the team developer persona, with the same acceptance-criteria format, so ingestion, tracing and evaluation are tested the same way as student-facing work.

### Question 8
What size should a story be?

A) **Small:** one story is one student-visible behavior that one owner can finish in about 1–2 days (expect roughly 35–50 stories)

B) **Medium:** one story is one step of a journey that can span owners, finishable in about 3–4 days (expect roughly 15–25 stories)

X) Other (please describe after [Answer]: tag below)

[Answer]: B) Medium: one story per journey step, about 15–25 stories. With the owner tags from Q6, cross-owner stories stay trackable, and this size keeps the stories reviewable as a team while still being split by the INVEST check if any story is too large.

### Question 9
How should non-functional requirements appear in the stories?

A) **As acceptance criteria in the stories they affect.** For example, the solver story includes "the separate checker re-validates every displayed schedule" (NFR-COR-01), and the upload story includes the 20 MB limit and the signature check (NFR-SEC-05). Cross-cutting NFRs that fit no single story are listed in a "Constraints" section that every story inherits

B) **As a separate set of NFR stories** ("As a student, I want answers within about 30 s…")

C) **Not in the stories.** The NFRs stay in `requirements.md` only

X) Other (please describe after [Answer]: tag below)

[Answer]: A) As acceptance criteria in the stories they affect, plus a "Constraints" section for cross-cutting NFRs that every story inherits.
