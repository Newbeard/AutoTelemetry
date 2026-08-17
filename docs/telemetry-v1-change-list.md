# Telemetry v1 frozen change list

This is a review list, not authorization to edit the v3.4 design. “KEEP” means conceptually reusable subject to electrical/automotive validation, not “already certified.” Evidence and detailed caveats are in [`rejsacan-analysis.md`](rejsacan-analysis.md).

## KEEP

- ESP32-S3 module architecture with BLE/Wi-Fi, native USB, TWAI, GPIO matrix, and sufficient N16R8 memory.
- Native TWAI mapping on GPIO13 RX / GPIO14 TX.
- CAN transceiver mode-control concept on GPIO38, updated to TCAN3404-Q1 standby/WUP.
- Input-voltage digital and analog monitoring concepts on GPIO8/GPIO9.
- Native USB-C development/recovery concept on GPIO19/GPIO20 with PROG and RESET access.
- microSD logging and the GPIO39/40/41 SPI bus; CS moves to GPIO11 and GPIO45 stays unloaded.
- I2C on GPIO1/GPIO2 and useful GPIO/UART test access.
- Switchable peripheral-power concept, implemented as named GNSS_3V3, SD_3V3, DISPLAY_3V3 and AUX5 domains.
- Cut/jumper-selectable CAN termination concept, changed to optional split 120 Ω DNP/OFF by default.

## MODIFY

- Implement the frozen Task 4.7 OBD input chain and calculate the remaining divider/inrush/filter passives; do not copy the upstream circuit or claim compliance.
- Replace standard/catalog U2 with TCAN3404DRQ1. Replace U4 with LMQ66420MC3RXBRQ1; use the same regulator silicon for switched AUX5, with independent exact passives and thermal validation.
- Add connector-local ESDCAN04-2BWY and an ACT45B-510-2P-TL003 footprint, DNP by default with 0 Ω bypasses.
- Make CAN termination default state explicit and likely normally open for an OBD stub; retain service/configurability only if justified.
- Rework power-tree current capacity and domains for simultaneous ESP32 RF, GNSS/active antenna, SD, display/backlight, shift-light, and buzzer loads.
- Use four TPS22919QDCKRQ1 switches for GNSS_3V3, SD_3V3, DISPLAY_3V3, and DISPLAY_5V. Control low-speed rail enables through TCA6408AQPWRQ1.
- Recalculate threshold detector, gated input divider, hysteresis and ADC protection for 12 V passenger vehicles; 24 V operation is not required.
- Keep MAIN_3V3 energized in parked sleep so TCAN3404-Q1 and ESP32 can provide CAN wake; do not add a separate hardware-off wake domain in v1.
- Validate USB-C ESD, impedance, CC network, dual-source isolation/back-feed, and shield strategy.
- Move microSD CS from GPIO45 to GPIO11; reserve GPIO45 as an unloaded strap pin.
- Share SPI between SD/display only with independent CS, MISO release, reset defaults, arbitration, cable-integrity review, and power-off isolation.
- Put MODE on GPIO10, SD_CS on GPIO11, and one default-off status LED on the I2C expander.
- Turn connector concepts into keyed, rated, protected interfaces with documented pin numbering and cable constraints.
- Grow/reshape the PCB and enclosure around the GNSS RF keep-out, U.FL, connectors, thermal needs, and strain relief.

## ADD

- Onboard NEO-M9N-00B using GPIO15/16 UART at 230,400 bit/s, switched ≥200 mA rail, V_BCKP following that rail, decoupling and test points. Record that this order code is professional-grade, not automotive-qualified.
- Hirose U.FL-R-SMT-1(10), active AQ3118E-01ETG, and 50 Ω RF section with the u-blox 100 nF/27 nH bias-T topology. Exact antenna and validated passive/active short-current limiter remain blockers.
- RF keep-out/placement rules separating GNSS from ESP32 antenna, buck switch node, CAN edges, SD/display clocks, and cables.
- Universal display connector: dual GND, switched 3.3 V/optional 5 V, SPI, CS, DC, reset, driven PWM/enable, optional I2C and INT/TE; initial ≤200 mm/20 MHz cable contract.
- TPS1H100BQPWPRQ1 protected 5 V/1 A fault envelope and CAHCT1G126QDCKRQ1 buffered data on GPIO6 for exactly ten shift pixels, a 0.50 A qualified load and cable ≤0.5 m.
- `APPROVED DIRECTION`: TPA2005D1TDGNRQ1 on GPIO17 with a 300 mA AUX5 envelope and provisional onboard 8 Ω, ≥1 W speaker.
- `APPROVED`: LP5814DRLR on I2C with P6 `STATUS_DRV_EN` and a provisional common-anode RGB status LED.
- Vehicle input using `0437002A`, bidirectional `SM8SF24CA-Q`, `LM74502QDDFRQ1`, and two `DMT6007LFGQ-7` MOSFETs. Freeze damped post-switch C-L-C topology; exact L/C/R and threshold/inrush values remain Task 5 calculations.
- USB input using USBLC6-2SC6Y, TPS2553QDBVRQ1 with 43.2 kΩ ILIM, and PMEG6030EP-Q reverse isolation; USB-only load ≤500 mA.
- MODE button on non-strap GPIO10.
- Named test points for GND, protected vehicle input, main/switchable rails, CAN-H/L, CAN logic RX/TX, and GNSS UART/power.
- Measurable current-isolation points or zero-ohm links where useful for sleep-current characterization.
- Hardware revision identification that does not consume unnecessary pins, if production needs it.
- Schematic notes for external load limits, strap states, connector pinouts, termination default, and do-not-populate options.

## REMOVE

Nothing is removed from the upstream reference files.

Candidates to omit from the **new derivative** after review:

- The legacy GPIO17 power-hold circuit; the frozen rail-on architecture uses GPIO17 for the onboard buzzer and manages optional rails independently.
- One or both general-purpose onboard LEDs if GPIO10/11, current, or board area is more valuable. Retain at least one unambiguous status indication if service requirements need it.
- External four-wire JTAG header/pads may be omitted if native USB-JTAG is accepted; GPIO42/MTMS remains reserved and is not used by the buzzer.
- Board-version strap scheme on GPIO4/GPIO5 if a more scalable hardware-ID method is selected.
- Populated 120 Ω CAN termination if the product is exclusively an OBD stub and serviceable termination is unnecessary; this requires CAN specialist review.
- The generic 3V3_SWITCHED output connector if it is replaced by named, current-limited GNSS/display rails.

## Review gates before any schematic edit

1. Review and approve the Task 4 component freeze and schematic architecture.
2. Close the explicitly listed schematic blockers: UV/OV/inrush/fuse/filter calculations, exact antenna limiter, card/socket mechanics, display connector/ESD, speaker/LED, and converter passive/thermal calculations.
3. Only then begin schematic capture as a separate task.
