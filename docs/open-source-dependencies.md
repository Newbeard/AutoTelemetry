# Open-source dependency and reference review

Status: architecture research performed 2026-08-14. This is an engineering inventory, not legal advice or a guarantee. No third-party source was copied or integrated during this task.

## Policy

Before adding any dependency or imported data:

1. identify the exact repository, version/commit, files, author, and license;
2. verify that the license covers the artifact type (code, hardware, CAD, data, documentation, fonts/assets);
3. record required copyright/license/NOTICE attribution and source-offer/disclosure duties;
4. review transitive dependencies and generated code;
5. approve architecture, maintenance, security, memory, and test impact;
6. pin the version and preserve provenance in a dependency manifest;
7. isolate development tools from shipped firmware/product dependencies;
8. do not use “public on GitHub” as permission.

If no explicit license is found, treat copying, modification, linking, redistribution, and product integration as **not permitted pending permission/legal review**. Public documentation may be studied for interoperability, but implementation must be original and must respect applicable terms, trademarks, patents, and local law.

## Candidate matrix

| Project | Role/source | License found | Copy/link/modify assessment | Disclosure/notice | Preliminary disposition |
|---|---|---|---|---|---|
| RaceChrono DIY BLE API/reference examples | [RaceChrono tutorial](https://racechrono.com/article/2572), [aollin/racechrono-ble-diy-device](https://github.com/aollin/racechrono-ble-diy-device) | No repository license file or GitHub license designation found in the reviewed root on 2026-08-14 | Do not copy or derive example source without permission. Independently implement interoperability from the public protocol documentation after terms review. | Unknown; permission/legal review required before any copied material | Protocol/reference only; required interoperability tests, no source integration |
| `rc_can_ble` hardware | [Sergey1560/rc_can_ble](https://github.com/Sergey1560/rc_can_ble) | No explicit repository license found in reviewed root | Public availability does not grant reuse. Do not copy KiCad/CAD/BOM/docs. | Unknown | Reference only; its NRF52/MCP2515/SN65HVD230 architecture is not the Telemetry v1 design |
| `rc_can_ble_fw` | [Sergey1560/rc_can_ble_fw](https://github.com/Sergey1560/rc_can_ble_fw) | No explicit license confirmed | Do not copy/link/modify pending permission | Unknown | Reference only; not a firmware candidate |
| MagnusThome `esp32_obd2` | [MagnusThome/esp32_obd2](https://github.com/MagnusThome/esp32_obd2) | MIT | Linking/copying/modifying/redistribution permitted subject to MIT notice; review inherited/fork provenance and dependencies | Preserve copyright and MIT permission notice; no general source-disclosure requirement | Candidate reference or library; evaluate API, ESP-IDF fit, scheduler/coexistence behavior, tests and dependency age before selection |
| RejsaCAN upstream content | [MagnusThome/RejsaCAN-ESP32](https://github.com/MagnusThome/RejsaCAN-ESP32) and this fork | No root license file found locally or on the reviewed repository page | Existing fork/reference use continues, but redistribution/derivative rights are unclear; do not assume open-source permission | Permission/legal clarification required | Block distribution/licensing decision; preserve upstream files and attribution |
| ESP-IDF / TWAI driver | [espressif/esp-idf](https://github.com/espressif/esp-idf) | Apache-2.0 for ESP-IDF, with component/submodule notices to review | Generally permits commercial use, modification and distribution with license/notice conditions and patent terms | Preserve Apache license, notices and modification notices; review third-party components | Preferred framework/TWAI candidate, subject to explicit version and support-policy selection |
| `isotp-c` | [lishen2/isotp-c](https://github.com/lishen2/isotp-c) | MIT | Linking/copying/modifying permitted with notice | Preserve MIT notice; no general source-disclosure requirement | ISO-TP candidate; validate timing, addressing, concurrency, error handling, tests and maintenance before selection |
| `uds-c` / current `iso14229` | [driftregion/iso14229](https://github.com/driftregion/iso14229) | MIT | Linking/copying/modifying permitted with notice | Preserve MIT notice; no general source-disclosure requirement | UDS candidate, but upstream states 0.x API may change; pin/review and use only behind project abstraction |
| OpenDBC | [commaai/opendbc](https://github.com/commaai/opendbc) | MIT at reviewed repository root | Code/data reuse appears permissive under MIT notice, but each imported vehicle artifact still needs provenance, semantic, safety, and any third-party-rights review | Preserve MIT notice for copied substantial portions; no general source-disclosure requirement | Optional host-tool/profile input only; not a firmware/runtime hard dependency |
| SparkFun u-blox GNSS v3 | [sparkfun/SparkFun_u-blox_GNSS_v3](https://github.com/sparkfun/SparkFun_u-blox_GNSS_v3) | MIT for code; its license separately uses CC BY-SA 4.0 for SparkFun hardware | Code may be linked/modified with MIT notice; do not confuse hardware-license obligations with code | Preserve MIT notice for code; CC BY-SA obligations apply if hardware material is reused | GNSS library candidate if NEO-M9N/UART/25 Hz needs and memory footprint pass tests |
| u-blox `ubxlib` | [u-blox/ubxlib](https://github.com/u-blox/ubxlib) | Apache-2.0 with listed exceptions/third-party components | License permits use subject to notices, but repository states it was discontinued and archived in 2024 | Preserve Apache/third-party notices; exception audit required | Reference only by default because it is archived; do not choose without maintenance plan |
| SavvyCAN | [collin80/SavvyCAN](https://github.com/collin80/SavvyCAN) | MIT at reviewed root; bundled third-party attributions also exist | May be used/modified under applicable notices | Preserve MIT and bundled asset/component attributions if redistributed | Development/reference tool only; use for CAN capture, visualization, DBC work and replay conversion, not firmware |

## RaceChrono-specific conclusion

RaceChrono's official tutorial points to the BLE DIY repository and documents GPS, CAN-Bus, and Monitor APIs. The reviewed repository exposes protocol details but no explicit license. Telemetry v1 may implement an independent compatibility adapter from normalized telemetry to the documented API; it must not paste reference example code. RaceChrono names/marks remain third-party identifiers, and compatibility must be validated against current official behavior.

The RaceChrono adapter is optional. It cannot define channel semantics, profiles, logger format, configuration, or the first-party protocol.

## DBC/OpenDBC conclusion

DBC is a format/ecosystem, not one library. Tooling should accept reviewed DBC input and convert it to the canonical profile schema with explicit warnings for lossy or unsupported constructs. OpenDBC's reviewed root is MIT, but:

- license notice and exact commit must accompany imported substantial content;
- vehicle data provenance and correctness are not guaranteed by the software license;
- actuation/control content is outside this telemetry product's default scope;
- imported signals require recorded-fixture tests and manual semantic review;
- the firmware must not require the OpenDBC Python package or repository layout.

## Standards and specifications

ISO-TP, UDS, OBD-II, CAN, USB, and Bluetooth specifications may have access, copyright, membership, logo, qualification, or patent conditions independent of an implementation library. An open-source implementation does not grant the underlying standard text or certification marks. Use legitimately obtained specifications and do not copy standards text into the repository.

## Dependency architecture rules

- Wrap selected libraries behind project-owned interfaces so replacement does not change the telemetry core.
- Prefer libraries with deterministic memory, bounded work, maintained releases, tests, explicit license, and ESP32-S3/ESP-IDF support.
- Keep diagnostic libraries beneath the centralized scheduler; a library may not transmit independently.
- Keep GNSS libraries beneath the GNSS producer; library-specific types do not enter normalized channels.
- Keep RaceChrono encoding inside `firmware/racechrono/`.
- Record exact versions/checksums and licenses in a future machine-readable dependency manifest plus human-readable notices.
- Run license/source scanning in CI once code/dependencies exist, but retain human review for data/CAD/assets and ambiguous results.

## Unresolved licensing questions

- Distribution/derivative permissions for the upstream RejsaCAN hardware and repository content because no explicit root license was found.
- Permission/terms for copying any RaceChrono DIY example code; current plan avoids copying it.
- License status of `rc_can_ble` and `rc_can_ble_fw`; current plan avoids reuse.
- Exact product license(s) for new firmware, hardware, profiles, protocol schemas, tools, app, and CAD.
- Treatment of customer/vehicle captures, potentially identifying VIN/location data, and proprietary manufacturer signal information.
- Required Bluetooth qualification/listing, USB identifiers, application-store terms, and use of third-party names/logos.

All findings require re-verification at the exact version chosen for implementation.
