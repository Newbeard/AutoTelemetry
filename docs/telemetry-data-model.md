# Normalized Telemetry Data Core

Status: conceptual architecture contract. No parser, registry, scheduler, serialization, or firmware implementation is defined here.

## Purpose and boundary

The Normalized Telemetry Data Core is the product's internal contract between acquisition and consumers. Raw CAN, Generic OBD-II, manufacturer diagnostics, vehicle profiles, GNSS, and future sensors publish candidate samples. RaceChrono, the first-party protocol, displays, shift-light, alarms, logger, and optional lap engine subscribe to normalized channels.

Consumers must not decode vehicle frames, issue diagnostic requests, parse u-blox messages, or depend on another consumer. The core does not know BLE, display pixels, log files, or BMW-specific identifiers.

```text
CAN frames ─┐
OBD-II ─────┤
UDS/ISO-TP ─┤─> decoders ─> candidate samples ─> arbitration ─> channel store/event stream
GNSS ───────┘                                                │
                                                              ├─ RaceChrono
                                                              ├─ first-party protocol
                                                              ├─ display / shift-light / alarms
                                                              ├─ logger
                                                              └─ optional lap engine
```

## Channel registry

Every public channel has a stable symbolic ID, one canonical value type, one canonical engineering unit, a semantic description, validity constraints, and a compatibility lifecycle. IDs are uppercase ASCII with underscores. Once released, an ID's meaning, type, and canonical unit do not change incompatibly.

New standard channels are added to a versioned registry. Experimental or vehicle-specific channels use namespaces such as `EXPERIMENTAL_*` or `VENDOR_BMW_*`; a vendor channel may later map to a new canonical channel, but is not silently renamed. Unknown channels pass through logs/protocols by ID where capability negotiation permits.

The initial registry is deliberately small:

| Channel ID | Conceptual type | Canonical unit | Notes |
|---|---|---|---|
| `RPM` | unsigned numeric | rpm | Engine or selected rotating-system speed; metadata identifies semantics |
| `VEHICLE_SPEED` | numeric | m/s | Selected vehicle-ground speed |
| `GPS_SPEED` | numeric | m/s | GNSS speed over ground; remains distinct from selected vehicle speed |
| `COOLANT_TEMP` | numeric | °C | Validity range belongs to source/profile, not this table |
| `OIL_TEMP` | numeric | °C | May be unavailable on many vehicles |
| `OIL_PRESSURE` | numeric | Pa | Display/protocol adapters may convert to bar/psi |
| `THROTTLE` | numeric | ratio | Canonical closed-to-open ratio; source semantics must identify commanded vs measured |
| `BRAKE_PRESSURE` | numeric | Pa | Not assumed available from Generic OBD-II |
| `STEERING_ANGLE` | numeric | rad | Sign convention must be defined by channel registry |
| `GEAR` | enum/integer | none | Includes neutral/reverse/unknown states |
| `BATTERY_VOLTAGE` | numeric | V | Source may be vehicle ECU or local ADC |
| `LATITUDE` | numeric | degree | WGS 84 latitude |
| `LONGITUDE` | numeric | degree | WGS 84 longitude |
| `ALTITUDE` | numeric | m | Reference datum must be part of channel definition |
| `HEADING` | numeric | rad | Course/heading distinction must remain explicit |
| `GNSS_ACCURACY` | structured/numeric | m | Exact horizontal/vertical representation remains to be finalized |
| `GNSS_SATELLITES` | unsigned integer | count | Used satellites; visible satellites may be a separate future channel |
| `ACCELERATION_X/Y/Z` | numeric | m/s² | Reserved for future sensor/GNSS/vehicle sources; coordinate frame is mandatory metadata |

The registry must distinguish values that look similar but have different semantics. For example, wheel speed, ECU vehicle speed, and GNSS speed remain separate source channels even when one is selected as `VEHICLE_SPEED`.

## Sample contract

A candidate or selected sample conceptually contains:

| Field | Requirement |
|---|---|
| `channel_id` | Stable registry ID |
| `value` | Typed value: signed/unsigned integer, floating/fixed numeric, boolean, enum, or bounded structure |
| `unit` | Registry unit; producers convert before publishing |
| `source_id` | Stable source instance such as passive profile signal, OBD PID, UDS DID, GNSS, or local ADC |
| `capture_time` | Monotonic device timestamp assigned as close to acquisition as practical |
| `source_time` | Optional source timestamp, e.g. GNSS time, with its domain identified |
| `timestamp_domain` | Monotonic, GNSS/UTC, vehicle/source, or explicitly unknown |
| `validity` | `VALID`, `STALE`, `SUSPECT`, `INVALID`, or `NOT_AVAILABLE` |
| `age` | Derived from current monotonic time and capture time; consumers do not invent their own clocks |
| `expected_rate` | Source/profile expectation used for health and staleness, not a promise of exact periodicity |
| `quality` | Optional comparable source quality/confidence plus reason flags; absence means unknown, not perfect |
| `sequence` | Monotonic per-source or per-channel sequence where useful for loss detection |

Raw provenance may additionally include CAN bus, arbitration ID, diagnostic request identity, GNSS fix type, decoder/profile version, and conversion rule. Provenance can be retained for logging/diagnostics without bloating every hot-path consumer message.

## Time model

- A monotonic device clock is authoritative for ordering, latency, age, timeout, and rate calculations.
- GNSS/UTC is a separately qualified wall-clock mapping. UTC jumps or loss of fix must not reorder samples.
- Source timestamps are preserved when useful but never silently treated as device monotonic time.
- Log/session formats must record clock-domain mapping events and uncertainty so offline tools can reconstruct time.
- Consumers specify maximum acceptable age. A numerically valid but old sample becomes `STALE` and is not refreshed by another channel's activity.

## Validity and quality

Validity describes whether a value may be used. Quality describes how strongly one healthy source should be preferred over another. Reasons are explicit, for example: timeout, transport error, diagnostic negative response, out-of-range decode, GNSS fix loss, profile mismatch, implausible rate-of-change, or source conflict.

Validation occurs in layers:

1. transport/frame integrity;
2. decoder bounds and reserved/sentinel handling;
3. profile-defined physical/temporal plausibility;
4. source health and freshness;
5. optional cross-source consistency checks.

A failed cross-check marks data `SUSPECT`; it does not rewrite the value to match another source.

## Source arbitration

Each normalized channel may have several candidate sources. Arbitration policy is data-driven by vehicle profile and user/system configuration. The policy contains:

- ordered preferences or conditional rules;
- required source health;
- freshness timeout and minimum useful rate;
- latency and quality thresholds;
- hysteresis/minimum hold time to prevent source flapping;
- fallback permission;
- recovery criteria and reason reporting.

The examples below illustrate defaults only, not universal priorities:

```text
RPM: passive decoded CAN -> manufacturer diagnostic -> Generic OBD-II
VEHICLE_SPEED: passive decoded CAN -> Generic OBD-II -> GNSS speed
```

A profile may reverse or remove any preference. Selection changes generate a status event and preserve the source ID in every selected sample. Consumers subscribe to `RPM` or `VEHICLE_SPEED` and do not change when the source changes.

When all sources fail, the selected channel becomes `STALE` and then `NOT_AVAILABLE` according to policy; the core must not hold a plausible-looking value indefinitely.

## Vehicle support layers

| Level | Source | Contract |
|---|---|---|
| 1 — Generic OBD-II | Supported-PID discovery and standard diagnostic PIDs | Provides only channels actually reported by the vehicle; absence is normal |
| 2 — known vehicle profile | Passive CAN/DBC-like signal definitions and known metadata | Adds vehicle-specific channels without exposing raw frame knowledge to consumers |
| 3 — extended manufacturer diagnostics | Profile-defined ISO-TP/UDS/manufacturer requests and response decoders | Adds missing channels under conservative scheduler/policy control |

BMW E81/N43 is the first fixture/profile, not a special case in the core. Missing or rejected Level 2/3 profiles fall back to Level 1 where the vehicle supports it. Failure of Generic OBD-II discovery produces explicit unavailable channels, not fabricated defaults.

## Declarative vehicle profile

The canonical profile format is a versioned project schema, not raw DBC. DBC can be imported for CAN signal definitions; OpenDBC can be an optional data source after license/provenance review. Diagnostic scheduling, source arbitration, validity, vehicle matching, and product metadata exceed ordinary DBC scope and remain in the project schema.

Conceptual example only:

```yaml
profile:
  schema_version: 1
  id: example.vehicle.powertrain
  metadata:
    make: Example
    model_family: Example-1
    years: [unknown]
    provenance: "review-required"
  matching:
    manual_select: true
    rules:
      - type: diagnostic_identity
        expected: "TBD"
  buses:
    powertrain:
      bitrate: 500000
  passive_signals:
    - source_id: can.powertrain.engine_speed
      bus: powertrain
      arbitration_id: 0x123
      frame_format: standard
      start_bit: 16
      bit_length: 16
      byte_order: little_endian
      signed: false
      scale: 0.25
      offset: 0
      unit: rpm
      destination: RPM
      expected_rate_hz: 50
      validity:
        raw_invalid: [65535]
        timeout_ms: 100
  diagnostics:
    - source_id: uds.powertrain.oil_temp
      transport: isotp
      addressing:
        mode: physical
        request_id: 0x700
        response_id: 0x708
      request:
        service: 0x22
        identifier: 0x1234
      response:
        service: 0x62
        data_offset: 3
        bit_length: 8
        scale: 1
        offset: -40
        unit: degC
        destination: OIL_TEMP
      schedule_class: medium
  source_policy:
    RPM:
      preferred: [can.powertrain.engine_speed, obd.service01.rpm]
      fallback: true
```

Every numeric value in this example is `ASSUMPTION` and deliberately fictitious; it must never be used for a real vehicle. A real profile requires provenance, review status, recorded fixtures, decoder tests, and safe-mode behavior.

Profile capabilities include standard/extended frames, bus bitrate, bit numbering convention, byte order, sign, scale, offset, unit, destination, expected rate, validity, request/response definitions, ISO-TP addressing, source policy, metadata, and matching/detection. Unsupported schema fields cause a clear compatibility error, never partial silent interpretation.

## DBC and OpenDBC compatibility

- DBC is an interchange/import format for named CAN frames and signals. Import must normalize bit-numbering, endianness, multiplexing, attributes, units, value tables, and comments into the canonical schema.
- DBC does not by itself define this product's diagnostic scheduler, source health, fallback, GNSS integration, configuration, or profile signing/version policy.
- OpenDBC is not a runtime or build hard dependency. Selected DBC data may be imported only after its license, vehicle provenance, signal semantics, safety scope, and test fixtures are reviewed.
- Exporting a profile to DBC may be lossy; tools must report omitted project-specific fields.
- Custom profiles use the versioned canonical schema and may reference, but do not embed, third-party content without the required notices/permissions.

## Publication and consumption

The core exposes two conceptual views:

- a latest-value store for dashboards, shift-light, alarms, configuration/status, and low-rate queries;
- a bounded event stream for logging, protocols, replay, and rate-sensitive consumers.

Subscriptions declare channel set, maximum rate, maximum age, and delivery policy. Slow consumers receive coalesced latest values or explicit drop/gap events; they cannot block CAN/GNSS acquisition. The logger may request loss-aware delivery but must still fail independently when storage cannot keep up.

## Extensibility and compatibility rules

1. Add fields as optional with defined defaults; never reinterpret an existing field.
2. Version channel registry, profile schema, configuration schema, protocol schema, and log schema independently.
3. Preserve unknown fields through tooling when practical.
4. Use capability discovery before a client subscribes or writes configuration.
5. Deprecate before removal; firmware and tools declare supported version ranges.
6. Channel conversion occurs once at producer/profile boundary; layouts and outputs must not contain vehicle scale/offset formulas.
7. Derived channels identify their input channels, algorithm/version, latency, and validity propagation.
8. Safety-relevant alarms fail visibly on missing/stale data; they never substitute zero without an explicit rule.

## Open questions

- Exact in-memory numeric representation and memory budget on ESP32-S3.
- Registry governance, numeric wire IDs, extension namespace allocation, and localization of names/units.
- Exact quality scale and whether covariance/accuracy needs structured types.
- Canonical profile serialization and signing/update trust model.
- Required profile matching confidence before enabling diagnostic transmission.
- Log schema and lossless mapping between internal samples and first-party protocol messages.
