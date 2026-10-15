# Task 2: Change Requests and Traceability Matrix

## 2.1 Version-controlled change log

| CR | Req. ID | Version | Status | Old text | New text | Reason |
| --- | --- | --- | --- | --- | --- | --- |
| CR-01 | SR-01 | v1.0 → v1.1 | Modified | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | The robot shall enter Safe Mode within 3 seconds when chamber or battery temperature exceeds the critical threshold or battery capacity falls below the emergency level. | Original had no time limit and undefined 'critical'; Mission Control needs a bounded, measurable response. |
| CR-02 | NFR-03 | v1.0 → v1.1 | Modified | The system shall support communication with multiple robots simultaneously. | The system shall support at least 20 simultaneously connected robots. | 'Multiple' has no number; pilot serves 20 clinics, so capacity must be a testable figure. |
| CR-03 | FR-01 | v1.0 → v1.1 | Modified | Only authenticated operators shall be permitted to issue robot commands. | The system shall require authenticated and role-authorised operators before accepting any robot command. | Authentication alone lets any logged-in user command a robot; role-based authorisation is required for every command. |
| CR-03 | FR-02 | v1.0 → v1.0 (impacted) | Unchanged, cross-referenced | (see Task 1) | (no text change) | FR-02 is the detailed realisation of the new authorisation wording in FR-01; its tests are re-run to confirm the ordering authenticate → authorise → validate. |

Baseline v1.0 stays in the repository; v1.1 is a new tagged revision. Status "Modified" marks the three requirements that changed text. The unchanged FR-02 is listed only because CR-03 affects its tests. Affected IDs: SR-01 (CR-01), NFR-03 (CR-02), FR-01 (CR-03).

### Rewritten requirements (v1.1)

| Req. ID | Requirement (v1.1) | Type | Priority | Verification Method | Status |
| --- | --- | --- | --- | --- | --- |
| SR-01 | The robot shall enter Safe Mode within 3 seconds when chamber or battery temperature exceeds the critical threshold or battery capacity falls below the emergency level. | Safety | Critical | Test + analysis | Modified (CR-01) |
| NFR-03 | The system shall support at least 20 simultaneously connected robots. | NFR | High | Load test | Modified (CR-02) |
| FR-01 | The system shall require authenticated and role-authorised operators before accepting any robot command. | FR | High | Test (negative tests; role × command matrix) | Modified (CR-03) |

## 2.2 Why each new wording is more measurable or testable

### CR-01

A 3-second limit gives a numeric pass/fail criterion. The original gave none, so a tester could not say whether 10 s was acceptable.

'Critical battery or thermal condition' is split into three named measurements: chamber temperature, battery temperature and battery capacity. Each can be tested separately with boundary values just below, at and just above its threshold.

'Critical threshold' and 'emergency level' are tied to configured values (P-04, P-05), so the oracle is a number, not an opinion.

The response time can be measured directly from the log: timestamp of the threshold crossing to timestamp of the SAFE_MODE transition (T20–T24).

### CR-02

'Multiple' could be 2 or 200. 'At least 20' matches the pilot (20 clinics) and gives one exact load level for the test.

'Simultaneously connected' defines a checkable state: 20 open sessions at the same time, each sending telemetry and accepting commands, while NFR-01 timing still holds.

Residual gap: there is no upper bound or growth margin. A later CR should state a design target (for example 30 robots) so that scalability can be tested beyond the pilot.

### CR-03

The original covered authentication only, so a logged-in Viewer could still move a robot. The new text adds role authorisation, which creates three clear test classes: unauthenticated, authenticated-but-unauthorised, and both satisfied.

'Before accepting' fixes the order and the observation point. A rejected command must cause no state change, return an error code and leave a log entry. Each of these is observable.

'Any robot command' means every command type must appear in the role × command matrix, so coverage can be counted instead of argued.

## 2.3 Test cases

Pipeline for command tests: command received → authenticated → authorised → validated → executed → status reported → logged. Column "Scenario steps" shows which steps each test exercises (✔ done, ✘ fails on purpose, – not applicable).

| TC ID | Test case | Scenario steps | Requirements | Expected result |
| --- | --- | --- | --- | --- |
| TC-01 | Valid command, end to end: authenticated Operator sends a valid command. | Received ✔ → Authenticated ✔ → Executed ✔ → Status reported ✔ → Logged ✔ | FR-01, FR-02, FR-03, FR-09, FR-10, NFR-01 | Command ACCEPTED, executed, DONE status visible, one log entry per step. |
| TC-02 | Command with no session or expired session. | Received ✔ → Authenticated ✘ → Executed – → Status reported ✔ (REJECTED) → Logged ✔ | FR-01, NFR-02 | Rejected, no state change, error code returned, rejection logged. |
| TC-03 | Authenticated Viewer sends DeliveryRequest. | Received ✔ → Authenticated ✔ → Authorised ✘ → Executed – → Status ✔ → Logged ✔ | FR-02 | AUTHZ_DENIED, robot state unchanged, event logged. |
| TC-04 | Stale (> 30 s), replayed or malformed command. | Received ✔ → Authenticated ✔ → Validated ✘ → Executed – → Status ✔ → Logged ✔ | FR-03 | Rejected with a coded reply; no execution. |
| TC-05 | DeliveryRequest in IDLE with battery above 20 %, full mission to IDLE (T01, T04, T05, T06). | Received ✔ → Authenticated ✔ → Executed ✔ → Status ✔ → Logged ✔ | FR-05 | NAVIGATING within 5 s; DELIVERING only at destination; IDLE on arrival. |
| TC-06 | Inject E16 and spurious DestinationReached in IDLE and LINK_LOST (T08, T10, T36). | Received ✔ → Authenticated ✔ → Executed ✘ (blocked) → Status ✔ → Logged ✔ | SR-04 | State stays IDLE / LINK_LOST; DELIVERING never entered. |
| TC-07 | Obstacle in NAVIGATING and RETURNING; clears; times out; stationary at destination (T02, T03, T13, T17, T19). | Executed ✔ → Status ✔ → Logged ✔ | FR-04 | Stop ≤ 1 s; resume after 2 s clear; timeout → RETURNING. |
| TC-08 | DestinationReached while AVOIDING_OBSTACLE (T09). | Executed ✘ (event deferred) → Status ✔ → Logged ✔ | FR-04, SR-04 | No DELIVERING while avoiding; after ObstacleCleared the robot delivers. |
| TC-09 | CriticalBattery in NAVIGATING, AVOIDING_OBSTACLE, DELIVERING, IDLE (T07, T11, T12, T15). | Executed ✔ → Status ✔ → Logged ✔ | FR-05, SR-01 | Abandon and return, or finish handover then return; requests rejected in IDLE. |
| TC-10 | Boundary test: chamber temperature, battery temperature and battery capacity just below, at and just above threshold (CR-01). | Executed ✔ → Status ✔ → Logged ✔ | SR-01 | SAFE_MODE within 3 s above threshold; none below; time measured from log. |
| TC-11 | SafeModeTrigger from every state; exit only with authorised reset to IDLE (T20–T27). | Received ✔ → Authenticated ✔ → Executed ✔ → Status ✔ → Logged ✔ | SR-01 | Motion disabled; requests rejected; only T25 leaves SAFE_MODE. |
| TC-12 | Heartbeat stops for 30 s, then restored; then not restored for 10 min (T28–T35, T37). | Executed ✔ → Status queued → Logged ✔ | SR-03, FR-09 | LINK_LOST at a safe point outside crossing; autonomous return after 10 min. |
| TC-13 | Crossing: barrier open with train approaching; barrier failed; sensors disagree (T38). | Executed ✔ → Status ✔ → Logged ✔ | SR-02 | Robot holds 5 m before barrier. |
| TC-14 | Crossing: barrier open, no train, fresh status; then status 3 s old; then ARLCCS link lost (T38, T39). | Executed ✔ → Status ✔ → Logged ✔ | SR-02 | Enters only on fresh safe status. |
| TC-15 | Out-of-range temperature and humidity: recovery succeeds; recovery fails; removal request in protection mode. | – | FR-06, FR-08 | 3 consecutive readings needed; protection mode at 10 min; alert ≤ 5 s; removal denied. |
| TC-16 | Door opens during conservation (authorised in DELIVERING, unauthorised while moving); door closes with sensors verified or not. | – | FR-07 | Suspension; alert ≤ 5 s; resume only after verification. |
| TC-17 | Load test: 20 simultaneously connected robots with 5 s link delay, commands and telemetry at the same time. | Received ✔ → Authenticated ✔ → Executed ✔ → Status ✔ → Logged ✔ (×20) | NFR-01, NFR-03 | All 20 sessions stay up; 95th percentile acknowledgement ≤ 12 s. |
| TC-18 | Link outage up to 72 h: telemetry and log store-and-forward. | Logged ✔ → Status ✔ (after restore) | FR-09, FR-10, NFR-04 | No lost entries; ordered and time-stamped on resend. |
| TC-19 | Replay, tamper, expired session, certificate check. | Received ✔ → Authenticated ✘ | NFR-02, FR-01, FR-03 | 100 % rejected; sessions end after 15 min idle. |
| TC-20 | Usability test with 10 operators; portal load time and accessibility audit. | – | NFR-05 | ≥ 90 % complete the task ≤ 30 s; portal ≤ 3 s; WCAG 2.1 AA. |
| TC-21 | Power loss with and without emergency supply. | – | FR-08 | Emergency power within 1 s; else incident recorded and safe shutdown. |
| TC-22 | Endurance run (availability, MTBF measurement). | – | NFR-04 | Availability ≥ 99.5 % per month; no unlogged failures. |
| TC-23 | Vibration above threshold then below for 60 s. | – | FR-07 | Risky activities suspended; resumed after 60 s stable. |

All 23 tests are designed, none are executed yet. Eight or more are traceable to the command pipeline (TC-01 to TC-06, TC-11, TC-17, TC-19).

## 2.4 Test cases to re-run for each change request

### CR-01 (SR-01)

| TC ID | Action and reason |
| --- | --- |
| TC-10 | Rewritten: new 3 s limit and three trigger variables. |
| TC-11 | Re-run: Safe Mode entry from every state now time-bounded. |
| TC-09 | Re-run: CriticalBattery (≤ 20 %) must still return, not trigger Safe Mode, above the emergency level. |
| TC-15 | Re-run: chamber temperature threshold interacts with correction and protection mode. |
| TC-12 | Re-run: Safe Mode priority over LINK_LOST (T35). |
| TC-18 | Re-run: Safe Mode events must be logged and forwarded. |

### CR-02 (NFR-03)

| TC ID | Action and reason |
| --- | --- |
| TC-17 | Rewritten: load level is now 20 robots. |
| TC-01 | Re-run under 20 concurrent sessions. |
| TC-12 | Re-run: independent link loss on several robots. |
| TC-18 | Re-run: 20 robots forwarding stored logs. |
| TC-19 | Re-run: 20 sessions and keys. |
| TC-22 | Re-run: availability and endurance under full load. |

### CR-03 (FR-01)

| TC ID | Action and reason |
| --- | --- |
| TC-01 | Re-run: success path now needs both authentication and authorisation. |
| TC-02 | Re-run: unauthenticated rejection. |
| TC-03 | Rewritten: new authorisation criterion. |
| TC-04 | Re-run: order of checks (authenticate → authorise → validate). |
| TC-05 | Re-run: DeliveryRequest passes the new gate. |
| TC-19 | Re-run: security tests of the gate. |
| TC-18 | Re-run: rejected commands must be logged. |
| TC-20 | Re-run: operator messages for AUTHZ_DENIED. |

## 2.5 Multi-layer Requirement Traceability Matrix

Chain: Requirement → State / Transition → Operation → Constraint → Test Case → Risk Level. T-IDs come from Task 3 (robot), CT-IDs and OP-IDs from Task 4 (chamber and system operations), CON-IDs from Task 5, TC-IDs from section 2.3. The matrix is shown at v1.1.

| Req. ID | State / Transition | Operation | Constraint | Test Case | Risk Level |
| --- | --- | --- | --- | --- | --- |
| FR-01 | Pipeline: RECEIVED → AUTHENTICATED (gate on every T-row) | OP-21, OP-22, OP-27 | CON-15 | TC-01, TC-02, TC-19 | High |
| FR-02 | Pipeline: AUTHENTICATED → AUTHORISED | OP-23, OP-27 | CON-15 | TC-01, TC-03 | High |
| FR-03 | Pipeline: AUTHORISED → VALIDATED → EXECUTING | OP-24, OP-25 | CON-15 | TC-01, TC-04, TC-19 | High |
| FR-04 | T02, T03, T13, T17, T19, T37 | OP-31, OP-27 | CON-13 | TC-07, TC-08 | High |
| FR-05 | T01, T04, T05, T06, T07, T11, T12, T14, T15, T16, T18 | OP-25, OP-26, OP-32 | CON-13, CON-14 | TC-05, TC-09 | High |
| FR-06 | CT-04, CT-05, CT-06 | OP-06, OP-07, OP-08, OP-09 | CON-18, CON-19 | TC-15 | High |
| FR-07 | CT-01, CT-02, CT-03, CT-07, CT-08, CT-09, CT-10, CT-13 | OP-01, OP-02, OP-03, OP-04, OP-05, OP-13, OP-14, OP-15, OP-16 | CON-16, CON-17, CON-22 | TC-16, TC-23 | High |
| FR-08 | CT-06, CT-11, CT-12, CT-14 | OP-10, OP-11, OP-12, OP-17, OP-18, OP-19, OP-20 | CON-19, CON-20, CON-21 | TC-15, TC-21 | High |
| FR-09 | All T and CT rows (status sent on each) | OP-26 | CON-23 | TC-01, TC-18 | Medium |
| FR-10 | All T and CT rows (record on each) | OP-27, OP-18 | CON-23 | TC-01, TC-18 | Medium |
| NFR-01 | Pipeline end to end | OP-21, OP-22, OP-23, OP-24, OP-25, OP-26 | None | TC-01, TC-17 | High |
| NFR-02 | Pipeline gate | OP-22, OP-23, OP-24 | CON-15 | TC-19 | High |
| NFR-03 | All robot sessions | OP-21, OP-26 | None | TC-17 | High |
| NFR-04 | CT-12, T33 | OP-27, OP-29 | CON-23 | TC-18, TC-22 | High |
| NFR-05 | None (UI) | OP-26 | None | TC-20 | Medium |
| SR-01 | T20, T21, T22, T23, T24, T25, T26, T27, T35 | OP-28, OP-32 | CON-12, CON-24 | TC-10, TC-11 | Critical |
| SR-02 | T38, T39 | OP-30 | CON-01, CON-02, CON-03, CON-04, CON-05, CON-06, CON-07, CON-08, CON-09, CON-10 | TC-13, TC-14 | Critical |
| SR-03 | T27, T28, T29, T30, T31, T32, T33, T34, T35, T37 | OP-29, OP-28 | CON-11 | TC-12, TC-18 | Critical |
| SR-04 | T08, T09, T10, T36 | OP-25, OP-31 | CON-13, CON-14 | TC-06, TC-08 | Critical |
