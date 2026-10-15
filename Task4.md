# Task 4: Operations and Operation Schema (Conservation Chamber)

## 4.1 Operations

Operations are verbs the system performs. State names (MONITORING, PROTECTION_MODE) are deliberately not used as operation names. OP-01 to OP-20 are chamber operations; OP-21 to OP-32 are system operations that support the command pipeline, Safe Mode and the crossing.

| ID | Operation | Description | Scope |
| --- | --- | --- | --- |
| OP-01 | RunPowerOnSelfCheck | Tests every sensor and actuator when the chamber is powered. | Chamber |
| OP-02 | EvaluateSensorHealth | Decides whether all essential sensors (temperature, humidity, vibration, door, power) work. | Chamber |
| OP-03 | ReadBatchIdentification | Reads the identification of a batch placed inside. | Chamber |
| OP-04 | LoadConservationProfile | Loads the required temperature and humidity limits for the identified batch. | Chamber |
| OP-05 | StartActiveConservation | Enables the control loop once the door is closed and the profile is loaded. | Chamber |
| OP-06 | SampleEnvironment | Reads temperature, humidity, vibration, door status and power every 5 s. | Chamber |
| OP-07 | CorrectTemperature | Commands cooling or heating to bring temperature back into the profile range. | Chamber |
| OP-08 | CorrectHumidity | Commands humidifying or drying to bring humidity back into range. | Chamber |
| OP-09 | VerifyRecovery | Checks sensor readings to confirm the corrected value is truly back in range, or declares failure at timeout. | Chamber |
| OP-10 | ActivateProtectionResponse | Starts the protection response after failed recovery. | Chamber |
| OP-11 | ReduceLightExposure | Closes the shutter or dims lighting around the batch. | Chamber |
| OP-12 | RaiseOperatorAlert | Sends or queues an alert to Mission Control. | Chamber |
| OP-13 | SuspendRiskyActivities | Blocks risky activities (door release, handling) while vibration is above threshold. | Chamber |
| OP-14 | ResumeAfterStabilisation | Re-enables risky activities after vibration stays below threshold for 60 s. | Chamber |
| OP-15 | HandleDoorOpened | Suspends normal operation when the door opens during conservation. | Chamber |
| OP-16 | VerifyAndResumeAfterDoor | Verifies door latch, sensors and conditions before restoring normal operation. | Chamber |
| OP-17 | SwitchToEmergencyPower | Moves the chamber to the emergency supply on power loss. | Chamber |
| OP-18 | RecordPowerIncident | Writes the power-loss incident to the non-volatile log. | Chamber |
| OP-19 | ShutDownSafely | Closes the door lock, stores state and powers down in an orderly way. | Chamber |
| OP-20 | AuthorizeBatchRemoval | Releases the door lock only when the chamber is safe and no protection response is active. | Chamber |
| OP-21 | ReceiveCommand | Accepts a command message from Mission Control (supports RTM step 'command received'). | System |
| OP-22 | AuthenticateOperator | Verifies the operator identity and session. | System |
| OP-23 | AuthoriseCommand | Checks the operator role against the command type. | System |
| OP-24 | ValidateCommand | Checks schema, robot ID, sequence number and age. | System |
| OP-25 | ExecuteCommand | Applies the command to the robot state machine if the guards hold. | System |
| OP-26 | ReportStatus | Sends the command reply and periodic telemetry. | System |
| OP-27 | LogEvent | Appends a log record (command, transition, alarm, excursion). | System |
| OP-28 | InitiateSafeMode | Stops motion, locks the door and raises the Safe Mode alert. | System |
| OP-29 | DetectLinkLoss | Tracks heartbeats and raises LinkLost / LinkTimeout. | System |
| OP-30 | EvaluateCrossingClearance | Checks the freshness and content of ARLCCS status before entry. | System |
| OP-31 | HandleObstacle | Stops, avoids, and decides when the path is clear. | System |
| OP-32 | MonitorBatteryAndThermal | Compares battery capacity and battery/chamber temperature with CR-01 thresholds. | System |

## 4.2 Operation Schemas

### OP-05 StartActiveConservation

| Field | Content |
| --- | --- |
| Description | Starts active temperature and humidity control for the loaded batch. |
| Inputs | Trigger: DoorClosed event or ProfileLoaded event. |
| Outputs | ConservationStarted status; log record. |
| Reads | Door_Status, Profile_Loaded, Batch_ID, Sensor_Health, Power_Source, Protection_Active. |
| Changes | Chamber_State: BATCH_READY → ACTIVE_CONSERVATION; control loop enabled; Conservation_Start_Time. |
| Sends / Alerts | Status message to Mission Control; no alert. |
| Assumes (precondition) | Self-check passed ∧ all essential sensors OK ∧ Door_Status = CLOSED ∧ Profile_Loaded = true ∧ Protection_Active = false. |
| Result (postcondition) | Chamber_State = ACTIVE_CONSERVATION; setpoints equal profile limits; control loop running; event logged. |

### OP-07 CorrectTemperature

| Field | Content |
| --- | --- |
| Description | Commands cooling or heating after temperature leaves the profile range. |
| Inputs | Temperature_Reading out of range. |
| Outputs | Actuator command (cooling/heating level); Correction_Started flag. |
| Reads | Temperature sensors, profile limits, Power_Source, Door_Status. |
| Changes | Chamber_State: ACTIVE_CONSERVATION → CORRECTING; actuator drive; Recovery_Timer started at 0. |
| Sends / Alerts | Excursion event to Mission Control (queued if the link is down). |
| Assumes (precondition) | Chamber_State = ACTIVE_CONSERVATION ∧ temperature sensor healthy ∧ temperature outside [T_min, T_max] ∧ Door_Status = CLOSED. |
| Result (postcondition) | Actuator commanded and Recovery_Timer running. The temperature is NOT assumed to be back in range; this is established only by OP-09. |

### OP-09 VerifyRecovery

| Field | Content |
| --- | --- |
| Description | Confirms from sensor readings that a corrected value has returned to range, or declares failure. |
| Inputs | Sensor reading stream; Recovery_Timer. |
| Outputs | Recovery_Result = SUCCESS or FAIL. |
| Reads | Temperature, humidity, Recovery_Timer, profile limits, Sensor_Health. |
| Changes | SUCCESS: CORRECTING → ACTIVE_CONSERVATION, timer cleared. FAIL: CORRECTING → PROTECTION_MODE (OP-10). |
| Sends / Alerts | Recovery-success notice, or failure event that triggers OP-10 and OP-12. |
| Assumes (precondition) | Chamber_State = CORRECTING ∧ Recovery_Timer running ∧ relevant sensors healthy. |
| Result (postcondition) | SUCCESS only if 3 consecutive readings (5 s apart) are within range; FAIL when Recovery_Timer reaches 10 min first. |

### OP-10 ActivateProtectionResponse

| Field | Content |
| --- | --- |
| Description | Puts the chamber into protection mode after failed recovery. |
| Inputs | Recovery_Result = FAIL. |
| Outputs | Protection_Active flag; protection alert message. |
| Reads | Chamber_State, Batch_ID, light sensor, shutter position. |
| Changes | Chamber_State → PROTECTION_MODE; Protection_Active = true; light exposure reduced; batch removal blocked. |
| Sends / Alerts | Protection alert to operator within 5 s (queued and retried if the link is down). |
| Assumes (precondition) | Recovery failed within the recovery period ∧ chamber powered (main or emergency). |
| Result (postcondition) | PROTECTION_MODE active; light level at or below the profile limit as read by the light sensor; alert sent or queued; OP-20 denies removal. |

### OP-13 SuspendRiskyActivities

| Field | Content |
| --- | --- |
| Description | Blocks risky activities while vibration is above the threshold. |
| Inputs | Vibration_Reading above threshold. |
| Outputs | Suspension flag; vibration alert. |
| Reads | Vibration sensor, threshold, Stabilisation_Timer, Chamber_State. |
| Changes | Chamber_State: ACTIVE_CONSERVATION → VIBRATION_HOLD; risky activities (door release, handover) blocked; Stabilisation_Timer reset. |
| Sends / Alerts | Vibration warning to Mission Control. |
| Assumes (precondition) | Vibration sensor healthy ∧ vibration > threshold. |
| Result (postcondition) | Risky activities blocked; OP-14 is allowed only after vibration stays below threshold for 60 s without interruption. |

### OP-15 HandleDoorOpened

| Field | Content |
| --- | --- |
| Description | Suspends normal operation when the door opens while the batch is under conservation. |
| Inputs | DoorOpened event. |
| Outputs | Suspension notice; log record. |
| Reads | Door sensor, Chamber_State, robot state, Authorised_Opening flag. |
| Changes | Chamber_State: ACTIVE_CONSERVATION → SUSPENDED_DOOR_OPEN; normal control suspended; Door_Open_Timer started. |
| Sends / Alerts | Operator alert within 5 s. If the robot is not in DELIVERING (unauthorised opening) a critical-fault request is sent to the robot (→ SAFE_MODE, T21–T24). |
| Assumes (precondition) | Chamber_State = ACTIVE_CONSERVATION ∧ door sensor healthy. |
| Result (postcondition) | Normal operation suspended; event logged; no resume until OP-16 succeeds. |

### OP-16 VerifyAndResumeAfterDoor

| Field | Content |
| --- | --- |
| Description | Verifies door closure, sensors and conditions before restoring normal operation. |
| Inputs | DoorClosed event. |
| Outputs | Resume decision (RESUMED or NOT_RESUMED). |
| Reads | Door latch sensor, temperature, humidity, vibration, Sensor_Health, Door_Open_Timer. |
| Changes | SUSPENDED_DOOR_OPEN → ACTIVE_CONSERVATION if verified; otherwise → CORRECTING (and OP-09 / OP-10 follow). |
| Sends / Alerts | Resume notice, or alert that conditions were not restored. |
| Assumes (precondition) | Chamber_State = SUSPENDED_DOOR_OPEN. |
| Result (postcondition) | ACTIVE_CONSERVATION only when the latch sensor reads CLOSED, all essential sensors are healthy, and temperature and humidity are within range. |

### OP-17 SwitchToEmergencyPower

| Field | Content |
| --- | --- |
| Description | Moves the chamber to the emergency supply on power loss. |
| Inputs | PowerLoss event. |
| Outputs | Power_Source = EMERGENCY; power alert. |
| Reads | Main power sensor, emergency supply status and charge. |
| Changes | Power_Source; non-essential loads shed; Chamber_State → EMERGENCY_POWER. |
| Sends / Alerts | Power-loss alert to operator; robot battery estimate updated (may raise CriticalBattery). |
| Assumes (precondition) | Main power lost ∧ emergency supply available with charge above minimum. |
| Result (postcondition) | Chamber powered from the emergency source within 1 s, confirmed by the power sensor. If not available: OP-18 then OP-19 (record incident, shut down safely). |

### OP-20 AuthorizeBatchRemoval

| Field | Content |
| --- | --- |
| Description | Allows the batch to be taken out only when it is safe to do so. |
| Inputs | Removal request (from the handover step in DELIVERING). |
| Outputs | GRANT or DENY. |
| Reads | Chamber_State, Protection_Active, Door_Status, robot state, vibration. |
| Changes | Door lock released on GRANT; otherwise unchanged. |
| Sends / Alerts | Grant/deny status to Mission Control; DENY triggers HandoverBlocked (T18). |
| Assumes (precondition) | Robot state = DELIVERING at the destination. |
| Result (postcondition) | GRANT only if Chamber_Safe ∧ ¬Protection_Active ∧ vibration below threshold. Result logged. |

Ten full schemas are given (the brief asks for at least six). Schemas for the remaining operations follow the same template.

## 4.3 How operations change chamber state and link to the Task 3 robot model

### Chamber state transitions

| ID | From | Event / guard | To | Operations | Link to robot state (Task 3) |
| --- | --- | --- | --- | --- | --- |
| CT-01 | SELF_CHECK | AllEssentialSensorsOK | MONITORING | OP-01, OP-02 | Robot IDLE (S1) only |
| CT-02 | MONITORING | BatchPlaced ∧ ProfileLoaded | BATCH_READY | OP-03, OP-04 | S1 (loading at warehouse) |
| CT-03 | BATCH_READY | DoorClosed | ACTIVE_CONSERVATION | OP-05 | Robot may leave S1 only after this |
| CT-04 | ACTIVE_CONSERVATION | TemperatureOrHumidityOutOfRange | CORRECTING | OP-06, OP-07, OP-08 | S2–S5: no robot change; excursion reported |
| CT-05 | CORRECTING | InRangeVerified (3 readings) | ACTIVE_CONSERVATION | OP-09 | None |
| CT-06 | CORRECTING | RecoveryTimeout (10 min) | PROTECTION_MODE | OP-09, OP-10, OP-11, OP-12 | S4: HandoverBlocked → T18; S2/S3: operator alert, SafeModeTrigger if above critical threshold (T21, T22) |
| CT-07 | ACTIVE_CONSERVATION | VibrationHigh | VIBRATION_HOLD | OP-13 | S4: handover held (T19) |
| CT-08 | VIBRATION_HOLD | StableForPeriod (60 s) | ACTIVE_CONSERVATION | OP-14 | None |
| CT-09 | ACTIVE_CONSERVATION | DoorOpened | SUSPENDED_DOOR_OPEN | OP-15 | S4: authorised handover (expected). S2, S3, S5: unauthorised → SafeModeTrigger (T21, T22, T24) |
| CT-10 | SUSPENDED_DOOR_OPEN | DoorClosed ∧ ConditionsVerified | ACTIVE_CONSERVATION | OP-16 | S4: DeliveryComplete allowed only after this |
| CT-11 | Any powered state | PowerLoss ∧ EmergencyAvailable | EMERGENCY_POWER | OP-17 | Battery estimate re-evaluated; may trigger E6 (T07, T11, T12) |
| CT-12 | Any powered state | PowerLoss ∧ ¬EmergencyAvailable | SAFE_SHUTDOWN | OP-18, OP-19 | Robot → SAFE_MODE (T20–T24) |
| CT-13 | SELF_CHECK | EssentialSensorFailed | FAULT | OP-01, OP-02 | Robot stays in S1; DeliveryRequest rejected (T15) |
| CT-14 | MONITORING or ACTIVE_CONSERVATION | BatchRemovalRequest ∧ ChamberSafe ∧ ¬ProtectionActive | MONITORING (batch removed) | OP-20 | S4 → DeliveryComplete (T05) |

### Worked examples

Door opened during conservation (OP-15): CT-09: ACTIVE_CONSERVATION → SUSPENDED_DOOR_OPEN

In S4 DELIVERING the opening is expected (handover) and the robot stays in S4. In S2, S3 or S5 the opening is an unauthorised fault, which raises SafeModeTrigger (T21, T22, T24, requirement SR-01). DeliveryComplete (T05) is allowed only after OP-16 verified the door and conditions (CT-10).

Correction fails → protection response (OP-10): CT-06: CORRECTING → PROTECTION_MODE

Batch removal is denied by OP-20, so a robot in S4 cannot complete the handover: T18 sends it to RETURNING with the batch quarantined. In S2 or S3 the operator is alerted; if the chamber temperature is above the critical threshold, SafeModeTrigger moves the robot to S6 within 3 s (CR-01).

Power loss → emergency power (OP-17): CT-11: any powered state → EMERGENCY_POWER

The chamber now draws on the shared battery, so the robot battery estimate falls faster and CriticalBattery (E6) can fire earlier: T07, T11 or T12 apply. If no emergency power exists, CT-12 shuts the chamber down and the robot goes to S6 SAFE_MODE.

## 4.4 Operations whose postcondition must be verified by sensors

### CorrectTemperature (OP-07)

The command only starts an actuator. Whether the temperature returns to range depends on the actuator working, the thermal mass of the batch, ambient heat and door seals, none of which the command can know.

Assuming success after the command would hide a failed compressor or a leaking seal, and the batch would be reported as safe while it is not. This is why the postcondition is checked in OP-09 with 3 consecutive in-range readings and a 10-minute recovery limit.

Sensor readings also protect against transient values: one in-range reading during oscillation does not prove recovery.

### VerifyAndResumeAfterDoor (OP-16)

A DoorClosed event only says the switch changed. The latch may be incomplete, a seal may be damaged, and the chamber may have warmed up while the door was open.

Resuming on the event alone would restart 'normal' operation on an unverified chamber and could mask an excursion. The postcondition requires the latch sensor to read CLOSED, all essential sensors healthy, and temperature and humidity in range.

The 5 s link delay makes remote confirmation too slow, so the check must run on board from sensor readings.

The same reasoning applies to CorrectHumidity (OP-08) and ReduceLightExposure (OP-11, shutter position sensor).
