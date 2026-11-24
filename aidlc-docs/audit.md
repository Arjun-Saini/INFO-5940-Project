# AI-DLC Audit Log

## Workflow Start - Initial User Request
**Timestamp**: 2026-11-24T11:35:03Z
**User Input**: "Using AI-DLC, I want to start a new project. Please read vision.md and technical-environment.md, then begin the AI-DLC workflow."
**AI Response**: "Displayed AI-DLC welcome message. Loaded common rules (process-overview, session-continuity, content-validation, question-format-guide) and extension opt-in files (security-baseline, resiliency-baseline, property-based-testing). Read vision.md, technical-environment.md, and README.md."
**Context**: Workflow initiation

---

## Workspace Detection
**Timestamp**: 2026-11-24T11:35:03Z
**User Input**: (none; automatic stage)
**AI Response**: "No aidlc-docs/aidlc-state.md found, so this is a new workflow. Workspace scan found no source code or build files, only README.md, vision.md, technical-environment.md, CLAUDE.md, .gitignore, and .aidlc-rule-details/. Classified as Greenfield. Reverse Engineering skipped. Created aidlc-docs/aidlc-state.md. Proceeding to Requirements Analysis."
**Context**: INCEPTION - Workspace Detection complete

---

## Requirements Analysis - Clarifying Questions Issued
**Timestamp**: 2026-11-24T11:35:03Z
**User Input**: (none yet)
**AI Response**: "Intent analysis: New Project; Clear request (detailed vision, technical environment, and README); System-wide scope; Complex. Selected Comprehensive depth (multi-component agent system, four-person team, measurable evaluation targets). Created aidlc-docs/inception/requirements/requirement-verification-questions.md with 14 questions: the vision's open questions, gaps found during completeness analysis, and the 3 extension opt-in questions. Awaiting user answers."
**Context**: INCEPTION - Requirements Analysis, Step 6 gate

---

## Requirements Analysis - User Answers Received
**Timestamp**: 2026-11-24T11:48:14Z
**User Input**: "We've answered all 14 requirement verification questions in requirement-verification-questions.md. Please read them, log our answers in audit.md, and continue with Requirements Analysis."
**Answers recorded in requirement-verification-questions.md (verbatim)**:
- Question 1: X) Fall 2026, used as a stand-in for the next term to plan. Its Class Roster data is complete and published, so the demo will not depend on a partially posted Spring 2027 roster. The term is a configuration value, so switching to Spring 2027 later only means re-running the seed script.
- Question 2: A) Small: about 15–20 courses. Every prerequisite rule is reviewed by hand, so a small set is what we can verify by December 16. It must include at least 3–4 ML electives so the "2 ML electives" demo request has real choices.
- Question 3: A) Minimal: CS core and CS electives only. This covers the demo request, and the UI will state which categories are checked. Practicum and technical electives are a stretch goal.
- Question 4: A) The current CS major checklist for the most recent catalog year, which matches our target users.
- Question 5: C) The seed script pulls Fall 2026 CS data once from the public Class Roster API (`httpx`) and commits the result to `data/` as a cached export; any field the API lacks is filled in by hand. Prerequisite text is converted into rules and reviewed by a teammate either way. The app never calls Class Roster at runtime.
- Question 6: B) Credit target plus a team-assigned `heavy` flag on known high-workload courses in the reviewed set, used as a soft penalty. Credits alone do not capture workload, and we are already reviewing every course in the set, so tagging heavy courses adds little effort.
- Question 7: A) The official term calendar's first day of instruction by default, with a per-course student override in the review table. Any date interpreted from the default anchor is shown for review before export.
- Question 8: B) A multiselect in the sidebar, populated from the course set. It is reliable and does not depend on a small local model parsing free text. Chat entry is a possible stretch goal.
- Question 9: C) Prefer course-level differences, and fall back to section-level differences when there are not enough feasible course combinations, so the student sees real alternatives without getting fewer than 3 schedules when more exist.
- Question 10: A) All times in America/New_York. Deadlines with a stated time become timed events; date-only deadlines become all-day events, so we never invent a time the syllabus did not state.
- Question 11: B) Soft target: routing under about 3 s and a full agent answer (including tool calls) under about 30 s on an Apple Silicon laptop with 16 GB of memory. The evaluation reports latency against this target, and the UI shows a progress indicator.
- Question 12: Security Extensions: A) Yes: enforce the SECURITY rules. Uploaded PDFs and model output are untrusted, and our technical environment already requires input validation, secret handling, parameterized SQL, and safe failure. Rules that do not apply to a local, single-user app with no network intermediaries will be marked N/A.
- Question 13: Resiliency Extensions: B) No: skip the resiliency baseline. CoursePilot is a local course demo with no availability, RTO, or RPO target.
- Question 14: Property-Based Testing Extension: B) Partial: enforce PBT rules for pure functions and serialization round-trips. This covers the schedule solver and checker invariants, prerequisite evaluation, and `.ics` export round-trips, which matches the `hypothesis` tests already planned in the technical environment.
**AI Response**: "Read and validated all 14 answers (all present and valid). Recorded extension configuration: Security Baseline = Yes, Resiliency Baseline = No, Property-Based Testing = Partial (PBT-02, PBT-03, PBT-07, PBT-08, PBT-09 blocking; others advisory). Loaded security-baseline.md and property-based-testing.md; resiliency-baseline.md not loaded. Contradiction and ambiguity check found no contradictions and 4 ambiguities: (1) choosing Fall 2026 (Q1) leaves the reference date for 'this week' deadline queries undefined at a December demo; (2) the minimal requirement categories (Q3) leave no deterministic way to identify an 'ML elective' for the hard constraint; (3) a sidebar multiselect from the 15-20 course set (Q2, Q8) cannot list prerequisite courses outside that set; (4) which Security Baseline rules are N/A in a local single-user deployment (Q12), especially SECURITY-01, SECURITY-03, and SECURITY-04. Created aidlc-docs/inception/requirements/requirement-clarification-questions.md with 4 questions."
**Context**: INCEPTION - Requirements Analysis, Step 6 answer analysis; awaiting clarification answers

---

## Requirements Analysis - Clarification Answers Received
**Timestamp**: 2026-11-24T11:58:24Z
**User Input**: "We've answered the 4 clarification questions in requirement-clarification-questions.md. Our answer to Question 2 records a team direction change: no manual curation anywhere in the product. Course data, prerequisite rules, topic tags, workload signals, and requirement categories must all be derived automatically from the course catalog and an official requirements document, so the product works for any term (or another institution) by changing configuration and re-running ingestion. This supersedes the manual parts of our earlier answers to Q2, Q3, Q5, and Q6. We have updated vision.md, README.md, and technical-environment.md to reflect this (see the latest commit). Please re-read all three, log the direction change and our answers in audit.md, and apply it throughout requirements.md. If this creates any new conflicts or ambiguities, ask us before writing the document."
**Answers recorded in requirement-clarification-questions.md (verbatim)**:
- Clarification Question 1:

A) A configurable `TODAY_OVERRIDE` date in `.env` (for the demo, a date in mid-October 2026). When it is unset, the real system date in America/New_York is used. Tests and evaluation always set it explicitly, so results do not depend on the day they run.

- Clarification Question 2:

X) **Team direction change: no manual curation anywhere in the product.** CoursePilot must build everything it needs automatically from the course data source (course names, descriptions, prerequisite text, meeting times, credits, components), so it works for any term, and for another institution, by changing configuration and re-running ingestion. No team-assigned tags, flags, or hand-reviewed rules.

For "ML electives": during ingestion, the LLM assigns topic tags (for example `ml`, `systems`, `theory`) to every course from its name and description, with a confidence and a one-line justification. Tags are validated with Pydantic, stored with the course, and cached so the same input always gives the same tags. The planner shows which courses it will count for a topic constraint in the interpretation panel, and the student can add or remove courses before solving. The solver and checker then enforce "N courses with the tag (after student edits)" deterministically. Low-confidence tags are shown as uncertain.

This supersedes the manual parts of our earlier answers:
- Q2: no team-reviewed set. The planning set is every course the ingestion pulls for the configured subjects and term.
- Q5: pull from the Class Roster API automatically, with no hand-built fallback. The data source sits behind an adapter interface so another institution's catalog can plug in.
- Q6: no team-assigned `heavy` flag. Workload is the credit target plus an automatically derived workload signal (credits, components such as labs or projects, and LLM-extracted workload cues in the description), used only as a soft penalty and shown to the student.
- Prerequisites: the LLM converts prerequisite text into the rule tree at ingestion. The result is validated and cached with its source text, and any prerequisite that cannot be parsed confidently evaluates to `needs_confirmation`, never `met`.
- Q3: requirement categories are extracted automatically from an official requirements document supplied as input (for Cornell CS, the major checklist), with the same validation, caching, and `needs_confirmation` fallback.

Deterministic Python still enforces every hard rule on the parsed data.

- Clarification Question 3:

X) Option C, adapted to automatic ingestion: the multiselect offers every course in the ingested roster export plus every course referenced by the automatically parsed prerequisite rules and requirement categories, plus a free-text field for courses not in the list. Free-text courses are stored and shown but never count as `met`; they can only produce `needs_confirmation`, matching `reported_equivalents` in `check_prereqs`.

- Clarification Question 4:

A) Use the N/A, adapted, and fully enforced split as listed, with each N/A and adaptation recorded with its reason in the security compliance summary. B and C add infrastructure that a local single-user demo does not need.

**AI Response**: "Read and validated all 4 clarification answers. Re-read the input documents through commit fb5c7f7 ('Update input docs: fully automatic course ingestion, no manual curation'), which changes vision.md, README.md, and technical-environment.md. Checked the updated documents against each other and against the clarification answers: they are consistent. Found 5 new ambiguities introduced by the direction change, so requirements.md is not written yet (per user instruction). Created aidlc-docs/inception/requirements/requirement-direction-change-questions.md."
**Context**: INCEPTION - Requirements Analysis, clarification round 1 answer analysis

---

## Requirements Analysis - Direction Change: No Manual Curation
**Timestamp**: 2026-11-24T11:58:24Z
**User Input**: (see the previous entry; the direction change is recorded in Clarification Question 2's answer and commit fb5c7f7)
**AI Response**: "Recorded direction change. Course data, prerequisite rules, topic tags, workload signals, and requirement categories are all derived automatically from the course catalog (Class Roster API via a CourseSource adapter) and an official requirements document, with no hand-curated data in the product. Supersedes: Q2:A (team-reviewed 15-20 course set) -> planning set is every ingested course for the configured TERM and SUBJECTS; Q3:A (hand-encoded core and electives) -> categories extracted from REQUIREMENTS_DOC; Q5:C (hand-built fallback) -> automatic API pull only; Q6:B (team-assigned heavy flag) -> derived workload signal used as a soft penalty; README section 7 hand-reviewed prerequisite rules -> LLM-derived rule trees with a confidence, where uncertain rules give needs_confirmation, never met. Unchanged: Q1, Q4, Q7-Q14 and clarifications 1, 3, and 4 (clarification 3's answer is already adapted to automatic ingestion). New evaluation targets were added to the input docs: prerequisite parsing (30 hand-labeled courses, >=85% exact, errors fail safe) and topic tagging (40 hand-labeled courses, >=85% precision for ml). Hand-labeled evaluation data is not product curation, so it does not conflict with the direction change."
**Context**: INCEPTION - Requirements Analysis, scope change

---

## Requirements Analysis - Direction-Change Answers Received
**Timestamp**: 2026-11-24T12:01:16Z
**User Input**: "We've answered the 5 direction-change questions in requirement-direction-change-questions.md. Please log our answers in audit.md and write requirements.md."
**Answers recorded in requirement-direction-change-questions.md (verbatim)**:
- Direction-change Question 1: A) Deterministic cross-checks plus the model's score. A rule is trusted only if every course code in the source text appears in the tree (and vice versa), the text contains no wording the grammar can't express, and the model's score is above the threshold; otherwise it is stored as uncertain and gives `needs_confirmation`. The cross-checks are what make the fail-safe target achievable, since a small model's self-reported confidence alone can't be trusted. We chose not to add self-consistency (B) because deriving every rule twice doubles ingestion time on a laptop for little extra safety beyond the deterministic checks; the parsing evaluation will tell us if that changes.
- Direction-change Question 2: B) Eligible with a soft penalty. Unconfirmed courses can appear, but the solver prefers schedules with fewer of them, and each schedule lists its outstanding conditions with the rule, source text, and reason. Only `unmet` courses are excluded. With automatic derivation many courses may be `needs_confirmation`, so excluding them (C) could leave too few options, while A could fill schedules with unconfirmed courses when confirmed alternatives exist.
- Direction-change Question 3: A) A fixed vocabulary in configuration (`TOPIC_TAGS`), enforced with a Pydantic enum, so tags can't drift. The model assigns tags from each course's name and description, and the planner maps student phrases such as "ML" or "AI" onto the vocabulary. This keeps tagging driven by course text alone and portable to institutions whose checklists define no vectors, and changing the vocabulary is configuration, not curation.
- Direction-change Question 4: A) CS core and CS electives must be extracted and evaluated correctly. Every other category in the checklist is extracted on a best-effort basis and shown with `needs_confirmation`, never counted as satisfied. This keeps the demo scope achievable while still showing the full checklist to the student.
- Direction-change Question 5: C) Derived automatically at ingestion as the most common start date among the term's lecture sections, stored with the term, with an optional `TERM_START` in `.env` that overrides it if the derived date is ever wrong. The per-course student override from Q7 still applies.
**AI Response**: "Read and validated all 5 answers. No contradictions with each other, with the earlier 14 answers, with the 4 clarifications, or with the input documents at commit fb5c7f7. The Step 6 gate is passed. Writing aidlc-docs/inception/requirements/requirements.md (comprehensive depth)."
**Context**: INCEPTION - Requirements Analysis, clarification round 2 answer analysis

---

## Requirements Analysis - Approval Prompt
**Timestamp**: 2026-11-24T12:01:16Z
**User Input**: "continue" (sent while requirements.md was being written; treated as an instruction to finish the stage, not as approval of the document)
**AI Response**: "Created aidlc-docs/inception/requirements/requirements.md: 15 decisions (DEC-01..15), functional requirements in 13 areas (FR-ING, PLN, ELG, REF, SEL, MAT, DL, RTE, AGT, MEM, TRC, EVAL, UI), non-functional requirements (PERF, COR, SEC, PRIV, PORT, TEST, USE), success metrics, out of scope, extension compliance, 4 assumptions to confirm (A-01..A-04), and a traceability table. Security compliance: 9 rules compliant (01, 03 and 04 adapted for local use), 6 N/A, 0 blocking findings. One noted conflict: SECURITY-09 (no default credentials) versus the coursepilot:coursepilot DATABASE_URL example in technical-environment.md; resolved by NFR-SEC-09, and the input doc needs updating. PBT Partial: PBT-02, 03, 07, 08, 09 compliant. Next stage: User Stories. Presented the approval prompt (Request Changes / Approve & Continue)."
**Context**: INCEPTION - Requirements Analysis, Step 9 approval gate

---

## Requirements Analysis - Change Request
**Timestamp**: 2026-11-24T12:06:29Z
**User Input**: "Request Changes on requirements.md:

FR-ING-02: distinguish "no prerequisites" from "missing data". If the catalog returns the prerequisite field and it is empty or explicitly says none, the course has no prerequisites and check_prereqs returns met. Only a field the source did not return at all is stored as missing and gives needs_confirmation. Update FR-ING-08, the trust gate, and the PBT invariants to match.
Add a section on priority for the December 16 deadline. Rank the requirements into tiers: Tier 1 is the critical path for the end-to-end demo (ingestion of CS prerequisites and tags, planning with the solver and checker, cited Q&A, deadline review and export); Tier 2 is everything else that is MUST; Tier 3 is the SHOULD items. Keep the MUST/SHOULD labels as they are.
We updated technical-environment.md (latest commit) to remove the default database password and add TOPIC_TAGS, DERIVE_CONFIDENCE_THRESHOLD, and TERM_START to .env.example. Please re-read it and remove the conflict notes in NFR-SEC-09, §9 SECURITY-09, and A-04.
We confirm assumptions A-01, A-02, and A-03."
**AI Response**: "Re-read technical-environment.md at commit da6c501 (placeholder POSTGRES_PASSWORD=change-me, DATABASE_URL takes the password from .env, plus TOPIC_TAGS, DERIVE_CONFIDENCE_THRESHOLD=0.8 and TERM_START). Updated requirements.md: (1) FR-ING-02 now has three prerequisite statuses: none (field returned and empty, or whole text matches NO_PREREQ_PATTERNS, decided by Python; text with any course code is never none) -> met; text -> derivation and trust gate; missing (field not returned) -> needs_confirmation. Updated FR-ING-07 (the gate applies only to text status), FR-ING-08 (status-to-result mapping), NFR-COR-02 (met only from none or a trusted tree), and PBT invariants NFR-TEST-05(c) and (d). Added DEC-19. (2) Replaced §10 Assumptions with §10 Delivery Priority. Tier 1 is the critical path, plus enabling dependencies FR-AGT-01-05, FR-RTE-01/02, FR-MEM-01/03 and FR-UI-01 (except the trace viewer)/03. Tier 2 is every other MUST. Tier 3 is FR-PLN-08, the only SHOULD. Extension-enforced NFRs constrain every tier. Verified that all 79 FRs are assigned to a tier, and MUST/SHOULD labels are unchanged. Added DEC-20. (3) Removed the conflict notes in NFR-SEC-09, §9 SECURITY-09 and A-04. Updated the §3.3 configuration table. (4) Recorded confirmed A-01, A-02 and A-03 as DEC-16, DEC-17 and DEC-18. Security compliance unchanged: 0 blocking findings. PBT Partial: compliant. Re-presented the approval prompt."
**Context**: INCEPTION - Requirements Analysis, change request applied; awaiting approval

---

## Requirements Analysis - Approval
**Timestamp**: 2026-11-24T12:09:38Z
**User Input**: "Approve & Continue"
**AI Response**: "Requirements approved. Marked Requirements Analysis complete in aidlc-state.md. Proceeding to User Stories (Part 1: Planning)."
**Context**: INCEPTION - Requirements Analysis complete

---

## User Stories - Assessment and Planning Questions Issued
**Timestamp**: 2026-11-24T12:09:38Z
**User Input**: (none; continuing after approval)
**AI Response**: "Loaded inception/user-stories.md. Assessment: Execute (high-priority criteria: new user-facing features, multiple personas, complex business rules, four-person team). Created aidlc-docs/inception/plans/user-stories-assessment.md and aidlc-docs/inception/plans/story-generation-plan.md, with breakdown options and 9 embedded questions. Awaiting answers (Step 8 gate)."
**Context**: INCEPTION - User Stories, Part 1 Planning

---

## User Stories - Planning Answers Received
**Timestamp**: 2026-11-24T12:12:54Z
**User Input**: "Answered all 9 questions in story-generation-plan.md. Please continue."
**Answers recorded in story-generation-plan.md (verbatim)**:
- Question 1: B) 4 personas: a junior choosing electives after finishing core (the demo persona), a sophomore still finishing core who hits more prerequisite edge cases (reported equivalents, uncertain or missing prerequisite data), the managing student, and the team developer. Splitting the planning student gives the prerequisite and `needs_confirmation` behavior its own realistic user instead of hiding it inside the demo persona.
- Question 2: A) Epics by journey: Epic 1 Plan and refine, Epic 2 Ask about materials, Epic 3 Review and export deadlines, Epic 4 Developer operations (ingestion, tracing, evaluation), each with stories tagged by tier so the Tier 1 path is still easy to see.
- Question 3: C) Given/When/Then for behavior, plus a short checklist for UI details that don't fit a scenario.
- Question 4: A) Concrete examples with real data, so the criteria can be reused as development examples and session-memory scripts. Course facts in the examples (codes, prerequisites, meeting times) are illustrative and must be checked against the ingested Fall 2026 data before they are reused as test cases.
- Question 5: A) Main failure cases inside each story's acceptance criteria, plus separate misuse stories for the security misuse cases (prompt injection in a syllabus or course description, malicious tool arguments, crafted PDFs), so NFR-SEC-11 has clear, testable coverage.
- Question 6: A) Yes: a primary owner area from the README Team table for each story, plus a secondary owner for stories that cross areas. This makes handoffs between owners explicit for Construction.
- Question 7: A) As user stories for the team developer persona, with the same acceptance-criteria format, so ingestion, tracing and evaluation are tested the same way as student-facing work.
- Question 8: B) Medium: one story per journey step, about 15–25 stories. With the owner tags from Q6, cross-owner stories stay trackable, and this size keeps the stories reviewable as a team while still being split by the INVEST check if any story is too large.
- Question 9: A) As acceptance criteria in the stories they affect, plus a "Constraints" section for cross-cutting NFRs that every story inherits.
**AI Response**: "Analyzed all 9 answers (Step 9) and found no vague, contradictory or combined-option answers, so no follow-up questions are needed. The Q8 medium size and the Q6 secondary owners fit together, because the INVEST check splits any story one primary owner can't finish in about 3-4 days. Added a 'Methodology Decided by the Answers in §3' table to story-generation-plan.md."
**Context**: INCEPTION - User Stories, Part 1 answer analysis

---

## User Stories - Plan Approval Prompt
**Timestamp**: 2026-11-24T12:12:54Z
**User Input**: (none yet)
**AI Response**: "Approval prompt: 'Story generation plan is ready at aidlc-docs/inception/plans/story-generation-plan.md. Do you approve this plan, so Part 2 (Generation) can create personas.md and stories.md? You may Request Changes to the plan, or Approve the plan to start generation.'"
**Context**: INCEPTION - User Stories, Step 13 plan approval gate

---

## User Stories - Plan Approval
**Timestamp**: 2026-11-24T12:15:02Z
**User Input**: "Approve the plan. One addition: at the end of stories.md, include a short "Demo script" section listing the Tier 1 stories in order as the end-to-end demo conversation (the exact student messages and expected results), using the P1 persona."
**AI Response**: "Plan approved with one addition. Added Step 11b (Demo script) to story-generation-plan.md. Starting Part 2: Generation."
**Context**: INCEPTION - User Stories, plan approved; Part 2 started

---

## User Stories - Generation Complete and Approval Prompt
**Timestamp**: 2026-11-24T12:24:40Z
**User Input**: (none; Part 2 executed after plan approval)
**AI Response**: "Executed every step of story-generation-plan.md and marked each [x]. Created inception/user-stories/personas.md (P1 Jordan, junior and demo persona; P2 Sam, sophomore with prerequisite edge cases; P3 Riley, managing student; P4 Team developer) and inception/user-stories/stories.md. stories.md contains 4 epics, 22 user stories and 3 misuse stories: Tier 1 is US-01, 02, 03, 04, 09, 11, 12, 14, 15, 17, 18 and MS-01, 02, 03; Tier 2 is US-05, 06, 07, 08, 13, 16, 19, 20, 21, 22; Tier 3 is US-10. Also added an inherited Constraints section, coverage tables (all 79 FRs and all listed NFRs mapped), INVEST notes, a persona map, and the P1 Tier 1 demo script (14 steps). Changed personas.md so P2 has completed MATH 1920, so the CS 4780 'CS 2110 or equivalent, and MATH 1920' example gives needs_confirmation. Fixed a demo inconsistency before presenting: with the ml set {CS 4780, CS 4782, CS 4740}, only one pair avoids CS 4782. Recorded 5 open points for Functional Design: OP-01 default credit range; OP-02 how non-topic slots are filled; OP-03 citation validity for computed relative dates; OP-04 week numbering; OP-05 explicit course inclusions, which FR-ELG-03 does not list. Approval prompt: Request Changes, or Approve & Continue to Workflow Planning."
**Context**: INCEPTION - User Stories, Step 19-20 approval gate

---

## User Stories - Change Request (decisions on OP-01 to OP-05)
**Timestamp**: 2026-11-24T12:21:34Z
**User Input**: "Request Changes:

OP-05, OP-02 and OP-01 affect the Tier 1 demo, so let's decide them now instead of in Functional Design. Please add them to the requirements decision record and amend the affected FRs (logging the change in audit.md):
OP-05: explicit course inclusions ("I also want to take CS 4820") are a hard constraint, the counterpart of exclusions. Amend FR-ELG-03, and the separate checker must enforce them.
OP-02: when the named constraints don't fill the credit range, the solver fills the remaining credits from eligible ingested courses (never unmet, never excluded ones), ranked by soft preferences and, once requirement categories exist (Tier 2), by remaining requirement gaps. The interpretation panel labels these as solver-chosen, and the student can pin or exclude any of them.
OP-01: the default credit range is 12–18, set by a configurable DEFAULT_CREDIT_RANGE, and shown as an editable item in the interpretation panel.
We confirm the assumptions in OP-03 (the citation covers the original wording; the computed date is labeled as an interpretation) and OP-04 (Monday–Sunday weeks, with Week 1 the week containing the anchor). Record them as decisions and remove them from Open Points.
Update the stories and the demo script to match, and change P2's representative request from "Plan my spring" to "Plan my next semester", since the term is the configured Fall 2026 stand-in."
**AI Response**: "Requirements change (approved requirements.md amended, per user instruction): added DEC-21 (DEFAULT_CREDIT_RANGE 12-18), DEC-22 (fill rule), DEC-23 (explicit inclusions), DEC-24 (relative-date citations), and DEC-25 (week numbering). Amended FR-PLN-01 and 02, FR-ELG-03, 06 and 08, FR-MAT-07, FR-DL-02 and 03, and NFR-TEST-05(a), and added DEFAULT_CREDIT_RANGE to the §3.3 configuration table, marked as not yet in technical-environment.md. No new FR IDs; the count stays at 79. Tier placement unchanged (requirement-gap ranking is marked Tier 2). Stories: replaced 'Open Points for Functional Design' with 'Resolved Open Points' (OP-01 to OP-05 -> DEC-21 to DEC-25). Updated US-02 (default credit range, inclusion), US-03 (inclusion check by the checker, fill, pin and exclude, [Tier 2] gap ranking), US-07, US-12, US-14 and US-16, and the coverage table. Demo script step 2 message is now 'I need 2 ML electives and no Friday classes, and I also want to take CS 4820.' (it drops the explicit 12-16, so the demo shows the default 12-18 range); step 4 notes that no fill is needed. P2's request now says 'Plan my next semester'. Correction note: the earlier entry 'User Stories - Generation Complete and Approval Prompt' carries timestamp 2026-11-24T12:24:40Z, which was not read from the system clock and is later than this entry. Its actual time was about 12:21Z. Per the append-only rule, that entry is left unchanged and corrected here."
**Context**: INCEPTION - User Stories, change request applied; awaiting approval

---
