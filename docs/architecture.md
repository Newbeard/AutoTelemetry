# Universal OBD/CAN + GNSS Telemetry Gateway architecture

Status: Task 5A.1 conditional prototype/capture re-freeze, 2026-08-17. Product/software boundaries remain architectural; the selected power partition below is controlled by [task5a1-power-architecture.md](task5a1-power-architecture.md) and remains conditional on its prototype gates. No compliance claim is made.

## Product and monorepo boundary

This repository is the monorepo for the complete Universal OBD/CAN + GNSS Telemetry Gateway:

- Telemetry v1 automotive electronics and later hardware revisions;
- ESP32-S3 firmware and vehicle-profile runtime;
- normalized telemetry contracts and device protocols;
- RaceChrono integration and a future first-party companion application;
- external displays, shift-light, alarms, logging, configuration, and optional lap timing;
- CAN/DBC/profile/log development tools and test fixtures;
- core enclosure, reusable display housings, vehicle-specific mounts, and manufacturing documentation.

The upstream RejsaCAN material remains a protected reference subtree. BMW E81/N43 is the first vehicle fixture. GC9A01 is the first display. RaceChrono is the first third-party client. None defines the core architecture.

Telemetry v1 hardware scope is deliberately narrow: 12 V passenger OBD, one Classical CAN channel, ESP32-S3, onboard GNSS, SD, USB-C/BLE, interchangeable display, short-cable shift light, onboard alarm, MODE/status/debug, and parked sleep/wake. TPMS, tire-temperature sensing, IMU, analog sensor hubs, external sensor networks, a second CAN channel, and unrelated additions require a later revision.

## System architecture

```text
ACQUISITION / VEHICLE SUPPORT
  Raw CAN ───────┐
  Generic OBD-II ├─> decoders/profile runtime ─┐
  ISO-TP / UDS ──┘                            │
  GNSS ───────────────────────────────────────┤
  local device status/sensors ────────────────┘
                                               ▼
                                  NORMALIZED TELEMETRY DATA CORE
                                  channel registry + timestamps
                                  validity + health + arbitration
                                               │
        ┌───────────────┬──────────────┬────────┼──────────┬───────────┐
        ▼               ▼              ▼        ▼          ▼           ▼
  RaceChrono       device API       display   logger   shift/alarm  lap engine
  adapter          BLE/Wi-Fi/USB    manager            outputs       optional
        │               │
        ▼               ▼
  RaceChrono app   future first-party app and tools
```

Acquisition producers publish candidates; the core selects one normalized source per channel; clients consume the same stable contract. No output owns acquisition. A RaceChrono disconnect, display reset, SD error, or future app absence cannot stop CAN/GNSS processing.

A cross-cutting diagnostic/health subsystem receives defined counters, state, last-success timestamps and fault/recovery events from services without owning their work. It makes health available to supervision, logger and expert/service interfaces. Normal telemetry channels and engineering diagnostics remain distinct contracts.

## Vehicle support layers

| Level | Behavior | Safety/default |
|---|---|---|
| 1 — Generic OBD-II | Discover supported standard PIDs and decode only supported responses | Available as fallback; no assumption that any example PID exists |
| 2 — known vehicle profile | Decode reviewed passive CAN signals and vehicle metadata | Prefer passive acquisition where profile policy and source health support it |
| 3 — extended manufacturer diagnostics | Use profile-defined ISO-TP/UDS/manufacturer requests for otherwise unavailable channels | Explicitly enabled scheduler policy; conservative rates; negative-response and coexistence handling |

Profile selection may be manual, configured, or detected with declared confidence. An absent/rejected Level 2/3 profile falls back to Level 1 where possible. A mismatch never enables speculative manufacturer requests.

## Vehicle-profile runtime

The canonical profile is a versioned project schema. It expresses bus properties, frame/signal decoding, validity, expected rate, diagnostic request/response/addressing, source priority, metadata, and matching rules. DBC is an import/export source for the subset it represents; it is not the complete runtime contract. OpenDBC is optional input after license/provenance review, not a hard dependency.

Canonical profile artifacts live under `profiles/`; firmware loading/validation belongs to `firmware/vehicle_profiles/`; conversion and validation tools belong to `tools/profile-tools/`. A profile is releasable only with provenance, schema validation, matching policy, recorded fixtures, decoder tests, and diagnostic-mode tests.

The conceptual schema and a fictitious example are in [`telemetry-data-model.md`](telemetry-data-model.md).

## CAN modes and diagnostic scheduler

### `LISTEN_ONLY`

- The controller/transceiver receives but firmware does not intentionally transmit CAN frames.
- It is the default safe acquisition mode for unknown vehicles, passive profiles, another connected diagnostic tool, or maximum non-interference.
- Hardware/driver configuration must use the ESP32 TWAI listen-only capability where applicable; application-level “do not send” alone is not the final safety control.

### `DIAGNOSTIC_POLLING`

- Transmission is allowed only through a centralized diagnostic scheduler.
- Requests are declared by the active profile or Generic OBD-II service, never emitted directly by a display, logger, output, or alarm.
- The scheduler owns arbitration, ISO-TP sessions, timeouts, negative responses, retry/backoff, utilization budget, duplicate-request coalescing, and cancellation during shutdown.

Conceptual classes are relative, not fixed frequencies:

| Class | Use | Policy |
|---|---|---|
| realtime/high-rate | Fast channels unavailable passively | Strict bus/load budget; disable first on contention |
| medium | Temperatures and driver-state channels | Poll only when subscribed/needed |
| slow | Voltage, health, slow status | Low duty cycle |
| one-time | VIN/identity/capability discovery | Session/profile activation only, cached with validity |

Rates are profile/configuration values and adapt to response latency, ECU busy/negative responses, bus utilization, active clients, power state, and evidence of another diagnostic client. No task may bypass the scheduler.

### Coexistence

Multiple passive listeners add electrical loading and processing but do not contend like active diagnostic clients. Multiple clients issuing OBD/UDS requests can collide, alter ECU session state, consume bus/ECU capacity, or receive each other's responses. Telemetry v1 therefore:

- never assumes exclusive OBD ownership;
- offers user-selectable `LISTEN_ONLY` and `DIAGNOSTIC_POLLING`;
- defaults unknown vehicles/profile mismatches to listen-only until explicitly configured;
- applies conservative polling, response ownership, backoff, and session cleanup;
- detects/reports possible contention where evidence exists;
- permits immediate suspension of diagnostics while keeping passive telemetry active.

Safe coexistence with OBDLink, scan tools, listeners, or other diagnostic clients cannot be guaranteed for every vehicle; it requires vehicle/profile validation.

## GNSS

The NEO-M9N producer publishes GNSS candidate channels directly into the core. RaceChrono, display, logger, device protocol, and optional lap engine subscribe independently. No GNSS data path is routed “through RaceChrono.”

Hardware decisions are authoritative in [`component-freeze.md`](component-freeze.md) and [`schematic-architecture.md`](schematic-architecture.md): NEO-M9N-00B on switched GNSS_3V3, V_BCKP following that rail, protected external active antenna, and 230,400-bit/s UART. Parser/library selection is unresolved.

## Outputs and optional services

### RaceChrono

`firmware/racechrono/` is an adapter from normalized vehicle/GNSS samples to the public RaceChrono DIY BLE contract. RaceChrono packet identifiers, units, filters, connection lifecycle, and BLE characteristics remain inside the adapter. The adapter is neither the data model nor a vehicle decoder. Disconnect/reconnect affects only that output.

### First-party device protocol and application

`firmware/protocol/` serves a transport-neutral command/telemetry service. BLE, Wi-Fi, and USB are transport bindings around the same operations. Versioned protocol specifications belong under `protocol/`; the preliminary capability model is in [`device-protocol-architecture.md`](device-protocol-architecture.md).

`apps/companion/` reserves the future first-party application. It may provide pairing, dashboards, GNSS maps, lap/sector analysis, session history, graphs, comparisons, configuration, updates, and log transfer. It consumes public device/log/profile contracts and must not require firmware-internal memory layouts.

### Display

```text
firmware/display/
  manager/     attachment, lifecycle, frame pacing, headless mode
  render/      drawing primitives/view model
  drivers/     GC9A01, ST7789, future controller/panel adapters
  layouts/     race_round, temperatures, minimal, street, future themes
```

Drivers know electrical/controller behavior but no vehicle channels. Layouts request normalized channels and render through the abstraction. The manager handles missing/unhealthy displays independently. `headless` is a supported configuration, not an error.

### Shift-light

The removable external shift-light contains exactly ten addressable RGB pixels, local bypass capacitors, optical wells/diffusion and cable strain relief. It subscribes to a configured normalized channel, normally `RPM`. Configuration owns activation, staged thresholds, flash threshold, brightness, day/night profile, output device, and missing/stale-data behavior. It does not depend on RaceChrono, a display, or a phone. Electrical and mechanical details are frozen in [`user-io-configuration.md`](user-io-configuration.md).

### Alarms

Alarm architecture has three layers:

1. **rule** — versioned condition over normalized channels, thresholds, duration/hysteresis, enablement, and stale/missing behavior;
2. **state** — inactive, pending, active, acknowledged/latched, cleared, or data-unavailable, with timestamps/reason;
3. **presentation** — independent buzzer, display, shift-light/LED, logger, and future-app sinks.

A presentation failure does not erase alarm state. A missing safety-relevant channel raises data-unavailable according to rule policy; it is not interpreted as a safe numeric value.

### Logger

The logger subscribes to normalized samples, GNSS, clock mappings, source/provenance changes, device health, configuration/profile versions, and errors. It owns buffering, rotation, session lifecycle, and storage fault reporting. The final file format remains open, but it must be versioned, self-describing enough for independent tools, and decodable without firmware structs. SD failure disables/restarts the logger without stopping normal telemetry.

### Optional lap engine

`firmware/lap_engine/` is reserved for autonomous start/finish, sectors, best/predicted lap, and live delta. It consumes normalized GNSS/telemetry and publishes derived channels/events. It can be absent or disabled; RaceChrono remains fully usable without it.

## Fault isolation and runtime model

Conceptual FreeRTOS tasks/services are separated by bounded queues or message buses:

- CAN receive and timestamp;
- diagnostic scheduler/ISO-TP/UDS;
- GNSS receive/parse;
- telemetry core/arbitration;
- outputs (RaceChrono/protocol/display/shift/alarm);
- logger;
- power/config/health supervision.

This is a responsibility model, not a frozen task count. Mandatory principles:

- acquisition paths never wait on BLE, display, SD, app, or rendering;
- queues are bounded and expose high-water/drop counters;
- latest-value consumers may coalesce; loss-aware consumers receive gap markers;
- each service reports heartbeat, error class, queue health, restart count, and last-success time;
- watchdog escalation distinguishes a restartable service from a whole-device deadlock;
- timeouts cancel owned work and release diagnostic/session resources;
- optional services reconnect/restart independently with bounded backoff;
- repeated failures degrade capability and remain visible through status/logs;
- GNSS failure cannot stop CAN; BLE failure cannot stop local outputs/logging; display failure cannot stop streaming; SD failure cannot stop acquisition;
- missing profile falls back to Generic OBD-II where possible, otherwise explicit unavailable channels.

Reserved health categories include CAN/diagnostic frames, drops, errors, bus-off/recovery and timeouts; GNSS fix/C/N0/gaps/reacquisition/parser health; BLE/Wi-Fi connections and failures; SD/logger write, mount and backpressure health; and system uptime, reset/brownout/watchdog, memory, queue and service-restart health. Exact data types, retention, rollover and transport are deferred. Metrics must be useful rather than collected indiscriminately.

Power management coordinates a quiesce barrier: stop new diagnostics, make outputs safe, flush/close logger within a bounded deadline, save eligible state, disable peripheral domains, configure wake, then sleep. A failed optional participant is timed out and reported; it cannot hold the vehicle awake indefinitely.

## Validation hooks and deterministic evidence

Production responsibilities expose test seams at stable boundaries:

- decoders and vehicle profiles accept provenance-controlled recorded/synthetic CAN fixtures;
- the normalized core accepts deterministic time and expected source/validity transitions;
- OBD/ISO-TP/UDS behavior can use virtual or protected physical ECU emulation;
- GNSS parser behavior can use replay while RF performance remains a separate physical test;
- outputs and logger accept deterministic normalized samples and injected failures;
- power, wake and optional services expose state/health needed by bench and future HIL automation.

Recorded CAN replay is a first-class regression mechanism. BMW E81/N43 is the first real fixture, not a branch in core logic; future vehicles add fixtures and profile expectations through the same contracts.

Prototype validation follows host/software → bench → stationary vehicle → road → track/slalom stress. `FULL_LOAD_INTERFERENCE_TEST` and controlled A/B cases exercise noisy/high-current consumers while observing power, CAN, GNSS, radios, storage and runtime health. Procedures and evidence records are controlled by [system-validation-plan.md](system-validation-plan.md).

## Configuration

A centralized configuration service owns:

- active vehicle profile and matching policy;
- CAN mode and diagnostic scheduler policy;
- display type/layout/brightness and headless state;
- shift-light source/thresholds/brightness;
- alarm rules/presentations;
- RaceChrono and first-party transport settings;
- GNSS, logger, protocol, and power/sleep settings.

Drivers receive validated configuration snapshots; constants are not scattered through modules. Configuration has independent schema version, transactional validation, migration, defaults, provenance, and rollback to the last-known-good copy. Unknown/new fields are preserved where practical. Writes are authenticated/authorized as required by the eventual threat model; transport connection alone is not authorization.

The future web UI and first-party app are clients of the same configuration manager and public API. Web setup is an explicitly requested, authenticated, time-limited SoftAP mode; it is not a second configuration store. Firmware and vehicle-profile updates stage, authenticate, validate and activate independently, always retaining a recovery path. The complete policy is in [`user-io-configuration.md`](user-io-configuration.md).

## Future tire module and second CAN

Tire pressure, tire temperature and left-to-right tread-temperature arrays are future normalized producers, not Telemetry v1 sensor hardware. A separate optional module is preferred and its transport remains open. Missing/stale/invalid data, link loss and either-side reboot degrade only tire channels; they cannot stop the main unit's CAN/GNSS acquisition or other consumers. RaceChrono naming must be verified against the target app release. ESP32-S3 has one native TWAI controller; neither a second CAN controller, Tire RF receiver nor an ESP32-C6 migration is authorized for v1.

## Hardware partition and states

Task 5A.1 conditionally re-freezes this vehicle-input chain for prototype capture:

```text
OBD pin 16
  -> 0437002.WRA 2 A fuse
  -> VBAT_FUSED_CLAMPED
     -> shunt common-anode SM15T47AY + SM15T33AY pair -> POWER_GND
  -> LM74720QDRRRQ1 reverse-blocking controller
  -> 2 x STL125N10F8AG 100 V back-to-back N-MOSFETs
  -> characterized damped input filter
  -> vehicle regulators and vehicle-only CAN/AUX domains
```

The OV network must be tolerance-bounded for no static trip through 18 V, a rising-input OV/PD-low command no later than 25.531 V and therefore before 26 V, and reconnect eligibility by 18 V. Completed disconnect/reconnect, dynamic downstream peak and timing remain capture gates. The older ≤20 V trip and `VEHICLE_PROTECTED` ≤24 V targets are superseded. The 2 A fuse, asymmetric clamp, MOSFET VDS/SOA, controller stress, inrush and filter remain subject to the exact analytical and prototype gates in [task5a1-power-architecture.md](task5a1-power-architecture.md); conditional capture status is not transient qualification.

`LMQ66420MC3RXBRQ1` is frozen as the vehicle converter producing `VEH_3V3`. USB uses its own `TPS62162QDSGRQ1` converter to produce `USB_3V3`; `TPS2116DRLR` selects the two regulated inputs and supplies the `MAIN_3V3` core domain. USB cannot power the vehicle CAN or AUX5 domains. TPS2116 is a catalog, non-AEC part whose published electrical limits used here extend through 105 °C, so board-level environmental qualification or an automotive replacement remains a production gate. `LMQ66420MC5RXBRQ1` remains a conditional vehicle-only AUX5 choice pending rail thermal and load-step evidence. The corrected maximum-load total is 8.27505 W; the historical 12.458 W aggregate is superseded.

Software power states remain `PARKED/SLEEP`, `WAKE`, `ACTIVE`, `SHUTDOWN_PENDING`, and `USB_DEBUG`. Full-system operation is not allowed throughout 6–18 V. MAX is limited to ≤10 s and ≤25% duty in any rolling 60 s where permitted. Below 10.0 V, MAX, full-white diagnostics and Wi-Fi are prohibited; below 9.5 V, request logger flush and stop starting optional work; after less than 9.0 V persists for 100 ms, shed optional loads and retain controlled core/CAN/wake survival down to 6 V. Restore shed loads only after greater than 10.0 V persists for 2 s. These exact timer/tolerance implementations remain design requirements pending measurement. CAN traffic plus profile/policy remains stronger activity evidence than vehicle voltage alone.

## Repository ownership

| Path | Authority |
|---|---|
| `docs/` | Product requirements, architecture, engineering evidence and reviews |
| `hardware/telemetry-v1/` | Future derivative CAD/BOM/manufacturing work after gates |
| `firmware/` | Device implementation, separated by service/domain |
| `profiles/` | Canonical declarative vehicle-profile packages and fixtures metadata |
| `protocol/` | Public device protocol schemas/specifications and compatibility records |
| `apps/companion/` | Reserved first-party application |
| `tools/` | Host-side DBC/CAN/profile/log tools; never linked implicitly into firmware |
| `enclosure/` | Parametric source CAD and derived outputs organized by reusable layer |
| `tests/` | Cross-domain test harnesses and recorded/synthetic fixtures |

Third-party source is never pasted into these paths without dependency approval, license notices, version pinning, provenance, and tests.

## Architecture gates and unresolved decisions

Before implementation:

- select firmware framework/build system and dependency baseline;
- freeze v1 channel registry/profile/config schema subsets;
- define memory, queue, task-priority, and watchdog budgets from measurements;
- define diagnostic coexistence limits and explicit enable policy;
- define first-party security/threat/update model before writable remote operations;
- choose protocol/log serialization only after transport and resource prototypes;
- define signed/trusted profile and firmware update policy;
- resolve root/upstream and third-party licensing before distribution;
- establish fixture privacy/redaction policy for recorded vehicle data.
- define diagnostic metric schemas, retention/reset semantics and expert/service access.
- refine revision-specific A/B and `FULL_LOAD_INTERFERENCE_TEST` limits from measured prototype baselines.

No schematic capture or implementation is authorized by this architecture document.
