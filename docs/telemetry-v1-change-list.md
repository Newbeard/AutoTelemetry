# Telemetry v1 preliminary change list

This is a review list, not authorization to edit the v3.4 design. “KEEP” means conceptually reusable subject to electrical/automotive validation, not “already certified.” Evidence and detailed caveats are in [`rejsacan-analysis.md`](rejsacan-analysis.md).

## KEEP

- ESP32-S3 module architecture with BLE/Wi-Fi, native USB, TWAI, GPIO matrix, and sufficient N16R8 memory.
- Native TWAI mapping on GPIO13 RX / GPIO14 TX.
- CAN transceiver mode-control concept on GPIO38, updated to TCAN3404-Q1 standby/WUP.
- Input-voltage digital and analog monitoring concepts on GPIO8/GPIO9.
- MCU power-hold concept on GPIO17.
- Native USB-C development/recovery concept on GPIO19/GPIO20 with PROG and RESET access.
- microSD logging and the GPIO39/40/41 SPI bus, subject to GPIO45 strap validation.
- I2C on GPIO1/GPIO2 and useful GPIO/UART test access.
- Switchable peripheral-power concept, implemented as named GNSS_3V3, SD_3V3, DISPLAY_3V3 and AUX5 domains.
- Cut/jumper-selectable CAN termination concept, changed to optional split 120 Ω DNP/OFF by default.

## MODIFY

- Recalculate/redesign the complete OBD input protection for the agreed vehicle transient and compliance profile; do not copy the existing circuit as proof of safety.
- Replace standard/catalog U2 with TCAN3404-Q1 (`DESIGN_REQUIREMENT`). Replace U4 with a ≥2 A low-IQ automotive buck; LMQ66420-Q1 is the preferred candidate pending full selection calculations.
- Add connector-local low-capacitance AEC-Q101 dual CAN TVS and a bypassed/DNP common-mode-choke footprint; exact parts remain pending pulse/SI review.
- Make CAN termination default state explicit and likely normally open for an OBD stub; retain service/configurability only if justified.
- Rework power-tree current capacity and domains for simultaneous ESP32 RF, GNSS/active antenna, SD, display/backlight, shift-light, and buzzer loads.
- Separate GNSS/display/SD power control where the parked-current, inrush, or fault budget requires it; do not overload reference `3V3_SWITCHED`.
- Recalculate threshold detector, gated input divider, hysteresis and ADC protection for 12 V passenger vehicles; 24 V operation is not required.
- Keep MAIN_3V3 energized in parked sleep so TCAN3404-Q1 and ESP32 can provide CAN wake; do not add a separate hardware-off wake domain in v1.
- Validate USB-C ESD, impedance, CC network, dual-source isolation/back-feed, and shield strategy.
- Validate or move microSD CS GPIO45 after strap analysis with all card/socket states.
- Share SPI between SD/display only with independent CS, MISO release, reset defaults, arbitration, cable-integrity review, and power-off isolation.
- Replace/reassign the blue/yellow status LED usage as needed: GPIO10 becomes MODE in the preliminary plan and GPIO11 becomes a spare.
- Turn connector concepts into keyed, rated, protected interfaces with documented pin numbering and cable constraints.
- Grow/reshape the PCB and enclosure around the GNSS RF keep-out, U.FL, connectors, thermal needs, and strain relief.

## ADD

- Onboard NEO-M9N circuit using GPIO15/16 UART at 230,400 bit/s, switched ≥200 mA rail, V_BCKP following that rail, decoupling and test points.
- U.FL and 50 Ω RF section with active-antenna bias/filtering/current protection designed from the current u-blox manual and selected antenna.
- RF keep-out/placement rules separating GNSS from ESP32 antenna, buck switch node, CAN edges, SD/display clocks, and cables.
- Universal display connector: dual GND, switched 3.3 V/optional 5 V, SPI, CS, DC, reset, driven PWM/enable, optional I2C and INT/TE; initial ≤200 mm/20 MHz cable contract.
- Protected 5 V/1 A shift-light branch and buffered addressable data controlled by GPIO6, with default-off bias, ESD and damping.
- ≤200 mA MOSFET buzzer driver controlled by GPIO42; clamp depends on selected load.
- Reverse-blocked OBD/USB source OR supporting all four source cases without back-feed.
- MODE button on a non-strap pin (preliminary GPIO10).
- Named test points for GND, protected vehicle input, main/switchable rails, CAN-H/L, CAN logic RX/TX, and GNSS UART/power.
- Measurable current-isolation points or zero-ohm links where useful for sleep-current characterization.
- Hardware revision identification that does not consume unnecessary pins, if production needs it.
- Schematic notes for external load limits, strap states, connector pinouts, termination default, and do-not-populate options.

## REMOVE

Nothing is removed from the upstream reference files.

Candidates to omit from the **new derivative** after review:

- One or both general-purpose onboard LEDs if GPIO10/11, current, or board area is more valuable. Retain at least one unambiguous status indication if service requirements need it.
- External four-wire JTAG header/pads if native USB-JTAG is accepted; the SPI display already conflicts with MTCK/MTDO/MTDI and the preliminary buzzer allocation uses MTMS.
- Board-version strap scheme on GPIO4/GPIO5 if a more scalable hardware-ID method is selected.
- Populated 120 Ω CAN termination if the product is exclusively an OBD stub and serviceable termination is unnecessary; this requires CAN specialist review.
- The generic 3V3_SWITCHED output connector if it is replaced by named, current-limited GNSS/display rails.

## Review gates before any schematic edit

1. Review and approve this power/wake architecture gate.
2. Establish the exact 12 V pulse/crank/ESD/temperature test profile.
3. Select exact antenna, card, display/connector, shift-light and buzzer loads.
4. Approve GPIO table and JTAG/LED trade-offs.
5. Complete exact protection, regulator, load-switch and source-OR component selection calculations before schematic capture.
