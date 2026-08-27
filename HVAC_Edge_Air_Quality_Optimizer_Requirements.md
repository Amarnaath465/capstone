# Capstone Project Requirements: Edge-Based HVAC Temperature and Air Quality Optimizer with Modbus Telemetry

## Honeywell Embedded Engineering Teams — Capstone Brief

---

## Document Revision

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-08-26 | Initial capstone requirements |

---

## Table of Contents

1. [Overview](#1-overview)
2. [Scenario](#2-scenario)
3. [Learning Objectives](#3-learning-objectives)
4. [System Architecture](#4-system-architecture)
5. [Functional Requirements](#5-functional-requirements)
6. [Non-Functional Constraints](#6-non-functional-constraints)
7. [Modbus Interface Contract](#7-modbus-interface-contract)
8. [Acceptance Criteria](#8-acceptance-criteria)
9. [Test Strategy Requirements](#9-test-strategy-requirements)
10. [Agentic/MCP Requirement](#10-agenticmcp-requirement)
11. [SDLC Deliverables & Repository Structure](#11-sdlc-deliverables--repository-structure)
12. [Suggested Milestones](#12-suggested-milestones)
13. [Grading Rubric / Definition of Done](#13-grading-rubric--definition-of-done)
14. [Stretch Goals (Optional)](#14-stretch-goals-optional)
15. [Dependencies and References](#15-dependencies-and-references)

---

## 1. Overview

The capstone brings together everything practiced in Modules 01–07 — spec-driven development, the seven-stage embedded SDLC, greenfield and brownfield discipline, test strategy and traceability, and MCP-enabled agentic engineering — and applies it to a single, self-contained project: a host-buildable edge controller that optimizes indoor temperature and air quality for one HVAC zone, and exposes/consumes its state over **Modbus**, the same class of field protocol used across Honeywell PA/IA/BA product lines.

Like `labs/module-02/sample-repo`, this is a **host-buildable stand-in** for real firmware — no physical hardware, MCU toolchain, or lab bench is required. Every sensor and actuator is a deterministic HAL simulation stub, so the whole thing builds and tests with `gcc`/`make`, the same way the Module 02–06 sample repository does. Teams may reuse that repository's HAL, coding standards, and layering conventions as a starting skeleton, or start a new repository that follows the same conventions — either is acceptable as long as the architecture rules in Section 4 hold.

This document is the capstone's top-level specification. It is deliberately more complete than the `spec.md` you'd write for a single module (Module 04's template) because it spans a multi-module system — but every sub-requirement in Section 5 should still be broken down into its own `spec.md` → `plan.md` → `tasks.md` chain before implementation, exactly as practiced in Module 04 and Module 05.

---

## 2. Scenario

> **HVACOPT-CAPSTONE** — A commercial building's BMS (Building Management System) needs an intelligent edge node for one HVAC zone (a single room, floor, or AHU-served area). Today, zone control is a dumb thermostat: it drives a setpoint and nothing else. Occupants complain of stuffiness and drowsiness even when temperature reads "comfortable," and facilities has no visibility into the zone from the BMS beyond current temperature.
>
> You are building the edge controller that replaces the dumb thermostat: it reads zone temperature, humidity, and an air-quality proxy (CO₂ and/or VOC), decides how to drive the zone's actuators (a heating/cooling stage and a fresh-air damper or ventilation fan) to keep occupants comfortable **and** the air within acceptable quality bounds, and reports its full state — and accepts supervisory commands — over Modbus, so it can be dropped into an existing BMS network without a proprietary integration.

This is deliberately under-specified in the same way EVENTLOG-01 (Module 05) and SENSOR-142 (Module 04) were: it names the problem and the constraints, not the design. Framing the problem and architecting the solution — module boundaries, interfaces, control strategy — is part of the capstone, not a given.

---

## 3. Learning Objectives

By the end of the capstone, you will have demonstrated:

1. **Spec-driven development at system scale** (Module 04) — decomposing one capstone-level problem statement into multiple module-level `spec.md → plan.md → tasks.md` chains, each independently implementable and testable.
2. **The seven-stage embedded SDLC** (Module 05) — problem framing, architecture/interfaces, module structure, implementation, test & static analysis, build & CI/CD, and release readiness — walked deliberately, not skipped.
3. **Layered embedded architecture discipline** — a HAL boundary, driver layer, control/logic layer, and integration/telemetry layer that respect the same layering rules as `sample-repo/docs/ARCHITECTURE.md`.
4. **Specification-to-test traceability and a real test pyramid** (Module 06) — unit tests with mocked HAL and a simulated Modbus master, integration tests across the control loop, and boundary/negative tests for out-of-range sensor and telemetry input.
5. **A working field-protocol integration** — a Modbus register map designed as a contract, implemented against, and validated the way `can_driver`'s state machine or `diag_formatter`'s error codes are: as a documented interface, not an incidental implementation detail.
6. **MCP-enabled / agentic engineering practice** (Module 07) — at least one reusable skill, MCP tool, or agent-assisted workflow used in service of building, testing, or documenting this project, with its governance (permissions, context boundaries, failure handling) considered, not just its output.
7. **Release-readiness judgment** — the ability to state, with evidence, what is and isn't true about this system before calling it "done."

---

## 4. System Architecture

Follow the same layered model used throughout this programme's sample repository, extended with a telemetry layer:

```
┌───────────────────────────────────────────────────────────┐
│  Integration / Telemetry Layer                              │
│  modbus_server.*  — register map, read/write dispatch,       │
│  telemetry_bridge.* — maps live control-loop state to/from   │
│  the register map                                            │
└───────────────┬───────────────────────┬─────────────────────┘
                │                       │
┌───────────────▼───────┐   ┌───────────▼─────────────────────┐
│  Control / Logic Layer  │   │  Diagnostics (cross-cutting)     │
│  hvac_optimizer.*        │   │  diag_formatter.* (reused or     │
│  (setpoint control,      │   │  extended from sample-repo)      │
│  air-quality-driven      │   └───────────────────────────────────┘
│  ventilation logic)      │
└───────────────┬─────────┘
                │
┌───────────────▼───────────────────────────────────────────┐
│  Driver Layer                                                │
│  sensor_poll.* (temp / humidity / air-quality),               │
│  actuator_driver.* (heat/cool stage, damper or fan)           │
└───────────────┬───────────────────────────────────────────┘
                │
┌───────────────▼───────────────────────────────────────────┐
│  HAL Layer (host-simulation stubs, no dynamic allocation)    │
│  hal_adc.*, hal_gpio.*, hal_pwm.* (as needed), hal_uart.*     │
│  or hal_tcp.* (Modbus transport)                              │
└───────────────────────────────────────────────────────────┘
```

### Layering Rules (extends `sample-repo/docs/ARCHITECTURE.md`)

1. **HAL is a leaf.** Only the driver layer touches `hal_*` functions directly.
2. **Drivers don't know about the optimizer.** `sensor_poll` and `actuator_driver` expose readings/commands; they have no dependency on `hvac_optimizer.h`.
3. **The optimizer doesn't know about Modbus.** `hvac_optimizer.c` reads sensor snapshots and produces actuator commands; it has no dependency on `modbus_server.h`. Translating optimizer state into register values (and register writes into optimizer commands/overrides) is the telemetry bridge's job — the same separation `fault_monitor` maintains from `can_driver` in Module 02–05's sample repo.
4. **The register map is a documented contract**, not a side effect of the implementation. Treat Section 7 the way `docs/CODING_STANDARDS.md` treats error codes: a table that implementation must match, not the other way around.
5. **Diagnostics are cross-cutting.** Reuse or extend `diag_formatter`-style error-code grouping for optimizer faults, sensor faults, and Modbus/comms faults — each in its own code range.

Decide, and document your reasoning (this feeds your top-level architecture doc), whether the Modbus server operates as **RTU** (serial, byte-stream framing, CRC) or **TCP** (MBAP header, no CRC) — or supports both behind a common transport interface. Either is acceptable; an undefended choice is not.

---

## 5. Functional Requirements

Each requirement below should become its own `spec.md` (Module 04 template) before implementation. IDs are provided for traceability; extend the numbering as sub-requirements emerge.

### HVACOPT-01 — Sensor Acquisition
- Poll zone temperature, relative humidity, and an air-quality proxy (CO₂ ppm and/or a VOC index) on a fixed, defined interval.
- Return a single snapshot struct per poll cycle with per-channel fault flags (matching `sensor_poll`'s existing pattern in the sample repo), not per-channel exceptions.
- Define and document out-of-range and sensor-fault behavior (e.g., stuck reading, out-of-bounds value, comms timeout to a simulated sensor) — a fault flag must never silently pass through as a plausible reading.

### HVACOPT-02 — Comfort & Air-Quality Optimization Logic
- Given a sensor snapshot and a current setpoint/target band (temperature range, humidity range, air-quality threshold), decide the heating/cooling stage command and the ventilation/damper command for this cycle.
- The optimizer must treat air quality as a first-class input, not an afterthought: define explicit logic for what happens when temperature is in-band but air quality is not (e.g., increase fresh-air ventilation even if that costs conditioning energy), and document the tradeoff you chose.
- No dynamic allocation; state carried between cycles must be a fixed-size struct owned by the caller (matching `docs/CODING_STANDARDS.md`'s no-`malloc` rule).
- Define hysteresis or deadband behavior to prevent actuator chatter (rapid on/off cycling) — state your thresholds and justify them.

### HVACOPT-03 — Actuator Driver
- Translate optimizer commands into HAL-level actuator calls (heat/cool stage, damper position or fan speed).
- Define safe defaults / fail-safe behavior if the optimizer layer stalls, faults, or produces an out-of-range command (e.g., hold last-known-safe state, or force a defined safe position — decide and document which).

### HVACOPT-04 — Modbus Telemetry Server
- Expose the full current state (all sensor readings, computed comfort/air-quality status, current actuator commands, active faults) as a documented Modbus register map (Section 7).
- Accept supervisory writes from a Modbus master: setpoint changes, mode overrides (e.g., manual ventilation override), and a defined "return to auto" command.
- Reject or safely clamp out-of-range writes (e.g., a setpoint outside a defined safe envelope) rather than applying them uncritically — this is a boundary/negative-test requirement, not just an implementation nicety.
- The Modbus layer must not directly mutate optimizer or driver state; it hands validated commands to the control layer through a defined interface (Layering Rule 3).

### HVACOPT-05 — Diagnostics & Fault Reporting
- Extend or reuse `diag_formatter`-style error codes for: sensor faults, actuator faults, and Modbus/comms faults, each in its own code range, following `docs/CODING_STANDARDS.md`'s existing range convention.
- Faults must be visible both locally (for the same kind of event-log/diagnostic retrieval built in Module 05) and over Modbus (a fault/status register or discrete-input bank), so a BMS operator does not need physical access to see why the zone is in a degraded mode.

### HVACOPT-06 — Device-Level State Machine (recommended)
- Model the zone controller's overall mode (e.g., `INIT → CALIBRATING → AUTO → MANUAL_OVERRIDE → FAULT → SAFE_SHUTDOWN`) as an explicit state machine, following the same discrete-event-driven pattern as `state_machine.c` in the sample repo, rather than encoding mode as scattered booleans.

---

## 6. Non-Functional Constraints

- **Memory:** No dynamic allocation (`malloc`/`free`) anywhere in the codebase — fixed-size buffers and caller-owned memory only, matching the programme-wide coding standard.
- **Timing:** Define and document your control-loop cadence (sensor poll → optimize → actuate) and your Modbus response-time budget. State what happens if a poll cycle overruns its budget.
- **Determinism:** The host build must be fully deterministic — no reliance on wall-clock time, real randomness, or real hardware for test runs, consistent with every HAL stub elsewhere in this programme.
- **Portability:** Business logic (optimizer, state machine, diagnostics) must not include transport-specific code (no Modbus framing, no UART/TCP calls) — only the HAL and driver layers touch transport or hardware specifics.
- **Toolchain:** Must build cleanly with `gcc`/`make` on the host, with warnings treated as build-blocking (`-Wall -Wextra`, at minimum), matching Module 05's build-validation checklist.

---

## 7. Modbus Interface Contract

Design and document your own concrete register map as part of your top-level architecture deliverable — the table below is a **starting skeleton**, not a fixed answer key. At minimum, your documented map must specify, per point: register type (coil / discrete input / holding register / input register), address, data type and scaling, unit, and read/write access.

| Point | Suggested Register Type | Suggested Data | Access | Notes |
|---|---|---|---|---|
| Zone temperature | Input Register | int16, ×0.1 °C | Read-only | Matches typical BMS scaling convention |
| Zone relative humidity | Input Register | uint16, ×0.1 %RH | Read-only | |
| Air quality (CO₂ or VOC index) | Input Register | uint16, raw units | Read-only | Document which proxy you chose and why |
| Temperature setpoint | Holding Register | int16, ×0.1 °C | Read/Write | Must be clamped to a documented safe envelope |
| Air-quality threshold | Holding Register | uint16 | Read/Write | Must be clamped |
| Heat/cool stage command | Holding Register or Coil(s) | enum or bitfield | Read/Write (supervisory override) | Normally driven by the optimizer; override is a documented exception path |
| Damper/fan command | Holding Register | uint16, 0–100% | Read/Write (supervisory override) | Same as above |
| Mode (AUTO/MANUAL/FAULT/…) | Holding Register | enum | Read/Write | Write triggers state-machine transition, not a silent flag flip |
| Active fault code | Input Register | uint16 | Read-only | Zero when no fault active |
| Fault discrete flags | Discrete Input | bitfield | Read-only | One bit per fault category (sensor/actuator/comms) |

Your documented register map is a contract deliverable — treat a change to it the same way Module 04 treats interface-contract changes: versioned, and reflected in your traceability matrix (Section 9).

---

## 8. Acceptance Criteria

- [ ] A documented top-level architecture (layer diagram + module responsibility table, in the style of `sample-repo/docs/ARCHITECTURE.md`) exists before implementation began.
- [ ] Each functional requirement in Section 5 has its own `spec.md`, reviewed `plan.md`, and ordered `tasks.md`.
- [ ] The full system builds cleanly (`make` / equivalent) with zero warnings under the flags stated in Section 6.
- [ ] The optimizer demonstrably changes ventilation/actuator behavior in response to air-quality input alone (temperature in-band, air quality out-of-band) — proven by a test, not just claimed.
- [ ] A simulated Modbus master can read every point in the documented register map and receives values consistent with the current sensor/optimizer state.
- [ ] A simulated Modbus master writing an out-of-envelope setpoint is rejected or clamped, and this is proven by a boundary test, not just implemented.
- [ ] A simulated sensor fault (stuck value, out-of-range, comms timeout) is detected, surfaces as a documented fault code both locally and over Modbus, and drives the zone to a documented safe actuator state.
- [ ] `make test` (or equivalent) is fully green, including unit, integration, and boundary/negative tests.
- [ ] A specification-to-test traceability matrix exists and every functional requirement maps to at least one test.
- [ ] At least one MCP tool, reusable skill, or agent-assisted workflow was used in building this project, and its use is documented (Section 10).
- [ ] A completed release-readiness self-audit (Module 05, Exercise 6 style) exists for the whole system.

---

## 9. Test Strategy Requirements

Following Module 06's framework, your test strategy document must cover:

1. **Traceability matrix** — every requirement ID from Section 5 mapped to the test(s) that prove it, per the Module 06 traceability workflow.
2. **Host-based unit tests** — for `sensor_poll`, `hvac_optimizer`, `actuator_driver`, and the register-map encode/decode logic, each in isolation.
3. **Mocks/stubs/fakes** — a mocked HAL (matching the sample repo's HAL-mocking pattern) and a **simulated Modbus master** capable of issuing reads and writes against your server for test purposes, so the telemetry layer is testable without real network hardware.
4. **Integration tests** — full control-loop cycles (sensor snapshot → optimizer decision → actuator command → register state) exercised end-to-end at the host level.
5. **Boundary and negative testing** — out-of-range sensor values, out-of-envelope Modbus writes, malformed/short Modbus frames, and a comms-timeout scenario must each have a dedicated test, per Module 06's boundary/negative testing framework.
6. **Regression protection** — if your capstone builds on or wires into any existing sample-repo module (e.g., reusing `diag_formatter` or `event_log` from Module 05), the existing test suite for that module must remain green and unmodified, per Module 05's minimal-change / regression-protection discipline.

---

## 10. Agentic/MCP Requirement

Per Module 07, incorporate at least one of the following into how you build this capstone, and document what you used, what it did, and where you drew the permission/context boundary:

- An MCP server/tool connected to an engineering resource relevant to this project (e.g., a register-map linter, a build/test runner exposed as a tool, or a documentation-lookup resource).
- A packaged, reusable skill (in the style of Module 07's Appendix A skill templates) for a repeatable task on this project — e.g., a "register-map-to-spec consistency checker" or a "traceability matrix auditor."
- A scoped Agent Mode task that respects this document's layering rules (Section 4) — the same way Module 02's Agent Mode exercise had to respect `sample-repo`'s layering rules to implement `fault_monitor` correctly.

State explicitly what tool access, permissions, and failure handling you configured (Module 07, Sections 6 and 8) — "I used Copilot" is not sufficient; name the boundary you set and why.

---

## 11. SDLC Deliverables & Repository Structure

Walk the same seven-stage SDLC as Module 05, once per functional requirement group, and once at the system level for integration:

| Stage | System-Level Artifact |
|---|---|
| Problem Framing | This document's Section 2, refined with your own written framing |
| Architecture & Interfaces | Layer diagram, module responsibility table, register map (Section 7) |
| Module Structure | Per-module `spec.md` / `plan.md` / `tasks.md` |
| Implementation | `include/`, `src/` per module |
| Test & Static Analysis | `tests/`, static-analysis output, traceability matrix |
| Build & CI/CD | `Makefile` (or equivalent), reproducible clean build |
| Release Readiness | Release-readiness self-audit (Section 8 acceptance criteria, signed off) |

Suggested repository layout (mirrors `labs/module-02/sample-repo`):

```
capstone/hvac-edge-optimizer/
├── docs/
│   ├── ARCHITECTURE.md
│   ├── CODING_STANDARDS.md
│   ├── REGISTER_MAP.md
│   └── TRACEABILITY.md
├── include/
│   ├── hal_adc.h, hal_gpio.h, hal_uart.h (or hal_tcp.h)
│   ├── sensor_poll.h, actuator_driver.h
│   ├── hvac_optimizer.h, state_machine.h
│   ├── modbus_server.h, telemetry_bridge.h
│   └── diag_formatter.h
├── src/
│   └── (matching .c files)
├── tests/
│   ├── minitest.h (or your chosen host-test framework)
│   ├── test_sensor_poll.c
│   ├── test_hvac_optimizer.c
│   ├── test_modbus_server.c
│   └── test_integration_control_loop.c
├── specs/
│   ├── HVACOPT-01-sensor-acquisition/{spec,plan,tasks}.md
│   ├── HVACOPT-02-optimizer/{spec,plan,tasks}.md
│   └── ... (one folder per requirement)
└── Makefile
```

---

## 12. Suggested Milestones

Adapt to your actual cohort schedule; the sequencing dependency matters more than the exact durations.

| Milestone | Focus | Depends On |
|---|---|---|
| M1 | Problem framing, architecture, register-map draft | Section 2, Section 4 |
| M2 | HAL + driver layer (sensors, actuators), host-buildable and unit-tested | M1 |
| M3 | Optimizer logic (HVACOPT-02) + state machine (HVACOPT-06), unit-tested | M2 |
| M4 | Modbus server + telemetry bridge (HVACOPT-04), simulated-master tests | M2, M3 |
| M5 | Diagnostics/fault reporting wired through all layers (HVACOPT-05) | M2–M4 |
| M6 | Full integration tests, boundary/negative tests, traceability matrix complete | M1–M5 |
| M7 | Release-readiness self-audit, MCP/agentic documentation, final review | M1–M6 |

---

## 13. Grading Rubric / Definition of Done

| Dimension | Evidence Expected |
|---|---|
| Spec-driven discipline | Every requirement has a spec/plan/tasks chain written *before* its implementation existed |
| Architecture integrity | No layering-rule violation anywhere in the final codebase (Section 4) |
| Functional completeness | All of Section 5 implemented and demonstrable |
| Protocol correctness | Register map matches documentation exactly; boundary writes proven rejected/clamped |
| Test rigor | Traceability matrix complete; unit + integration + boundary/negative tests all green |
| Fault handling | At least one injected fault (sensor, actuator, comms) proven caught and safely handled |
| Agentic practice | MCP/skill/agent usage documented with explicit permission/context reasoning |
| Release judgment | Self-audit completed honestly, including any known gaps or deferred items |

A capstone that is functionally complete but skips the spec/plan/tasks trail, the traceability matrix, or the release-readiness self-audit is **not** done — process evidence is graded as seriously as working code, consistent with every prior module in this programme.

---

## 14. Stretch Goals (Optional)

- Multi-zone support (N independent optimizer instances behind one Modbus server, addressed by a zone-index scheme in the register map).
- A second telemetry transport (e.g., MQTT bridge alongside Modbus) sharing the same telemetry-bridge abstraction, proving the integration layer's transport-agnostic design.
- Basic security hardening for the Modbus TCP path (source-address allowlisting, write-command rate limiting) with a documented threat model.
- An auto-tuning or adaptive comfort/air-quality tradeoff strategy (e.g., adjusting the ventilation-vs-conditioning tradeoff based on recent occupancy patterns), clearly separated from the baseline optimizer so the baseline remains independently testable.

---

## 15. Dependencies and References

- `labs/module-02/sample-repo/` — reusable HAL stubs, coding standards, and architecture conventions this capstone extends.
- `guides/Module04_Spec_Driven_Development.md` — spec/plan/tasks template and traceability matrix conventions used throughout this document.
- `guides/Module05_Embedded_Development_SDLC.md` — seven-stage SDLC, repository archaeology, and release-readiness checklist.
- `guides/Module06_Embedded_Test_Strategy_Validation.md` — test pyramid, mocking, boundary/negative testing, and traceability workflow.
- `guides/Module07_MCP_Agentic_Engineering.md` — MCP architecture, tool governance, and reusable skill patterns.
