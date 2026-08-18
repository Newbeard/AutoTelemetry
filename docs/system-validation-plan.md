# AutoTelemetry system validation plan

Status: Task 5A.1 conditional prototype/capture re-freeze, 2026-08-17. This is a controlled plan for future evidence, not a record of executed tests, measurements, certification or regulatory compliance.

## Scope and evidence rules

This plan defines the sequence, observability and record structure for AutoTelemetry prototype validation. Later tasks may add exact fixtures, procedures, instrument settings, expected results, acceptance limits and measured evidence without weakening these rules.

- Each stage complements earlier stages; a later stage does not replace deterministic earlier evidence.
- Passing a bench test does not imply passing stationary-vehicle, road or track testing.
- Passing functional tests does not establish EMC, transient, environmental or regulatory compliance.
- Add numerical PASS/FAIL degradation limits only when a component requirement, measurement capability or prototype baseline supports them.
- Every result identifies hardware revision, firmware version, profile/configuration, fixture revision, instruments, environment, procedure, raw evidence and reviewer.
- The authoritative Task 5A.1 power partition, numerical requirements, assumptions, and conditional selections are controlled by [task5a1-power-architecture.md](task5a1-power-architecture.md). Passing internal gates below does not establish automotive qualification or regulatory compliance.

## Validation flow

~~~text
HOST / SOFTWARE
  -> BENCH
  -> VEHICLE — STATIONARY
  -> ROAD
  -> TRACK / SLALOM STRESS
  -> PROFESSIONAL EMC / PRE-COMPLIANCE / TRANSIENT TESTING
~~~

### Phase 1 — host and software

Validate deterministic logic without target hardware where practical:

- parsers, scaling, units, validity and source arbitration;
- vehicle-profile schema, matching and recorded CAN fixtures;
- Generic OBD-II, ISO-TP/UDS state machines and bounded scheduling;
- configuration migration, interrupted writes, rollback and safe defaults;
- RaceChrono/device-protocol packet construction;
- logger format, backpressure and fault behavior;
- display, shift-light and alarm state logic;
- diagnostic-counter accounting and reset/rollover behavior;
- fault isolation with injected timeouts, queue saturation and task/service failures.

### Phase 2 — bench

Validate the assembled prototype with controlled power, deterministic CAN/OBD stimuli and instrumented outputs:

- every power rail, current state, startup/shutdown and load step;
- parked current and sleep/wake;
- vehicle-only, USB-only, both-source and unpowered behavior;
- physical CAN, Generic OBD, diagnostic scheduling and recovery;
- GNSS digital operation and live antenna reception/replay where appropriate;
- BLE, configuration-mode Wi-Fi and supported coexistence combinations;
- display, ten-pixel shift-light, sounder and SD;
- programmed faults and cross-subsystem isolation;
- individual A/B interference tests and FULL_LOAD_INTERFERENCE_TEST.

### Phase 3 — vehicle, stationary

Validate OBD attachment, actual vehicle grounds, startup, engine start/crank, charging voltage, shutdown, CAN wake, explicitly authorized diagnostics, accessory/ignition noise, ECU coexistence and other OBD/CAN devices where practical.

### Phase 4 — road

Validate continuous CAN/OBD acquisition, GNSS update continuity, BLE/RaceChrono, display, shift-light, logging, sleep/wake transitions, long-duration operation, vibration, temperature and recorded fault/recovery evidence.

### Phase 5 — track or slalom stress

Exercise high CAN traffic and engine speed, shift-light activity, target-rate GNSS, BLE telemetry, SD logging, display updates, alarms/sounder, elevated temperature and vibration simultaneously. Road testing does not substitute for this combined workload.

### Phase 6 — professional validation

Before a serious commercial release, define the applicable product/market plan and obtain qualified evidence for conducted transients, ESD, radiated/conducted emissions, immunity and relevant thermal/environmental/vibration testing. Internal tests must never be represented as CISPR 25, ISO 7637, ISO 10605, UNECE R10 or automotive-qualification evidence.

## Conceptual bench

~~~text
development workstation
  |-- USB --------------------> AutoTelemetry under test
  |-- USB-CAN / OBD emulator -> protected physical CAN bench
  |-- CAN/GNSS replay -------> deterministic inputs
  |-- automated host tests --> sequencing and evidence collection
  |
  +-- attached DUT loads
      |-- GNSS antenna or approved replay fixture
      |-- display
      |-- ten-pixel flexible shift-light
      |-- sounder
      +-- SD card

current-limited supply and measurement equipment observe power and signals
~~~

Bench scenarios should be deterministic and repeatable where practical. The harness documents termination, common ground/isolation, source limits, power sequence and emergency disconnect.

## Task 5A.1 power validation gates

All gates below require retained waveforms, temperatures, currents, fixture configuration, component lots, preconditions, and PASS/FAIL disposition. They apply before the conditional power architecture can be considered prototype-proven:

1. **Exact transient profiles:** reproduce every frozen positive, negative, sustained, and interrupted supply case with its specified amplitude, duration, source impedance, repetition, rise/fall behavior, polarity, temperature, and source-state precondition. Include +26 V/60 s, the exact +38 V suppressed-load-dump source, −14 V/60 s, the +50 V/2 Ω/0.05 ms reference, the −100 V/10 Ω/2 ms reference, and every additional purchased-standard/OEM pulse later controlled by [task5a1-power-architecture.md](task5a1-power-architecture.md). Record both raw input and every affected protected/regulated node.
2. **TVS electrothermal and fuse coordination:** validate the common-anode `SM15T47AY` + `SM15T33AY` pair for both polarities, including dynamic clamp voltage, forward-leg stress, current and energy sharing, junction-temperature accumulation, cooling/repetition, leakage, and fault aftermath. Coordinate the 2 A fuse using tolerance- and temperature-bounded current, I²t, clearing time, and DC interrupt capability; nominal I²t alone is insufficient.
3. **Controller/MOSFET VDS, SOA, and overshoot:** measure LM74720QDRRRQ1 A, C, C-to-A, PD, both gates, and both `STL125N10F8AG` devices during normal operation, +26 V, +38 V, +50 V, −14 V, −100 V, cutoff, source crossover, output-precharge/held-output cases, shorts, and recovery. Bound each VDS/VGS, avalanche exposure, transient SOA, dissipation, ringing and layout-induced overshoot with temperature and part tolerances; prove downstream survival through 25.531 V plus measured overshoot.
4. **OV, inrush, and reverse-current blocking:** demonstrate no OV nuisance trip through 18 V, the rising OV/PD-low command by 25.531 V before 26 V, and reconnect eligibility by 18 V across tolerance and temperature. Separately measure completed isolation/reconnection time, dynamic downstream peak, startup/hot-plug inrush, dV/dt, fuse stress, output discharge/recovery, reverse-battery isolation, and reverse current/back-feed with the output energized from USB.
5. **Filter impedance, EMI, and DC-bias characterization:** prove the combined direct capacitance remains 9.4–20.68 µF over the controlled 0–25.6 V envelope, then measure or correlate damped-filter impedance across the relevant frequency range and operating states. Include capacitance versus DC bias/temperature/tolerance/aging, ESR/ESL, inductor DCR and saturation, damping-resistor pulse stress, converter negative input impedance/stability, hot plug, conducted/radiated noise observations, and source/harness impedance. This is engineering evidence, not an EMC-compliance test.
6. **Four-state source injection:** exercise vehicle-only, USB-only, both sources, and neither source through every order while observing TPS2553/TPS62162/TPS2116, `MAIN_3V3`, vehicle-only domains and connector pins. Prove no back-feed, CAN/AUX5 power or source chatter. Test the exact USB population, including 60.4 kΩ RILIM, both TPS62162 capacitors/inductor and the 100 µF TPS2116 VOUT bulk. Require a prototype source/cable that advertises and sustains ≥500 mA at 4.75 V; verify post-ramp `USB_ENUM` ≤100 mA `MAIN_3V3` (about 82 mA VBUS), 350 mA VBUS/400 mA MAIN_3V3 configured ceilings, limiter tolerance, current-limited startup, clamp/RCB behavior, reverse leakage, 105 °C limitation and every handoff/load combination. Do not treat this test as proof of generic legacy USB 2.0 pre-enumeration compliance.
7. **Hot parked-current budget:** measure complete input current across the parked voltage and temperature matrix, including controller, regulators, mux, CAN standby, MCU sleep, monitors, divider/filter leakage, protection leakage and disabled branches. Compare against the 408.026 µA (0.408 mA) 12 V paper envelope and the 0.476/0.438/0.419/0.408/0.402/0.400 mA 6/8/10/12/14.4/18 V model; verify the <1.0 mA release limit and separately report the <0.50 mA at 12 V/25 °C stretch target.
8. **Rail thermal and load steps:** validate the complete MC3/`VEH_3V3` bucket and conditional AUX5 at 12/14.4/18 V, 85 °C ambient, STREET/TRACK/MAX duty, startup, load release and simultaneous permitted peaks using the exact shared `XGL5030-222MEC`, CIN/COUT/CVCC and CBOOT-DNP population. Separately exercise the 1.050 A 3.3 V-display MC3 capacity case and decide whether a 105 °C enclosure requirement is needed. Measure effective capacitance, efficiency, dropout, droop/overshoot, ripple, stability, component/junction/board temperatures, recovery, sequencing, current limiting and branch faults. Use 8.27505 W only for the simultaneous aggregate; never add the mutually exclusive display cases.
9. **Low-voltage state transitions:** prove MAX load is prohibited below 10.0 V, logger flush is requested below 9.5 V, optional loads shed only after less than 9.0 V persists for 100 ms, controlled core/CAN/wake survives at 6 V, and shed loads recover only after greater than 10.0 V persists for 2 s. Sweep and step through thresholds with ripple/noise and source handover while checking timer tolerance, hysteresis, SD integrity, CAN behavior, reset loops and deterministic recovery. Exact timer/tolerance implementation remains a design requirement pending measurement.

Failure or inconclusive evidence at any gate reopens the affected selection; it does not authorize an undocumented threshold, component substitution, or broader operating claim.

## FULL_LOAD_INTERFERENCE_TEST

FULL_LOAD_INTERFERENCE_TEST is a mandatory prototype/release validation gate. At minimum operate simultaneously:

- ten-pixel shift-light at maximum permitted brightness;
- aggressive full-strip transitions or repeated flash pattern;
- sounder at maximum configured TRACK output;
- continuous SD writes;
- active display updates;
- BLE telemetry streaming;
- GNSS at the target high update rate;
- representative CAN traffic;
- diagnostic polling when enabled and safe for the fixture.

Run configuration-mode Wi-Fi traffic separately and, where technically meaningful, as an additional combined case. Wi-Fi remains normally disabled outside its intended workflow.

Monitor and retain evidence for:

| Domain | Required observations |
|---|---|
| Power | Raw/protected input, MAIN_3V3, AUX5, GNSS rail and SHIFT5 droop, overshoot, ripple, ringing, oscillation and recovery |
| System | Reset reason, brownout, watchdog, task restart, starvation, memory and queue/backpressure events |
| CAN | Frames, drops/overflow, errors, TX failures, bus-off, recovery and diagnostic timeouts |
| GNSS | Fix state, satellites, C/N0 summary, accuracy, continuity, lost fix, reacquisition, gaps and UART/parser errors |
| BLE/Wi-Fi | Connections, disconnects, reconnects, send failures, configuration/OTA failures and recovery |
| Storage/display | SD mount/write/backpressure errors, dropped log samples and display corruption |

Passing means meeting the later revision-specific acceptance criteria with no unexplained event. This document intentionally does not invent final ripple, RF-degradation or packet-loss limits.

## Controlled A/B interference method

Use a stable baseline, then change one stressor at a time before the combined case:

~~~text
BASELINE
  -> shift-light steady
  -> shift-light worst-case flashing
  -> sounder maximum
  -> SD continuous write
  -> display high activity
  -> BLE high activity
  -> Wi-Fi active
  -> FULL_LOAD_INTERFERENCE_TEST
~~~

Keep power source, antenna, fixture, configuration, software build, sampling interval and environment controlled. Compare measured behavior against baseline; do not record only “device still works.”

## GNSS and RF coexistence validation

For baseline, each individual stressor and the combined test, collect where available:

- fix state and satellite count;
- per-satellite or summarized C/N0 statistics;
- reported position/velocity accuracy;
- navigation-update continuity and gaps;
- fix-loss and reacquisition counts;
- UART/parser errors.

Test GNSS with BLE, with Wi-Fi, with BLE plus Wi-Fi where supported/meaningful, and with full RF/peripheral load. Replay establishes digital behavior only; it does not establish RF immunity or antenna performance.

## Shift-light interference validation

Use the frozen functional concept: ten addressable RGB LEDs, flexible remote module, SHIFT5, SHIFT_DATA_5V, GND, cable no longer than 0.5 m, software brightness control and approximately 0.50 A qualified output envelope.

Exercise all-off, normal progression, maximum permitted brightness, rapid full-strip transitions, worst supported RGB pattern and repeated flashing. Observe SHIFT5, AUX5, MAIN_3V3, accessible GNSS power, shift data integrity, CAN, GNSS and BLE to detect conducted and radiated coupling.

## Sounder interference validation

Exercise low volume, normal STREET volume, maximum TRACK volume, representative tones/frequencies and rapid alarm patterns. Observe AUX5, MAIN_3V3, GNSS, CAN, BLE, supply modulation, resets and diagnostic events.

## Power-integrity measurement

Use oscilloscope-based validation on raw/protected vehicle input, both regulated inputs to the TPS2116 core mux, the selected core rail, vehicle-only CAN supply, AUX5, GNSS rail, SHIFT5 and other sensitive switched rails where useful during startup, shutdown, source transitions, power cycling, load steps, shift-light/sounder activation, SD writes, BLE/Wi-Fi activity and combined stress.

Look for droop, overshoot, ringing, excessive ripple, oscillation, slow recovery and cross-domain coupling. Exact limits remain tied to later component, interface and prototype requirements.

## CAN validation

Future tests monitor received/transmitted frames, drops/overflow, controller error counters where available, TX failures, bus-off, recovery and diagnostic timeouts. Exercise CAN while high-current/noisy subsystems are active. Where practical, inspect CAN-H/CAN-L waveforms on the protected bench without adding harmful stubs.

## Runtime diagnostic and health metrics

The exact representation is deferred. Non-trivial subsystems reserve useful counters/state:

| Subsystem | Reserved metrics |
|---|---|
| CAN/diagnostics | RX/TX frames, dropped/overflow frames, error count/state, TX failures, bus-off, recovery and diagnostic request timeouts |
| GNSS | Fix state, satellites, C/N0 summary where available, fix loss, reacquisition, parser/UART errors and update gaps |
| BLE | Connection, disconnect and reconnect counts; telemetry-send failures/drops |
| Wi-Fi | Configuration-session count, unexpected disconnects and relevant OTA/configuration failures |
| SD/logger | Write errors, mount failures, queue overflow/backpressure and dropped log samples |
| System | Uptime, reset reason, detectable brownouts, watchdog events, service/task restart/recovery, useful minimum/free memory and queue overflow/backpressure |

Metrics have defined meaning and rollover/reset behavior. Do not collect a metric solely because an API exposes it.

## Diagnostic access

Development/service diagnostics may be exposed through serial/debug, an expert Web UI page, the first-party Device Protocol, a future AutoTelemetry application and/or an SD diagnostic log. Normal dashboards expose only actionable user health; engineering counters remain in a clearly separated expert/service view.

## Fault-isolation validation

Future firmware and bench tests prove:

- GNSS failure does not stop CAN telemetry;
- BLE failure does not stop display, shift-light or logging;
- display failure does not stop RaceChrono telemetry;
- SD failure does not stop acquisition or other outputs;
- Wi-Fi/configuration failure does not stop normal telemetry mode;
- optional Tire Module absence/failure does not stop the main unit;
- noisy shift-light or sound operation does not destabilize the core device.

## Emulator, replay and future HIL

### Magnus OBD emulator

MagnusThome's ESP32_OBD2_Emulator remains an INITIAL PHYSICAL-CAN SMOKE-TEST REFERENCE / DEVELOPMENT TOOL concept for supported-PID discovery, RPM, speed, basic Generic OBD values and communication smoke tests. Its unlicensed source is not integrated or copied. It is not a complete ECU qualification environment.

### CAN replay

Recorded CAN traffic is a first-class regression mechanism:

~~~text
recorded vehicle fixture
  -> replay / decoder test
  -> vehicle profile
  -> normalized telemetry core
  -> expected channels and values
~~~

BMW E81/N43 is the first real vehicle fixture, not a special-case architecture. Every future supported vehicle can add provenance-controlled fixtures and expected outputs under the same schema.

### Future HIL

Reserve a workstation-controlled bench combining USB control/logging, USB-CAN or ECU emulation, GNSS replay/simulation, controllable power, automated tests and measurement equipment. Future automation may control stimuli, power cycling, sequencing and evidence collection. This task selects or purchases no HIL equipment.

## Future Tire Module validation

The optional AutoTelemetry Tire Module remains a separate future product outside the Telemetry v1 main PCB. No second CAN controller, tire RF receiver or other main-board hardware is added solely for it.

Future tests cover missing module/node, stale pressure, stale temperature, invalid sensor, RF/link loss, module reboot, main-unit reboot, data recovery and RaceChrono forwarding. All failure modes remain isolated from core CAN/GNSS telemetry and other outputs.

## EMI / GNSS / POWER PLACEMENT REVIEW

Before final PCB routing or manufacturing release, a dedicated review considers:

- GNSS module, RF path, U.FL and antenna cable;
- ESP32 antenna and keep-out;
- DC/DC converters, inductors and switching nodes;
- CAN transceiver and connector path;
- USB and microSD;
- display and shift-light cables;
- sound amplifier;
- high-current return paths and ground-plane continuity;
- cable common-mode radiation risk.

The review records placement rationale, keep-outs, return paths, cable exits, coupling risks and unresolved validation needs. It occurs before routing/release, not only after prototype failure.

## Mechanical EMC implications

Future mechanical review considers GNSS antenna-cable routing, shift-light/display/USB cable exits, distance from noisy wiring, enclosure geometry and any later justified grounding/shielding provisions. No CAD is created by this plan.

## Instrumentation categories

Useful future categories include a current-limited laboratory supply, quality multimeter, oscilloscope, logic analyzer, CAN interface/emulator, GNSS monitoring/logging, optional near-field probes and optional spectrum analyzer. Exact equipment selection or purchasing is outside this task.

## Professional validation boundary

Bench, vehicle, road and track testing are engineering evidence only. They do not prove CISPR 25 compliance, ISO 7637 compliance, ISO 10605 compliance, UNECE R10 approval or automotive qualification. A serious commercial release must evaluate appropriate qualified conducted-transient, ESD, emissions, immunity, thermal, environmental and vibration testing for the target product and market.

## Test record template

| Field | Required content |
|---|---|
| Identity | Test ID/name, stage, date, operator/reviewer |
| Product | Hardware revision/serial, firmware build, bootloader, profile/configuration and peripheral versions |
| Fixture | Harness/emulator/replay/HIL revision, fixture manifest/hash and licenses/provenance |
| Environment | Supply settings, temperature, vehicle state, antenna/cables and relevant installation geometry |
| Instrumentation | Instrument/model, channel/probe connection, calibration status and acquisition settings |
| Procedure | Preconditions, ordered actions, duration, injected faults and safety limits |
| Criteria | Evidence-backed expected behavior and PASS/FAIL limits, or explicitly TBD |
| Results | Raw logs/waveforms/captures, summary, diagnostic counters and anomalies |
| Disposition | PASS, FAIL or INCONCLUSIVE; issue links, retest need and approvals |

## Open refinement items

- Define revision-specific acceptance thresholds only after measurement capability and baseline data exist.
- Define project-owned emulator-scenario and replay-manifest schemas.
- Define bench harness safety, termination and instrumentation procedures.
- Establish legal storage/redaction policy for BMW and future vehicle captures.
- Select professional laboratory scope only after target market and release intent are known.
