# Task 3: State Modelling and Transition Verification

## 3.1 States of the delivery robot

### Base model (five states from the scenario)

| ID | State | Meaning | Entry condition | Exit condition |
| --- | --- | --- | --- | --- |
| S1 | IDLE | Robot is powered on, stationary at the warehouse, chamber monitoring active, waiting for a request. | Power-on self-check passed; WarehouseReached from RETURNING; SafeModeCleared; LinkRestored (origin IDLE). | Valid DeliveryRequest (→ S2); SafeModeTrigger (→ S6); LinkLost (→ S7). |
| S2 | NAVIGATING | Robot is driving the route to the destination and checking for obstacles and crossings. | DeliveryRequest accepted; ObstacleCleared with Resume_Target = NAVIGATING; LinkRestored after re-validation. | ObstacleDetected (→ S3); DestinationReached (→ S4); CriticalBattery (→ S5); SafeModeTrigger (→ S6); LinkLost (→ S7). |
| S3 | AVOIDING_OBSTACLE | Navigation is suspended and the robot manoeuvres around an obstacle. Robot must not deliver here. | ObstacleDetected from S2 or S5 (Resume_Target stored). | ObstacleCleared (→ Resume_Target); CriticalBattery (→ S5); AvoidanceTimeout (→ S5); SafeModeTrigger (→ S6). |
| S4 | DELIVERING | Robot is stationary at the destination handing over the package. | DestinationReached while in S2 only. | DeliveryComplete (→ S5); HandoverBlocked (→ S5); SafeModeTrigger (→ S6). |
| S5 | RETURNING | Robot is travelling back to the warehouse (after delivery, or after abandoning it). | DeliveryComplete; CriticalBattery from S2 or S3; AvoidanceTimeout; HandoverBlocked; LinkTimeout. | WarehouseReached (→ S1); ObstacleDetected (→ S3); SafeModeTrigger (→ S6). |

### Extended model (3.4: Safe Mode and Link Lost added)

| ID | State | Meaning | Entry condition | Exit condition |
| --- | --- | --- | --- | --- |
| S6 | SAFE_MODE | Robot is stopped (outside any crossing zone), motion disabled, chamber monitoring and alerts continue. No delivery or navigation is possible. | SafeModeTrigger (chamber or battery temperature above critical threshold, battery capacity below emergency level, or critical fault) from any state. | SafeModeCleared by an authorised operator with conditions verified normal (→ S1 only). |
| S7 | LINK_LOST | No valid heartbeat for 30 s. Robot is stopped at a safe point outside any crossing zone, still monitoring and logging. | LinkLost from S1 or S2 (from S3 after the obstacle is cleared). | LinkRestored (→ origin state after re-validation); LinkTimeout 10 min (→ S5); SafeModeTrigger (→ S6). |

## 3.2 Events and conditions

| ID | Event / condition | Kind | Meaning |
| --- | --- | --- | --- |
| E1 | DeliveryRequest | Event | Valid, authenticated, authorised delivery command arrives. |
| E2 | ObstacleDetected | Event | Obstacle sensors report an object on the path. |
| E3 | ObstacleCleared | Event | Path clear for 2 consecutive seconds. |
| E4 | DestinationReached | Event | Position within the delivery radius of the destination. |
| E5 | DeliveryComplete | Event | Handover confirmed (batch removal authorised and door closed). |
| E6 | CriticalBattery | Condition | Battery capacity ≤ 20 % (P-03). Level condition, latched until recharge. |
| E7 | WarehouseReached | Event | Robot arrives at the warehouse. |
| E8 | SafeModeTrigger | Condition | Chamber or battery temperature above critical threshold, or battery < 10 % (CR-01), or critical fault. |
| E9 | SafeModeCleared | Event | Authorised operator reset and all trigger conditions verified normal. |
| E10 | LinkLost | Condition | No valid heartbeat for 30 s. |
| E11 | LinkRestored | Event | Valid heartbeat and session re-established. |
| E12 | LinkTimeout | Event | 10 min in LINK_LOST. |
| E13 | AvoidanceTimeout | Event | Obstacle not cleared within 90 s. |
| E14 | HandoverBlocked | Event | Batch removal denied (protection active) or recipient absent for 5 min. |
| E15 | CrossingNotSafe / CrossingSafe | Condition | ARLCCS status stale, barrier not open, train approaching, or the reverse. |
| E16 | StartDelivery (illegal) | Event | Fault-injection event: an unguarded 'deliver now' trigger. Must never move the robot into DELIVERING. |

### State versus event

A state is a situation the robot stays in over time. It has duration, and the robot can be asked "which state are you in now?" (NAVIGATING lasts until something changes it).

An event is something that happens at one moment and may cause a change of state (ObstacleDetected). A condition is a level that stays true (CriticalBattery) and is re-checked on state entry so it cannot be lost.

A transition links them: Current State + Event [guard] → Next State. Confusing the two causes errors such as modelling "Delivering" as an event, which is how a robot can be told to deliver from anywhere.

Simultaneous events are resolved in this order: E8 SafeModeTrigger > E6 CriticalBattery > E10 LinkLost > E2 ObstacleDetected > E4 DestinationReached > E3 ObstacleCleared. Level conditions (E6, E8, E10, E15) are re-evaluated on entry to every state, so a condition that fired while the robot was busy is not lost.

## 3.3 State Transition Table

Valid = allowed by the scenario or safety rules. INVALID rows are forbidden transitions that must not exist in the implementation and are tested as negative cases. "Added" rows are transitions the scenario leaves undefined, found by the verifier in 3.5 and added to close gaps.

| ID | Current State | Event [guard] | Next State | Req. ID | Valid? | Verification evidence |
| --- | --- | --- | --- | --- | --- | --- |
| T01 | IDLE | E1 DeliveryRequest [authenticated ∧ authorised ∧ battery > 20 %] | NAVIGATING | FR-05 | Valid | Scenario sentence 2; TC-05; guard G1 |
| T02 | NAVIGATING | E2 ObstacleDetected | AVOIDING_OBSTACLE | FR-04 | Valid | Scenario sentence 3; TC-07 |
| T03 | AVOIDING_OBSTACLE | E3 ObstacleCleared | Resume_Target (NAVIGATING or RETURNING) | FR-04 | Valid | Scenario sentence 3; TC-07 |
| T04 | NAVIGATING | E4 DestinationReached | DELIVERING | FR-05, SR-04 | Valid | Scenario sentence 4; TC-05; only entry to DELIVERING |
| T05 | DELIVERING | E5 DeliveryComplete | RETURNING | FR-05 | Valid | Scenario sentence 4; TC-05 |
| T06 | RETURNING | E7 WarehouseReached | IDLE | FR-05 | Valid | Scenario sentence 6; TC-05 |
| T07 | NAVIGATING | E6 CriticalBattery | RETURNING | FR-05 | Valid | Scenario sentence 5; TC-09 |
| T08 | IDLE | E16 StartDelivery (illegal) | DELIVERING | SR-04 | INVALID: must not exist | Scenario sentence 7; TC-06; unreachable in model check |
| T09 | AVOIDING_OBSTACLE | E4 DestinationReached | DELIVERING | SR-04 | INVALID: must not exist | Scenario sentence 7; TC-08; event deferred, re-checked on re-entry to NAVIGATING |
| T10 | IDLE | E4 DestinationReached (spurious) | DELIVERING | SR-04 | INVALID: must not exist | TC-06; GPS drift at warehouse |
| T11 | AVOIDING_OBSTACLE | E6 CriticalBattery | RETURNING | FR-05 | Added (gap fix) | Verifier finding 5(d); TC-09 |
| T12 | DELIVERING | E6 CriticalBattery | DELIVERING (finish handover, then T05) | FR-05 | Added (gap fix) | Verifier finding 5(d); TC-09 |
| T13 | RETURNING | E2 ObstacleDetected | AVOIDING_OBSTACLE (Resume_Target = RETURNING) | FR-04 | Added (gap fix) | Finding F5; TC-07 |
| T14 | RETURNING | E6 CriticalBattery | RETURNING (no change) | FR-05 | Added (gap fix) | Finding F5; model completeness |
| T15 | IDLE | E6 CriticalBattery / E1 with low battery | IDLE (request rejected, alert sent) | FR-05 | Added (gap fix) | TC-09 |
| T16 | NAVIGATING | E1 DeliveryRequest | NAVIGATING (rejected: BUSY) | FR-03 | Added (gap fix) | Finding F6; TC-04 |
| T17 | AVOIDING_OBSTACLE | E13 AvoidanceTimeout | RETURNING | FR-04 | Added (gap fix) | Verifier finding 5(b); TC-07 |
| T18 | DELIVERING | E14 HandoverBlocked | RETURNING (batch quarantined) | FR-05 | Added (gap fix) | Finding F9; TC-15 |
| T19 | DELIVERING | E2 ObstacleDetected | DELIVERING (hold, no motion) | FR-04 | Added (gap fix) | Finding F10; TC-07 |
| T20 | IDLE | E8 SafeModeTrigger | SAFE_MODE | SR-01 | Valid | CR-01; TC-10, TC-11 |
| T21 | NAVIGATING | E8 SafeModeTrigger | SAFE_MODE | SR-01 | Valid | CR-01; TC-10, TC-11 |
| T22 | AVOIDING_OBSTACLE | E8 SafeModeTrigger | SAFE_MODE | SR-01 | Valid | CR-01; TC-11 |
| T23 | DELIVERING | E8 SafeModeTrigger | SAFE_MODE (door locked, handover aborted) | SR-01 | Valid | CR-01; TC-11 |
| T24 | RETURNING | E8 SafeModeTrigger | SAFE_MODE | SR-01 | Valid | CR-01; TC-11 |
| T25 | SAFE_MODE | E9 SafeModeCleared | IDLE (never NAVIGATING or DELIVERING) | SR-01 | Valid | TC-11 |
| T26 | SAFE_MODE | E1 DeliveryRequest | SAFE_MODE (rejected) | SR-01 | Valid | TC-11 |
| T27 | SAFE_MODE | E10 LinkLost | SAFE_MODE (flag set) | SR-01, SR-03 | Valid | TC-11, TC-12 |
| T28 | IDLE | E10 LinkLost | LINK_LOST | SR-03 | Valid | TC-12 |
| T29 | NAVIGATING | E10 LinkLost | LINK_LOST (stop at safe point outside crossing zone) | SR-03 | Valid | TC-12 |
| T30 | AVOIDING_OBSTACLE | E10 LinkLost | AVOIDING_OBSTACLE (latched; after T03 → LINK_LOST) | SR-03 | Valid | TC-12 |
| T31 | DELIVERING | E10 LinkLost | DELIVERING (flag set; after T05 continue return) | SR-03 | Valid | TC-12 |
| T32 | RETURNING | E10 LinkLost | RETURNING (flag set, autonomous return continues) | SR-03 | Valid | TC-12 |
| T33 | LINK_LOST | E11 LinkRestored | Origin state (IDLE or NAVIGATING) after command re-validation | SR-03 | Valid | TC-12 |
| T34 | LINK_LOST | E12 LinkTimeout | RETURNING | SR-03 | Valid | TC-12 |
| T35 | LINK_LOST | E8 SafeModeTrigger | SAFE_MODE | SR-01, SR-03 | Valid | TC-11, TC-12 |
| T36 | LINK_LOST | E4 DestinationReached / E16 | DELIVERING | SR-04 | INVALID: must not exist | TC-06, TC-12; no unsupervised delivery |
| T37 | LINK_LOST | E2 ObstacleDetected | LINK_LOST (hold, event logged) | FR-04, SR-03 | Valid | TC-12 |
| T38 | NAVIGATING | E15 CrossingNotSafe | NAVIGATING (sub-state HOLD_AT_STOP_LINE) | SR-02 | Valid | TC-13, TC-14 |
| T39 | NAVIGATING (HOLD) | E15 CrossingSafe [status ≤ 2 s old] | NAVIGATING (proceed) | SR-02 | Valid | TC-14 |

## 3.4 Safe Mode and Link Lost: which states may enter them

| Existing state | May enter SAFE_MODE? | May enter LINK_LOST? | Reason |
| --- | --- | --- | --- |
| IDLE | Yes (T20) | Yes (T28) | A thermal fault can occur at the warehouse. With no motion, link loss only blocks new requests. |
| NAVIGATING | Yes (T21) | Yes (T29) | Moving robot must stop in a known safe state. LINK_LOST stop point is never inside a crossing zone. |
| AVOIDING_OBSTACLE | Yes (T22) | Deferred (T30) | Avoidance is autonomous and must finish; stopping mid-manoeuvre is riskier. Enters LINK_LOST after ObstacleCleared. |
| DELIVERING | Yes (T23) | No (T31) | Handover is local. It completes, then the robot returns autonomously; LINK_LOST would strand a half-finished handover. |
| RETURNING | Yes (T24) | No (T32) | Returning to the warehouse is already the safe fallback action, so the robot keeps going. |
| SAFE_MODE | n/a | No (T27) | Safe Mode has higher priority; link loss is recorded as a flag. |
| LINK_LOST | Yes (T35) | n/a | Safety condition overrides link handling. |

Rule: SAFE_MODE has priority over LINK_LOST. Neither state may stop the robot inside a railway-crossing zone (CON-10, CON-11); the robot clears the crossing first.

## 3.5 Verifier investigation

### (a) Can IDLE → DELIVERING happen?

In the described behaviour there is no such transition: IDLE leaves only through DeliveryRequest to NAVIGATING. An implementation could still allow it through (i) an unguarded 'deliver now' command accepted in IDLE (E16), (ii) a spurious DestinationReached while at the warehouse, for example GPS drift when the warehouse is also a stored destination, or (iii) a DeliveryRequest handler that jumps straight to DELIVERING when the robot is already within the delivery radius.

Requirement violated: SR-04 (and the scenario rule 'a waiting robot must never begin delivering directly'), also constraint CON-14. Consequence: the batch is handed over (or the chamber door unlocked) at the wrong place without navigation, checks or crossing clearance.

Resolution: T08 and T10 are marked INVALID. Guard: Start_Delivering requires previous state = NAVIGATING ∧ At_Destination. Test TC-06 injects E16 and a spurious E4 in IDLE and expects the state to stay IDLE. Model check: DELIVERING must be unreachable from IDLE without passing through NAVIGATING.

### (b) What if AVOIDING_OBSTACLE has no transition back? Can the robot still deliver?

No, the robot cannot deliver: DELIVERING is only reachable from NAVIGATING (T04), and without a way back the robot never returns to NAVIGATING. The delivery safety rule holds, but the model fails a different property: it is a dead-end (trap) state.

Consequences: the package is never delivered, the robot never returns, the chamber keeps running on a draining battery, and with no CriticalBattery or Safe Mode transition out of the state the robot can freeze with a batch whose temperature is no longer controlled. This is safe with respect to SR-04 but unsafe for the cold chain.

Resolution: exits T03 (ObstacleCleared), T17 (AvoidanceTimeout 90 s → RETURNING), T11 (CriticalBattery), T22 (SafeModeTrigger) and T30 (LinkLost, latched). Every state must have at least one exit; this is checked mechanically in the verification table.

### (c) Can AVOIDING_OBSTACLE → DELIVERING occur?

It must not (T09 is INVALID). It can still occur by accident in a realistic way: an obstacle appears when the robot is just before the destination, the robot drifts inside the delivery radius while avoiding, and DestinationReached fires while in AVOIDING_OBSTACLE. An implementation with a shared handler or an edge-triggered flag could go straight to DELIVERING.

Requirement violated: SR-04 and the scenario rule 'must never deliver while handling an obstacle'; constraint CON-13 (mutual exclusion).

Resolution: ignore E4 in S3 (event deferred). On re-entry to NAVIGATING (T03) re-evaluate At_Destination as a level condition, so E4 fires again and T04 then moves to DELIVERING legitimately. TC-08 covers it.

### (d) What happens on CriticalBattery in AVOIDING_OBSTACLE or DELIVERING (missing transition)?

With no transition the event is silently dropped (undefined behaviour). In AVOIDING_OBSTACLE the robot finishes avoiding and resumes NAVIGATING (T03) with the battery condition forgotten, so it continues toward the destination on a critical battery and may strand on the route. In DELIVERING it completes the handover and returns, but nothing guarantees the return is feasible, and there is no handling if the battery then drops below the emergency level.

Requirements affected: FR-05 (abandon delivery on critical battery) and SR-01 (Safe Mode on emergency level).

Resolution: T11 (AVOIDING_OBSTACLE → RETURNING) and T12 (DELIVERING: finish handover, never extend, then T05). Emergency level goes to SAFE_MODE from both states (T22, T23). CriticalBattery is a latched level condition re-checked on every state entry, so it cannot be lost by timing.

### Additional findings from the review of the table

| ID | Problem found | Fix |
| --- | --- | --- |
| F5 | No transition for ObstacleDetected or CriticalBattery in RETURNING. | Robot could hit an obstacle on the way back. Fixed by T13 (Resume_Target = RETURNING) and T14. |
| F6 | DeliveryRequest arriving in a non-IDLE state is undefined. | A second request could overwrite the active mission. Fixed by T16 (reject BUSY), T26, T15. |
| F7 | Simultaneous events are not ordered. | Safety conditions could lose against comfort events. Fixed by the priority rule under the event table. |
| F8 | SAFE_MODE could exit straight into NAVIGATING. | T25 allows SAFE_MODE → IDLE only, with authorised reset. |
| F9 | Handover can be blocked by protection mode (batch removal not authorised, constraint CON-20), leaving DELIVERING with no exit. | Fixed by T18 (HandoverBlocked → RETURNING, batch quarantined). |
| F10 | Obstacle appearing during DELIVERING (a person walking up). | T19 holds the handover; the robot never enters AVOIDING_OBSTACLE while stationary at the destination. |
