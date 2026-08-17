# ChatGPT engineering review packet

## Task completed

Completed Task 5A-DOC — System Validation, Interference Testing and Diagnostics Architecture Amendment. Created the permanent staged system-validation plan; made FULL_LOAD_INTERFERENCE_TEST, useful runtime diagnostics, deterministic replay/emulation testability and the EMI/GNSS/power placement review permanent gates; synchronized product test, requirements, architecture, reference-tool and user-I/O documentation.

No test was executed and no measurement, PASS/FAIL limit or compliance result was invented. No Task 5A power blocker was solved or changed. No KiCad, PCB, firmware, CAD or manufacturing file was created or modified.

## Files changed

- Created docs/system-validation-plan.md.
- Updated CODEX.md.
- Updated docs/test-architecture.md.
- Updated docs/requirements.md.
- Updated docs/architecture.md.
- Updated docs/test-reference-architecture.md.
- Updated docs/user-io-configuration.md.
- Updated docs/chatgpt-review.md.

## Engineering findings

- Product validation now progresses through host/software, bench, stationary vehicle, road, track/slalom stress and professional testing where applicable. Later stages complement rather than replace earlier evidence.
- FULL_LOAD_INTERFERENCE_TEST is a mandatory prototype/release gate. It combines maximum permitted ten-pixel shift-light stress, TRACK sounder, continuous SD writes, active display, BLE streaming, target-rate GNSS, representative CAN and diagnostic polling when enabled/safe. Wi-Fi is tested separately and in combination where meaningful.
- Controlled A/B testing compares each likely noise/current source and the combined case against a stable baseline. Required observations include power integrity, resets/watchdogs, CAN errors/drops/bus-off, GNSS quality/continuity, radio connections, storage/display errors and runtime queue/task health.
- GNSS interference evidence reserves fix state, satellite count, C/N0, reported accuracy, continuity, fix loss/reacquisition, gaps and UART/parser health without inventing final degradation limits.
- Runtime diagnostics reserve meaningful CAN/diagnostic, GNSS, BLE, Wi-Fi, SD/logger and system health counters/state. Engineering metrics remain separate from the normal user dashboard.
- MagnusThome/ESP32_OBD2_Emulator remains an initial physical-CAN smoke-test reference/development-tool concept only. Its source was not integrated, and it does not cover malformed traffic, timeouts, ISO-TP, UDS, multiple ECUs or negative responses.
- Recorded CAN traffic is a first-class regression input. BMW E81/N43 is the first real fixture but does not create special-case core logic; future vehicles use the same profile/replay contracts.
- Future HIL remains a reserved workstation/CAN/GNSS/power/automation architecture. No equipment was selected or purchased.
- The optional Tire Module remains outside Telemetry v1 main-PCB scope. Its absence, stale/invalid data, link loss and reboot must be isolated from the main unit; no second CAN controller or Tire RF hardware was added.
- An EMI / GNSS / POWER PLACEMENT REVIEW is mandatory before final PCB routing or manufacturing release and covers RF paths/keep-outs, converters, switch nodes, buses, storage, cable exits, high-current returns, ground continuity and common-mode radiation risk.
- Bench, vehicle, road and track evidence is not regulatory compliance or automotive qualification.

## Decisions made

- Make the staged validation sequence permanent.
- Make FULL_LOAD_INTERFERENCE_TEST and controlled baseline/A/B evidence mandatory.
- Reserve useful runtime health metrics and expert/service diagnostic access.
- Preserve replay, emulation and future HIL as first-class test seams.
- Preserve Tire Module isolation and the one-CAN Telemetry v1 boundary.
- Add the EMI/GNSS/power placement gate before final routing/manufacturing release.
- Keep all Task 5A vehicle-input, TVS/FET/fuse, AUX5 and filter decisions unresolved.

## Assumptions

- Exact instruments, fixtures and automation equipment will be selected by later implementation tasks.
- Revision-specific ripple, GNSS-degradation, packet-loss and thermal acceptance limits require measurement capability and baseline evidence.
- Track/slalom validation is performed only under an approved safety procedure.
- Professional laboratory scope depends on target market and seriousness of the intended release.
- Diagnostic counters are exposed only where their semantics and resource cost justify them.

## Uncertainties / unresolved questions

- Exact test procedures, durations, sample rates, instrument settings and acceptance limits.
- Bench harness termination, grounding/isolation, source limits and emergency-disconnect procedure.
- Project-owned emulator scenario and replay-manifest schemas.
- Storage, privacy, permission and redaction rules for BMW and future vehicle captures.
- Runtime diagnostic data types, retention, rollover/reset semantics, resource budgets and protocol representation.
- Exact expert/service UI authorization and visibility policy.
- HIL equipment, GNSS simulation method and automation framework.
- Applicable professional transient/EMC/environmental laboratory scope for the final product/market.
- Every Task 5A electrical blocker listed in task5a-power-calculations.md.

## Risks

- A prototype can pass isolated functional tests while failing under simultaneous high-current/RF/storage load.
- Switching, LED, audio, SD, display or radio activity can degrade GNSS, CAN or supply integrity without obvious total failure.
- Missing counters can hide frame loss, bus-off recovery, GNSS gaps, storage backpressure or watchdog/service restarts.
- Vehicle-only testing can be non-repeatable and can miss deterministic edge cases.
- Bench/road evidence may be overstated as compliance if the professional-validation boundary is ignored.
- Poor fixture provenance or capture handling can create legal/privacy exposure or non-reproducible vehicle support.
- Deferring the placement review until after routing can make EMI/GNSS corrections expensive or impractical.

## GPIO / peripheral changes

| Resource | Result |
|---|---|
| GPIO, CAN controller, radio, GNSS, storage and power peripherals | No allocation or hardware change |

## Power / CAN / RF impact

- Automotive power: no component, threshold, regulator, filter or requirement resolution. Existing Task 5A blockers remain explicitly reopened.
- CAN: no hardware or transmit-policy change. Validation now requires counters, physical-bench observation, emulator smoke tests, replay regression, coexistence and combined-load testing.
- GNSS/RF: no hardware change. Baseline/A/B C/N0 and fix-continuity evidence plus pre-routing placement review are now required.
- USB: no hardware change. Bench validation includes vehicle-only, USB-only, both-source and unpowered states.
- ESP32 boot/strapping: no change.

## Datasheets / primary sources consulted

- No new component selection or numerical hardware decision was made.
- Repository-controlled CODEX.md, test-architecture.md, test-reference-architecture.md, architecture.md, requirements.md and the Task 5A calculation/blocker record were the authoritative inputs.
- The previously audited MagnusThome/ESP32_OBD2_Emulator record and its no-license limitation were preserved; no source was copied or executed.
- Previously catalogued CISPR 25, ISO 7637, ISO 10605 and UNECE R10 boundaries were used only to prohibit unsupported compliance claims.

## Recommended next step

Resolve the existing Task 5A vehicle-input and power decision set before resuming KiCad capture.

## STOP condition

Task 5A-DOC is complete. Stop before power-correction work, KiCad capture, firmware implementation, PCB layout, CAD, manufacturing generation, equipment purchasing or physical testing.
