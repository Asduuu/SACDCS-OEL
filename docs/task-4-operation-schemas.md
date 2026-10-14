# Task 4 — Conservation Chamber Operations

## 4.1 Operation inventory
Operation names describe actions, not states.

| Operation ID | Operation name | Purpose |
|---|---|---|
| OP-01 | RunPowerOnSelfCheck | Test essential sensors, controller and actuators |
| OP-02 | IdentifyMedicineBatch | Read/validate batch identifier |
| OP-03 | LoadConservationProfile | Load approved temperature/humidity limits |
| OP-04 | VerifyDoorClosure | Check door switch and independent closure evidence |
| OP-05 | StartConservationControl | Enable control only after all preconditions pass |
| OP-06 | SampleEnvironmentalSensors | Acquire timestamped temperature/humidity readings |
| OP-07 | CorrectTemperature | Command approved cooling/heating correction |
| OP-08 | CorrectHumidity | Command approved humidity correction |
| OP-09 | VerifyEnvironmentalRecovery | Confirm fresh readings are within profile for required interval |
| OP-10 | ActivateProtectionResponse | Reduce light exposure, inhibit risky activity and alert operator |
| OP-11 | DetectAndStabiliseVibration | Suspend risky activities until vibration stays below threshold |
| OP-12 | HandleDoorOpening | Suspend normal conservation activity and trigger re-verification |
| OP-13 | SwitchToEmergencyPower | Transfer load to emergency supply if available |
| OP-14 | RecordPowerLossAndShutdown | Record incident and safely shut down if backup is unavailable |
| OP-15 | AuthoriseBatchRemoval | Permit removal only when safe and no protection response is active |
| OP-16 | NotifyMissionControl | Send alarm/status with timestamp and batch/robot identity |

## 4.2 Complete operation schemas

### OP-01 — RunPowerOnSelfCheck
- **Description:** Tests essential temperature, humidity, door, vibration and power sensors and checks control outputs.
- **Inputs:** Sensor diagnostics, controller health, calibration/validity metadata.
- **Outputs:** Pass/fail result, failed component list, diagnostic timestamp.
- **Reads:** Sensor status, calibration dates, controller/actuator health.
- **Changes:** Records diagnostic outcome; enables monitoring only on complete pass.
- **Sends/Alerts:** Reports failures to Mission Control/operator.
- **Assumes (precondition):** Chamber controller has power and diagnostic mode is available.
- **Result (postcondition):** Monitoring is enabled only if all essential sensors and required actuators pass; otherwise conservation start is inhibited and a fault is recorded.

### OP-02 — IdentifyMedicineBatch
- **Description:** Validates the batch label/identifier and associates it with the chamber load.
- **Inputs:** Barcode/RFID/manual identifier, batch registry record.
- **Outputs:** Validated batch ID or rejection reason.
- **Reads:** Batch registry, expiry/status and permitted storage profile reference.
- **Changes:** Stores the active batch identity after validation.
- **Sends/Alerts:** Alerts operator if unknown, expired or mismatched.
- **Assumes (precondition):** Batch is presented and identification reader is operational.
- **Result (postcondition):** Exactly one validated batch identity is associated with the load; invalid identity cannot start conservation.

### OP-03 — LoadConservationProfile
- **Description:** Loads approved temperature and humidity limits for the identified batch.
- **Inputs:** Batch ID, profile ID/version.
- **Outputs:** Loaded profile and validation status.
- **Reads:** Approved profile registry, batch identity.
- **Changes:** Sets active permitted ranges and verification intervals.
- **Sends/Alerts:** Logs profile version; alerts on missing/invalid profile.
- **Assumes (precondition):** Batch identity has been validated.
- **Result (postcondition):** Profile is active only if it matches the batch and has approved limits; otherwise active conservation remains disabled.

### OP-04 — VerifyDoorClosure
- **Description:** Confirms the chamber door is physically closed before active conservation or resumption.
- **Inputs:** Door switch, latch sensor and timestamp.
- **Outputs:** Closed/open/uncertain status.
- **Reads:** Door and latch sensors, sensor-health flags.
- **Changes:** Updates door-verification record.
- **Sends/Alerts:** Reports open, disagreement or sensor fault.
- **Assumes (precondition):** Door sensors are powered and have valid diagnostics.
- **Result (postcondition):** Closure is confirmed only when valid sensor evidence agrees; uncertainty is treated as not closed.

### OP-05 — StartConservationControl
- **Description:** Starts closed-loop environmental monitoring/control for a validated batch.
- **Inputs:** Valid batch, valid profile, self-check pass, door-closed confirmation.
- **Outputs:** Start accepted/rejected and controller status.
- **Reads:** Profile, batch identity, self-check record, door state, power health.
- **Changes:** Enables control loop and starts periodic sensor sampling.
- **Sends/Alerts:** Records start time and batch/profile references.
- **Assumes (precondition):** All essential sensors pass self-check, batch/profile are valid, door is closed and power is adequate.
- **Result (postcondition):** Conservation control runs only if all preconditions are true; otherwise it remains inhibited.

### OP-08 — CorrectHumidity
- **Description:** Requests a controlled adjustment when humidity is outside the permitted range.
- **Inputs:** Current humidity, target range, actuator limits and recovery deadline.
- **Outputs:** Actuator command result and correction attempt ID.
- **Reads:** Humidity readings, profile limits, actuator status, power state.
- **Changes:** Commands humidifier/dehumidifier within approved limits.
- **Sends/Alerts:** Logs the correction attempt and deadline.
- **Assumes (precondition):** Sensor reading is fresh and valid, profile is loaded, and actuator operation is allowed.
- **Result (postcondition):** A correction command has been issued; this alone does **not** mean humidity has recovered. OP-09 must verify the physical result.

### OP-09 — VerifyEnvironmentalRecovery
- **Description:** Verifies that temperature/humidity has physically returned to the permitted range.
- **Inputs:** Fresh sensor samples, permitted range, stabilisation interval and recovery deadline.
- **Outputs:** Recovered/not-recovered result with evidence.
- **Reads:** Timestamped temperature/humidity samples, sensor health, active profile.
- **Changes:** Marks recovery verified only after all acceptance conditions pass.
- **Sends/Alerts:** Reports recovery or invokes OP-10 when recovery fails before the deadline.
- **Assumes (precondition):** Profile is valid and the required sensors are healthy.
- **Result (postcondition):** Recovery is true only if fresh readings remain in range for the complete verification interval; timeout or sensor uncertainty means failure.

### OP-10 — ActivateProtectionResponse
- **Description:** Applies protective measures after unsuccessful correction or another critical chamber fault.
- **Inputs:** Failure reason, batch ID, current readings, recovery timeout.
- **Outputs:** Protection activation result.
- **Reads:** Environmental readings, door state, power and fault log.
- **Changes:** Reduces light exposure, suspends risky activities and marks protection response active.
- **Sends/Alerts:** Sends high-priority alert and records evidence.
- **Assumes (precondition):** A critical out-of-range or fault condition has been detected.
- **Result (postcondition):** Protection actions are commanded and acknowledged where feedback exists; operator alert and event record are generated.

### OP-12 — HandleDoorOpening
- **Description:** Suspends normal chamber activity when the door opens during conservation.
- **Inputs:** Door-open event and sensor timestamps.
- **Outputs:** Suspension result and re-verification request.
- **Reads:** Door/latch sensors, current chamber mode, environmental samples.
- **Changes:** Suspends normal conservation-dependent risky activity; invalidates prior door-closed confirmation.
- **Sends/Alerts:** Alerts operator and logs door-open duration.
- **Assumes (precondition):** Door event is received or sensor reports open.
- **Result (postcondition):** Normal activity does not resume until the door is reclosed, sensor health and environmental conditions are reverified, and policy permits resumption.

### OP-13 — SwitchToEmergencyPower
- **Description:** Transfers chamber load to emergency power after primary power loss.
- **Inputs:** Power-failure event, backup availability and transfer status.
- **Outputs:** Transfer success/failure.
- **Reads:** Primary and backup voltage, battery reserve, load status.
- **Changes:** Selects emergency supply and records transfer.
- **Sends/Alerts:** Reports power event and remaining reserve.
- **Assumes (precondition):** Emergency supply is installed and transfer controller is responsive.
- **Result (postcondition):** Emergency operation is claimed only after output voltage and load continuity are measured as acceptable; otherwise OP-14 executes.

### OP-14 — RecordPowerLossAndShutdown
- **Description:** Records power loss and safely disables operation when no usable backup exists.
- **Inputs:** Power-loss event, backup failure/unavailability.
- **Outputs:** Incident ID and shutdown result.
- **Reads:** Power sensors, last valid readings, event clock and controller status.
- **Changes:** Flushes essential incident data, disables nonessential loads and performs controlled shutdown.
- **Sends/Alerts:** Attempts an alert while communication/power permits; stores a local event record.
- **Assumes (precondition):** Primary power is lost and backup is unavailable or failed.
- **Result (postcondition):** Incident data is committed to nonvolatile storage where available, and the chamber shuts down in the safest achievable condition.

### OP-15 — AuthoriseBatchRemoval
- **Description:** Determines whether a batch can be removed safely.
- **Inputs:** Current chamber safety status, door status, active protection flag, batch ID.
- **Outputs:** Permit/deny decision and reason.
- **Reads:** Sensor health, environmental status, protection-response status and batch record.
- **Changes:** Records removal authorisation; does not physically unlock unless permit is true.
- **Sends/Alerts:** Logs decision and alerts on denied removal if appropriate.
- **Assumes (precondition):** Current safety status is available and not stale.
- **Result (postcondition):** Removal is allowed only when chamber safe is confirmed and no protection response is active; otherwise removal is denied.

## 4.3 State-model links
The chamber has its own lifecycle, which interacts with robot states:
- `CH-01 SELF_CHECK → MONITORING_READY`: OP-01 passes all essential diagnostics.
- `CH-02 MONITORING_READY → CONSERVING`: OP-02 and OP-03 succeed, OP-04 confirms door closed, and OP-05 starts control.
- `CH-03 CONSERVING → CORRECTING`: OP-06 detects out-of-range conditions; OP-07/OP-08 command correction.
- `CH-04 CORRECTING → CONSERVING`: OP-09 verifies recovery from fresh sensor readings.
- `CH-04 CORRECTING → PROTECTION`: recovery deadline expires; OP-10 activates protection.
- `CH-05 CONSERVING → DOOR_OPEN_SUSPENDED`: OP-12 detects door opening; resumption requires closure and re-verification.
- `CH-06 POWER_FAILURE → EMERGENCY_POWER`: OP-13 verifies successful transfer; if it fails, OP-14 records and shuts down.

Robot integration: if chamber status is unsafe, robot must not enter `DELIVERING` or authorise batch removal. If door opens during delivery, delivery is suspended and the robot enters a controlled hold or `SAFE_MODE` according to risk policy.

## 4.4 Operations requiring physical verification
1. **OP-07 CorrectTemperature / OP-08 CorrectHumidity:** An actuator acknowledgement proves only that a command was received, not that the physical environment reached the target. OP-09 must use fresh sensor measurements and a stability interval.
2. **OP-13 SwitchToEmergencyPower:** A relay/controller acknowledgement does not prove power continuity under load. Verify measured output voltage, load continuity and reserve before claiming success.
3. **OP-04 VerifyDoorClosure:** A command to close a door is not proof of closure; independent, healthy door/latch sensor evidence is required.
