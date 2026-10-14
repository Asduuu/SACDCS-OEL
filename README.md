# Smart Autonomous Cold-Chain Delivery & Control System (SACDCS)
## Open-Ended Lab — Software Quality Engineering

**Repository:** [Asduuu/SACDCS-OEL](https://github.com/Asduuu/SACDCS-OEL)  
**Scope:** Requirements, change control, traceability, state modelling, chamber operations, railway-crossing constraints, and verification.

## System overview
SACDCS transports vaccines and temperature-sensitive medicines from a central warehouse to 20 remote clinics using autonomous robots. Each robot has a monitored conservation chamber and communicates with Mission Control over a low-bandwidth, delayed link. Commands require authentication and role authorisation. Critical conditions trigger Safe Mode. Railway crossings are coordinated with ARLCCS.

## Repository contents
- `docs/task-1-requirements.md` — functional, non-functional and high-risk requirements
- `docs/task-2-traceability.md` — change requests, change log and multi-layer RTM
- `docs/task-3-state-modelling.md` — robot states, transitions and verification
- `docs/task-4-operation-schemas.md` — chamber operations and schemas
- `docs/task-5-constraints.md` — safety constraints, logic and violations
- `diagrams/state-machine.mmd` — Mermaid state diagram
- `tests/test-cases.md` — test cases and regression mapping

## Verification note
Thresholds and timing values below are proposed pilot acceptance criteria for this OEL. The health authority and safety engineers must approve actual clinical, battery, environmental and railway thresholds before deployment. This document is a design/verification exercise, not deployment authorization.

## Suggested review sequence
1. Review requirements and acceptance thresholds.
2. Approve CR changes and update traceability.
3. Verify all state transitions, including invalid and missing transitions.
4. Review chamber operations and sensor-confirmed postconditions.
5. Execute normal, negative, fault-injection and regression tests.
