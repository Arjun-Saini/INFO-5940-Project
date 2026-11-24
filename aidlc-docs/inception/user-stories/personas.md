# CoursePilot: Personas

These four personas come from requirements §3.1 and the answer to story-plan Q1:B. Names are fictional. Course histories are **illustrative**: check them against the ingested Fall 2026 data before reusing them as test cases.

---

## P1: Jordan, junior choosing electives (demo persona)

| Attribute | Detail |
|---|---|
| **Year and major** | Junior, CS major (College of Engineering) |
| **Completed** | CS 1110, CS 2110, CS 2800, CS 3110, CS 3410, MATH 1910, MATH 1920, MATH 2940 (CS core is done) |
| **Context** | Planning next semester while working on a large outside project (a research or startup project of about 10 hours a week). Wants two ML electives. Has a standing Friday commitment and can't attend Friday classes |
| **Goals** | Get a conflict-free schedule with 2 ML electives and no Friday classes, without reading every course description. See why each option fits, then keep one and use it all semester |
| **Frustrations** | Cross-checking Class Roster times by hand. Not knowing which electives are "heavy". Searching several syllabi to answer "when is HW3 due?" |
| **Technical comfort** | High. Comfortable with chat UIs, and will open a citation to check it |
| **Epics** | E1 Plan and refine (main), E2 Ask about materials, E3 Review and export deadlines |
| **Representative request** | "I need 2 ML electives and no Friday classes." |

---

## P2: Sam, sophomore still finishing core

| Attribute | Detail |
|---|---|
| **Year and major** | Sophomore, intended CS major |
| **Completed** | CS 1110, MATH 1910 and MATH 1920 at Cornell. Took an object-oriented Java course at a community college that they believe is equivalent to CS 2110, but it hasn't been formally transferred |
| **Context** | Still has core courses left. Many courses that look interesting have prerequisites they may or may not meet. Some catalog entries have unclear wording ("or permission of instructor"), and some have missing prerequisite data |
| **Goals** | Learn which courses they can take, which they can't, and which need confirmation with an advisor, with the rule and the source text for each |
| **Frustrations** | Prerequisite text that is vague or says "or equivalent". Tools that silently assume eligibility. Plans that say "no schedule" without explaining why |
| **Technical comfort** | Medium. Uses the sidebar more than free-form chat |
| **Epics** | E1 Plan and refine (prerequisite and infeasibility paths) |
| **Representative request** | "Plan my next semester with CS 2800 and an intro systems course, 15 credits max." |

---

## P3: Riley, student managing the current semester

| Attribute | Detail |
|---|---|
| **Year and major** | Junior or senior CS student, taking 5 courses |
| **Context** | Has an active schedule and syllabi for each course, often in different formats. Late policies and slip-day rules vary by course. Assignments are often given as "Friday of Week 6" rather than a date |
| **Goals** | Quick answers about deadlines and policies that can be checked against the syllabus. One calendar holding only deadlines Riley has verified |
| **Frustrations** | Wrong dates in the calendar. Answers with no source. Having to re-read a 12-page syllabus to find the late policy |
| **Technical comfort** | High. Imports `.ics` files into Google Calendar |
| **Epics** | E2 Ask about materials (main), E3 Review and export deadlines (main) |
| **Representative requests** | "What's the late policy for CS 4780?", "When is Homework 3 due?", "What's due this week?" |

---

## P4: Team developer

| Attribute | Detail |
|---|---|
| **Role** | One of the four CoursePilot teammates (any owner area) |
| **Context** | Runs ingestion once per term on their own laptop with Ollama. Runs evaluation before the deadline. Diagnoses wrong answers during development and the demo |
| **Goals** | Re-run ingestion for a new term with configuration changes only. See which derived rules are uncertain and why. Trace a wrong answer to its cause (routing, retrieval or answer composition). Produce a reproducible evaluation report |
| **Frustrations** | Long ingestion runs that start over after a crash. LLM output that differs between runs. Failures that can't be reproduced. Laptops with too little memory for the 8B model |
| **Technical comfort** | Expert. Uses the terminal, Docker and pytest |
| **Epics** | E4 Developer operations |
| **Representative tasks** | `uv run python -m app.ingest --term FA26 --subjects CS`, `uv run python -m eval.run`, opening a trace in the trace viewer |
