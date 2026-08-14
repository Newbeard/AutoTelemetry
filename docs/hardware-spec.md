# Telemetry v1 preliminary hardware specification

This is a classified design target, not an approved schematic or an automotive-safety claim. Source calculations are in [`power-budget.md`](power-budget.md) and [`power-wake-review.md`](power-wake-review.md).

## Processing and storage

- ESP32-S3 module class with BLE, Wi-Fi, native USB, and TWAI.
- Reference population: ESP32-S3-WROOM-1-N16R8, 16 MB Quad-SPI flash and 8 MB Octal-SPI PSRAM (`VERIFIED_DATASHEET`, Espressif module data sheet v1.8, Table 1-1). Review its stated ambient-temperature limitation for R8 variants against enclosure thermal requirements.
- Retain microSD for removable CAN/GNSS logs; provide card-detect only if it does not consume a more valuable pin or disturb boot.

## CAN/OBD

- One TCAN3404-Q1 3.3 V automotive CAN FD-capable transceiver connected to native TWAI (`DESIGN_REQUIREMENT`; device limits are `VERIFIED_DATASHEET` in `power-wake-review.md`). Telemetry v1 protocol scope remains Classical CAN.
- OBD-II: 16 = vehicle battery, 4/5 = ground, 6 = CAN-H, 14 = CAN-L.
- Optional split 120 Ω termination is DNP/OFF by default (`DESIGN_REQUIREMENT`). `CALCULATED`: another 120 Ω across an already terminated 60 Ω bus produces 40 Ω effective loading.
- Add/validate a low-capacitance AEC-Q101 dual CAN TVS close to the connector and a bypassed/DNP common-mode-choke option (`DESIGN_REQUIREMENT`).

## GNSS

- u-blox NEO-M9N on switched ≥200 mA GNSS_3V3, UART to ESP32-S3 at 230,400 bit/s, target 20–25 Hz (`DESIGN_REQUIREMENT`; throughput calculation and source limits in `power-wake-review.md`).
- U.FL connector with 50 Ω RF path and layout/keep-out exactly reconciled with the current u-blox integration manual.
- External active antenna bias with filtering, current/fault protection, and DC separation as required by the selected supply scheme and antenna.
- Provide GNSS TX/RX and power test points away from the RF trace.
- Tie V_BCKP to switched GNSS power for v1 and investigate u-blox host save/restore (`DESIGN_REQUIREMENT`); provide isolation/DNP flexibility only if schematic review finds it low risk.

## External display

- Initial display: GC9A01, 240×240, 4-wire SPI; future ST7789/AMOLED supported by software and connector policy.
- Required logical signals: two GND returns, switched 3.3 V, optional switched 5 V, SCLK, MOSI, optional MISO, CS, DC, RESET, BL_PWM/BL_EN, optional I2C and INT/TE.
- Backlight must use a suitable transistor/driver when connector current exceeds GPIO limits. Add local bulk/decoupling guidance for the display module and define cable length.
- Do not expose raw vehicle battery. DISPLAY_3V3 capacity is ≥400 mA and DISPLAY_5V capacity is ≥600 mA within AUX5 (`DESIGN_REQUIREMENT`); incompatible rails must be keyed/configured.

## Outputs

- Shift-light: protected, current-limited switched 5 V/1 A plus translated/buffered single-wire data for a short 8–10 pixel module (`DESIGN_REQUIREMENT`); intelligent/differential external module remains the long-cable option.
- Buzzer: PWM-capable MOSFET driver with ≤200 mA branch envelope (`DESIGN_REQUIREMENT`). Define magnetic/piezo and active/passive construction before selecting the clamp.
- Neither external load is driven directly by an ESP32 GPIO.

## Expansion and debug

- USB-C USB 2.0 device/Serial-JTAG on GPIO19/20, with CC pull-downs and ESD reviewed.
- I2C 3.3 V SDA/SCL plus GND; connector power and pull-up ownership must be explicit.
- Preserve UART0 GPIO43/44 pads as optional serial debug/recovery access; USB remains primary.
- Test points: GND, protected vehicle input/VCC, 3.3 V, each switchable rail, CAN-H, CAN-L, CAN_RX, CAN_TX, GNSS TX/RX, and GNSS rail. RF test provisions require RF-review approval.

## Mechanical and environmental

- RejsaCAN v3.4 reference outline is approximately 31.50 mm × 49.53 mm (`CALCULATED` from repository PCB edge coordinates) with an antenna-end notch; Telemetry v1 dimensions remain `TBD` and will grow for GNSS/RF and connectors.
- RF connector/antenna cable, OBD strain relief, display cable, enclosure, ventilation, ingress, vibration, and service access are TBD.
- Temperature, transient, ESD, EMC, and qualification targets must be set before part selection is frozen.

## Product-interface implications

- Hardware exposes capabilities; it does not encode a vehicle, RaceChrono, display layout, alarm rule, logger format, or companion-app transport into the electrical design.
- CAN mode control must permit a verifiable non-transmitting LISTEN_ONLY state and a supervised DIAGNOSTIC_POLLING state. Exact TCAN3404-Q1/TWAI reset, standby, and failure behavior remains a schematic/firmware verification item.
- Independent failures of display, shift-light, buzzer, SD, GNSS, USB/client connectivity, or switched peripheral rails must not cause uncontrolled CAN transmission.
- External interfaces must support the bounded-queue, timeout, health, and isolated-restart architecture in [`architecture.md`](architecture.md); hardware watchdog/reset-domain implications remain unresolved.
- Enclosure, display, and vehicle-mount CAD must consume controlled PCB/connector/keep-out drawings and the limits in [`enclosure-architecture.md`](enclosure-architecture.md), not create new electrical requirements implicitly.
