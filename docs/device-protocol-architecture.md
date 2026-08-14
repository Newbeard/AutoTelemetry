# First-party device protocol architecture

Status: capability and layering proposal. No wire format, BLE service, Wi-Fi port, USB class, cryptographic suite, or compatibility promise is frozen.

## Goals

The first-party protocol provides one product API independent of RaceChrono and independent of transport. It supports a future companion application, command-line tools, manufacturing/service tools, and automated tests without exposing firmware memory layouts.

```text
normalized telemetry core / configuration / logger / update manager / health
                               │
                     versioned service model
                               │
                 framing + request/event semantics
                  ┌────────────┼────────────┐
                  ▼            ▼            ▼
                 BLE         Wi-Fi         USB
```

Transport adapters carry the same logical operations but may use different framing, MTU, discovery, flow control, and authentication. RaceChrono remains a separate adapter and does not share first-party internal messages.

## Capability groups

| Service | Conceptual capabilities |
|---|---|
| Device | identification, hardware/firmware versions, serial/revision, uptime, reset reason |
| Capabilities | supported protocol/schema versions, transports, channel registry, profile/config/log/update features and limits |
| Telemetry | channel catalogue, subscribe/unsubscribe, snapshot, streaming, requested rate/age, gap/overrun reporting |
| GNSS | normalized GNSS subscriptions plus fix/time/accuracy metadata; no dependency on RaceChrono encoding |
| Configuration | schema discovery, read, validate, transactional write, migration result, rollback/status |
| Vehicle profiles | list, inspect metadata, validate compatibility, upload/stage, activate, rollback, delete under policy |
| Display | capabilities, type/layout/brightness/configuration; headless state |
| Shift-light | source, thresholds, brightness/day-night settings and test hook subject to safe-state policy |
| Alarms | rules, enablement, state, acknowledgement and presentation settings |
| Sessions/logs | list, metadata, begin/end where allowed, range/chunk download, integrity, delete under policy |
| Update | firmware metadata, compatibility/preflight, staged upload, integrity/authenticity verification, activate/rollback hooks |
| Diagnostics/status | service health, queue/drop counters, CAN mode, active source map, power state, errors and support bundle |

Capabilities are discoverable. A client must not infer feature support from a model name or firmware version string.

## Message semantics

The future protocol requires:

- request/response correlation and explicit typed errors;
- asynchronous events/streams;
- operation idempotency or idempotency keys for retryable writes;
- bounded payloads and chunking for profiles, logs, and firmware;
- stream sequence numbers, timestamps, loss/gap reporting, and backpressure;
- cancellation and timeout semantics;
- atomic configuration/profile activation separate from upload/staging;
- integrity metadata for transferred artifacts;
- diagnostics that expose unsupported version/capability rather than silently ignoring required fields.

The serialization decision remains open. Evaluation must compare binary schema evolution, code/RAM/flash cost, tooling support, deterministic bounds, unknown-field behavior, transport overhead, and licensing. Firmware C/C++ structs are not a protocol.

## Telemetry subscriptions

A subscription conceptually declares:

- channel IDs or groups;
- snapshot plus/or streaming mode;
- maximum requested rate and acceptable age;
- change-only/deadband option where semantically valid;
- inclusion of source/quality/provenance;
- batching preference within transport limits.

The device accepts, rejects, or clamps each request against capability and resource limits and reports the effective subscription. A slow client receives coalesced latest values or explicit gaps; it cannot backpressure the telemetry core. GNSS uses the same normalized subscription semantics even if a transport later optimizes its representation.

## Versioning and backward compatibility

Version these independently:

1. logical protocol/service version;
2. transport binding/framing version;
3. channel registry version;
4. configuration schema version;
5. vehicle-profile schema version;
6. log format version;
7. firmware/update manifest version.

Compatibility rules:

- negotiate a mutually supported major/minor range before state-changing operations;
- incompatible major versions fail clearly;
- additive optional fields/capabilities do not change existing meanings;
- required fields are explicitly marked;
- unknown optional fields are ignored or preserved as specified, never reinterpreted;
- deprecation is advertised before removal;
- logs/profiles/configuration carry their own schema versions;
- golden compatibility fixtures cover at least the current and supported previous release contracts.

No version number in this document is a frozen wire value.

## Transport considerations

### BLE

Suitable for pairing, configuration, moderate-rate telemetry, and status. The binding must handle MTU variation, connection interval, notification credits, reconnection, mobile background behavior, and coexistence with the separate RaceChrono BLE service. Advertising/service discovery must expose capabilities without leaking sensitive vehicle data.

### Wi-Fi

Suitable for high-rate telemetry, logs, profile/firmware transfer, and development. AP/client behavior, discovery, TLS, credentials, local-network trust, and power policy remain unresolved. Wi-Fi absence must not affect acquisition or local outputs.

### USB

Suitable for deterministic development/service access, logs, configuration, and recovery. The binding must coexist with native ESP32-S3 USB debugging/update needs and the hardware power-isolation policy. USB presence is not automatically administrative authorization.

## Security and safety boundary

Writable remote operations create a security and vehicle-safety boundary. Before implementation, define a threat model covering unauthorized telemetry access, profile/config tampering, malicious diagnostic schedules, firmware rollback, credential loss, BLE/Wi-Fi pairing, USB physical access, denial of service, and privacy of recorded location/vehicle data.

Mandatory architectural rules:

- authentication, authorization, transport confidentiality, artifact signing, and key provisioning are explicit decisions, not deferred defaults;
- entering `DIAGNOSTIC_POLLING` or changing diagnostic profiles requires policy authorization;
- update/profile/config artifacts are verified before activation and support last-known-good rollback;
- firmware update cannot run while vehicle/power state makes it unsafe;
- protocol clients cannot directly transmit arbitrary CAN frames through the production API;
- diagnostic/development escape hatches, if any, are separately gated, visible, and disabled in production configuration;
- log/profile deletion and factory reset are explicit destructive operations;
- secrets and private keys never appear in logs or exported support bundles.

## Resource isolation

Protocol handling is an optional output/service. It uses bounded queues, per-client subscription/resource limits, transfer windows, and timeouts. Malformed messages, a stalled transfer, an excessive subscription, or repeated reconnects must not block CAN/GNSS acquisition, local alarms, shift-light, display, or shutdown.

## Session and log portability

Log downloads return a documented versioned artifact plus integrity metadata. The future application and `tools/log-tools/` parse public schemas rather than firmware internals. Session metadata may reference vehicle/profile/config/channel-registry versions and clock mapping without requiring the exact firmware binary that recorded it.

## Update hooks

The API reserves discover/preflight/stage/verify/activate/status/rollback operations. It does not define OTA implementation. Update architecture must later specify signed manifests, compatible hardware revisions, anti-rollback policy, A/B or recovery behavior, power-loss safety, bootloader trust, and factory recovery.

## Open questions

- Serialization/framing and maximum resource budgets.
- BLE service coexistence and mobile-platform constraints.
- Wi-Fi topology and secure discovery.
- USB class/binding alongside Serial/JTAG and recovery.
- Device identity, provisioning, authentication, authorization, and ownership transfer.
- Firmware/profile trust roots and offline recovery.
- Supported compatibility window and deprecation cadence.
- Whether third-party clients receive a stable public SDK or only protocol schemas.
