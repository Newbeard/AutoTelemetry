# Telemetry v1 frozen hardware specification

This is the Task 5A.1 conditional automotive-electrical, component, and interface re-freeze for prototype schematic capture. It is not a tested design, PCB/production release, or automotive/standards-compliance claim. Exact ordering codes and status are in [`component-freeze.md`](component-freeze.md); authoritative Task 5A.1 source evidence, equations, filter values, and validation gates are in [`task5a1-power-architecture.md`](task5a1-power-architecture.md).

Task 5A.1 status: **conditionally re-frozen for exact prototype capture**. The prior input-power block is superseded. Capture must preserve the named parts, orientations, thresholds, source boundaries, AUX policy, load shedding, and authoritative filter values without substitution. Hardware testing, ERC, schematic review, PCB review, and every documented laboratory gate remain required before any stronger release status.

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
- Conditionally freeze LMQ66420MC5RXBRQ1 for AUX5 under STREET=0.56001 A, TRACK=0.92001 A, and MAX=1.16001 A. MAX is limited to ≤10 s and ≤25% rolling 60 s; 2 A continuous operation is not claimed.
- Approve LP5814DRLR on MAIN_3V3 for one common-anode RGB status LED. It shares I2C; expander P6 becomes `STATUS_DRV_EN`. Exact LED and optical implementation remain provisional.
- MODE remains GPIO10, button-to-ground, 47 kΩ starting pull-up, separate from RESET and BOOT. User-visible behavior and safe configuration entry are controlled by [`user-io-configuration.md`](user-io-configuration.md).

- Neither the external shift load nor the onboard sounder is powered directly by an ESP32 GPIO. Longer/noisier shift installations require a different intelligent or differential module and are outside v1.

## Automotive input and ground

- Conditionally freeze the main-current path `VBAT_OBD_RAW -> 0437002.WRA -> VBAT_FUSED_CLAMPED -> LM74720QDRRRQ1 + 2× back-to-back STL125N10F8AG -> shared damped post-switch filter -> FILTERED_VEHICLE`. At `VBAT_FUSED_CLAMPED`, the common-anode `SM15T47AY` + `SM15T33AY` pair is a shunt branch to `POWER_GND`: the 47 V-leg cathode connects to fused VBAT, both anodes connect at `TVS_MID`, and the 33 V-leg cathode connects to ground.
- `FILTERED_VEHICLE` feeds both frozen `LMQ66420MC3RXBRQ1` and normally-off, conditional `LMQ66420MC5RXBRQ1`. The corrected maximum simultaneous case is `3.3 V×0.750 A + 5 V×1.16001 A = 8.27505 W`. A separate 3.3 V-display case loads the MC3 output to 1.050 A/3.465 W and produces 6.765 W total rail power; 1.313 A is sizing margin only, and 1.050 A controls MC3 thermal validation. The 0.750 A vehicle 3.3 V bucket includes the direct vehicle-only TCAN branch; physical muxed `MAIN_3V3` excludes CAN.
- Use two 249 kΩ OV-top resistors in series and 28.0 kΩ bottom, all 0.1%. With the explicit ±1 µA leakage model, the rising OV/PD-low command is 20.690–25.531 V and falling reconnect eligibility is 18.815–23.366 V. This keeps the path on through 18 V, commands opening before +26 V, and permits reconnect after return to 18 V; completed switching, overshoot, inrush and recovery time require capture.
- The previous cutoff-by-20 V and protected-node-≤24 V limits are deliberately superseded. Review downstream input ratings against the 25.531 V modeled endpoint plus measured dynamic overshoot.
- The LM74720/FET path provides true reverse-current blocking while enabled. AUX5 MAX is prohibited below 10 V; its authoritative 10 V screen is 0.9064 A nominal/0.9545 A with sensitivity. Below 9 V for 100 ms the permitted steady state is CORE/SHED, 0.1211 A nominal at 6 V. The 6 V TRACK 1.0535/1.1102 A result is a pre-shed transient screen only. Retain the 2 A fuse only with enforced shedding and complete pulse/fault validation.
- Public-data screens cover 12/14.4/18 V, +26 V/60 s, the defined suppressed +38 V source, +50 V/2 Ω fast reference, −100 V/10 Ω/2 ms fast reference, and −14 V/60 s reverse. These are prototype design inputs, not standards-compliance claims; actual TVS clamp/forward voltage, fuse response, controller/FET stress, temperature, and layout overshoot require measurement.
- Freeze controlled reset/automatic recovery during crank; full ride-through is not required.
- OBD4 and OBD5 join once at entry into one continuous `POWER_GND`; high-current/ESD return geometry is controlled without split ground planes.
- Copy the exact shared damped-filter values and placement constraints only from `task5a1-power-architecture.md`; do not revive the superseded Task 4.7 envelope or invent substitutions.

## Expansion and debug

- USB-C USB 2.0 device/Serial-JTAG stays on GPIO19/20. D+/D- pass through the `USBLC6-2SC6Y` I/O pins; connector VBUS branches to its VBUS shunt/reference pin and separately to `TPS2553QDBVRQ1 -> TPS62162QDSGRQ1 fixed 3.3 V -> USB_3V3 -> TPS2116DRLR VIN2`. CC1/CC2 use separate USB Type-C sink terminations. `VEH_3V3` connects to VIN1 and mux VOUT is `MAIN_3V3`; CAN VCC and AUX5 are vehicle-only.
- Populate TPS2553 `RILIM` with `CRCW060360K4FKEA`, 60.4 kΩ ±1%; TI Section 8.5 equations with resistor tolerance screen a 387.2–491.3 mA fault/current-limit population band, not a load contract. The prototype startup source/cable must advertise and sustain ≥500 mA at 4.75 V. After ramp, `USB_ENUM` is ≤100 mA `MAIN_3V3` (about 82 mA VBUS), not an inrush ceiling; provisionally limit configured operation to 350 mA VBUS steady and 400 mA `MAIN_3V3`, with MAX prohibited. This does not establish generic legacy USB 2.0 pre-enumeration compliance. Capture the exact TPS62162 input capacitor, inductor, output capacitor and TPS2116 VOUT bulk frozen in `task5a1-power-architecture.md`; startup/capacitance sequencing and bench validation remain mandatory. The former `PMEG6030EP-Q` raw source-OR and 43.2 kΩ value are superseded.
- I2C 3.3 V SDA/SCL plus GND; connector power and pull-up ownership must be explicit.
- Preserve UART0 GPIO43/44 pads as optional serial debug/recovery access; USB remains primary.
- Test points: GND, protected vehicle input/VCC, 3.3 V, each switchable rail, CAN-H, CAN-L, CAN_RX, CAN_TX, GNSS TX/RX, and GNSS rail. RF test provisions require RF-review approval.

## Mechanical and environmental

- RejsaCAN v3.4 reference outline is approximately 31.50 mm × 49.53 mm (`CALCULATED` from repository PCB edge coordinates) with an antenna-end notch; Telemetry v1 dimensions remain `TBD` and will grow for GNSS/RF and connectors.
- RF connector/antenna cable, OBD strain relief, display cable, enclosure, ventilation, ingress, vibration, and service access are TBD.
- Connector mechanics, enclosure temperature, exact purchased-standard/OEM test severities, ESD/EMC setup, and product qualification targets remain validation gates. The Task 5A.1 vehicle-input, source-mux, and AUX5 selections are conditional prototype decisions, not production or compliance approval.

## Product-interface implications

- Hardware exposes capabilities; it does not encode a vehicle, RaceChrono, display layout, alarm rule, logger format, or companion-app transport into the electrical design.
- CAN mode control must permit a verifiable non-transmitting LISTEN_ONLY state and a supervised DIAGNOSTIC_POLLING state. Exact TCAN3404-Q1/TWAI reset, standby, and failure behavior remains a schematic/firmware verification item.
- Independent failures of display, shift-light, buzzer, SD, GNSS, USB/client connectivity, or switched peripheral rails must not cause uncontrolled CAN transmission.
- External interfaces must support the bounded-queue, timeout, health, and isolated-restart architecture in [`architecture.md`](architecture.md); hardware watchdog/reset-domain implications remain unresolved.
- Enclosure, display, and vehicle-mount CAD must consume controlled PCB/connector/keep-out drawings and the limits in [`enclosure-architecture.md`](enclosure-architecture.md), not create new electrical requirements implicitly.
