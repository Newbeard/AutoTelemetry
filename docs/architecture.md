# Universal OBD/CAN + GNSS Telemetry Gateway architecture

Status: product architecture baseline for the third pre-implementation phase. This document defines boundaries and responsibilities, not firmware tasks, wire formats, schemas, or code.

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

Hardware assumptions remain authoritative in [`power-budget.md`](power-budget.md) and [`power-wake-review.md`](power-wake-review.md): switched GNSS rail, v1 backup supply off with host save/restore investigated, external active antenna, and preliminary 230,400-bit/s UART. Parser/library selection is unresolved.

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

The shift-light subscribes to a configured normalized channel, normally `RPM`. Configuration owns activation, staged thresholds, flash threshold, brightness, day/night profile, output device, and missing/stale-data behavior. It does not depend on RaceChrono, a display, or a phone.

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

Power management coordinates a quiesce barrier: stop new diagnostics, make outputs safe, flush/close logger within a bounded deadline, save eligible state, disable peripheral domains, configure wake, then sleep. A failed optional participant is timed out and reported; it cannot hold the vehicle awake indefinitely.

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

## Hardware partition and states

The approved pre-schematic hardware direction remains the hybrid rail-on architecture: protected/source-isolated OBD and USB input, low-IQ MAIN_3V3 for ESP32/TCAN, independent GNSS/SD/display switches, and normally-off AUX5. The complete numerical evidence remains in `power-budget.md` and `power-wake-review.md`; this software phase changes no GPIO, rail, CAN, RF, or USB decision.

Software power states remain `PARKED/SLEEP`, `WAKE`, `ACTIVE`, `SHUTDOWN_PENDING`, and `USB_DEBUG`. CAN traffic plus profile/policy is stronger activity evidence than vehicle voltage alone.

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

No schematic capture or implementation is authorized by this architecture document.
