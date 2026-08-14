## Task completed

Completed the third pre-implementation Telemetry v1 architecture phase. Defined the product monorepo, normalized telemetry core, three vehicle-support levels, declarative profiles, CAN modes and diagnostic scheduling, independent outputs, first-party device protocol, fault isolation, centralized configuration, dependency policy, enclosure/mount architecture, and product-wide test strategy. Added ownership-only scaffolds. No schematic, PCB, firmware/app implementation, source dependency, CAD geometry, or manufacturing output was created.

## Files changed

- `CODEX.md`
- `docs/architecture.md`
- `docs/requirements.md`
- `docs/hardware-spec.md`
- `docs/interfaces.md`
- `docs/telemetry-data-model.md`
- `docs/device-protocol-architecture.md`
- `docs/open-source-dependencies.md`
- `docs/enclosure-architecture.md`
- `docs/test-architecture.md`
- `docs/chatgpt-review.md`
- `hardware/telemetry-v1/README.md`
- `hardware/telemetry-v1/kicad/README.md`
- `hardware/telemetry-v1/bom/README.md`
- `hardware/telemetry-v1/manufacturing/README.md`
- `firmware/README.md`
- `firmware/main/README.md`
- `firmware/core/README.md`
- `firmware/can/README.md`
- `firmware/obd/README.md`
- `firmware/isotp/README.md`
- `firmware/uds/README.md`
- `firmware/gnss/README.md`
- `firmware/vehicle_profiles/README.md`
- `firmware/racechrono/README.md`
- `firmware/protocol/README.md`
- `firmware/display/README.md`
- `firmware/display/manager/README.md`
- `firmware/display/render/README.md`
- `firmware/display/drivers/README.md`
- `firmware/display/layouts/README.md`
- `firmware/shiftlight/README.md`
- `firmware/alarms/README.md`
- `firmware/logger/README.md`
- `firmware/lap_engine/README.md`
- `firmware/power/README.md`
- `firmware/config/README.md`
- `profiles/README.md`
- `protocol/README.md`
- `apps/companion/README.md`
- `tools/README.md`
- `tools/dbc/README.md`
- `tools/can-analysis/README.md`
- `tools/profile-tools/README.md`
- `tools/log-tools/README.md`
- `enclosure/core_device/README.md`
- `enclosure/displays/README.md`
- `enclosure/displays/gc9a01/README.md`
- `enclosure/vehicle_mounts/README.md`
- `enclosure/vehicle_mounts/bmw_e81/README.md`
- `tests/README.md`
- `tests/fixtures/README.md`

## Engineering findings

- The product needs one central Normalized Telemetry Data Core. Producers publish typed candidates with canonical channel ID, unit, source, capture/source time, validity, age, rate and quality; outputs consume the selected canonical sample and never decode vehicle frames.
- Source arbitration is deterministic and channel-specific: eligibility/validity, configured priority, source quality, freshness, hysteresis/minimum hold, then stable source identity. Loss of the selected source triggers fallback rather than substituting zero or an unmarked stale value.
- Vehicle support is layered: Level 1 Generic OBD-II, Level 2 known-profile passive CAN, and Level 3 explicitly enabled manufacturer diagnostics over ISO-TP/UDS. A weak/mismatched profile must not enable speculative requests.
- Profiles require frame ID, 11/29-bit format, bit rate, bit extraction, byte order, signedness, scale, offset, unit, destination, rate, validity, diagnostic addressing/request/response, source priority, matching, provenance and metadata. The YAML example is fictitious and conceptual.
- DBC represents much passive CAN extraction but not the full contract. It can be reviewed/imported into the canonical schema. OpenDBC remains optional host-side input, not a runtime dependency.
- LISTEN_ONLY is non-transmitting and must ultimately use controller/driver enforcement. In DIAGNOSTIC_POLLING, only one scheduler may transmit; it owns budgets, ISO-TP sessions, timeouts, negative responses, cancellation, backoff and coexistence.
- Active diagnostic clients may contend or alter ECU state. Universal coexistence cannot be guaranteed; the device must expose mode/state, schedule conservatively, report suspected contention, and suspend diagnostics while retaining passive acquisition.
- RaceChrono is a replaceable normalized-data output. The first-party API is transport-neutral with independent API/schema/transport versions and BLE, Wi-Fi and USB bindings.
- Display manager, renderer, driver and layout are independent; headless operation is supported. Shift-light is independent. Alarm rules/state/presentations are separate. Logger and optional lap engine are independent consumers.
- Conceptual FreeRTOS responsibilities use bounded queues. Acquisition never waits on display, storage or clients; services expose pressure/health, use deadlines, bounded retry, isolated restart and watchdog escalation.
- Configuration is centralized, versioned, validated, atomic, migratable, provenance-aware and recoverable to last-known-good/safe defaults.
- Core-device enclosure, display enclosure and vehicle mount are separate parametric products. Native CAD is authoritative, STEP is interchange, and meshes are derived. Electrical drawings/limits control CAD.
- Tests cover core arbitration, profile qualification, replay, diagnostics/coexistence, GNSS, outputs, transport parity, migrations, overload/fault isolation, power transitions, HIL, and separately approved electrical/RF/mechanical plans.
- The customer-mount workflow records vehicle/year/trim/location and selected display, then uses traceable drawings, photos/scales, caliper measurements, scans or characterized photogrammetry; it proceeds through parametric modeling, prototype, fit/safety validation, revision, controlled output, and reusable-library release.
- RaceChrono's tutorial links its DIY repository, but no explicit license was found in the reviewed root. `rc_can_ble` and `rc_can_ble_fw` likewise have no confirmed license. They are reference-only; no source was copied.
- `esp32_obd2`, `isotp-c`, `iso14229`, OpenDBC and SavvyCAN declare MIT licenses. ESP-IDF declares Apache-2.0. SparkFun u-blox GNSS v3 separates MIT code from CC BY-SA hardware. `ubxlib` is Apache-2.0 with exceptions/notices and is archived, so it is reference-only by default.
- No root license was found for the reviewed RejsaCAN upstream/fork. Distribution and derivative permissions remain unresolved.
- Final architecture scaffold:

```text
hardware/telemetry-v1/{kicad,bom,manufacturing}
firmware/{main,core,can,obd,isotp,uds,gnss,vehicle_profiles,racechrono,protocol,display,shiftlight,alarms,logger,lap_engine,power,config}
firmware/display/{manager,render,drivers,layouts}
profiles/
protocol/
apps/companion/
tools/{dbc,can-analysis,profile-tools,log-tools}
enclosure/{core_device,displays/gc9a01,vehicle_mounts/bmw_e81}
tests/fixtures/
```

## Decisions made

- Adopt the normalized core as the only shared data boundary.
- Keep canonical profiles under `profiles/`, with runtime handling and host tools separated; keep DBC/OpenDBC optional at the import boundary.
- Default unknown/mismatched vehicles to LISTEN_ONLY and route all diagnostics through one scheduler; prohibit output-triggered/arbitrary production CAN transmission.
- Keep RaceChrono, first-party protocol, display, shift-light, alarms, logger and lap engine replaceable and fault-isolated.
- Reserve a transport-independent first-party protocol, centralized configuration owner, and independent core/display/mount mechanical layers.
- Create README-only architecture scaffolds, including hardware `kicad`, `bom` and `manufacturing`, without authorizing implementation.
- Preserve every earlier power, CAN, GNSS/RF, USB and GPIO hardware decision unchanged.

## Assumptions

- ESP-IDF/FreeRTOS is the likely target because Telemetry v1 uses ESP32-S3, but framework/version selection remains open.
- Runtime responsibilities may map to tasks or services; count/priorities are not fixed.
- Generic OBD-II is a fallback only where vehicle and supported PIDs permit it.
- A user/configuration mechanism will select CAN mode and confirm uncertain profile matches.
- BLE, Wi-Fi and USB are candidate protocol bindings; simultaneous availability is not promised.
- All conceptual profile identifiers, bit positions, scaling, rates, request bytes, priorities and matching data are fictitious `ASSUMPTION` values.
- The initial channel registry is extensible and is not a frozen wire enumeration.
- Display cable <=200 mm, SPI <=20 MHz, and shift-light long-cable review above 0.5 m remain prior `DESIGN_REQUIREMENT` values.
- BMW E81/N43, GC9A01 and RaceChrono are first fixtures/integrations, not architectural special cases.
- A future CAD toolchain can provide native parametric CAD, STEP, and optional derived 3MF/STL.

## Uncertainties / unresolved questions

- Firmware/build system, exact dependencies, update/rollback mechanism and product license.
- Frozen v1 channel registry, profile/config syntax, wire encoding, log format and migration policy.
- Task/queue/heap/stack/watchdog budgets and measured deadlines.
- Diagnostic rate/bus budgets, tester detection, cleanup and per-vehicle safe service allowlists.
- Threat model, pairing/authentication, authorization, key storage, profile/firmware signing and writable API policy.
- RaceChrono interoperability details and permission for example reuse; current design avoids reuse.
- RejsaCAN derivative/distribution permission and exact licenses/notices for every library, DBC, capture, asset and transitive dependency.
- First-party app platforms and concrete transport bindings.
- Fixture provenance, redistribution permission, VIN/location redaction and storage policy.
- PCB mechanical data, enclosure environment/material/process, mount load cases, exact display geometry and BMW E81 measurements.
- Bench/HIL equipment, test/CI framework, tolerances, vehicle safety procedure and compliance targets.

## Risks

- Incorrect matching or diagnostic definitions can create unintended traffic or ECU state changes.
- Poor arbitration thresholds can oscillate or conceal degraded data.
- Unbounded/shared blocking work or broad watchdog resets can let an optional output disrupt acquisition.
- Transport-specific or RaceChrono-derived internal models would cause permanent coupling.
- Unlicensed examples, CAD, DBCs or captures can block lawful distribution; public access is not permission.
- Vehicle captures and GNSS logs can expose VIN, driving and location data.
- Generic mechanical assumptions can obstruct safety systems/controls, create projectiles, overheat, fatigue, or falsely claim compatibility.
- The architecture may exceed ESP32-S3 resources until concurrency is measured.
- Coexistence detection cannot prove another tester is absent on every vehicle.

## GPIO / peripheral changes

No GPIO or peripheral allocation changed. The preliminary allocation in `docs/interfaces.md` remains intact. No driver was implemented and no pin was reserved, released or reassigned.

## Power / CAN / RF impact

- **Automotive power:** no topology, component, capacity, parked-current or wake decision changed; only shutdown/test responsibilities were added.
- **CAN:** no hardware or bit-rate decision changed; behavior now formally separates LISTEN_ONLY from centralized DIAGNOSTIC_POLLING.
- **GNSS/RF:** no module, antenna, UART, supply or layout decision changed; GNSS is an independent producer and CAD must respect RF/cable constraints.
- **USB:** no electrical decision changed; USB is a candidate protocol binding subject to existing source isolation.
- **ESP32 boot/strapping:** no change; existing GPIO45/microSD and other strap constraints remain.

## Datasheets / primary sources consulted

- Repository `CODEX.md`, current architecture/hardware/interface/power analysis documents, and protected RejsaCAN v3.4 reference material.
- RaceChrono tutorial and DIY reference: https://racechrono.com/article/2572 and https://github.com/aollin/racechrono-ble-diy-device
- `rc_can_ble` and firmware: https://github.com/Sergey1560/rc_can_ble and https://github.com/Sergey1560/rc_can_ble_fw
- `esp32_obd2`: https://github.com/MagnusThome/esp32_obd2
- ESP-IDF/TWAI: https://github.com/espressif/esp-idf
- `isotp-c`: https://github.com/lishen2/isotp-c
- `iso14229`: https://github.com/driftregion/iso14229
- OpenDBC: https://github.com/commaai/opendbc
- SparkFun u-blox GNSS v3: https://github.com/sparkfun/SparkFun_u-blox_GNSS_v3
- u-blox `ubxlib`: https://github.com/u-blox/ubxlib
- SavvyCAN: https://github.com/collin80/SavvyCAN

No new numerical hardware decision was made, so no additional component datasheet was required.

## Recommended next step

Freeze the v1 channel registry, vehicle-profile/config schema subset, dependency baseline, and measured ESP32-S3 runtime resource budgets in a firmware-platform architecture review before implementing any module.

## STOP condition

The requested third pre-implementation architecture phase is complete. Stop here and wait for review; do not begin schematic capture, PCB layout, firmware/app implementation, CAD geometry, dependency integration, procurement, or manufacturing output.
