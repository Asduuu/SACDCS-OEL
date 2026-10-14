# Task 2 — Change Requests and Requirements Traceability Matrix

## 2.1 Version-controlled change log

| CR | Requirement ID | Version | Old text | New text | Reason | Status |
|---|---|---|---|---|---|---|
| CR-01 | SAFE-01 / NFR-01 | 0.2.0 | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | The robot shall enter Safe Mode within 3 seconds when chamber or battery temperature exceeds the critical threshold or battery capacity falls below the emergency level. Hazardous motion shall be inhibited while Safe Mode is active. | Adds a deadline, explicit trigger categories and observable response. Motion inhibition makes the safety intent testable. | Modified |
| CR-02 | NFR-02 | 0.2.0 | The system shall support communication with multiple robots simultaneously. | The system shall support at least 20 simultaneously connected robots while preserving per-robot identity and telemetry separation. | “Multiple” is ambiguous; 20 matches the pilot fleet and enables a repeatable load test. | Modified |
| CR-03 | FR-03 / NFR-03 | 0.2.0 | Only authenticated operators shall be permitted to issue robot commands. | The system shall require an authenticated identity and a role authorisation that permits the requested action before accepting any robot command. Rejected commands shall be logged. | Authentication proves identity; authorisation checks permission. Both checks and rejection logging can be tested. | Modified |

## 2.2 Why the changed wording is more testable
- **CR-01:** A test can inject each defined trigger, timestamp detection and Safe Mode entry, and assert elapsed time is no more than 3 seconds. “When detected” alone did not define response time or explicitly enumerate temperature/capacity conditions.
- **CR-02:** A load test can connect 20 distinct robot clients simultaneously and check that identities and telemetry do not cross. “Multiple” had no measurable minimum.
- **CR-03:** Positive and negative tests can vary identity validity and role permission independently. Authentication alone does not ensure the operator is allowed to issue that command.

## 2.3 Multi-layer RTM

| Requirement | State/transition | Operation | Constraint | Test case | Risk |
|---|---|---|---|---|---|
| FR-01 | ST-01 startup → IDLE | OP-01 Initialise robot | CON-01 Startup state is IDLE | TC-01 | Medium |
| FR-02 | ST-02 IDLE → NAVIGATING | OP-02 Validate delivery request | CON-02 Only eligible robot accepts request | TC-02 | High |
| FR-03 / NFR-03 | ST-12 command accepted/rejected | OP-03 Authenticate and authorise command | CON-03 No command without authentication and permission | TC-03, TC-04 | Critical |
| FR-04 | ST-04 NAVIGATING → AVOIDING_OBSTACLE → NAVIGATING | OP-04 Detect and avoid obstacle | CON-04 No delivery during obstacle avoidance | TC-05 | Critical |
| FR-05 | ST-06 NAVIGATING → DELIVERING → RETURNING | OP-05 Confirm destination and handover | CON-05 Delivery requires destination and safe chamber | TC-06 | High |
| FR-06 / SAFE-01 | ST-10 any permitted state → SAFE_MODE; ST-11 low battery → RETURNING | OP-06 Evaluate battery/thermal safety | CON-06 Safe Mode deadline and motion inhibit | TC-07, TC-08 | Critical |
| FR-07 | CH-01 READY only after self-check/profile/door checks | OP-07 Load batch profile | CON-07 Conservation requires door closed and profile valid | TC-09 | Critical |
| FR-08 / SAFE-03 | CH-03 correction → sensor-confirmed recovery or protection | OP-08 Correct and verify environment | CON-08 Recovery needs fresh in-range readings | TC-10 | Critical |
| FR-09 | ST-12 event emitted from every important transition | OP-09 Append audit event | CON-09 Safety events are timestamped and attributable | TC-11 | High |
| FR-10 / SAFE-02 | ST-13 crossing wait → crossing allowed only on valid clearance | OP-10 Validate crossing authority | CON-10 No entry on train/closed barrier/stale permission | TC-12 | Critical |
| SAFE-04 | ST-14 connected → LINK_LOST | OP-11 Detect link loss and apply local policy | CON-11 Stale commands are never executed | TC-13 | Critical |
| FR-09 | ST-15 recovery/reporting | OP-12 Report status and telemetry age | CON-12 Stale telemetry is labelled stale | TC-14 | High |

## 2.4 Regression tests by change request
- **CR-01:** Re-run TC-07 and TC-08; also TC-10 if chamber thermal triggers share the same safety supervisor. Test all thermal and battery trigger combinations and the 3-second deadline.
- **CR-02:** Re-run TC-14 plus the 20-client load test; verify connection count, identity isolation, message delivery/loss marking and latency under load.
- **CR-03:** Re-run TC-03, TC-04 and TC-11. Test valid identity/allowed role, valid identity/denied role, invalid identity and expired session. Verify every rejection is logged without executing the command.

## 2.5 Traceability maintenance rule
Any changed requirement must be followed through its state/transition, operation, constraint, test case and risk entry. No requirement is closed until its linked tests have evidence and failures are resolved or formally accepted.
