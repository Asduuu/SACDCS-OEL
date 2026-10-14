# Task 1 — Requirement Extraction

## Assumptions and acceptance conventions
The values below are proposed pilot acceptance criteria for testing, not externally approved medical or railway limits. “Critical threshold”, “emergency level”, allowed chamber ranges and authorised roles must be configured from approved safety data. Unless stated otherwise, timing is measured by monotonic system timestamps and logs are checked after the event.

## 1.1 Functional requirements

| Req. ID | Requirement | Type | Priority | Verification method |
|---|---|---|---|---|
| FR-01 | On startup, the robot shall initialise in `IDLE` and publish its identity and health status to Mission Control. | FR | High | Startup test; inspect state and telemetry |
| FR-02 | On receiving a delivery request, the robot shall validate request ID, destination, payload assignment and current eligibility before transitioning from `IDLE` to `NAVIGATING`. | FR | High | Positive/negative command tests |
| FR-03 | The command service shall accept a robot command only after operator authentication and role-authorisation checks both succeed. | FR/Security | Critical | Role matrix and negative API tests |
| FR-04 | While navigating, the robot shall detect an obstacle, suspend forward navigation and enter `AVOIDING_OBSTACLE`; after clearance is confirmed, it shall resume the planned route. | FR/Safety | Critical | Obstacle simulation and transition-log review |
| FR-05 | On destination confirmation, the robot shall perform the assigned handover, record delivery evidence and then transition to `RETURNING`. | FR | High | End-to-end delivery test |
| FR-06 | If critical battery is detected during navigation or obstacle avoidance, the robot shall cancel delivery and initiate a safe return or controlled stop according to the approved recovery policy. | FR/Safety | Critical | Battery fault injection |
| FR-07 | The chamber shall load the batch identity and approved temperature/humidity profile, then begin active conservation only after profile validation and door-closed confirmation. | FR | Critical | Chamber integration test |
| FR-08 | The chamber shall attempt correction of out-of-range temperature/humidity and shall confirm recovery using fresh sensor readings before reporting normal conditions. | FR/Safety | Critical | Sensor-in-loop test; inject failed correction |
| FR-09 | The system shall record timestamped commands, command decisions, robot state changes, chamber alarms, crossing decisions and delivery outcomes in an auditable event log. | FR | High | Log completeness and sequence test |
| FR-10 | The robot shall request crossing permission from ARLCCS and shall enter the crossing only when permission is valid, the barrier is confirmed open and no train-approach indication is active. | FR/Safety | Critical | Crossing simulator and negative tests |

## 1.2 Non-functional requirements

| Req. ID | Requirement | Type | Priority | Verification method |
|---|---|---|---|---|
| NFR-01 | For CR-01, after a critical chamber/battery temperature or emergency battery-capacity trigger is detected, the robot shall enter `SAFE_MODE` within 3 seconds. | NFR/Performance | Critical | Timestamped fault-injection test |
| NFR-02 | The pilot shall support at least 20 simultaneously connected robots while preserving per-robot identity and telemetry separation. | NFR/Scalability | High | 20-client load test |
| NFR-03 | The command API shall reject commands lacking a valid authenticated identity or sufficient role permission, with 100% rejection in the defined negative-test suite. | NFR/Security | Critical | Security/authorisation test suite |
| NFR-04 | Under the defined pilot load of 20 robots and the specified low-bandwidth test profile, at least 99% of telemetry messages shall be delivered or explicitly marked lost/stale within 10 seconds; no stale message may be treated as current. | NFR/Reliability | High | Network impairment test |
| NFR-05 | An operator shall be able to identify each robot's state, active alarms, last telemetry age and command outcome from the portal within 3 user actions in a usability test. | NFR/Usability | Medium | Task-based usability test |

## 1.3 High-risk requirements

| Req. ID | Requirement | Type | Priority | Verification method |
|---|---|---|---|---|
| SAFE-01 | A critical thermal or battery condition shall cause `SAFE_MODE` entry within 3 seconds; delivery and other hazardous motion shall be inhibited until a defined safe recovery procedure authorises exit. | Safety | Critical | Fault injection, timer and motion-inhibit checks |
| SAFE-02 | Robot motion shall not enter a railway crossing unless crossing permission is fresh and valid, the barrier is confirmed open, and no train is approaching or present. | Safety | Critical | ARLCCS simulator; invalid/stale signal tests |
| SAFE-03 | The chamber shall not declare temperature/humidity recovery until independent fresh readings remain within the configured profile for the verification interval. | Safety | Critical | Sensor fault and recovery test |
| SAFE-04 | Loss of command/telemetry link shall be detected; stale or unauthenticated commands shall not execute, and the robot shall apply its approved local safe policy. | Safety | High | Link interruption and replay tests |

## 1.4 Requirement quality checks
- Each requirement uses an observable action or measurable acceptance condition.
- `SAFE-01` and `NFR-01` overlap intentionally: one is a safety obligation and the other supplies a measurable performance target.
- The actual thermal, humidity, vibration, battery and crossing thresholds must come from approved system configuration; they must not be invented by software developers.
- Verify each requirement by its listed method and retain timestamped evidence.
