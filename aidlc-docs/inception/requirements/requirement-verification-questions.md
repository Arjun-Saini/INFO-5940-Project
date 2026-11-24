# Requirements Verification Questions

`vision.md`, `technical-environment.md` and `README.md` already settle most of the requirements. These questions cover the vision's **Open Questions**, a few gaps found while checking the documents for completeness, and the three AI-DLC extension opt-ins.

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe your answer after the tag. Answer as a team, since several of these questions affect more than one owner's area.

---

## Question 1
Which term will CoursePilot cache and plan for? (Vision open question 1)

A) Spring 2027 (the next term to be planned as of late November 2026)

B) Fall 2026 (the current term, which already has complete published data, used as a stand-in)

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Fall 2026, used as a stand-in for the next term to plan. Its Class Roster data is complete and published, so the demo will not depend on a partially posted Spring 2027 roster. The term is a configuration value, so switching to Spring 2027 later only means re-running the seed script.

## Question 2
How large should the team-reviewed CS course set be for the demo? (Vision open question 1)

A) Small: about 15–20 courses (core and popular electives, including enough ML electives for the demo request)

B) Medium: about 30–40 courses (most 3000/4000-level CS courses offered that term)

C) Large: every CS course offered that term, with prerequisite rules reviewed only for a subset and the rest marked `needs_confirmation`

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Small: about 15–20 courses. Every prerequisite rule is reviewed by hand, so a small set is what we can verify by December 16. It must include at least 3–4 ML electives so the "2 ML electives" demo request has real choices.

## Question 3
Which CS requirement categories should `check_requirements` encode? (Vision open question 2)

A) Minimal: CS core courses and CS electives only

B) Moderate: CS core, CS electives, practicum/project, and technical electives

C) Broad: everything in option B plus the vectors/concentrations (for example, the ML vector) and the external specialization

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Minimal: CS core and CS electives only. This covers the demo request, and the UI will state which categories are checked. Practicum and technical electives are a stretch goal.

## Question 4
Which checklist version should requirement rules come from? (Vision open question 2)

A) The current CS major checklist for students entering in the most recent catalog year

B) One specific older catalog year that matches the demo persona (please name it under Other)

X) Other (please describe after [Answer]: tag below)

[Answer]: A) The current CS major checklist for the most recent catalog year, which matches our target users.

## Question 5
How will the team get the term's meeting times, components, credits and prerequisites? (Vision open question 3)

A) The seed script pulls once from the public Class Roster API (`httpx`), and the result is committed in `data/` as a cached export

B) Hand-built JSON/CSV export in `data/` (no API use at all)

C) Try the API first and fall back to a hand-built export if the API lacks the needed fields

X) Other (please describe after [Answer]: tag below)

[Answer]: C) The seed script pulls Fall 2026 CS data once from the public Class Roster API (`httpx`) and commits the result to `data/` as a cached export; any field the API lacks is filled in by hand. Prerequisite text is converted into rules and reviewed by a teammate either way. The app never calls Class Roster at runtime.

## Question 6
How should "too heavy" workload be represented? (Vision open question 4)

A) Credit target only: "nothing too heavy" becomes a lower soft credit target that the student can edit

B) Credit target plus a team-assigned `heavy` flag on known high-workload courses in the reviewed set, used as a soft penalty

C) Credit target plus a student-marked "heavy" flag on any course during the session, used as a soft penalty

X) Other (please describe after [Answer]: tag below)

[Answer]: B) Credit target plus a team-assigned `heavy` flag on known high-workload courses in the reviewed set, used as a soft penalty. Credits alone do not capture workload, and we are already reviewing every course in the set, so tagging heavy courses adds little effort.

## Question 7
Where does the week-one anchor for relative deadlines ("Friday of Week 6") come from when the syllabus doesn't state it? (Vision open question 5)

A) The official term calendar (first day of instruction) by default, with a per-course student override in the review table

B) Always ask the student for the anchor before interpreting any relative date

C) Leave relative dates unresolved (status: needs review) until the student supplies a date for each one

X) Other (please describe after [Answer]: tag below)

[Answer]: A) The official term calendar's first day of instruction by default, with a per-course student override in the review table. Any date interpreted from the default anchor is shown for review before export.

## Question 8
How do students enter completed courses each session, since there are no long-term profiles? (Vision open question 6)

A) Free text in chat ("I've taken CS 2110, 2800, 3110…"), which the planner parses and shows back for confirmation

B) A multiselect widget in the sidebar, populated from the course set

C) Both: the sidebar multiselect plus chat entry, kept in sync in session state

D) Paste or upload a plain-text course list (for example, copied from a transcript)

X) Other (please describe after [Answer]: tag below)

[Answer]: B) A multiselect in the sidebar, populated from the course set. It is reliable and does not depend on a small local model parsing free text. Chat entry is a possible stretch goal.

## Question 9
What does "distinct" mean when returning up to 3 schedules?

A) Each schedule differs from the others in at least one **course** (different section times of the same courses don't count)

B) Each schedule differs in at least one **section** (alternate times of the same courses count as distinct)

C) Prefer course-level differences, and fall back to section-level differences when there aren't enough feasible course combinations

X) Other (please describe after [Answer]: tag below)

[Answer]: C) Prefer course-level differences, and fall back to section-level differences when there are not enough feasible course combinations, so the student sees real alternatives without getting fewer than 3 schedules when more exist.

## Question 10
Which time zone and event format should the deadline extraction and `.ics` export use?

A) All times in America/New_York (Ithaca). Deadlines with a time become timed events; date-only deadlines become all-day events

B) America/New_York, but every deadline becomes a timed event (date-only deadlines default to 11:59 PM)

X) Other (please describe after [Answer]: tag below)

[Answer]: A) All times in America/New_York. Deadlines with a stated time become timed events; date-only deadlines become all-day events, so we never invent a time the syllabus did not state.

## Question 11
What response-time target should the app meet on a teammate's laptop with the default models? (No latency target is set yet.)

A) No hard target. Report latency in evaluation, and show a progress indicator in the UI

B) Soft target: routing under ~3 s, and a full agent answer (including tool calls) under ~30 s on an Apple Silicon laptop with 16 GB of memory

C) Strict target: full agent answer under ~15 s, which may force smaller models or fewer tool calls

X) Other (please describe after [Answer]: tag below)

[Answer]: B) Soft target: routing under about 3 s and a full agent answer (including tool calls) under about 30 s on an Apple Silicon laptop with 16 GB of memory. The evaluation reports latency against this target, and the UI shows a progress indicator.

## Question 12: Security Extensions
Should security extension rules be enforced for this project?

A) Yes: enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)

B) No: skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Yes: enforce the SECURITY rules. Uploaded PDFs and model output are untrusted, and our technical environment already requires input validation, secret handling, parameterized SQL, and safe failure. Rules that do not apply to a local, single-user app with no network intermediaries will be marked N/A.

## Question 13: Resiliency Extensions
Should the resiliency baseline be applied to this project?

**What this extension is.** Enabling it applies a set of **directional, design-time best practices** for building resilient systems, derived from the **AWS Well-Architected Framework (Reliability Pillar)** and resilience-review guidance. It steers requirements, design, and code toward fault tolerance, high availability, observability, and recoverability. It covers 15 practice areas across business goals, change management, observability, high availability, disaster recovery, and continuous improvement.

**What this extension is NOT.** Enabling it does **not** make your workload production-ready, nor does it certify or guarantee any availability, RTO, or RPO target. It is a **starting point** that scaffolds good resiliency decisions early. It is not a substitute for a formal **AWS Well-Architected Review** of the built system.

A) Yes: apply the resiliency baseline as directional best practices and design-time guidance (recommended for business-critical workloads, as an informed starting point that you can validate and harden before go-live)

B) No: skip the resiliency baseline (suitable for PoCs, prototypes, and experimental projects where rapid iteration matters more than reliability)

X) Other (please describe after [Answer]: tag below)

[Answer]: B) No: skip the resiliency baseline. CoursePilot is a local course demo with no availability, RTO, or RPO target.

## Question 14: Property-Based Testing Extension
Should property-based testing (PBT) rules be enforced for this project?

A) Yes: enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization, or stateful components)

B) Partial: enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)

C) No: skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin integration layers with no significant business logic)

X) Other (please describe after [Answer]: tag below)

[Answer]: B) Partial: enforce PBT rules for pure functions and serialization round-trips. This covers the schedule solver and checker invariants, prerequisite evaluation, and `.ics` export round-trips, which matches the `hypothesis` tests already planned in the technical environment.
