# AI-DLC State Tracking

## Project Information
- **Project Name**: CoursePilot
- **Project Type**: Greenfield
- **Start Date**: 2026-11-24T11:35:03Z
- **Current Stage**: INCEPTION - Workflow Planning (execution-plan.md written; awaiting approval)

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

## Stage Progress
### 🔵 INCEPTION PHASE
- [x] Workspace Detection
- [ ] Reverse Engineering (SKIPPED: greenfield)
- [x] Requirements Analysis (approved 2026-11-24T12:09:38Z)
- [x] User Stories (approved 2026-11-24T12:24:23Z: 4 personas, 22 user stories + 3 misuse stories)
- [ ] Workflow Planning (execution-plan.md written; awaiting approval)
- [ ] Application Design - EXECUTE
- [ ] Units Generation - EXECUTE

### 🟢 CONSTRUCTION PHASE
- [ ] Functional Design - EXECUTE (per unit, units with business logic)
- [ ] NFR Requirements - EXECUTE (minimal, once)
- [ ] NFR Design - EXECUTE (per unit, security-critical units)
- [ ] Infrastructure Design - SKIP (local Docker Compose fixed in technical-environment.md)
- [ ] Code Generation - EXECUTE
- [ ] Build and Test - EXECUTE

### 🟡 OPERATIONS PHASE
- [ ] Operations (PLACEHOLDER)
