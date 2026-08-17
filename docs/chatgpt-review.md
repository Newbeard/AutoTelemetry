# Telemetry v1 Task 4.6 review

## Task completed

Completed the documentation-only User I/O, Configuration, Tire Module Provision and Test-Reference Freeze. Researched primary manufacturer material and relevant open-source projects; froze the ten-pixel shift-light contract, MODE behavior, centralized configuration/recovery model, OTA/profile-update principles, future tire-data boundary and layered test strategy. Two hardware changes are explicitly proposals pending review. No schematic, PCB, firmware, CAD, manufacturing file or third-party code was created or modified.

## Files changed

- `CODEX.md`: added the permanent research, licensing and reuse-classification rule.
- `docs/user-io-configuration.md`: created the detailed User I/O, configuration, OTA, profile, tire and dual-CAN decision record.
- `docs/test-reference-architecture.md`: created the reference-project audit and host/replay/emulator/HIL strategy.
- `docs/requirements.md`: added Task 4.6 requirements and updated output envelopes.
- `docs/architecture.md`: integrated the configuration, update, tire-module and second-CAN boundaries.
- `docs/hardware-spec.md`: recorded the output contracts and proposed sound/status changes.
- `docs/interfaces.md`: updated the shift/sound/status interface contracts.
- `docs/power-budget.md`: replaced the informal LED estimate and recalculated AUX5.
- `docs/power.md`: reconciled the rail summary and input-power calculation.
- `docs/power-wake-review.md`: reconciled the output comparison and remaining selection gates.
- `docs/telemetry-data-model.md`: reserved future normalized tire concepts.
- `docs/device-protocol-architecture.md`: defined shared configuration, OTA and profile-update ownership.
- `docs/open-source-dependencies.md`: added project/license/classification findings.
- `docs/enclosure-architecture.md`: added removable shift-light and sound/status mechanical constraints.
- `docs/test-architecture.md`: added deterministic emulator and update/configuration gates.
- `docs/component-freeze.md`: recorded proposed components and corrected input-power calculations.
- `docs/schematic-architecture.md`: updated controlled pre-schematic requirements only; no schematic was created.
- `docs/telemetry-v1-change-list.md`: reconciled the Task 4.6 output changes/proposals.
- `docs/chatgpt-review.md`: replaced the Task 4.5 packet with this Task 4.6 review.

## Engineering findings

### Frozen current-v1 decisions

1. Prefer the Worldsemi WS2812 V6-class family for the shift light; freeze the family/function, but keep the exact ordering code `PROVISIONAL` until a controlled English data sheet, availability and flex-assembly evidence are captured.
2. Exactly ten pixels are required. At 12 mA/color, all-white current is `10 × 3 × 12 mA + ≤0.010 mA = 360.010 mA`; 25% margin gives 450 mA, rounded to a 0.50 A qualified load. A representative one-color/50%-PWM pattern is 60.010 mA.
3. Retain `CAHCT1G126-Q1` at 5 V, safe-disabled with the output rail, a 33 Ω starting series resistor (22–47 Ω measurement range), connector-side low-capacitance ESD and local 100 nF/pixel.
4. Freeze the external contract as `SHIFT5`, `SHIFT_DATA_5V`, `GND`, cable ≤0.5 m, ≥26 AWG power/ground (24 AWG preferred), ≥28 AWG data, connector ≥1.5 A, qualified load 0.50 A and retained TPS1H100-Q1 1 A protection/fault envelope.
5. AUX5 named load is `0.50 + 0.50 + 0.30 = 1.30 A`; with 25% margin it is 1.625 A, so the frozen 2 A converter remains sufficient on paper.
6. MODE remains GPIO10, non-strap, button-to-ground with a 47 kΩ starting pull-up. Short press acknowledges/silences an active alarm or advances local indication; ≥3 s requests configuration only in a safe state. It never silently erases configuration or enables CAN transmission.
7. RESET/EN and BOOT/GPIO0 remain distinct recessed service controls. BOOT retains its strap-only role.
8. Alarm rule, alarm state and presentation remain separate. Thresholds, persistence/hysteresis, mute/acknowledgement and STREET/TRACK presentation are configuration, never global hard-coded constants.
9. Configuration Manager is the only persistent-settings owner: schema-versioned validation/migration, immutable runtime snapshots, atomic stage/commit, last-known-good recovery, safe defaults, explicit factory reset and secret redaction.
10. Web configuration is a physically requested, authenticated, 10-minute-inactivity SoftAP session with explicit local URL/mDNS; Wi-Fi is normally off and turns off on exit. A captive portal is not required. Normal CAN/GNSS/alarm work continues with bounded resources.
11. OTA uses ESP-IDF HTTPS OTA, server-certificate validation, signed image/manifest, compatibility checks, inactive-slot programming, pending-image self-test and rollback. Anti-rollback eFuses wait for a qualified key/manufacturing/recovery process.
12. Vehicle profiles may update independently only as declarative, non-executable, authenticated artifacts with schema/runtime compatibility, staging/dry-run, atomic activation, last-known-good rollback and built-in Generic OBD fallback. The package format remains open.

### Future-module reservations

13. Reserve a separate future AutoTelemetry Tire Module that produces four-wheel pressure, overall/internal temperature, left-to-right tread-temperature arrays and health; the main unit normalizes and forwards them to RaceChrono, logger, display and app.
14. Telemetry v1 needs no current hardware change for Tire Module support. Evaluate BLE/ESP-NOW first; wired UART/RS-485/CAN remains a later module/system decision.
15. A second CAN controller is not justified in Telemetry v1. ESP32-S3 has one TWAI instance and all present requirements use one vehicle bus.
16. If two independent buses become necessary, prefer a separate tire/sensor gateway first. A later main-board redesign must explicitly compare ESP32-C6 dual TWAI against ESP32-S3 plus an external controller; C6 does not preserve the frozen S3 N16R8 resource architecture.

### Open-source/reference findings

17. Keep the following in project documentation: MagnusThome `ESP32_OBD2_Emulator` as unlicensed `REFERENCE_ONLY` basic smoke-test concept; `esp32_obd2` as MIT `REFERENCE_ONLY`/possible later dependency; `ESP32S3RET` as MIT development reference; the local C6 dual-CAN example and Autosport Labs ESP32-CAN-X2 as unclear-license `REFERENCE_ONLY`; `lbenthins/ecu-simulator` as the first MIT development-tool candidate; Ircama ELM327-emulator as non-commercial evaluation only; archived MPL-2.0 `limiter121` as reference only; `mdabrowski1990/uds` and SavvyCAN as possible MIT development tools. No code was copied.

Magnus's simple emulator is suitable only for an initial physical-CAN OBD smoke test. Deterministic project-owned scenarios plus SocketCAN/replay, then a physical emulator and later HIL, are required for supported-PID, multi-ECU, malformed, timeout, ISO-TP and UDS qualification.

RaceChrono publicly documents per-position tread names `Tyre temperature <position> 1…8` from left to right. No authoritative public canonical tire-pressure name was found; capture and verify it against the target app release before freezing an adapter mapping.

## Decisions made

- The 17 numbered recommendations above are the Task 4.6 decisions.
- The shift-light exact load is frozen without changing its already frozen protection silicon or GPIO.
- The existing 2 A AUX5 silicon remains frozen after recalculation.
- No Tire Module electronics, additional main-board reservation, second CAN controller or MCU change is approved.
- Open-source projects remain references/tools unless a later dependency review explicitly changes classification.

## Assumptions

- The proposed 1 W sound calculation assumes 75% end-to-end amplifier efficiency; `1 W/(5 V×0.75)+2.8 mA = 269.5 mA`, rounded to 300 mA.
- Cable-drop screening uses Belden 1213A 26 AWG at 0.146 Ω/m, a 0.5 m one-way run, 25% resistance allowance and 50 mV contacts: approximately 0.141 V at 0.5 A.
- The realistic shift pattern uses one color channel at 50% PWM on all ten pixels; actual patterns are configuration-dependent.

## Uncertainties / unresolved questions

- Exact Worldsemi V6-class pixel/order code and controlled data sheet; exact shift connector, ESD part, flex stack-up, diffuser and adhesive.
- Exact 8 Ω speaker, acoustic opening/back volume, ingress treatment and measured in-cabin STREET/TRACK acceptance.
- Exact common-anode RGB LED, optical path and the acceptability of a non-AEC LP5814 in this product environment.
- Full configuration ownership/threat model, credential lifecycle, serialization, firmware/profile manifest formats and signing-key operations.
- RaceChrono tire-pressure naming and tread overlay behavior in the target application release.
- Tire-module transport and any future justification for two independent CAN buses.
- Project-owned emulator scenario format, fixture redistribution permissions and HIL equipment.

## Risks

- Addressable-pixel variants share family names but differ in thresholds, current, temperature and quiescent behavior; a substitute is not qualified by name alone.
- A 1 A protection setting does not qualify a strip above the 0.50 A load contract; software brightness limiting is not a substitute for hardware qualification.
- Maximum sound pressure depends strongly on the chosen speaker and enclosure. No motorsport audibility claim exists until measured in representative cabins.
- SoftAP, OTA and remotely writable profiles enlarge the attack surface and can affect RF/power scheduling; implementation needs explicit security and resource gates.
- Unlicensed or non-commercial reference code cannot be incorporated merely because it is technically useful.
- RaceChrono channel names are compatibility behavior, not the canonical internal data model.

## GPIO / peripheral changes

No MCU GPIO number changes are proposed. GPIO6 remains shift data, GPIO10 MODE, GPIO17 sound waveform/control, GPIO0 BOOT and EN RESET. `PROPOSED CHANGE`: TCA6408A P6 changes from direct `STATUS_LED_N` to `STATUS_DRV_EN`; LP5814 joins the existing I²C bus and drives one common-anode RGB LED. P7 remains reserved.

## Power / CAN / RF impact

The shift-light is a 0.50 A qualified load with a separate retained 1 A protected fault envelope. The proposed speaker amplifier uses a 300 mA AUX5 branch. The recalculated 1.625 A AUX5 requirement remains below the frozen 2 A rating. The simultaneous named-rail input calculation becomes 12.458 W, 1.298 A at 12 V and 80% assumed efficiency, leaving 54.1% arithmetic margin to the 2 A fuse before inrush/time-current/thermal qualification.

There is no change to the one-channel Classical CAN architecture, transceiver, termination default, ESP32-S3, GNSS RF path, BLE/Wi-Fi antenna or USB isolation. Configuration-mode Wi-Fi coexistence and RF/power load must be measured later.

## Proposed hardware changes

- `PROPOSED CHANGE`: replace the frozen `2N7002KQ-7` buzzer MOSFET/`BAS21WQ` clamp concept with `TPA2005D1-Q1`, retaining GPIO17, and use a provisional onboard 8 Ω, ≥1 W speaker. Reason: programmable tone/volume and materially stronger acoustic potential with protected BTL drive.
- `PROPOSED CHANGE`: add `LP5814DRLR` on MAIN_3V3 and repurpose expander P6 to `STATUS_DRV_EN` for one provisional common-anode RGB LED. Reason: independent current/PWM control without consuming three MCU GPIOs.
- Neither proposal is approved for schematic capture by this document.

## Datasheets / primary sources consulted

- Worldsemi WS2812 family page and WS2812B-2020 v1.3 data sheet; OPSCO/DigiKey SK6812MINI-E data; Brightek official 2020 ICLED product pages.
- TI `CAHCT1G126-Q1`, `TPS1H100-Q1`, `TPA2005D1-Q1` and `LP5814` product data; PUI AS01808AO reference speaker data; Belden 1213A cable data.
- Espressif ESP-IDF ESP32-S3 TWAI, HTTPS OTA, OTA, Secure Boot v2 and Wi-Fi security documentation; ESP32-C6 data sheet.
- RaceChrono official DIY-device tutorial and maintainer tire-channel guidance.
- Exact upstream repository pages and license files recorded in `open-source-dependencies.md` and `test-reference-architecture.md`.

## Recommended next step

Conduct one formal Task 4.6 architecture review and explicitly approve or reject the two proposed sound/status hardware changes before authorizing schematic capture.

## STOP condition

Task 4.6 is complete. Stop here and wait for review; do not begin schematic/PCB work, firmware, Web UI, OTA, Tire Module electronics, flex layout, enclosure CAD or manufacturing output.
