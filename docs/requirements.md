# Telemetry v1 requirements

Status: pre-schematic requirements baseline, audited 2026-08-14. Items marked **TBD** require review or measurement before schematic capture. Numerical evidence uses `VERIFIED_DATASHEET`, `CALCULATED`, `DESIGN_REQUIREMENT`, and `ASSUMPTION` as defined in [`power-budget.md`](power-budget.md).

## Scope and invariants

- Use RejsaCAN v3.4 as a reference, while keeping every upstream schematic, PCB, Gerber, BOM, placement, and manufacturing file unchanged.
- Create derivative hardware only under `hardware/telemetry-v1/` after this document set is reviewed.
- Connect to OBD-II pin 16 (battery), pins 4/5 (ground), pin 6 (CAN-H), and pin 14 (CAN-L).
- Support permanent installation with average parked input current <1.0 mA over the specified parked voltage/temperature range (`DESIGN_REQUIREMENT`); <0.50 mA at 12 V and 25 °C is the stretch target (`DESIGN_REQUIREMENT`).
- Retain USB firmware download, recovery, configuration, and debugging, plus microSD logging.
- Do not claim automotive robustness until the design has defined and passed relevant electrical, EMC, thermal, and environmental tests.

## Functional requirements

| ID | Requirement | Initial acceptance criterion |
|---|---|---|
| CAN-01 | One Classical CAN channel using the ESP32-S3 TWAI controller | Receive and transmit 11/29-bit CAN at required vehicle bit rates; listen-only mode supported |
| CAN-02 | Generic OBD-II, ISO-TP, UDS, and raw/vehicle-profile decoding | Hardware must not couple protocol choice to vehicle or display choice |
| CAN-03 | Configurable CAN termination | Optional split 120 Ω (`DESIGN_REQUIREMENT`), DNP/OFF by default for OBD use; bench-only population documented |
| GNSS-01 | Onboard u-blox NEO-M9N with external active antenna | UART communication; target navigation rate 20–25 Hz subject to configured constellations/messages and link budget |
| GNSS-02 | U.FL antenna interface and active-antenna bias | Implement only from the current u-blox integration manual and selected antenna data sheet |
| GNSS-03 | Power control | Software-controlled GNSS power or backup strategy, with cold/warm-start trade-off documented |
| DSP-01 | Interchangeable external display | Initial GC9A01 240×240 SPI display; later ST7789/AMOLED drivers without core redesign |
| DSP-02 | Display connector | Ground, protected/specified power, SPI SCLK/MOSI/(optional MISO), CS, DC, reset, and PWM-capable backlight control |
| SHF-01 | External shift-light output | ESP32-controlled, independent of RaceChrono, with a driver sized for a specified external module rather than GPIO load current |
| ALM-01 | Audible alarm | Onboard buzzer or external connector driven through a transistor/MOSFET; alarm load and acoustic target TBD |
| LOG-01 | microSD logging | Concurrent CAN/GNSS logging without electrical bus contention with the display |
| BLE-01 | RaceChrono BLE link | BLE profile/protocol to be confirmed against current RaceChrono documentation |
| EXP-01 | Expansion and debug | Expose I2C, UART/debug access, useful spare GPIO, and named test points for CAN-H/L, vehicle input, 3.3 V, GNSS UART, and ground |
| PWR-01 | Permanent OBD installation | Verify active, transient and complete parked current against [`power-budget.md`](power-budget.md); release limit <1.0 mA, stretch <0.50 mA at 12 V/25 °C (`DESIGN_REQUIREMENT`) |
| PWR-02 | Wake sources | Rail-on ESP32/CAN standby wake from CAN, timer, MODE, vehicle-voltage hint and USB (`DESIGN_REQUIREMENT`) |
| PWR-03 | Power domains | ≥2.0 A MAIN_3V3 and ≥2.0 A switched AUX5 capacities, with separately switchable GNSS, SD and display branches (`DESIGN_REQUIREMENT`, derived in `power-budget.md`) |
| PWR-04 | USB source isolation | Support vehicle-only, USB-only, simultaneous and unpowered cases with no back-feed to OBD pin 16 or USB VBUS (`DESIGN_REQUIREMENT`) |
| FW-01 | Modular firmware | Independent CAN, OBD-II, ISO-TP, UDS, vehicle profile, GNSS, normalized data core, BLE, display, shift-light, alarms, logger, and power modules |

## Safety and validation requirements

- Preserve ESP32-S3 GPIO19/GPIO20 for native USB and do not load GPIO0/GPIO3/GPIO45/GPIO46 without a strap analysis.
- Validate OBD input against a written 12 V passenger-vehicle transient profile. 24 V operation is explicitly not a Telemetry v1 requirement (`DESIGN_REQUIREMENT`). An input-voltage range is not a transient-survival specification.
- Review CAN protection, common-mode range, ESD, termination, grounding, and non-automotive-qualified reference components.
- Perform regulator worst-case input/transient, load, thermal, stability, startup, shutdown, reverse-polarity, and back-power analyses.
- Establish RF keep-outs, controlled-impedance rules, antenna bias filtering/protection, and conducted/radiated noise targets before GNSS layout.
- Define connector pin numbering, load limits, short-circuit behavior, cable length, ESD protection, and hot-plug behavior for every external interface.
- Add test points without creating high-stub or antenna structures on CAN, USB, SPI, or GNSS RF nets.

## Deferred decisions

- Exact automotive pulse severity, temperature grade, enclosure, cable environment, compliance markets, and production volume.
- Whether a later hardware-off CAN wake variant is worth its added 5 V/AON sequencing; it is not required for Telemetry v1 unless rail-on measurements fail.
- Display supply voltage/current and cable/connector family.
- Shift-light and buzzer voltage, current, wiring, and fault protection.
- Whether GNSS backup supply and retained ephemeris are worth their parked-current cost.

## Audit disposition

- Earlier 5–24 V wording was a repository/reference claim, not a Telemetry v1 requirement; it is no longer used as a design fact.
- Earlier termination, switched-load capacity, start threshold, display current and peripheral-current wording is superseded by the classified values or explicit unresolved items in `power-budget.md` and `power-wake-review.md`.
- GPIO numbers are evidence from the v3.4 schematic/pinout and are classified in `interfaces.md`; they are not new electrical calculations.
