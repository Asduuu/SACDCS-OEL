# Task 3 — State Modelling and Transition Verification

## 3.1 States

| State ID | State | Meaning | Entry condition | Exit condition |
|---|---|---|---|---|
| ST-01 | IDLE | Powered, waiting for a valid delivery request at warehouse or after return | Startup checks complete, or warehouse arrival and reset complete | Valid delivery request → NAVIGATING |
| ST-02 | NAVIGATING | Following route to assigned clinic | Valid request accepted | Obstacle, destination, critical battery, Safe Mode, link loss or crossing hold |
| ST-03 | AVOIDING_OBSTACLE | Forward route motion suspended while obstacle is handled | Obstacle detected during navigation | Obstacle clearance confirmed → NAVIGATING; critical battery/Safe Mode/link loss handled by higher-priority transition |
| ST-04 | DELIVERING | Performing approved package handover at confirmed destination | Destination reached and delivery preconditions satisfied | Handover confirmed → RETURNING; fault → SAFE_MODE |
| ST-05 | RETURNING | Travelling to warehouse; delivery may be cancelled after a fault | Delivery complete or critical battery recovery policy chooses return | Warehouse reached → IDLE; critical fault → SAFE_MODE |
| ST-06 | SAFE_MODE | Safety-controlled state that inhibits hazardous movement and delivery | Critical battery/thermal/fault condition or unsafe system condition | Only approved recovery, fault clear and authorised reset → previous safe state or IDLE |
| ST-07 | LINK_LOST | Communication heartbeat expired; local autonomy follows pre-approved safe policy | Link-loss timer expires or link is declared unavailable | Link restored and authenticated resynchronisation completed → safe prior state or SAFE_MODE review |
| ST-08 | CROSSING_HOLD | Robot waits outside railway crossing for fresh, valid ARLCCS clearance | Route reaches crossing boundary | Valid clearance → resume NAVIGATING; train/barrier/fault/link uncertainty → remain stopped or SAFE_MODE |
| ST-09 | FAULT_STOP | Controlled stop for a non-recoverable or ambiguous local fault | Fault cannot be safely handled by normal policy | Maintenance/authorised recovery and self-test → IDLE or SAFE_MODE |

**Priority:** `SAFE_MODE` and controlled stopping override delivery and ordinary navigation. Link restoration alone must never automatically resume hazardous motion without state reconciliation.

## 3.2 Events and conditions
- `PowerOn`: robot power-on event.
- `SelfCheckPassed`: startup diagnostic completion event.
- `DeliveryRequest`: request received from Mission Control.
- `RequestValid`: request schema, destination, payload and eligibility checks pass.
- `ObstacleDetected`: obstacle sensor reports an obstacle within the stopping envelope.
- `ObstacleCleared`: obstacle-free path is confirmed for the configured clearance interval.
- `DestinationReached`: localisation confirms the assigned destination/geofence.
- `HandoverConfirmed`: delivery handover evidence is verified.
- `WarehouseReached`: localisation confirms the warehouse geofence.
- `CriticalBattery`: battery capacity crosses the approved emergency threshold.
- `CriticalThermal`: battery or chamber temperature crosses its approved critical threshold.
- `FaultDetected`: a critical onboard fault is reported.
- `LinkHeartbeatExpired`: no valid heartbeat is received within the configured timeout.
- `LinkRestored`: valid communication resumes and identity/session reconciliation succeeds.
- `CrossingClear`: ARLCCS permission is fresh, barrier is confirmed open and train sensors indicate no approach/presence.
- `RecoveryAuthorised`: required self-checks pass and an authorised reset is received.

A **state** describes the current operating mode over an interval. An **event** is an occurrence that may cause a transition. A **condition/guard** is a Boolean predicate that must be true for a transition to be allowed.

## 3.3 Transition table

| Current state | Event/guard | Next state | Req. ID | Valid? | Verification evidence |
|---|---|---|---|---|---|
| Power-off | PowerOn and SelfCheckPassed | IDLE | FR-01 | Valid | TC-01; startup log |
| IDLE | DeliveryRequest and RequestValid and command authorised | NAVIGATING | FR-02, FR-03 | Valid | TC-02, TC-03 |
| IDLE | DeliveryRequest but invalid/unauthorised | IDLE | FR-02, FR-03 | Valid; reject | TC-04; rejection audit event |
| IDLE | Direct request to deliver without destination/route completion | DELIVERING | SAFE-05 | **Invalid** | Negative transition test; must be blocked |
| NAVIGATING | ObstacleDetected | AVOIDING_OBSTACLE | FR-04 | Valid | TC-05; navigation suspended |
| AVOIDING_OBSTACLE | ObstacleCleared and route revalidated | NAVIGATING | FR-04 | Valid | TC-05; resume event |
| NAVIGATING | DestinationReached and delivery preconditions pass | DELIVERING | FR-05 | Valid | TC-06 |
| DELIVERING | HandoverConfirmed | RETURNING | FR-05 | Valid | TC-06; delivery record |
| RETURNING | WarehouseReached and reset checks pass | IDLE | FR-01 | Valid | TC-01 |
| NAVIGATING | CriticalBattery | RETURNING or SAFE_MODE per approved safe policy | FR-06, SAFE-01 | Valid only if policy explicitly selects safe return; otherwise SAFE_MODE | TC-07 |
| AVOIDING_OBSTACLE | CriticalBattery | SAFE_MODE / controlled stop, then safe return when authorised | SAFE-01 | Required | TC-08 |
| DELIVERING | CriticalBattery or CriticalThermal | SAFE_MODE; inhibit delivery | SAFE-01 | Required | TC-07, TC-08 |
| Any active state | CriticalThermal or critical fault | SAFE_MODE | SAFE-01 | Valid; highest priority | TC-07 |
| Any communicating state | LinkHeartbeatExpired | LINK_LOST; apply local safe policy | SAFE-04 | Valid | TC-13 |
| LINK_LOST | LinkRestored and resynchronisation passes | Safe prior state or SAFE_MODE review | SAFE-04 | Valid only after reconciliation | TC-13 |
| NAVIGATING | Crossing boundary reached without clearance | CROSSING_HOLD | FR-10, SAFE-02 | Required | TC-12 |
| CROSSING_HOLD | CrossingClear | NAVIGATING | FR-10, SAFE-02 | Valid | TC-12 |
| CROSSING_HOLD | Train approaching/present, barrier closed, stale permission or sensor disagreement | CROSSING_HOLD or SAFE_MODE | SAFE-02 | Valid; remain outside crossing | TC-12 |
| SAFE_MODE | RecoveryAuthorised and fault cleared/self-check passed | IDLE or approved safe recovery state | SAFE-01 | Valid only after all guards pass | TC-07, TC-08 |
| AVOIDING_OBSTACLE | No clearance event/transition | AVOIDING_OBSTACLE indefinitely | FR-04 | **Incomplete model / liveness risk** | Timeout test; alert and controlled safe policy required |
| Any state | Unlisted event | Same state or FAULT_STOP according to event policy | SAFE-01, SAFE-04 | Must be explicitly defined | Unexpected-event test |

`SAFE-05` is a model-verification defect ID, not an approved requirement: add a negative requirement/test to ensure IDLE cannot jump directly to DELIVERING.

## 3.4 Model extension: Safe Mode and Link Lost
- From `NAVIGATING`, `AVOIDING_OBSTACLE`, `DELIVERING`, `RETURNING` and `CROSSING_HOLD`, critical thermal/fault conditions may enter `SAFE_MODE`.
- `IDLE` may enter `SAFE_MODE` if a critical battery, thermal or diagnostic fault is detected while waiting.
- `LINK_LOST` may be entered from any state that depends on remote communication. Local safety functions remain active; crossing clearance cannot be assumed from a lost link.
- If the robot is at a railway crossing when the link is lost, it must remain stopped outside the crossing unless a formally approved local protocol safely governs the complete crossing. It must not enter based on stale permission.
- Safe Mode exit requires cleared fault, passing self-check, a fresh and valid state snapshot, and authorised recovery. Link restoration alone is insufficient.

## 3.5 Verifier investigation
**(a) Can IDLE → DELIVERING happen?** It must not. It bypasses request validation, route navigation and destination confirmation, violating FR-02/FR-05 and the model safety invariant. Reject and log the transition.

**(b) If AVOIDING_OBSTACLE has no transition back, can the robot still deliver?** No, not on that execution path. It can remain stuck indefinitely, violating liveness/availability. Add `ObstacleCleared → NAVIGATING`, plus a timeout/escalation path if the obstacle never clears.

**(c) Can AVOIDING_OBSTACLE → DELIVERING occur?** No. Delivery is permitted only after the robot resumes navigation and confirms the destination. A direct transition violates FR-04/FR-05 and the “no delivery while avoiding” invariant.

**(d) CriticalBattery in AVOIDING_OBSTACLE or DELIVERING?** If no transition is defined, behaviour is unspecified and potentially unsafe. Add explicit transitions: from either state, inhibit delivery/forward motion and enter SAFE_MODE or an approved controlled-return procedure. Do not assume the generic navigation transition covers these states.

## 3.6 Invariants
1. `state = IDLE` implies no active delivery operation.
2. `state = DELIVERING` implies destination confirmed, obstacle handling inactive, chamber safe, door closed and batch profile valid.
3. `state = AVOIDING_OBSTACLE` implies delivery actuator disabled and forward navigation suspended.
4. `state = SAFE_MODE` implies hazardous motion and delivery commands are inhibited.
5. Crossing entry implies fresh ARLCCS clearance, barrier open, no train approach/presence and sensor health valid.
