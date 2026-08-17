# Product test architecture

Status: Task 4.6 verification architecture. It defines evidence and fixture boundaries, not implemented tests or released pass limits.

## Principles

- Test the normalized telemetry contract before testing individual consumers.
- Use deterministic clocks and recorded inputs where timing, staleness, arbitration, or scheduling is involved.
- Keep public/synthetic fixtures separate from restricted vehicle captures; redact identifiers and personal location data.
- A passing software replay does not establish electrical, RF, EMC, thermal, automotive, or vehicle safety compliance.
- Every released vehicle profile, protocol revision, configuration migration, enclosure, and hardware revision has traceable compatibility evidence.
- Use [`test-reference-architecture.md`](test-reference-architecture.md) for the audited emulator/tool matrix, fixture ownership and staged host-to-HIL topology.

## Test layers

| Layer | Evidence required |
|---|---|
| Unit | Parsers, scaling, byte order, signedness, validity predicates, conversions, alarm states, and configuration migrations. |
| Normalized core | Timestamp ordering, validity/age, quality flags, source arbitration, priorities, hysteresis, fallback, and subscription behavior. |
| Vehicle profile | Schema validation, matching safety, declared support level, expected channels, rates, and fixture provenance. |
| Passive CAN replay | 11/29-bit frame decode at declared bit rate without transmitting; malformed, missing, duplicate, delayed, and bus-load cases. |
| Generic OBD-II | Request/response parsing, supported-PID bitmaps, unsupported PIDs, multiple ECUs, timeouts, negative/malformed responses, rate limiting and recovery through deterministic virtual and physical emulators. |
| ISO-TP/UDS | Segmentation, flow control, addressing, negative responses, pending responses, timeouts, scheduler budgets, and cancellation. |
| Vehicle coexistence | LISTEN_ONLY remains non-transmitting; DIAGNOSTIC_POLLING respects budgets and backs off when another tester or congestion is detected. |
| GNSS | Parser and rate fixtures, fix/accuracy transitions, monotonic/GNSS time mapping, stale fixes, reconnect, and privacy handling. |
| RaceChrono output | Adapter maps only normalized channels; reconnect, subscription/rate control, missing values, and compatibility capture. |
| First-party protocol | Capability discovery, independent schema/API/transport versions, BLE/Wi-Fi/USB parity, web/config authentication, interrupted atomic writes, OTA/profile rollback, malformed requests and backpressure. |
| Display | Manager/renderer/driver/layout separation, headless mode, unsupported display, partial updates, rate limiting, and disconnect. |
| Shift-light, status and alarms | Ten-pixel mapping/current limits, independent operation, source loss, sound waveform/acoustic acceptance, MODE behavior, rule hysteresis/debounce/latching, presentation failure, reset defaults and output rate limits. |
| Logger | Bounded buffering, media absence/full/removal, record/version integrity, power loss, and slow storage without starving acquisition. |
| Log parser/tools | Golden versioned logs, malformed/truncated records, integrity failures, schema migration, unknown channels/fields, and exports independent of firmware structs. |
| Optional lap engine | Explicit enablement, GNSS-quality gates, deterministic geometry, and no coupling to core acquisition. |
| Fault injection | Queue saturation, task stall/crash, corrupted input/config, peripheral timeout, heap/storage pressure, watchdog escalation, and isolated restart. |
| Power/sleep | Wake causes, shutdown ordering, state persistence, brownout recovery, parked-current states, and no unintended transmission during transitions. |
| Hardware-in-loop | CAN transceiver/mode behavior, timing/load, USB paths, GNSS UART, display/shift/alarm outputs, SD, sleep/wake, and programmed fault states. |
| Electrical/mechanical | Power transient, thermal, ESD/EMC, RF, cable, connector, vibration, enclosure fit, and vehicle-mount safety plans with separately approved limits. |

## Repository and fixture model

```text
tests/
  fixtures/        # synthetic or redistributable CAN/GNSS/protocol/config data
  unit/            # future host-side module tests
  integration/     # future cross-module deterministic tests
  replay/          # future CAN/GNSS replay harnesses
  emulator/        # future project-owned scenarios/adapters
  hil/             # future hardware-in-loop definitions and evidence links
```

Every fixture manifest should state origin, permission/license, vehicle/profile applicability, capture tooling, timestamp model, redaction, expected results, and integrity hash. Raw proprietary DBCs or customer captures must not be committed without redistribution permission.

## Vehicle-profile qualification

1. Validate the profile schema and configuration-version compatibility.
2. Verify matching rules cannot select the profile from weak or ambiguous evidence without confirmation.
3. Replay known-good and adversarial fixtures in LISTEN_ONLY mode.
4. Compare each decoded channel against an independent reference and record unit, range, rate, and validity behavior.
5. If diagnostics are declared, test addressing and request/response definitions on a simulator or bench before a vehicle.
6. Verify scheduler rate/bus-load budgets, timeout/backoff, and coexistence with another scan tool.
7. Verify source arbitration when generic OBD, passive CAN, diagnostics, and GNSS provide overlapping channels.
8. Verify downstream RaceChrono, logger, display, shift-light, alarms, and first-party protocol consume only normalized data.
9. Perform controlled in-vehicle validation with a rollback/disable path and no safety-critical actuation.
10. Record evidence, known limitations, supported model-year/trim scope, profile revision, and reviewer approval.

## Timing and overload

Tests use an injectable monotonic clock and explicit scheduler seeds. Acceptance cases cover counter wrap, timestamp discontinuity, delayed callbacks, and stale-data transitions. Producer/consumer queues are bounded; tests prove the declared drop/coalesce policy and show that slow storage, display, BLE, or app clients cannot block CAN/GNSS acquisition.

## CI and release gates

- Documentation/schema validation can run for architecture-only changes.
- Firmware entry requires unit, core-contract, profile-schema, config-migration, and protocol compatibility gates.
- A vehicle profile cannot be marked supported without qualification evidence.
- Hardware release requires separately approved ERC/DRC, BOM, electrical, RF, mechanical, and manufacturing reviews.
- Failures must identify the artifact revision, fixture, seed, configuration, and platform sufficiently for reproduction.

## Unresolved inputs

- Test framework, project-owned emulator scenario schema, host language, CI platform, coverage policy, and resource budgets.
- Legal/redistribution status and provenance of future CAN/GNSS fixtures.
- Bench/HIL equipment and authoritative comparison instruments.
- Vehicle access, safety procedure, acceptance tolerances, and compliance test plans.
