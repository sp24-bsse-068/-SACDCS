# Task 5: Constraints, Formalisation and Violations (Railway Crossing)

## 5.1 Constraints the combined system must always satisfy

| ID | Area | Constraint in simple English | Why it is necessary |
| --- | --- | --- | --- |
| CON-01 | Railway | The barrier must not be open while a train is present in the crossing zone. | An open barrier with a train present allows vehicles or robots onto the track. |
| CON-02 | Railway | When a train is approaching the barrier must be closing or closed. | Barriers need time to close; they must start as soon as a train approaches. |
| CON-03 | Railway | The barrier may open only after the crossing is confirmed clear and no train is approaching. | Premature opening exposes road users to a second train or a train still passing. |
| CON-04 | Railway | Warning lights and the audible alarm must be on whenever a train approaches or is present, or the barrier is moving. | Warns people and robots that cannot see the barrier state. |
| CON-05 | Railway / Robot | The robot must not enter the crossing unless the barrier is open and no train is approaching or present. | This is the core rule stated in the scenario; breaking it risks a collision with a train. |
| CON-06 | Railway / Robot | The robot may enter only on crossing status no older than 2 s. | Old status can show 'open' after a train has been detected; link delay makes stale data likely. |
| CON-07 | Railway / Robot | On sensor failure, barrier failure, ARLCCS communication loss or an invalid reading the crossing is treated as unsafe and the robot must not enter. | The scenario requires the ARLCCS to handle these failures; the only safe default is to stop. |
| CON-08 | Railway | On sensor or barrier failure the ARLCCS must keep lights and alarm on and alert the operator. | People near the crossing must be warned even when the safety unit has lost a sensor or barrier. |
| CON-09 | Railway | If redundant train-detection sensors disagree, a train is assumed present. | An incorrect reading must not produce a false 'clear'. |
| CON-10 | Railway / Robot | A robot already inside the crossing when a train approaches must vacate it. | A robot trapped on the track is a collision hazard; stopping is not safe there. |
| CON-11 | Link / Robot | On link loss the robot must not stop inside a crossing zone. | A robot stopped on the track cannot be moved remotely. |
| CON-12 | Safe Mode | In Safe Mode the robot neither navigates nor delivers, and chamber monitoring continues (except inside a crossing, where CON-10 comes first). | Safe Mode must be a stable, safe condition, but must not trap the robot on the track. |
| CON-13 | Robot | The robot is never DELIVERING and AVOIDING_OBSTACLE at the same time. | Scenario rule: no delivery while handling an obstacle. |
| CON-14 | Robot | Delivery may start only from NAVIGATING at the destination. | Scenario rule: a waiting robot must never begin delivering directly. |
| CON-15 | Command | A command is accepted only if the operator is authenticated, role-authorised and the command is valid. | CR-03: unauthorised or malformed commands could move a robot or unlock a batch. |
| CON-16 | Chamber | Active conservation requires door closed, profile loaded and essential sensors working. | Without these the chamber cannot control or measure conditions. |
| CON-17 | Chamber | While the door is open normal operation is suspended, and it resumes only after sensors and conditions are verified. | An open door changes the chamber environment; resuming unverified hides an excursion. |
| CON-18 | Chamber | A correction is declared successful only when sensor readings are in range for 3 consecutive samples. | A command is not proof; the value must be measured. |
| CON-19 | Chamber | If a value is out of range when the recovery period ends, protection mode is active, light is reduced and the operator is alerted. | Failed recovery means the batch is at risk and a human must decide. |
| CON-20 | Chamber | A batch may be removed only when the chamber is safe and no protection response is active. | Removing a batch in protection mode could expose it or hide a failure. |
| CON-21 | Chamber | On power loss the chamber is on emergency power if available; otherwise the incident is recorded and the chamber shuts down safely. | The batch must stay under control or the loss must be on record. |
| CON-22 | Chamber | While vibration is above threshold risky activities are suspended, and resume only after 60 s below threshold. | Vibration can damage the batch; a single quiet reading is not stability. |
| CON-23 | Logging | Every command, transition, alarm and excursion is logged. | Traceability and audit for a safety-critical medical system. |
| CON-24 | Safe Mode | If chamber/battery temperature exceeds the critical threshold or battery is below the emergency level for 3 s, the robot is in Safe Mode. | CR-01 time limit; protects the batch and the robot. |

24 constraints are listed (the brief asks for at least 10). They cover the railway crossing, the robot state model, command security, chamber behaviour and logging.

## 5.2 Formalisation

Notation: ∧ (and), ∨ (or), ¬ (not), → (implies). Variables are Boolean predicates on the current system state. All 24 constraints are formalised; the first eight railway constraints are the core set.

| ID | Formal expression |
| --- | --- |
| CON-01 | Train_Present → ¬Barrier_Open |
| CON-02 | Train_Approaching → (Barrier_Closing ∨ Barrier_Closed) |
| CON-03 | Barrier_Opening → (Crossing_Clear ∧ Clear_Confirmed ∧ ¬Train_Approaching) |
| CON-04 | (Train_Approaching ∨ Train_Present ∨ Barrier_Moving) → (Lights_On ∧ Alarm_On) |
| CON-05 | Robot_Entering → (Barrier_Open ∧ ¬Train_Approaching ∧ ¬Train_Present) |
| CON-06 | Robot_Entering → Status_Fresh |
| CON-07 | (Sensor_Fault ∨ Barrier_Fault ∨ ARLCCS_Comm_Lost ∨ Reading_Invalid) → (Assume_Unsafe ∧ ¬Robot_Entering) |
| CON-08 | (Sensor_Fault ∨ Barrier_Fault) → (Lights_On ∧ Alarm_On ∧ Alert_Operator) |
| CON-09 | Sensors_Disagree → Assume_Train_Present |
| CON-10 | (Robot_In_Crossing ∧ Train_Approaching) → Robot_Vacating |
| CON-11 | Robot_Link_Lost → ¬Robot_Stopped_In_Crossing |
| CON-12 | (Safe_Mode ∧ ¬Robot_In_Crossing) → (¬Navigating ∧ ¬Delivering ∧ Chamber_Monitoring) |
| CON-13 | ¬(State_Delivering ∧ State_Avoiding_Obstacle) |
| CON-14 | Start_Delivering → (Prev_State_Navigating ∧ At_Destination) |
| CON-15 | Command_Accepted → (Authenticated ∧ Authorised ∧ Command_Valid) |
| CON-16 | Active_Conservation → (Door_Closed ∧ Profile_Loaded ∧ Sensors_OK) |
| CON-17 | (Door_Open → ¬Normal_Operation) ∧ (Resume_Normal → (Door_Closed ∧ Sensors_Verified ∧ Conditions_In_Range)) |
| CON-18 | Correction_Success → (Reading_In_Range ∧ Stable_3_Readings) |
| CON-19 | (Out_Of_Range ∧ Recovery_Expired) → (Protection_Mode ∧ Light_Reduced ∧ Operator_Alerted) |
| CON-20 | Remove_Batch → (Chamber_Safe ∧ ¬Protection_Active) |
| CON-21 | Power_Lost → ((Emergency_Available ∧ Emergency_Power_Active) ∨ (¬Emergency_Available ∧ Incident_Recorded ∧ Safe_Shutdown)) |
| CON-22 | (Vibration_High → Risky_Suspended) ∧ (Risky_Resume → Below_Threshold_60s) |
| CON-23 | Event_Occurred → Event_Logged |
| CON-24 | (Chamber_Temp_Critical ∨ Battery_Temp_Critical ∨ Battery_Below_Emergency) ∧ Elapsed_3s → Safe_Mode |

Definitions: Robot_Entering is true from the moment the robot passes the stop line until it leaves the crossing zone. Status_Fresh means the latest ARLCCS message is at most 2 s old. Assume_Unsafe forces Barrier_Open and ¬Train_Approaching to be treated as false when evaluating CON-05.

## 5.3 Violation scenarios and how they are detected

Each scenario shows a violation of the constraints, the consequence, the mitigation and the test that covers it.

| ID | Violation scenario | Constraints violated | Consequence | Mitigation / detection | Test |
| --- | --- | --- | --- | --- | --- |
| VS-01 | Train-detection sensor fails and reports 'no train' while a train approaches. | CON-02, CON-05 | Barrier may not close; robot may see a clear crossing. | Redundant sensors with voting (CON-09); safety-monitoring unit flags fault; robot also requires fresh status (CON-06). | TC-13 |
| VS-02 | Barrier jams open while a train approaches. | CON-01 | Open barrier with train present. | Barrier failure raises alarm and lights (CON-08), crossing marked unsafe (CON-07), robot holds. | TC-13 |
| VS-03 | ARLCCS link drops; last message ('open, no train') is 4 s old. | CON-06, CON-07 | Robot would enter on stale data. | Freshness check; robot holds at the stop line. | TC-14 |
| VS-04 | Barrier reports open but track sensor reports a train. | CON-05, CON-09 | Conflicting readings. | Disagreement treated as train present; robot holds. | TC-13 |
| VS-05 | Barrier opens before the crossing is confirmed clear. | CON-03 | Robot or road users enter while a train is still passing. | ARLCCS interlock; independent clear confirmation. | TC-14 |
| VS-06 | Robot enters, then barrier closes with robot inside. | CON-10 | Robot trapped on track. | Vacate rule; barrier timing leaves clearance time; robot ignores obstacle hold until clear. | TC-13 |
| VS-07 | Link to Mission Control is lost while the robot is on the crossing. | CON-11 | Robot stops on the track. | LINK_LOST stop point is chosen outside the crossing zone; robot completes the crossing first. | TC-12 |
| VS-08 | Safe Mode triggers while the robot is on the crossing. | CON-12 vs CON-10 | Two constraints conflict: stop versus vacate. | Rule: vacate first, then enter Safe Mode (reflected in CON-12 wording). | TC-10, TC-13 |
| VS-09 | Unguarded 'deliver now' trigger accepted in IDLE. | CON-14 | Delivery without navigation. | Guard on previous state; transition T08 forbidden. | TC-06 |
| VS-10 | Door opens while robot is navigating. | CON-17 | Batch exposed during transit. | OP-15 raises SafeModeTrigger (T21). | TC-16 |

Resolved conflict: CON-12 (Safe Mode stops the robot) versus CON-10 (vacate the crossing). The robot vacates first, then enters Safe Mode. This is the single priority rule that must be agreed with the ARLCCS owner.

# Appendix: Coverage check

| Brief item | Where covered |
| --- | --- |
| Task 1: 10 FR, 5 NFR, 4 high-risk, format Req. ID \\| Requirement \\| Type \\| Priority \\| Verification | 1.1 (FR-01 to FR-10, NFR-01 to NFR-05, SR-01 to SR-04) |
| Task 2.1: change log with old text, new text, reason, Modified | 2.1 |
| Task 2.2: why more measurable | 2.2 |
| Task 2.3: multi-layer RTM | 2.5 |
| Task 2.4: at least 8 test cases and re-run per CR | 2.3 (23 test cases) and 2.4 |
| Task 3.1 to 3.5: states, events, transition table, extension, verifier (a) to (d) | 3.1 to 3.5 |
| Task 4.1 to 4.4: 12+ operations, 6+ schemas, state change links, sensor-verified postconditions | 4.1 (32 operations) to 4.4 (10 schemas) |
| Task 5.1 and 5.2: 10+ constraints, 8+ formalised | 5.1 (24) and 5.2 (24 formalised) |
| Test-case evidence | Test cases are designed, not executed. Results columns will be added after execution. |
