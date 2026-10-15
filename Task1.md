# Task 1: Requirement Extraction

Requirements are rewritten as testable statements, not copied from the scenario. Priority scale: Critical (life or batch safety), High, Medium. Verification method is the main way each requirement is checked.

## 1.1 Requirement register (baseline v1.0)

| Req. ID | Requirement | Type | Priority | Verification Method |
| --- | --- | --- | --- | --- |
| FR-01 | Only authenticated operators shall be permitted to issue robot commands. | FR | High | Test (negative test: no session, expired session) |
| FR-02 | Each command type shall be mapped to the roles allowed to issue it (Viewer, Operator, Supervisor, Administrator). A command from an authenticated operator whose role lacks the permission shall be rejected with error code AUTHZ_DENIED and logged. | FR | High | Test (role × command matrix) |
| FR-03 | Before executing a command the robot shall check its schema, target robot ID, sequence number (no replay) and age (≤ 30 s since the operator timestamp). An invalid command shall be rejected with a coded reply and not executed. | FR | High | Test + fault injection |
| FR-04 | When ObstacleDetected occurs while moving, the robot shall stop forward motion within 1 s, suspend navigation, perform the avoidance manoeuvre, and resume navigation only after the obstacle has been clear for 2 consecutive seconds. | FR | High | Simulation / hardware-in-the-loop test |
| FR-05 | A valid DeliveryRequest received in IDLE with battery > 20 % shall start navigation within 5 s. At the destination the robot shall deliver the package, then return to the warehouse and become IDLE on arrival. If battery falls to ≤ 20 % during navigation the robot shall abandon the delivery and return. | FR | High | Scenario test |
| FR-06 | When chamber temperature or humidity leaves the batch profile limits the chamber shall start correction and declare success only after 3 consecutive in-range sensor readings (5 s apart). If the value is not back in range within 10 min, the correction shall be reported as failed. | FR | High | Test (sensor value injection) |
| FR-07 | At power-on the chamber shall run a self-check and enter MONITORING only if all essential sensors pass. Active conservation shall start only when the door is closed and the batch profile is loaded. Vibration above threshold shall suspend risky activities until it stays below threshold for 60 s. A door opening during conservation shall suspend normal operation, alert the operator within 5 s, and not resume until sensors and conditions are verified. | FR | High | Test (sensor injection, door simulation) |
| FR-08 | If correction fails within the recovery period the chamber shall enter protection mode, reduce light exposure and alert the operator within 5 s. On power loss it shall switch to emergency power within 1 s if available; otherwise it shall record the incident and shut down safely. | FR | High | Fault-injection test |
| FR-09 | The robot shall send telemetry (state, position, battery %, chamber temperature, humidity, vibration, door, power) at least every 10 s, each message ≤ 2 KB, and a status reply (ACCEPTED, REJECTED, EXECUTING, DONE, FAILED) for every command. | FR | Medium | Test with link emulator |
| FR-10 | The system shall write an append-only log entry for every command, authentication result, state transition, alarm and chamber excursion, containing UTC timestamp, robot ID, operator ID and outcome, retained for ≥ 12 months. Entries made during a link outage shall be stored and forwarded when the link returns. | FR | Medium | Inspection + test |
| NFR-01 | Performance: the time from command submission to operator-visible acknowledgement shall be ≤ 12 s at the 95th percentile with a 5 s one-way link delay. The robot shall reach an accept/reject decision for a received command in ≤ 1 s. | NFR | High | Load test with link emulator |
| NFR-02 | Security: all Mission Control–robot traffic shall be encrypted (TLS 1.3 or equivalent) with mutual device authentication; operator sessions shall expire after 15 min idle; 100 % of replayed or tampered messages in a penetration test shall be rejected. | NFR | High | Penetration test + inspection |
| NFR-03 | The system shall support communication with multiple robots simultaneously. | NFR | High | Load test |
| NFR-04 | Reliability: Mission Control availability ≥ 99.5 % per month; chamber control and onboard monitoring MTBF ≥ 1000 h; no log entry lost during a link outage of up to 72 h. | NFR | High | Analysis + endurance test |
| NFR-05 | Usability: after 4 h of training, ≥ 90 % of 10 test operators shall issue a delivery command and read robot status within 30 s unaided. Safe Mode and protection alerts shall be visually and audibly distinct. Static portal pages (robots, operators, artefacts) shall load in ≤ 3 s and meet WCAG 2.1 AA. | NFR | Medium | Usability test + accessibility audit |
| SR-01 | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | Safety | Critical | Test + analysis |
| SR-02 | The robot shall enter a railway level-crossing only when the latest ARLCCS status (≤ 2 s old) reports barrier open and no train approaching; otherwise it shall hold at the stop line 5 m before the barrier. | Safety | Critical | Hardware-in-the-loop test + formal analysis |
| SR-03 | If no valid heartbeat is received for 30 s the robot shall enter LINK_LOST, stop at the nearest safe stopping point outside any crossing zone, keep chamber monitoring and logging, never start a delivery, and return autonomously after 10 min if the link is not restored. | Safety | Critical | Test with link emulator |
| SR-04 | The robot shall never begin DELIVERING from IDLE or from AVOIDING_OBSTACLE. DELIVERING may be entered only from NAVIGATING on DestinationReached. | Safety | Critical | Model checking + test |

Summary: 10 Functional Requirements (FR-01 to FR-10), 5 Non-Functional Requirements (NFR-01 to NFR-05), 4 High-Risk / Safety Requirements (SR-01 to SR-04). NFR-03 and SR-01 are intentionally written in their original, less measurable form because Task 2 changes them (CR-02, CR-01). FR-01 is likewise the baseline text that CR-03 changes.

## 1.2 Why the four safety requirements are high risk

SR-01 (Safe Mode): a missed or late entry lets a batch overheat or a battery fail on the route.

SR-02 (railway crossing): an error can put a robot in front of a train; it depends on external ARLCCS data and on the delayed link.

SR-03 (link lost): the 5 s link delay and outages make remote supervision unreliable, so the robot must act safely alone.

SR-04 (no delivery from IDLE or while avoiding): a logic fault gives an uncontrolled handover; this is verified in Task 3.
