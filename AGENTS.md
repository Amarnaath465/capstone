# Agent Guidance

## Source Of Truth

- Treat [HVAC_Edge_Air_Quality_Optimizer_Requirements.md](HVAC_Edge_Air_Quality_Optimizer_Requirements.md) as the top-level specification.
- Preserve the requirement IDs `HVACOPT-01` through `HVACOPT-06` in design notes, tests, and traceability documentation.
- Do not invent a build, test, or deployment command until the repository adds the corresponding tooling. The requirements call for a deterministic host build using `gcc`/`make` with at least `-Wall -Wextra` and a green `make test` equivalent.

## Architecture Boundaries

- Keep the layers in the requirements document: HAL, drivers, control/logic, and integration/telemetry; diagnostics are cross-cutting.
- HAL APIs are used directly only by drivers. Drivers expose sensor snapshots and actuator commands without depending on the optimizer.
- The optimizer consumes sensor snapshots and produces actuator commands without depending on Modbus or transport code.
- Modbus read/write handling belongs in the integration layer. Validate supervisory writes and pass commands through an explicit control or telemetry-bridge interface; do not mutate optimizer or driver state directly from the Modbus layer.
- Keep the register map as a versioned documented contract. Any register-map change must update its documentation and the traceability matrix together.

## Embedded Constraints

- Do not use dynamic allocation. Prefer fixed-size, caller-owned state and buffers.
- Keep host tests deterministic: no wall-clock dependence, real randomness, or hardware assumptions.
- Define and document control-loop cadence, response-time budgets, overrun behavior, hysteresis/deadband thresholds, and actuator fail-safe behavior before relying on them.
- Use explicit state-machine transitions for device modes rather than scattered mode booleans.
- Represent sensor, actuator, and Modbus/comms faults distinctly, expose them locally and over Modbus, and test degraded-mode behavior.

## Delivery Workflow

- Before implementing each functional requirement, create its `spec.md`, reviewed `plan.md`, and ordered `tasks.md` chain as required by the top-level specification.
- Add isolated unit tests for drivers, optimizer logic, actuator translation, and register encoding/decoding; add end-to-end control-loop tests and dedicated boundary/negative tests for malformed frames, invalid sensor values, invalid writes, and timeouts.
- Keep architecture, coding standards, register-map, traceability, MCP/agentic usage, and release-readiness evidence under `docs/` as those artifacts are introduced. Link to those documents from agent guidance instead of duplicating their contents here.
- When using an MCP tool, skill, or agent workflow, document its permission boundary, context boundary, purpose, and failure handling in the project documentation.
- Prefer small, requirement-scoped changes. Run the narrowest available validation after each change, then the full host build and test suite when those tools exist.