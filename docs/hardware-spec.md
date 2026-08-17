# Telemetry v1 frozen hardware specification

This is the Task 4.7 automotive-electrical, component and interface research freeze, not an approved schematic, PCB release, or automotive-safety claim. Exact ordering codes, qualification, exceptions, and source evidence are in [`component-freeze.md`](component-freeze.md); circuit boundaries are in [`schematic-architecture.md`](schematic-architecture.md).

Task 5A status: **BLOCKED before KiCad project or schematic-sheet creation**; no Task 5A schematic capture, ERC run or review PDF was produced. The capture gate remains closed because the LM74502-Q1 guaranteed UV thresholds have no overlap between 6 V operation and USB/vehicle crossover, its guaranteed OV thresholds have no overlap between 18 V operation and cutoff by 20 V, SM8SF24CA-Q has 24 V `VRWM` against the required +26 V/60 s jump-start, the 2 A fuse needs a defined low-voltage/hot load policy or must be reopened, DMT6007LFGQ-7 negative-pulse stress remains unresolved, and AUX5 plus the damped input filter still require qualification. See [`task5a-power-calculations.md`](task5a-power-calculations.md).

Telemetry v1 is limited to a 12 V passenger-car OBD device with one Classical CAN channel, protected power and parked wake/sleep, ESP32-S3, onboard GNSS, microSD, USB-C, BLE, an interchangeable external display, a short-cable shift light, onboard audible alarm, MODE input, status indication, and debug. TPMS, tire-temperature sensing, IMUs, analog sensor hubs, external sensor networks, and a second CAN channel are explicitly outside this revision.

## Processing and storage

- Freeze ESP32-S3-WROOM-1-N16R8: 16 MB Quad-SPI flash and 8 MB Octal-SPI PSRAM (`VERIFIED_DATASHEET`). PSRAM ECC is mandatory because Espressif limits R8 operation without ECC to +65 °C; ECC consumes 1/16 of PSRAM, leaving about 7.5 MB (`CALCULATED`). This does not make the module automotive-qualified.
- Freeze microSD in SPI mode with CS on GPIO11 and card detect through the TCA6408A expander. GPIO45 remains unloaded as a boot strap.

## CAN/OBD

- Freeze TCAN3404DRQ1, SOIC-8, connected to native TWAI. The silicon is CAN FD-capable, but Telemetry v1 protocol scope is Classical CAN only.
- OBD-II: 16 = vehicle battery, 4/5 = ground, 6 = CAN-H, 14 = CAN-L.
- Optional split 120 Ω termination is DNP/OFF by default (`DESIGN_REQUIREMENT`). `CALCULATED`: another 120 Ω across an already terminated 60 Ω bus produces 40 Ω effective loading.
- Freeze ESDCAN04-2BWY at the connector. Provide an ACT45B-510-2P-TL003 footprint, DNP by default with 0 Ω bypasses.

## GNSS

- Freeze NEO-M9N-00B on switched ≥200 mA GNSS_3V3, UART at 230,400 bit/s, target 20–25 Hz. It is professional-grade, not automotive-qualified; this is an accepted prototype risk, not a qualification claim.
- Freeze Hirose U.FL-R-SMT-1(10) for the prototype and active Littelfuse AQ3118E-01ETG at the connector; use a controlled 50 Ω RF path. The earlier ST ESDAXLC6-1BT2Y is NRND.
- Freeze the u-blox R10 bias-T topology with 100 nF supply filtering and a 27 nH choke meeting the published RF impedance/current guidance. Reopen the earlier 22 Ω/136 mA limiter: Task 5 must select a passive or active current limiter against VCC_RF/module impedance and the exact 5–20 mA antenna. The antenna remains a schematic-release blocker.
- Provide GNSS TX/RX and power test points away from the RF trace.
- Tie V_BCKP to switched GNSS power for v1 and investigate u-blox host save/restore (`DESIGN_REQUIREMENT`); provide isolation/DNP flexibility only if schematic review finds it low risk.

## External display

- Initial display: GC9A01, 240×240, 4-wire SPI; future ST7789/AMOLED supported by software and connector policy.
- Freeze a 14-position logical connector contract: two GND, DISP_3V3, DISP_5V, SCLK, MOSI, MISO, CS, DC, RESET_N, BL_PWM/EN, SDA, SCL, and INT/TE. The physical family remains a mechanical schematic-release blocker.
- Freeze 2N7002KQ open-drain backlight control, cable ≤200 mm, and initial SPI clock ≤20 MHz.
- Do not expose raw vehicle battery. DISPLAY_3V3 capacity is ≥400 mA and DISPLAY_5V capacity is ≥600 mA within AUX5 (`DESIGN_REQUIREMENT`); incompatible rails must be keyed/configured.

## Outputs

- Freeze a ten-pixel, ≤0.5 m removable shift-light contract. Qualify the WS2812-compatible load to 0.50 A while retaining the existing 1 A TPS1H100-Q1 protected fault envelope, CAHCT1G126-Q1 5 V data buffer, connector-side ESD and ≥1.5 A connector rating.
- Approve TPA2005D1TDGNRQ1, the −40…+105 °C order code, replacing the low-side buzzer MOSFET and driving one onboard 8 Ω, ≥1 W speaker from a 300 mA AUX5 branch. Exact speaker, acoustic port and enclosure volume remain provisional.
- Approve LP5814DRLR on MAIN_3V3 for one common-anode RGB status LED. It shares I2C; expander P6 becomes `STATUS_DRV_EN`. Exact LED and optical implementation remain provisional.
- MODE remains GPIO10, button-to-ground, 47 kΩ starting pull-up, separate from RESET and BOOT. User-visible behavior and safe configuration entry are controlled by [`user-io-configuration.md`](user-io-configuration.md).

- Neither the external shift load nor the onboard sounder is powered directly by an ESP32 GPIO. Longer/noisier shift installations require a different intelligent or differential module and are outside v1.

## Automotive input and ground

- The Task 4.7 historical candidate chain was `VBAT_OBD_RAW -> 0437002A -> SM8SF24CA-Q -> LM74502QDDFRQ1 + 2×DMT6007LFGQ-7 -> damped post-switch filter -> VEHICLE_PROTECTED`. Task 5A reopens its threshold implementation, TVS/FET/fuse coordination and exact filter population; it is not approved for capture.
- Retain 6–18 V full operation, +26 V/60 s jump survival, +38 V suppressed-load-dump survival, −14 V/60 s reverse survival and measured ≤24 V `VEHICLE_PROTECTED` as `DESIGN_REQUIREMENT` inputs. The frozen LM74502-Q1/SM8SF24CA-Q implementation cannot meet all of them with the required USB crossover, so no exact UV/OV values or substitute parts are approved. Severe unsuppressed load dump remains outside the v1 guarantee.
- Freeze controlled reset/automatic recovery during crank; full ride-through is not required.
- OBD4 and OBD5 join once at entry into one continuous `POWER_GND`; high-current/ESD return geometry is controlled without split ground planes.
- Retain a damped post-switch filter as the architectural direction, but reopen exact L/C/R, leakage, UV/OV, `Cdvdt`, fuse/SOA and EMI values pending the Task 5A decisions.

## Expansion and debug

- USB-C USB 2.0 device/Serial-JTAG stays on GPIO19/20. Freeze USBLC6-2SC6Y ESD, TPS2553QDBVRQ1 source limiting with 43.2 kΩ ILIM, and PMEG6030EP-Q reverse isolation. USB-only configured load remains ≤500 mA. GCT USB4105 is provisional pending mechanical confirmation.
- I2C 3.3 V SDA/SCL plus GND; connector power and pull-up ownership must be explicit.
- Preserve UART0 GPIO43/44 pads as optional serial debug/recovery access; USB remains primary.
- Test points: GND, protected vehicle input/VCC, 3.3 V, each switchable rail, CAN-H, CAN-L, CAN_RX, CAN_TX, GNSS TX/RX, and GNSS rail. RF test provisions require RF-review approval.

## Mechanical and environmental

- RejsaCAN v3.4 reference outline is approximately 31.50 mm × 49.53 mm (`CALCULATED` from repository PCB edge coordinates) with an antenna-end notch; Telemetry v1 dimensions remain `TBD` and will grow for GNSS/RF and connectors.
- RF connector/antenna cable, OBD strain relief, display cable, enclosure, ventilation, ingress, vibration, and service access are TBD.
- Connector mechanics, enclosure temperature, exact purchased-standard/OEM test severities, ESD/EMC setup, and product qualification targets remain validation blockers. Unaffected Task 4.7 system requirements remain inputs, while the Task 5A vehicle-input and AUX5 implementation decisions above are explicitly reopened.

## Product-interface implications

- Hardware exposes capabilities; it does not encode a vehicle, RaceChrono, display layout, alarm rule, logger format, or companion-app transport into the electrical design.
- CAN mode control must permit a verifiable non-transmitting LISTEN_ONLY state and a supervised DIAGNOSTIC_POLLING state. Exact TCAN3404-Q1/TWAI reset, standby, and failure behavior remains a schematic/firmware verification item.
- Independent failures of display, shift-light, buzzer, SD, GNSS, USB/client connectivity, or switched peripheral rails must not cause uncontrolled CAN transmission.
- External interfaces must support the bounded-queue, timeout, health, and isolated-restart architecture in [`architecture.md`](architecture.md); hardware watchdog/reset-domain implications remain unresolved.
- Enclosure, display, and vehicle-mount CAD must consume controlled PCB/connector/keep-out drawings and the limits in [`enclosure-architecture.md`](enclosure-architecture.md), not create new electrical requirements implicitly.
