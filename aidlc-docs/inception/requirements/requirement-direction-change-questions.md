# Requirements Questions: No-Manual-Curation Direction Change

The direction change (Clarification Q2, commit `fb5c7f7`) is consistent across `vision.md`, `README.md` and `technical-environment.md`. However, removing human review raises five questions that the earlier answers didn't need to settle, because a teammate used to catch these cases. Each question explains what changed and which earlier answers it touches.

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe your answer after the tag.

---

## Conflict 1: "Every error fails safe" versus model-reported confidence
The new prerequisite-parsing target says every incorrect rule must **fail safe**: it may produce `needs_confirmation`, but never a false `met`. The documents say low-confidence output is flagged, but they don't say where the confidence comes from. Small local models are often confidently wrong. For example, a parse that silently drops "and MATH 1920" could carry high self-reported confidence. If confidence is only the model's own score, the fail-safe target can't be guaranteed.

### Question 1
How should CoursePilot decide that a derived rule (prerequisite tree, requirement category, or topic tag) is confident enough to trust?

A) **Deterministic cross-checks plus the model's score.** A rule is trusted only if it passes Python checks. Every course code in the source text must appear in the tree, and every tree course must appear in the text. The text must not contain wording the rule grammar can't express ("permission of instructor", "corequisite", "recommended", "or equivalent" without a `course_or_equiv` branch). The model's own score must also be above the configured threshold. A failed check means the rule is stored as uncertain, which gives `needs_confirmation`

B) **Option A plus self-consistency.** The rule is derived twice (for example, once directly and once with a reordered or rephrased prompt), and it is trusted only if both trees are identical

C) **The model's self-reported confidence only**, compared with a configured threshold

X) Other (please describe after [Answer]: tag below)

[Answer]: A) Deterministic cross-checks plus the model's score. A rule is trusted only if every course code in the source text appears in the tree (and vice versa), the text contains no wording the grammar can't express, and the model's score is above the threshold; otherwise it is stored as uncertain and gives `needs_confirmation`. The cross-checks are what make the fail-safe target achievable, since a small model's self-reported confidence alone can't be trusted. We chose not to add self-consistency (B) because deriving every rule twice doubles ingestion time on a laptop for little extra safety beyond the deterministic checks; the parsing evaluation will tell us if that changes.

---

## Ambiguity 2: How the solver treats `needs_confirmation` courses
The solver enforces "verified eligibility". With a 15–20 course set reviewed by hand, few courses were `needs_confirmation`. With every rule derived automatically and uncertain rules failing safe, many more courses may be `needs_confirmation`. The planning journey also says each schedule shows "any condition that still needs information", which suggests such courses can appear in schedules.

### Question 2
How should the solver treat a course whose prerequisite status is `needs_confirmation`?

A) **Eligible but flagged.** It can appear in schedules, and each schedule lists the outstanding conditions with the rule, source text, and reason. Only `unmet` courses are excluded

B) **Eligible with a soft penalty.** Same as A, but the solver prefers schedules with fewer unconfirmed courses, so they appear only when needed

C) **Excluded until confirmed.** The interpretation panel lists them, and the student must confirm eligibility before the solver can use them

X) Other (please describe after [Answer]: tag below)

[Answer]: B) Eligible with a soft penalty. Unconfirmed courses can appear, but the solver prefers schedules with fewer of them, and each schedule lists its outstanding conditions with the rule, source text, and reason. Only `unmet` courses are excluded. With automatic derivation many courses may be `needs_confirmation`, so excluding them (C) could leave too few options, while A could fill schedules with unconfirmed courses when confirmed alternatives exist.

---

## Ambiguity 3: Where the topic tag vocabulary comes from
The tag examples (`ml`, `systems`, `theory`) are written in the documents, but nothing says where the list of allowed tags comes from. The planner must map a student's "ML" to a tag. A model that tags freely can drift ("machine-learning", "ai", "ml"). The official CS checklist may define vectors (for example, an ML vector) that already list courses.

### Question 3
Where should the topic tag vocabulary come from?

A) **A fixed vocabulary in configuration** (for example, `TOPIC_TAGS=ml,ai,systems,theory,security,graphics,hci,pl,data`). The model may only assign tags from this list (enforced by a Pydantic enum), and the planner maps student phrases to it. Changing the vocabulary is configuration, not curation

B) **Extracted from the requirements document.** Vectors or concentrations in the official checklist become the tags and the courses they list. The model tags only courses the document doesn't list

C) **Both.** Checklist vectors are authoritative where they exist. The configured vocabulary covers topics the checklist doesn't define, and those tags are model-assigned

X) Other (please describe after [Answer]: tag below)

[Answer]: A) A fixed vocabulary in configuration (`TOPIC_TAGS`), enforced with a Pydantic enum, so tags can't drift. The model assigns tags from each course's name and description, and the planner maps student phrases such as "ML" or "AI" onto the vocabulary. This keeps tagging driven by course text alone and portable to institutions whose checklists define no vectors, and changing the vocabulary is configuration, not curation.

---

## Ambiguity 4: Which requirement categories must the extraction support
Your original answer limited categories to CS core and CS electives (Q3:A). The direction change replaces hand encoding with extraction from the official checklist (Clarification Q2). The updated `vision.md` open question 2 asks which categories the extraction must handle for the demo. The answer affects scope: the CS checklist has rules such as "any CS 4000+ course except…" and "3 credits from this list", and some of these need a richer rule format than course lists.

### Question 4
Which requirement categories must extraction handle correctly for the demo?

A) **CS core and CS electives must be correct.** Every other category in the document is extracted on a best-effort basis and shown with `needs_confirmation`, never counted as satisfied

B) **Every category in the checklist must be supported and evaluated**, including practicum, technical electives, external specialization, and vectors

C) **Only CS core and CS electives are extracted.** Other categories are listed by name as "not checked"

X) Other (please describe after [Answer]: tag below)

[Answer]: A) CS core and CS electives must be extracted and evaluated correctly. Every other category in the checklist is extracted on a best-effort basis and shown with `needs_confirmation`, never counted as satisfied. This keeps the demo scope achievable while still showing the full checklist to the student.

---

## Ambiguity 5: Where the week-one anchor date comes from without curation
You chose the official term calendar's first day of instruction as the default anchor for relative deadlines (Q7:A). With no hand-entered data, something still has to supply that date. The Class Roster API reports start and end dates for sections. A configured date is configuration, not curation, but it is one more value to keep correct for each term.

### Question 5
Where should the default week-one anchor (first day of instruction) come from?

A) **Derived automatically at ingestion** as the most common start date among the term's lecture sections, stored with the term, and shown in the review table

B) **A `TERM_START` value in `.env`**, taken from the official academic calendar

C) **Derived automatically (as in A)**, with an optional `TERM_START` in `.env` that overrides it. The per-course student override from Q7 still applies

X) Other (please describe after [Answer]: tag below)

[Answer]: C) Derived automatically at ingestion as the most common start date among the term's lecture sections, stored with the term, with an optional `TERM_START` in `.env` that overrides it if the derived date is ever wrong. The per-course student override from Q7 still applies.
