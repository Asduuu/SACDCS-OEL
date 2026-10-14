# Verification Test Cases

## Test conventions
Use a simulator or controlled test rig. Do not test safety failures on a live railway crossing or with real medicines. Record build/version, configuration, timestamps, tester, observed output and evidence location for every run. Timing tests use monotonic timestamps.

| Test ID | Scenario / steps | Expected result | Traceability | Risk |
|---|---|---|---|---|
| TC-01 | Power on with all required diagnostics passing; inspect initial state and status | Robot enters IDLE; identity/health status published | FR-01, ST-01, CON-01 | Medium |
| TC-02 | Send valid delivery request to eligible idle robot | Request validated; robot enters NAVIGATING; mission ID logged | FR-02, ST-02, CON-02 | High |
| TC-03 | Authenticated operator with permitted role sends valid command | Command accepted, executed once and outcome logged | FR-03, CON-03 | Critical |
| TC-04 | Try unauthenticated command, then authenticated but unauthorised command | Both rejected; no actuation; reason and identity/session data logged appropriately | FR-03, NFR-03, CON-03 | Critical |
| TC-05 | While navigating, inject obstacle; then clear it | Forward motion suspends; enters AVOIDING_OBSTACLE; delivery stays disabled; after verified clearance resumes NAVIGATING | FR-04, ST-03, CON-04 | Critical |
| TC-06 | Simulate destination reached with chamber safe, door closed and profile valid; confirm handover | Enters DELIVERING only after guards pass; handover evidence recorded; then RETURNING | FR-05, CON-05 | High |
| TC-07 | Inject critical battery and critical temperature separately; timestamp trigger and Safe Mode entry | Each trigger causes Safe Mode within 3 seconds; hazardous motion/delivery inhibited | CR-01, SAFE-01, NFR-01, CON-06 | Critical |
| TC-08 | Trigger critical battery during AVOIDING_OBSTACLE and during DELIVERING | No undefined transition; delivery/motion inhibited; Safe Mode or approved controlled stop occurs | FR-06, SAFE-01, CON-06 | Critical |
| TC-09 | Fail one essential sensor; then load a valid batch/profile with door open | Conservation cannot start unless self-check, profile and door conditions all pass | FR-07, OP-01–OP-05, CON-07 | Critical |
| TC-10 | Command temperature/humidity correction but keep simulated sensor reading out of range until recovery deadline | System does not claim recovery; enters protection, reduces light exposure and alerts operator | FR-08, SAFE-03, CON-08 | Critical |
| TC-11 | Trigger critical event and inspect audit log | Event includes timestamp, robot/batch identity, event ID, decision and outcome; missing fields cause audit test failure | FR-09, CON-09 | High |
| TC-12 | Simulate crossing with barrier closed, train approaching, stale permission, failed sensor, and fully clear crossing | Robot never enters for any unsafe/unknown case; enters only for fresh valid clearance with barrier open and no train; decisions logged | FR-10, SAFE-02, CON-10–CON-12 | Critical |
| TC-13 | Interrupt link during navigation and at crossing approach; restore link with stale queued command | Link loss detected; local safe policy applied; stale command rejected; state reconciliation required before resuming | SAFE-04, CON-12 | Critical |
| TC-14 | Connect 20 distinct robot clients simultaneously under low-bandwidth/delay profile | All identities remain separate; telemetry is delivered or marked lost/stale within the NFR target; no cross-robot command/data mix | CR-02, NFR-02, NFR-04 | High |
| TC-15 | Open chamber door during conservation/delivery | Chamber activity suspends; delivery interlock prevents unsafe handover; resume only after closure, sensor and environmental re-verification | OP-12, CON-05, CON-13 | Critical |
| TC-16 | Simulate primary power loss with backup available, then unavailable | First case verifies transfer under load; second records incident and shuts down safely | OP-13, OP-14 | High |
| TC-17 | Request batch removal while chamber unsafe or protection active | Removal is denied and reason is logged | OP-15, CON-14 | Critical |
| TC-18 | Mark an essential sensor unhealthy while status is otherwise in range | System never claims safe status and inhibits dependent actions | CON-15 | Critical |

## Regression mapping
- **CR-01:** TC-07, TC-08, and TC-10 (thermal integration); verify 3-second deadline for each defined trigger.
- **CR-02:** TC-14 and telemetry/link impairment portions of TC-13.
- **CR-03:** TC-03, TC-04 and TC-11 (command decision audit).
- **Cross-cutting regression:** TC-05, TC-06, TC-09, TC-12 and TC-15 after any state/interlock change.

## Evidence template
For each run, record:
- Test ID and requirement/configuration version
- Preconditions and simulator/robot identifier
- Input events and exact timestamps
- Expected result and actual result
- Pass/Fail and defect ID
- Log, screenshot or trace location
- Tester and execution date
