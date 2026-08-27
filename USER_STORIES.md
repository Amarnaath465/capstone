# HVAC Edge Optimizer User Stories

These stories decompose the top-level requirements into implementation-sized slices. Each story should become a `spec.md` -> `plan.md` -> `tasks.md` chain before implementation, with tests added to the traceability matrix.

## Story Summary

| ID | Story | Requirement trace | Primary layer |
|---|---|---|---|
| US-01 | Collect a deterministic sensor snapshot | HVACOPT-01 | Driver / HAL |
| US-02 | Detect and represent sensor faults | HVACOPT-01, HVACOPT-05 | Driver / Diagnostics |
| US-03 | Control temperature and humidity | HVACOPT-02 | Control / Logic |
| US-04 | Prioritize air-quality ventilation | HVACOPT-02 | Control / Logic |
| US-05 | Prevent actuator chatter | HVACOPT-02, HVACOPT-06 | Control / Logic |
| US-06 | Apply actuator commands safely | HVACOPT-03 | Driver / HAL |
| US-07 | Manage explicit controller modes | HVACOPT-06 | Control / Logic |
| US-08 | Publish controller state over Modbus | HVACOPT-04 | Integration / Telemetry |
| US-09 | Accept safe supervisory Modbus commands | HVACOPT-04, HVACOPT-06 | Integration / Telemetry |
| US-10 | Expose faults and prove behavior | HVACOPT-05 | Diagnostics / Tests |

## US-01: Collect A Deterministic Sensor Snapshot

**As** the control loop, **I want** temperature, relative humidity, and the selected air-quality proxy polled at a defined interval into one fixed-size snapshot, **so that** each optimization cycle uses a consistent input set.

**Acceptance criteria**

- The poll API returns one caller-owned snapshot containing temperature, humidity, air quality, validity/fault flags, and the sample status for every channel.
- The polling interval is a documented constant or configuration value and is not derived from wall-clock time in host tests.
- The HAL simulation returns repeatable values for the same configured inputs.
- A unit test proves a complete snapshot is produced without dynamic allocation.

**Traceability:** HVACOPT-01; Non-Functional Constraints: memory and determinism.

## US-02: Detect And Represent Sensor Faults

**As** a controller, **I want** out-of-range, stuck, and communication-timeout sensor conditions identified per channel, **so that** an invalid reading cannot silently influence control decisions.

**Acceptance criteria**

- Each sensor channel has an independent fault flag and documented valid range.
- Out-of-range values, an injected stuck-value condition, and a simulated timeout produce a faulted snapshot.
- Faulted values are either excluded from optimization or replaced by a documented safe behavior; they are never treated as valid measurements.
- Dedicated negative tests cover all supported sensor fault classes.

**Traceability:** HVACOPT-01, HVACOPT-05; Acceptance Criteria: simulated sensor fault handling.

## US-03: Control Temperature And Humidity

**As** a zone occupant, **I want** the optimizer to compare valid sensor data with configured comfort bands, **so that** the heating/cooling command maintains a comfortable zone.

**Acceptance criteria**

- Given a valid snapshot and configured target bands, the optimizer returns a bounded heating, cooling, or idle command and a ventilation command.
- Temperature below the lower band requests heating, temperature above the upper band requests cooling, and an in-band temperature does not request unnecessary conditioning.
- Humidity status is included in the computed control/status result, with the chosen humidity response documented.
- The optimizer has no dependency on Modbus, UART/TCP, HAL, or driver implementation headers.
- Unit tests cover below-band, in-band, above-band, and invalid-input cases.

**Traceability:** HVACOPT-02; Non-Functional Constraints: portability.

## US-04: Prioritize Air-Quality Ventilation

**As** a building operator, **I want** poor air quality to increase ventilation even when temperature is comfortable, **so that** occupants are not exposed to stale or polluted air merely because the thermostat is satisfied.

**Acceptance criteria**

- The selected air-quality proxy and threshold are documented with units and scaling.
- When temperature is in-band and air quality is above the threshold, the optimizer requests increased ventilation.
- When air quality is within bounds, ventilation follows the documented baseline or comfort policy.
- The tradeoff between ventilation energy and air quality is stated and covered by a test that changes air quality while holding temperature constant.

**Traceability:** HVACOPT-02; Acceptance Criteria: air-quality-only actuator response.

## US-05: Prevent Actuator Chatter

**As** an HVAC equipment owner, **I want** hysteresis or deadband around control thresholds, **so that** small sensor fluctuations do not rapidly switch equipment on and off.

**Acceptance criteria**

- Heating, cooling, and ventilation transition thresholds are explicit, bounded, and documented.
- State carried between cycles is stored in a fixed-size caller-owned structure.
- Repeated readings that remain inside the deadband do not cause command toggling.
- Tests demonstrate both entry into and exit from each relevant control state.

**Traceability:** HVACOPT-02; Non-Functional Constraints: memory and timing.

## US-06: Apply Actuator Commands Safely

**As** a controller, **I want** optimizer commands translated into validated HAL calls, **so that** physical outputs remain bounded and enter a defined safe state when control fails.

**Acceptance criteria**

- Heat/cool stage and damper/fan commands are validated before reaching the HAL.
- Out-of-range commands are rejected or clamped according to a documented policy.
- If the optimizer stalls, reports a fault, or misses the control-loop budget, the actuator driver applies the documented fail-safe behavior.
- HAL calls can be observed through a deterministic mock, and unit tests cover normal, invalid, and fail-safe commands.

**Traceability:** HVACOPT-03; Acceptance Criteria: safe actuator state.

## US-07: Manage Explicit Controller Modes

**As** a facilities operator, **I want** the zone controller to expose explicit modes and validated transitions, **so that** automatic operation, manual override, faults, and shutdown are unambiguous.

**Acceptance criteria**

- The controller implements documented states such as `INIT`, `AUTO`, `MANUAL_OVERRIDE`, `FAULT`, and `SAFE_SHUTDOWN`, with any additional states justified.
- Events cause explicit transitions; mode is not represented only by unrelated boolean flags.
- Invalid transitions are rejected or routed to a documented safe state.
- A sensor, actuator, or communications fault produces the documented degraded mode and actuator behavior.
- State transition tests cover startup, automatic operation, manual override, return to auto, fault entry, and recovery/shutdown paths.

**Traceability:** HVACOPT-06, HVACOPT-03, HVACOPT-04.

## US-08: Publish Controller State Over Modbus

**As** a BMS master, **I want** to read the zone's live measurements, statuses, commands, mode, and faults through a documented Modbus map, **so that** I can supervise the zone without proprietary integration.

**Acceptance criteria**

- The chosen transport, RTU or TCP, and its rationale are documented behind an appropriate transport boundary.
- Every published point identifies register type, address, data type, scaling, unit, and access mode in a versioned register-map document.
- A simulated Modbus master can read all required points and receives values consistent with the current control-loop state.
- Malformed or short requests return a documented error without corrupting controller state.
- Register encode/decode tests cover scaling, signed values, bounds, and read-only behavior.

**Traceability:** HVACOPT-04; Modbus Interface Contract and Test Strategy Requirements.

## US-09: Accept Safe Supervisory Modbus Commands

**As** a BMS master, **I want** to change approved setpoints and overrides and return the controller to auto mode, **so that** I can supervise the zone remotely without bypassing control safeguards.

**Acceptance criteria**

- Supported writes include setpoint changes, air-quality threshold changes, documented manual overrides, and a return-to-auto command.
- Setpoints and thresholds outside the safe envelope are rejected or clamped according to the documented contract, and the result is observable to the caller.
- A mode write triggers a state-machine event rather than silently flipping a flag.
- The Modbus server validates and queues commands through a control or telemetry-bridge interface; it does not directly mutate optimizer or driver state.
- Boundary tests cover out-of-envelope writes, unsupported writes, malformed frames, and override-to-auto behavior.

**Traceability:** HVACOPT-04, HVACOPT-06; Acceptance Criteria: safe supervisory writes.

## US-10: Expose Faults And Prove Behavior

**As** a BMS operator and release reviewer, **I want** sensor, actuator, and communications faults visible locally and over Modbus with tests proving the response, **so that** degraded operation is diagnosable and release claims are evidence-based.

**Acceptance criteria**

- Sensor, actuator, and Modbus/comms diagnostics use distinct documented code ranges.
- Active fault codes and category flags are available through the local diagnostic interface and the Modbus fault/status points.
- An injected fault is logged or retrievable locally, exposed remotely, and drives the documented safe actuator/mode behavior.
- Unit, integration, boundary, and negative tests cover the fault path and the full sensor snapshot -> optimizer -> actuator -> register flow.
- A traceability matrix maps every `HVACOPT-*` requirement to at least one test, and the release-readiness audit records any remaining gaps.

**Traceability:** HVACOPT-01 through HVACOPT-06; Acceptance Criteria and Test Strategy Requirements.

## Suggested Delivery Order

Implement the stories in dependency order:

1. US-01 and US-02: establish valid inputs and fault semantics.
2. US-03 through US-05: implement and test the control policy.
3. US-06 and US-07: enforce output safety and explicit modes.
4. US-08 and US-09: add the versioned Modbus contract and supervisory path.
5. US-10: complete diagnostics, end-to-end evidence, traceability, and release review.