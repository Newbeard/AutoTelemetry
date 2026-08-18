# Telemetry v1 frozen change list

This is a review list, not authorization to edit the v3.4 design. “KEEP” means conceptually reusable subject to electrical/automotive validation, not “already certified.” Evidence and detailed caveats are in [`rejsacan-analysis.md`](rejsacan-analysis.md).

Status: Task 5A.1 conditionally re-freezes the power architecture for prototype capture, subject to every gate in [task5a1-power-architecture.md](task5a1-power-architecture.md) and [system-validation-plan.md](system-validation-plan.md). It does not authorize PCB layout, manufacturing output, or a compliance claim.

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

- Replace the superseded Task 4.7 OBD-input candidates with the conditionally selected Task 5A.1 chain: `0437002.WRA` 2 A fuse, common-anode `SM15T47AY` + `SM15T33AY`, `LM74720QDRRRQ1`, and two 100 V `STL125N10F8AG`; retain exact transient, fuse/TVS, VDS/SOA/overshoot, inrush and filter gates.
- Replace standard/catalog U2 with TCAN3404DRQ1. Freeze `LMQ66420MC3RXBRQ1` as the vehicle converter producing `VEH_3V3`; retain `LMQ66420MC5RXBRQ1` only as a conditional vehicle-only AUX5 choice. Both use the exact shared `XGL5030-222MEC`, CIN/COUT/CVCC and CBOOT-DNP population frozen in `task5a1-power-architecture.md`; effective capacitance, thermal and load-step validation remain gates.
- Add connector-local ESDCAN04-2BWY and an ACT45B-510-2P-TL003 footprint, DNP by default with 0 Ω bypasses.
- Make CAN termination default state explicit and likely normally open for an OBD stub; retain service/configurability only if justified.
- Rework power-tree current capacity and domains for simultaneous ESP32 RF, GNSS/active antenna, SD, display/backlight, shift-light, and buzzer loads.
- Use four TPS22919QDCKRQ1 switches for GNSS_3V3, SD_3V3, DISPLAY_3V3, and DISPLAY_5V. Control low-speed rail enables through TCA6408AQPWRQ1.
- Implement the Task 5A.1 OV thresholds with tolerance proof: no static trip through 18 V, rising-input OV/PD-low command by 25.531 V before 26 V, and reconnect eligibility by 18 V. Completed isolation/recovery and the dynamic downstream peak require capture. The older ≤20 V trip and `VEHICLE_PROTECTED` ≤24 V targets are superseded. Recalculate the gated input divider, hysteresis and ADC protection for 12 V passenger vehicles; 24 V operation is not required.
- Keep the vehicle buck, `VEH_3V3`, the muxed `MAIN_3V3`, TCAN3404-Q1
  standby and the ESP32 sleep domain energized in vehicle-powered PARKED so
  CAN wake remains possible; TCAN is physically on `VEH_3V3`, not the mux
  output. Do not add a separate hardware-off wake domain in v1.
- Implement dual regulated core sources: `VEH_3V3` and USB-derived `USB_3V3` from `TPS62162QDSGRQ1`, selected by `TPS2116DRLR` to produce `MAIN_3V3`. Keep CAN and AUX5 vehicle-only, then validate USB-C ESD, impedance, CC network, all four source states, isolation/back-feed, transition behavior, the mux's non-AEC/105 °C limitation and shield strategy.
- Move microSD CS from GPIO45 to GPIO11; reserve GPIO45 as an unloaded strap pin.
- Share SPI between SD/display only with independent CS, MISO release, reset defaults, arbitration, cable-integrity review, and power-off isolation.
- Put MODE on GPIO10, SD_CS on GPIO11, and one default-off status LED on the I2C expander.
- Turn connector concepts into keyed, rated, protected interfaces with documented pin numbering and cable constraints.
- Grow/reshape the PCB and enclosure around the GNSS RF keep-out, U.FL, connectors, thermal needs, and strain relief.
- Enforce the low-voltage policy: MAX limited to ≤10 s and ≤25% rolling-60-s duty where permitted; MAX, full-white diagnostics and Wi-Fi prohibited below 10.0 V; logger flush requested below 9.5 V; nonessential-load shedding after less than 9.0 V for 100 ms; controlled core/CAN/wake survival at 6 V; and recovery only after greater than 10.0 V for 2 s. Exact timer/tolerance implementation remains a design requirement pending measurement.
- Use the corrected 8.27505 W simultaneous total for aggregate/fuse/harness work; separately exercise the 1.050 A/3.465 W MC3 output case for regulator and thermal validation. Do not reuse the superseded 12.458 W figure or add mutually exclusive display cases.

## ADD

- Onboard NEO-M9N-00B using GPIO15/16 UART at 230,400 bit/s, switched ≥200 mA rail, V_BCKP following that rail, decoupling and test points. Record that this order code is professional-grade, not automotive-qualified.
- Hirose U.FL-R-SMT-1(10), active AQ3118E-01ETG, and 50 Ω RF section with the u-blox 100 nF/27 nH bias-T topology. Exact antenna and validated passive/active short-current limiter remain blockers.
- RF keep-out/placement rules separating GNSS from ESP32 antenna, buck switch node, CAN edges, SD/display clocks, and cables.
- Universal display connector: dual GND, switched 3.3 V/optional 5 V, SPI, CS, DC, reset, driven PWM/enable, optional I2C and INT/TE; initial ≤200 mm/20 MHz cable contract.
- TPS1H100BQPWPRQ1 protected 5 V/1 A fault envelope and CAHCT1G126QDCKRQ1 buffered data on GPIO6 for exactly ten shift pixels, a 0.50 A qualified load and cable ≤0.5 m.
- `APPROVED DIRECTION`: TPA2005D1TDGNRQ1 on GPIO17 with a 300 mA AUX5 envelope and provisional onboard 8 Ω, ≥1 W speaker.
- `APPROVED`: LP5814DRLR on I2C with P6 `STATUS_DRV_EN` and a provisional common-anode RGB status LED.
- Vehicle input using `0437002.WRA`, common-anode ST `SM15T47AY` + `SM15T33AY` asymmetric TVS pair, `LM74720QDRRRQ1`, and two `STL125N10F8AG` 100 V MOSFETs. The damped post-switch filter requires impedance, EMI, DC-bias, saturation and stability characterization before release.
- USB core power uses USBLC6-2SC6Y, TPS2553 with 60.4 kΩ ±1% ILIM, the exact TPS62162 passive population and TPS2116 with 100 µF VOUT bulk frozen in `task5a1-power-architecture.md`. The limiter's 387.2–491.3 mA band is not a load contract. Require a prototype startup source/cable that advertises and sustains ≥500 mA at 4.75 V; post-ramp `USB_ENUM` ≤100 mA MAIN_3V3 (about 82 mA VBUS); provisional 350 mA VBUS/400 mA MAIN_3V3 configured ceilings; and no MAX/CAN/AUX5. This is not a generic legacy USB 2.0 pre-enumeration compliance claim. Startup/effective-capacitance proof remains conditional; the old 43.2 kΩ/PMEG topology is superseded.
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

1. Review and approve the Task 5A.1 conditional power re-freeze in [task5a1-power-architecture.md](task5a1-power-architecture.md).
2. Carry its selected parts and tolerance-bounded requirements into a separately authorized schematic-capture task without silently restoring the superseded Task 4 chain or 12.458 W load total.
3. Before prototype release, close every Task 5A.1 gate: exact transients; TVS electrothermal/fuse coordination; MOSFET VDS/SOA/overshoot; OV/inrush/reverse-current blocking; filter impedance/EMI/DC bias; four-state source injection; hot parked current; rail thermal/load steps; and low-voltage state transitions.
4. Keep the exact antenna limiter, card/socket mechanics, display connector/ESD, speaker/LED, converter passives/thermal evidence, ERC, review and all later layout/manufacturing gates separate and explicit.
