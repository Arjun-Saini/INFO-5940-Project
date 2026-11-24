# AI-DLC State Tracking

## Project Information
- **Project Name**: CoursePilot
- **Project Type**: Greenfield
- **Start Date**: 2026-11-24T11:35:03Z
- **Current Stage**: INCEPTION complete. Construction not started; it will start in the final project

## Workspace State
- **Existing Code**: No (only documentation: README.md, vision.md, technical-environment.md, CLAUDE.md)
- **Programming Languages**: None yet (planned: Python 3.12)
- **Build System**: None yet (planned: uv + pyproject.toml, Docker Compose)
- **Project Structure**: Empty (planned: single Streamlit app with router, two agents, tools, Postgres)
- **Reverse Engineering Needed**: No
- **Workspace Root**: /Users/arjun/Documents/Cornell/FA26/INFO 5940

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules

## Input Documents
- `vision.md`: product vision, journeys, MVP scope, success metrics, open questions
- `technical-environment.md`: stack, deployment model, testing rules, prohibited libraries, security basics (treated as constraints)
- `README.md`: project proposal with planned techniques, tools, timeline (final deadline December 16)

## Extension Configuration
| Extension | Enabled | Decided At |
|---|---|---|
| Security Baseline | Yes. N/A: 02, 06, 07, 08, 12, 14. Adapted for local use: 01, 03, 04. Fully enforced: 05, 09, 10, 11, 13, 15 (clarification Q4:A) | Requirements Analysis |
| Resiliency Baseline | No | Requirements Analysis |
| Property-Based Testing | Partial (blocking: PBT-02, PBT-03, PBT-07, PBT-08, PBT-09; others advisory) | Requirements Analysis |

## Current Status
- **Lifecycle Phase**: INCEPTION (complete)
- **Last Completed**: Units Generation (approved 2026-11-24T12:45:16Z)
- **Next Stage**: CONSTRUCTION, starting with U0 Foundation (Functional Design, minimal)
- **Status**: Paused at the user's request. Construction will start in the final project. Do not begin Construction until the team resumes the workflow.

## Stage Progress
### 🔵 INCEPTION PHASE
- [x] Workspace Detection
- [ ] Reverse Engineering (SKIPPED: greenfield)
- [x] Requirements Analysis (approved 2026-11-24T12:09:38Z)
- [x] User Stories (approved 2026-11-24T12:24:23Z: 4 personas, 22 user stories + 3 misuse stories)
- [x] Workflow Planning (approved; see inception/plans/execution-plan.md)
- [x] Application Design (approved 2026-11-24T12:39:07Z)
- [x] Units Generation (approved 2026-11-24T12:45:16Z)

### 🟢 CONSTRUCTION PHASE
- [ ] Functional Design - EXECUTE (per unit, units with business logic)
- [ ] NFR Requirements - EXECUTE (minimal, once)
- [ ] NFR Design - EXECUTE (per unit, security-critical units)
- [ ] Infrastructure Design - SKIP (local Docker Compose fixed in technical-environment.md)
- [ ] Code Generation - EXECUTE
- [ ] Build and Test - EXECUTE

### 🟡 OPERATIONS PHASE
- [ ] Operations (PLACEHOLDER)

## Unit Progress (Construction)
Each coding-agent session reads this table first. Updates follow the one-at-a-time merge rule for `aidlc-docs/` (unit plan Q4 and Q5).

| Unit | Owner | Pass | Functional Design | NFR Requirements | NFR Design | Code Generation | Status |
|---|---|---|---|---|---|---|---|
| U0 Foundation | All (one integrator) | Tier 1 | Minimal | Yes (project-wide) | Yes | Yes | Not started |
| U1 Course Ingestion | DATA (+UX wk 1) | Tier 1 | Yes | — | Yes | Yes | Not started |
| U1 Course Ingestion | DATA | Tier 2 | Update | — | Update | Yes | Not started |
| U2 Eligibility and Solver | DATA | Tier 1 | Yes | — | Skip | Yes | Not started |
| U2 Eligibility and Solver | DATA | Tier 2 | Update | — | Skip | Yes | Not started |
| U3 Planner, Router, Memory | PLAN | Tier 1 | Yes | — | Light | Yes | Not started |
| U3 Planner, Router, Memory | PLAN | Tier 2 (+3) | Update | — | Light | Yes | Not started |
| U4 Materials and Deadlines | MAT | Tier 1 | Yes | — | Yes | Yes | Not started |
| U4 Materials and Deadlines | MAT | Tier 2 | Update | — | Update | Yes | Not started |
| U5 UI and Evaluation | UX | Tier 1 | Skip | — | Skip | Yes | Not started |
| U5 UI and Evaluation | UX | Tier 2 | Minimal (evaluation harness) | — | Skip | Yes | Not started |
