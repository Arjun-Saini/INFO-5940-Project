# Requirements Clarification Questions

All 14 answers are complete and none contradict each other. Four combinations of answers still leave something undefined that the requirements document needs to pin down. Each question below explains the gap and which original answers it comes from.

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe your answer after the tag.

---

## Ambiguity 1: The reference date for "this week" under a Fall 2026 term
You chose Fall 2026 as the stand-in term (Q1:X), and the official Fall 2026 calendar as the week-one anchor (Q7:A). The demo runs in mid-December, when Fall 2026 classes are over. If "What's due this week?" uses the real system date, the demo will usually return no deadlines. The deadline tests would also change result depending on the day they run.

### Clarification Question 1
How should CoursePilot decide what "today" is for relative date questions ("this week", "next Friday")?

A) A configurable `TODAY_OVERRIDE` date in `.env` (for example, a date in mid-October 2026). When it is unset, the real system date in America/New_York is used. Tests and evaluation always set it explicitly

B) Always use the real system date, and phrase demo questions with explicit dates ("What's due the week of October 12?")

C) A date picker in the Streamlit sidebar that sets "today" for the session, with the real date as its default

X) Other (please describe after [Answer]: tag below)

[Answer]: A) A configurable `TODAY_OVERRIDE` date in `.env` (for the demo, a date in mid-October 2026). When it is unset, the real system date in America/New_York is used. Tests and evaluation always set it explicitly, so results do not depend on the day they run.

---

## Ambiguity 2: How an "ML elective" is identified
You chose to encode only the CS core and CS electives categories (Q3:A), and the reviewed set will include 3–4 ML electives (Q2:A). The demo request treats "2 ML electives" as a **hard constraint** that the solver and checker must enforce. However, "ML" is not one of the encoded categories, so there is no deterministic way to decide which courses count as ML.

### Clarification Question 2
How should the solver and checker decide whether a course counts toward "ML electives"?

A) Team-assigned topic tags (for example `ml`, `systems`, `theory`) on each course in the reviewed set, stored with the course data. A topic constraint means "N courses that count as CS electives **and** carry the tag", and the solver and checker enforce it

B) The planner proposes ML courses through semantic description search and shows the list for the student to confirm. The solver then enforces "N of these confirmed courses"

C) Both: team tags are the default set that the solver enforces, and the student can add or remove courses from the set in the interpretation panel before solving

X) Other (please describe after [Answer]: tag below)

[Answer]: X) **Team direction change: no manual curation anywhere in the product.** CoursePilot must build everything it needs automatically from the course data source (course names, descriptions, prerequisite text, meeting times, credits, components), so it works for any term, and for another institution, by changing configuration and re-running ingestion. No team-assigned tags, flags, or hand-reviewed rules.

For "ML electives": during ingestion, the LLM assigns topic tags (for example `ml`, `systems`, `theory`) to every course from its name and description, with a confidence and a one-line justification. Tags are validated with Pydantic, stored with the course, and cached so the same input always gives the same tags. The planner shows which courses it will count for a topic constraint in the interpretation panel, and the student can add or remove courses before solving. The solver and checker then enforce "N courses with the tag (after student edits)" deterministically. Low-confidence tags are shown as uncertain.

This supersedes the manual parts of our earlier answers:
- Q2: no team-reviewed set. The planning set is every course the ingestion pulls for the configured subjects and term.
- Q5: pull from the Class Roster API automatically, with no hand-built fallback. The data source sits behind an adapter interface so another institution's catalog can plug in.
- Q6: no team-assigned `heavy` flag. Workload is the credit target plus an automatically derived workload signal (credits, components such as labs or projects, and LLM-extracted workload cues in the description), used only as a soft penalty and shown to the student.
- Prerequisites: the LLM converts prerequisite text into the rule tree at ingestion. The result is validated and cached with its source text, and any prerequisite that cannot be parsed confidently evaluates to `needs_confirmation`, never `met`.
- Q3: requirement categories are extracted automatically from an official requirements document supplied as input (for Cornell CS, the major checklist), with the same validation, caching, and `needs_confirmation` fallback.

Deterministic Python still enforces every hard rule on the parsed data.

---

## Ambiguity 3: Completed courses outside the reviewed set
You chose a sidebar multiselect "populated from the course set" (Q8:B) and a reviewed set of 15–20 courses (Q2:A). Prerequisite rules for those courses refer to courses outside the set (for example CS 2110 or MATH 1920). A multiselect limited to the 15–20 planning courses would have no way to enter those prerequisites.

### Clarification Question 3
Which courses should the completed-courses multiselect offer?

A) The reviewed planning set **plus** every course mentioned in their reviewed prerequisite rules and requirement categories (generated by the seed script)

B) Every CS, MATH, and other course in the cached Fall 2026 roster export, even if it is not in the planning set

C) Option A, plus a free-text field for courses not in the list, which are stored and shown but never count as `met` (they can only produce `needs_confirmation`)

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Option C, adapted to automatic ingestion: the multiselect offers every course in the ingested roster export plus every course referenced by the automatically parsed prerequisite rules and requirement categories, plus a free-text field for courses not in the list. Free-text courses are stored and shown but never count as `met`; they can only produce `needs_confirmation`, matching `reported_equivalents` in `check_prereqs`.

---

## Ambiguity 4: Which Security Baseline rules are N/A
You enabled the Security Baseline and said rules that do not fit a local, single-user app with no network intermediaries will be marked N/A (Q12:A). Security rules are **blocking**, so the boundary needs to be fixed now. Some rules don't fit cleanly:
- **SECURITY-01** requires encryption at rest and TLS for every data store, and the plan uses local Postgres in Docker.
- **SECURITY-03** requires logging to a centralized log service.
- **SECURITY-04** requires HTTP security headers, which Streamlit does not let apps set directly.

### Clarification Question 4
How should the Security Baseline rules be applied?

A) **N/A, with a documented reason:** 02 (no load balancers, gateways or CDNs), 06 and 07 (no cloud IAM or network), 08 and 12 (no accounts or authentication, single user on localhost), and 14 (no alerting infrastructure for a local demo).
**Adapted to local use:** 01 uses disk encryption on the laptop (FileVault/BitLocker) plus a Postgres port that is not published to the host, instead of TLS. 03 sends structured logs (timestamp, trace_id, level, message, no document text) to stdout and the traces table. 04 is met only on a best-effort basis, through Streamlit's XSRF protection, because Streamlit can't set these headers.
**Enforced fully:** 05, 09, 10, 11, 13, 15

B) Same as A, but enforce SECURITY-01 fully: TLS between the app and `db` containers, plus encryption at rest for the database volume

C) Same as A, but put a local reverse proxy (for example, Caddy) in front of Streamlit so SECURITY-04 headers are fully set

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Use the N/A, adapted, and fully enforced split as listed, with each N/A and adaptation recorded with its reason in the security compliance summary. B and C add infrastructure that a local single-user demo does not need.
