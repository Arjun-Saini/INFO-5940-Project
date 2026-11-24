# Unit of Work: Story Map

Every story from `stories.md` is assigned to one **primary unit**, which is responsible for its acceptance criteria. **Contributing units** supply parts of it. The tier comes from `stories.md`. Criteria marked **[Tier 2]** inside a Tier 1 story are built in the unit's Tier 2 pass.

| Story | Title | Tier | Primary unit | Contributing units | Notes |
|---|---|---|---|---|---|
| US-01 | Enter completed courses | 1 | U3 | U5 (sidebar), U0 (state) | `set_completed_courses`. The multiselect lists courses from U1 data |
| US-02 | State goals and review the interpretation | 1 | U3 | U5 (panel), U0 (loop) | `INTERPRET_SPEC`, defaults, topic-set mapping |
| US-03 | Get up to three checked schedules | 1 | U2 | U3 (`solve_confirmed`, `explain`), U5 (comparison view) | Progress indicator and gap ranking are [Tier 2] |
| US-04 | See prerequisite status with its source | 1 | U2 | U1 (records), U5 (display) | |
| US-05 | "Nothing too heavy" | 2 | U3 | U1 (workload signal), U2 (penalty) | |
| US-06 | See requirement progress | 2 | U2 | U1 (categories), U5 (display) | |
| US-07 | Refine by chat | 2 | U3 | U2 (`solve_tool`) | `REFINE_SPEC` |
| US-08 | Understand an infeasible request | 2 | U2 | U3 (explanation) | |
| US-09 | Keep a schedule | 1 | U3 | U5 (button), U0 (state) | IP-3 handoff |
| US-10 | Enter completed courses in chat | 3 | U3 | U5 | Only if Tier 2 is done |
| US-11 | Upload a syllabus | 1 | U4 | U5 (upload widget), U0 (ownership) | |
| US-12 | Ask a question and get a cited answer | 1 | U4 | U3 (router), U5 (citation viewer) | |
| US-13 | Ask follow-ups and switch topics | 2 | U4 | U3 (re-routing, summary) | Rewriting with its on/off switch |
| US-14 | Review extracted deadlines | 1 | U4 | U5 (review page), U1 (derived anchor) | |
| US-15 | Export confirmed deadlines to `.ics` | 1 | U4 | U5 (download), U3 (router intent) | |
| US-16 | Ask "What's due this week?" | 2 | U4 | U3 (router intent) | |
| US-17 | Ingest a term's catalog | 1 | U1 | U0 (Docker, seed hook) | |
| US-18 | Derive prerequisite rules and topic tags safely | 1 | U1 | | |
| US-19 | Derive workload signals and requirement categories | 2 | U1 | | |
| US-20 | Re-run ingestion for a new term or a smaller laptop | 2 | U1 | U0 (config) | |
| US-21 | Trace a wrong answer | 2 | U5 | U0 (tracing) | |
| US-22 | Run the evaluation | 2 | U5 | U1, U2, U3, U4 (suites and cases, Q8:A) | |
| MS-01 | Prompt injection in a syllabus or course description | 1 | U4 | U1 (description path) | |
| MS-02 | Malicious or malformed tool arguments | 1 | U0 | U2, U3, U4 (their tools' ownership checks and bounds) | Allowlist, `AgentContext` and the ownership checks live in U0 |
| MS-03 | Crafted or malformed PDF upload | 1 | U4 | U5 (error display) | |

---

## Per-Unit View

| Unit | Primary stories | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|---|
| U0 | MS-02 | 1 | 0 | 0 |
| U1 | US-17, 18, 19, 20 | 2 | 2 | 0 |
| U2 | US-03, 04, 06, 08 | 2 | 2 | 0 |
| U3 | US-01, 02, 05, 07, 09, 10 | 3 | 2 | 1 |
| U4 | US-11, 12, 13, 14, 15, 16, MS-01, MS-03 | 6 | 2 | 0 |
| U5 | US-21, 22 | 0 | 2 | 0 |
| **Total** | **25** | **14** | **10** | **1** |

U5 has no Tier 1 *primary* stories, but it contributes the UI for 10 Tier 1 stories (US-01, 02, 03, 04, 09, 11, 12, 14, 15 and MS-03).

---

## Tier 1 Demo Script to Units

| Demo step | Story | Units involved |
|---|---|---|
| Setup | US-17, 18 | U1 |
| 1 Completed courses | US-01 | U3, U5 |
| 2–3 Planning message, then Find schedules | US-02 | U3, U5 |
| 4 Schedules A, B and C | US-03 | U2, U3, U5 |
| 5 Condition detail | US-04 | U2, U5 |
| 6 Keep a schedule | US-09 | U3, U5 |
| 7 Upload | US-11 | U4, U5 |
| 8–10 Questions | US-12 | U4, U3, U5 |
| 11–12 Review and confirm | US-14 | U4, U5 |
| 13–14 Export and import | US-15 | U4, U5 |

Every demo step is covered by units in their Tier 1 pass.

---

## Validation

| Check | Result |
|---|---|
| Every component in exactly one unit | Yes. Shared repositories are split by side: `courses_repo` and `derived_repo` read side in U0, write side in U1 |
| Every story in at least one unit | Yes, 25 of 25 |
| Unit boundaries respect R1–R12 | Yes. For example, U2's checker shares only U0's models (R4), and U5's UI imports only the service layer (R1) |
| No unit dependency cycles | Yes |
| Owner workload balance | **U4 (MAT) is the heaviest**, with 6 Tier 1 stories (4 user stories plus MS-01 and MS-03). DATA holds two units, with UX helping on U1 in week 1. **Mitigation:** after week 1, UX's U1 time moves to U4's Tier 1 UI-adjacent work and evaluation cases, and the team reviews the balance at the first weekly meeting after U0 |
| Component ownership changes from `components.md` | T-1 `search_courses` is placed in **U3** (PLAN), per Q1:A, although `components.md` lists its owner as DATA. Course search is consumed only by the planner. DATA reviews its queries |
