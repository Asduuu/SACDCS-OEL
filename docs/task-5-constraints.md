# Task 5 — Constraints, Formalisation and Violations

## 5.1 Constraints in simple English

| Constraint ID | Constraint | Why necessary |
|---|---|---|
| CON-01 | A robot shall start in IDLE only after required startup checks pass. | Prevents operation with uninitialised controls. |
| CON-02 | A delivery request is accepted only if valid, assigned and the robot is eligible. | Prevents unintended or malformed missions. |
| CON-03 | A command executes only when operator authentication and role authorisation both pass. | Prevents unauthorised control. |
| CON-04 | A robot in AVOIDING_OBSTACLE shall not deliver and shall suspend forward route motion. | Prevents collision during avoidance. |
| CON-05 | Delivery requires destination confirmation, a valid batch profile, healthy chamber sensors and a closed door. | Protects payload and people. |
| CON-06 | Critical thermal/battery trigger shall cause Safe Mode within 3 seconds and inhibit hazardous motion. | Makes emergency response measurable. |
| CON-07 | Chamber conservation is active only when essential sensors pass, profile is valid and door is closed. | Prevents uncontrolled storage. |
| CON-08 | Environmental recovery is declared only from fresh sensor readings within approved limits for the required interval. | Prevents false recovery reports. |
| CON-09 | Safety-critical commands, transitions, alarms and crossing decisions are timestamped and logged. | Supports audit and incident investigation. |
| CON-10 | Robot enters railway crossing only with fresh ARLCCS clearance, open barrier, no approaching/present train and healthy sensors. | Prevents train collision. |
| CON-11 | If train is present/approaching, barrier shall not be reported open and robot shall remain outside crossing. | Enforces crossing protection. |
| CON-12 | Loss of communication or stale crossing data shall never be interpreted as permission. | Fail-safe behaviour under link failure. |
| CON-13 | If chamber door is open, robot shall not be in active DELIVERING operation. | Prevents unsafe handling/exposure. |
| CON-14 | Batch removal is denied whenever chamber is unsafe or a protection response is active. | Protects medicine and operator. |
| CON-15 | If essential sensor health is false/unknown, the system shall not claim a safe condition. | Avoids decisions based on bad readings. |

## 5.2 Logical formalisation
Notation: `→` means implies, `∧` means AND, `∨` means OR, `¬` means NOT. Variables are Boolean unless noted.

1. **CON-03 — command access:**  
   `Command_Execute → (Authenticated ∧ Role_Authorised)`

2. **CON-04 — obstacle avoidance:**  
   `State_Avoiding → (¬Delivery_Active ∧ ¬Forward_Motion)`

3. **CON-05 — delivery preconditions:**  
   `State_Delivering → (Destination_Confirmed ∧ Chamber_Safe ∧ Door_Closed ∧ Profile_Valid ∧ Sensors_Healthy)`

4. **CON-06 — Safe Mode deadline:**  
   `(Critical_Thermal ∨ Critical_Battery) → (SafeMode_Entry_Time − Trigger_Time ≤ 3s)`

5. **CON-07 — conservation prerequisites:**  
   `Conservation_Active → (Sensors_Healthy ∧ Profile_Valid ∧ Door_Closed)`

6. **CON-08 — verified recovery:**  
   `Recovery_Declared → (Fresh_Readings ∧ Temp_In_Range ∧ Humidity_In_Range ∧ Stable_For_Interval)`

7. **CON-10 — crossing entry:**  
   `Robot_Enters_Crossing → (Permission_Fresh ∧ Permission_Valid ∧ Barrier_Open ∧ ¬Train_Approaching ∧ ¬Train_Present ∧ Crossing_Sensors_Healthy)`

8. **CON-11 — train protection:**  
   `(Train_Present ∨ Train_Approaching) → (¬Barrier_Open ∧ ¬Robot_Enters_Crossing)`

9. **CON-12 — communications fail-safe:**  
   `(Link_Lost ∨ Crossing_Data_Stale) → ¬Crossing_Entry_Authorised`

10. **CON-13 — robot/chamber coordination:**  
    `Door_Open → ¬State_Delivering`

11. **CON-14 — batch removal:**  
    `Batch_Removal_Allowed → (Chamber_Safe ∧ ¬Protection_Active)`

12. **CON-15 — sensor failure:**  
    `¬Sensors_Healthy → ¬Safe_Status_Claimed`

13. **CON-01 — initial state:**  
    `PowerOn_Complete → (Startup_Checks_Passed → State_IDLE)`

14. **CON-09 — audit:**  
    `Safety_Event → (Timestamp_Present ∧ Event_ID_Present ∧ Robot_ID_Present ∧ Log_Write_Acknowledged)`

For CON-06, implementation must use monotonic timestamps and define the exact trigger-detection point. Thresholds and measurement uncertainty require approval.

## 5.3 Violation scenarios, detection and expected response

| Constraint | Observed values (violation) | Detection | Expected response |
|---|---|---|---|
| CON-03 | `Command_Execute=TRUE`, `Authenticated=TRUE`, `Role_Authorised=FALSE` | Command gateway evaluates identity and permission separately | Reject command; no actuator effect; alert/log denial |
| CON-04 | `State_Avoiding=TRUE`, `Delivery_Active=TRUE` | State invariant monitor sees conflicting flags | Stop delivery actuator; controlled stop/Safe Mode if motion risk; log invariant breach |
| CON-05 | `State_Delivering=TRUE`, `Door_Closed=FALSE` | Robot/chamber interlock evaluates live door state | Suspend handover; controlled hold or Safe Mode; alert operator and log |
| CON-06 | `Critical_Thermal=TRUE`, `Trigger_Time=10.000s`, `SafeMode_Entry_Time=13.500s` | Monotonic timestamp comparison gives 3.5 seconds | Fail test; inhibit motion immediately when detected; raise critical alarm and log deadline breach |
| CON-07 | `Conservation_Active=TRUE`, `Profile_Valid=TRUE`, `Door_Closed=TRUE`, `Sensors_Healthy=FALSE` | Chamber controller checks sensor-health flags | Suspend normal conservation claim; protection response and operator alert |
| CON-08 | `Recovery_Declared=TRUE`, `Fresh_Readings=FALSE`, `Temp_In_Range=TRUE`, `Humidity_In_Range=TRUE` | Recovery verifier rejects stale sample timestamp | Revoke recovery status; continue correction or enter protection on timeout |
| CON-10 | `Robot_Enters_Crossing=TRUE`, `Barrier_Open=TRUE`, `Train_Approaching=TRUE`, `Permission_Fresh=TRUE` | ARLCCS interlock/robot crossing monitor sees train indication | Stop before crossing; command barrier protection; critical alarm and event log |
| CON-11 | `Train_Present=TRUE`, `Barrier_Open=TRUE` | Redundant train detector and barrier-position feedback disagree with invariant | Treat as critical crossing fault; alarm, inhibit robot entry, command barrier close if safe and possible |
| CON-12 | `Link_Lost=TRUE`, `Crossing_Entry_Authorised=TRUE` | Heartbeat timeout and permission-age monitor | Revoke entry authority; robot remains stopped outside crossing; log link loss |
| CON-13 | `Door_Open=TRUE`, `State_Delivering=TRUE` | Door sensor is wired into delivery interlock | Suspend handover; safe hold, alert and log |
| CON-14 | `Batch_Removal_Allowed=TRUE`, `Chamber_Safe=FALSE`, `Protection_Active=TRUE` | Removal authoriser evaluates both flags | Deny unlock/removal; operator alert and audit record |
| CON-15 | `Sensors_Healthy=FALSE`, `Safe_Status_Claimed=TRUE` | Health supervisor compares diagnostic flags and status output | Clear safe claim; inhibit unsafe actions and enter protection/Safe Mode as appropriate |
| CON-01 | `PowerOn_Complete=TRUE`, `Startup_Checks_Passed=FALSE`, `State_IDLE=TRUE` | Startup sequence monitor | Do not accept missions; enter fault handling and log failed check |
| CON-09 | `Safety_Event=TRUE`, `Timestamp_Present=FALSE` | Audit-log integrity checker | Mark event record invalid, raise logging fault and preserve local evidence if possible |

## 5.4 Detection and response principles
- **Fail safe:** Unknown, stale or contradictory values are not treated as safe.
- **Independent confirmation:** Train, barrier, door, temperature and humidity claims use healthy sensor feedback, not command acknowledgements alone.
- **Layered response:** Detect, inhibit hazardous action, alert the operator, record an event and transition to Safe Mode/controlled stop where required.
- **No silent failures:** If the logging or communication path is unavailable, preserve a local event where hardware supports it and report the gap after recovery.
- **Approval boundary:** Railway and medical-storage thresholds must be supplied and approved by responsible safety authorities before real-world use.
