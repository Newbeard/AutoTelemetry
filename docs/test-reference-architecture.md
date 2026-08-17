# Test and reference architecture

Status: Task 5A-DOC validation amendment, 2026-08-17. No third-party code was copied, linked, vendored or executed. Repository activity is a review snapshot and must be rechecked before use.

## Purpose

Testing progresses from deterministic host behavior to physical CAN and finally hardware-in-loop. A vehicle is not the first test fixture.

```text
host unit/config/profile tests
          |
CAN and GNSS replay + deterministic time
          |
virtual/physical ECU or OBD emulator
          |
AutoTelemetry bench integration
          |
hardware-in-loop fault/power/radio tests
          |
controlled stationary-vehicle validation
          |
road validation
          |
track/slalom stress
          |
professional validation where applicable
```

Each layer has versioned inputs, expected outputs and failure cases. Passing a higher layer does not replace the lower deterministic evidence.

## Reference-project record

| Project | Purpose and activity snapshot | License / classification | Reuse recommendation and limitations |
|---|---|---|---|
| [MagnusThome/ESP32_OBD2_Emulator](https://github.com/MagnusThome/ESP32_OBD2_Emulator) | Four-commit Arduino sketch replying to OBD requests with dummy data; special handling for RPM/speed and a one-byte ramp for most other PIDs | No license file/designation found. `REFERENCE_ONLY`; potential independently implemented `DEVELOPMENT_TOOL` concept | Useful as an initial physical-CAN smoke-test concept only. It explicitly lacks correct multi-byte/multi-packet behavior and does not cover deterministic malformed, timeout, negative-response, ISO-TP or UDS cases. Do not copy or extend its code without permission. |
| [MagnusThome/esp32_obd2](https://github.com/MagnusThome/esp32_obd2) | 35-commit Arduino OBD-II read library, rewritten from sandeepmistry's library around collin80 CAN dependencies | MIT; currently `REFERENCE_ONLY`, possible future `LIBRARY_DEPENDENCY` only after review | Preserve MIT notice if reused. Useful API/behavior reference, but Arduino/legacy CAN dependency, scheduler ownership, ESP-IDF v6 fit, bounded memory, tests, ISO-TP/UDS and coexistence need review. It may not bypass AutoTelemetry's diagnostic scheduler. |
| [MagnusThome/ESP32S3RET](https://github.com/MagnusThome/ESP32S3RET) | 36-commit S3 fork of ESP32RET for CAN reverse engineering, SavvyCAN/GVRET, LAWICEL and ELM327-style work; README calls it a quick hack, removes Classic Bluetooth and comments FastLED | MIT; `DEVELOPMENT_TOOL` / `REFERENCE_ONLY` | Retain as a lab-tool reference, not product firmware. Old Arduino IDE/dependencies, incomplete I/O and project-specific hacks require isolation and validation. Preserve MIT notice if any future reuse is approved. |
| Local RejsaCAN v6.x C6 dual-CAN self-test | ESP-IDF example instantiating/exercising both ESP32-C6 TWAI controllers; present under `Code Examples/RejsaCAN v6.x - ESP32-C6 - dual CAN - Self tester/` | No separate license and RejsaCAN root license remains unclear; `REFERENCE_ONLY` | Evidence that Magnus's C6 board/example exercises dual TWAI. Do not copy into product firmware. It does not prove S3 resource equivalence, product robustness, transceiver protection or a need for dual CAN. |
| [Autosport Labs ESP32-CAN-X2](https://github.com/autosportlabs/ESP32-CAN-X2) | 74-commit examples/hardware support for S3 native TWAI plus MCP2515 SPI controller; official wiki documents 10 MHz SPI and IRQ | No root license visible in the reviewed repository; `REFERENCE_ONLY` | Mature topology reference for external second CAN. Do not copy examples/CAD pending license clarification. Demonstrates the SPI/GPIO/controller/transceiver cost that Telemetry v1 avoids. |
| [lbenthins/ecu-simulator](https://github.com/lbenthins/ecu-simulator) | Linux/SocketCAN ECU simulator for OBD-II and UDS over ISO-TP, including physical/functional OBD addressing | MIT; candidate `DEVELOPMENT_TOOL` | Best current open-source candidate for deterministic virtual/physical ECU tests. Pin exact commit and dependencies; its older Raspbian/kernel ISO-TP setup, root/setup scripts, normal addressing only and maintenance state require sandboxed evaluation before adoption. |
| [Ircama/ELM327-emulator](https://github.com/ircama/ELM327-emulator) | Active Python multi-ECU ELM327 emulator with stateless OBD and stateful UDS/ISO-TP; v3.0.5 was released 2025-12-12 | CC BY-NC-SA 4.0; `DEVELOPMENT_TOOL` evaluation only | Technically more capable but non-commercial restriction is incompatible with casual product integration/distribution. It emulates an ELM/client-facing path rather than necessarily being the physical CAN ECU fixture. Keep isolated and obtain legal approval before business use. |
| [limiter121/esp32-obd2-emulator](https://github.com/limiter121/esp32-obd2-emulator) | Wi-Fi-controlled ESP32/CAN OBD-II emulator; repository is archived and last pushed in 2018 | MPL-2.0; `REFERENCE_ONLY` | Historical comparison only. File-level MPL obligations and inactivity make it a poor baseline. |
| [mdabrowski1990/uds](https://github.com/mdabrowski1990/uds) | Python UDS client/server simulation and monitoring abstraction | MIT; possible `DEVELOPMENT_TOOL` | Useful server-side protocol reference/host fixture after exact-version and maintenance review. It does not itself qualify physical CAN timing or AutoTelemetry behavior. |
| [SavvyCAN](https://github.com/collin80/SavvyCAN) | CAN capture, visualization, DBC and replay engineering tool | MIT with bundled third-party notices; `DEVELOPMENT_TOOL` | Retain for manual lab analysis and capture conversion. Automated acceptance still requires project-owned deterministic fixtures and scripts. |

Absence of a license is not permission. Classification records intended use, not a conclusion that the project is safe, supported, or legally approved.

## OBD emulator role

Magnus's simple emulator is classified as `INITIAL PHYSICAL-CAN SMOKE-TEST REFERENCE / DEVELOPMENT TOOL`. Its bounded purpose is to prove that a bench node can receive a functional request and return plausible supported-PID/RPM/speed/basic Generic OBD responses over physical Classical CAN. It is not a production-code base, complete ECU environment or protocol-qualification fixture. No third-party source is integrated by this amendment.

The long-term fixture must generate scripted, reproducible cases for:

- supported-PID discovery bitmaps;
- RPM, speed, coolant, throttle and other selected Generic OBD values;
- multiple ECUs and unrelated traffic;
- 11-bit and any explicitly supported 29-bit addressing;
- single- and multi-frame ISO-TP;
- flow-control block size, separation time and timeout boundaries;
- UDS positive, negative, response-pending and session cases;
- malformed length, wrong source ID, wrong service/PID, duplicate, delayed, out-of-order and truncated responses;
- missing replies, busy ECU, bus-off/recovery and another tester;
- deterministic value ramps, steps and boundary/sentinel values.

A project-owned scenario/expectation format must control these cases without importing third-party code or copyrighted vehicle captures. `lbenthins/ecu-simulator` is the first tool to evaluate for host/SocketCAN work; a small independently written ESP-IDF fixture may later provide the physical bench endpoint if needed.

## Test layers

### Host-side deterministic tests

Run parsers, profile decoding, unit conversion, normalized arbitration, alarms, configuration validation/migration, RaceChrono packet construction, first-party messages and update-manifest checks on the workstation. Use injectable time, deterministic scheduling seeds and golden expected results. No ESP32 or CAN interface is required.

Configuration tests cover every supported schema migration, interrupted commit, corrupt primary copy, last-known-good recovery, factory-safe fallback, unknown optional fields, rejected incompatible required fields and secret redaction.

### CAN/GNSS replay

Replay fixtures into the same decoder/acquisition boundaries used by target firmware. CAN replay is a first-class regression mechanism: recorded fixture → vehicle profile → normalized core → expected channels/values. Assert decoded sample value/unit/source/time/validity and every drop/backoff event. A replay manifest records provenance, permission, redaction, profile applicability, capture tool/version, timestamp domain, hash and expected output. Synthetic fixtures are preferred for public CI.

BMW E81/N43 is the first real vehicle fixture, not a special-case code or test architecture. Future vehicles add fixtures and expectations through the same contracts.

GNSS replay covers message mix/rate, fix loss/recovery, UTC mapping, malformed messages and 20-25 Hz UART load. Replay does not establish RF performance.

### Virtual and physical ECU

Use SocketCAN/vcan first for fast deterministic protocol cases. Then connect a separately powered, correctly terminated and protected physical emulator to AutoTelemetry under test. The bench harness defines bit rate, termination at exactly two ends, common ground/isolation, power sequence and emergency disconnect. The product remains in LISTEN_ONLY except tests explicitly authorizing bounded diagnostic transmission.

### Future HIL

```text
development workstation
  |-- USB control/log/status --> AutoTelemetry under test
  |-- CAN interface ----------> physical CAN bench / ECU emulator
  |-- vcan/SocketCAN ---------> virtual protocol scenarios
  |-- GNSS UART replay -------> isolated GNSS digital input fixture
  |-- programmable supply ----> OBD power/crank/brownout cases
  +-- pytest controller ------> scenario, time, assertions, evidence
```

Future HIL also observes shift data/power/fault, status RGB, sounder waveform/acoustic output, display SPI, SD behavior, CAN TX inhibition, wake/sleep and current. Electrical transient, ESD/EMC, RF and acoustic validation remain separate qualified tests even when orchestrated from the same workstation.

No HIL equipment selection or purchase is required by Task 5A-DOC.

## Acceptance ownership

| Evidence | Minimum gate |
|---|---|
| Host unit tests | Every change to parser, profile, configuration migration, protocol packet or alarm state |
| Replay | Every released vehicle profile and regression fix, including negative fixtures |
| Emulator | Generic OBD/ISO-TP/UDS scheduler changes before vehicle use |
| RaceChrono compatibility | Captured against the target app release; tire channel naming and overlay verified explicitly |
| HIL | Firmware release candidate and hardware revision, with repeatable fixture revisions |
| Bench integration | Every prototype revision before stationary-vehicle use, including A/B and full-load interference evidence |
| Vehicle stationary | OBD, crank/start/shutdown, wake, grounds, accessories and ECU/tool coexistence under an approved procedure |
| Road | Long-duration functional, transition, vibration and temperature evidence after stationary validation |
| Track/slalom | Highest realistic combined workload; road evidence is not a substitute |
| Professional | Applicable pre-compliance/compliance/transient/environmental evidence for serious commercial release; internal testing is not compliance |

## Repository reservation

```text
tests/
  fixtures/
    synthetic/
    restricted/        # manifests only; protected data stored outside Git
  unit/
  integration/
  replay/
  emulator/
  hil/
```

Directories remain conceptual until implementation needs them. Do not add empty scaffolding solely for this freeze.

## Open items

- Select the host language/framework and exact ESP-IDF host-test boundary.
- Evaluate `lbenthins/ecu-simulator` in an isolated development environment and record a commit/license/dependency lock before use.
- Define the project-owned emulator scenario schema and physical bench safety/termination procedure.
- Acquire redistribution permission or create synthetic replacements for every vehicle fixture.
- Select HIL CAN interface, programmable supply and acoustic instrumentation only when a later implementation task requires them.
