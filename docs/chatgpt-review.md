# Telemetry v1 Task 4 review packet

## Task completed

Completed the component-freeze and schematic-architecture gate for Telemetry v1. Audited the Task 2 power/wake work, selected exact major devices where evidence permitted, defined the schematic sheet hierarchy and power states, froze the GPIO/peripheral map, recorded lifecycle/availability/cost evidence, and identified the remaining release blockers. No schematic, PCB, firmware, CAD, manufacturing, procurement, or copied third-party design files were created or changed.

## Files changed

- Created `docs/component-freeze.md`.
- Created `docs/schematic-architecture.md`.
- Updated `CODEX.md`.
- Updated `docs/architecture.md`.
- Updated `docs/hardware-spec.md`.
- Updated `docs/interfaces.md`.
- Updated `docs/power.md`.
- Updated `docs/power-budget.md`.
- Updated `docs/power-wake-review.md`.
- Updated `docs/requirements.md`.
- Updated `docs/rejsacan-analysis.md`.
- Updated `docs/telemetry-v1-change-list.md`.
- Replaced this review packet, `docs/chatgpt-review.md`.

## Engineering findings

- The 1.313 A MAIN_3V3 and 1.955 A AUX5 calculated capacities support freezing 2 A LMQ66420-Q1 silicon, but exact inductors, effective capacitance, losses, stability, copper area, and enclosure thermal rise remain schematic calculations.
- Replacing the former 5 µA input-protection placeholder with the LM74502H-Q1 110 µA maximum operating-current bound changes the parked subtotal from 80.3 µA to 185.3 µA. A 100% allowance produces 370.6 µA, or 0.371 mA at 12 V. This remains below 1 mA and leaves 0.129 mA to the 0.5 mA stretch target.
- The calculated simultaneous output envelope is 14.108 W. At an assumed 80% efficiency and 12 V input, current is 1.470 A; a 2 A fuse therefore has 36.1% nominal current margin. This does not establish fuse hot hold/trip or pulse performance.
- TCAN3404DRQ1 provides the required 3.3 V rail-on wake architecture with 17 µA maximum standby. The OBD node must remain high impedance: split termination is DNP/OFF because adding 120 Ω to a normally terminated 60 Ω vehicle bus yields 40 Ω.
- NEO-M9N-00B meets the function target but is professional-grade rather than automotive-grade. The RF connector is frozen for the prototype only; the exact active antenna remains unresolved.
- ESP32-S3-WROOM-1-N16R8 needs PSRAM ECC for the documented +85 °C ambient extension. Usable PSRAM becomes approximately 7.5 MB; the module still is not automotive-qualified.
- The source evidence and calculations are detailed in `docs/component-freeze.md`, `docs/schematic-architecture.md`, `docs/power-budget.md`, and `docs/power-wake-review.md`.

## Decisions made

- Froze ESP32-S3-WROOM-1-N16R8, TCAN3404DRQ1, ESDCAN04-2BWY, LMQ66420MC3RXBRQ1, LMQ66420MC5RXBRQ1, TPS22919QDCKRQ1, TCA6408AQPWRQ1, LM74502HQDDFRQ1, DMT6007LFGQ-7, 0437002A/WRA, NEO-M9N-00B, ESDAXLC6-1BT2Y, USBLC6-2SC6Y, TPS2553QDBVRQ1, PMEG6030EP-Q, TPS1H100BQPWPRQ1, CAHCT1G126QDCKRQ1, and 2N7002KQ-7 for their stated roles.
- Froze ACT45B-510-2P-TL003 as a DNP CAN-choke footprint with populated 0 Ω bypasses. Split CAN termination is two 60.4 Ω resistors plus 4.7 nF, all DNP and isolated by normally open service links.
- Kept LDP01-28AY/input filtering, microSD socket, USB-C receptacle, physical display connector, active antenna, and buzzer transducer provisional because required electrical or mechanical inputs are absent.
- Froze the hybrid rail-on state architecture and the 14-position display electrical contract.
- Permanently excluded TPMS, tire-temperature sensing, IMU, analog sensor hubs, external sensor networks, a second CAN channel, and unrelated features from Telemetry v1.

## Assumptions

- The input-current calculation assumes 80% conversion efficiency at the constructed simultaneous maximum.
- The parked conversion calculation retains the Task 2 60% low-load efficiency assumption where 3.3 V rail currents are referred to 12 V.
- The preliminary major-electronics price subtotal is an approximately ±20% budget snapshot, not a quote or complete manufacturing cost.
- The passive GNSS short limiter assumes approximately 2.2 Ω additional series resistance beyond its 22 Ω resistor.
- Connector and enclosure requirements will not increase the already frozen rail-current contracts without reopening this gate.

## Uncertainties / unresolved questions

- The approved 12 V passenger-vehicle crank, jump-start, reverse, ISO 7637-2 pulse, load-dump source impedance/energy, ISO 10605 ESD, and temperature profile is missing; therefore LDP01-28AY and the EMI filter cannot be finalized.
- Exact LMQ66420 magnetics, capacitors, feedback/configuration, stability, losses, junction temperature, copper area, and transient headroom are uncalculated.
- The final OBD/display/shift-light connector mechanics, production microSD socket, active GNSS antenna, and onboard buzzer are not selected.
- Vehicle-activity sensing, protected ADC divider, PWR_FAULT_N, connector-side ESD arrays, and branch-discharge details require exact schematic calculations.
- GPIO13 CAN wake, GPIO8 vehicle-activity wake, power-off signal isolation, USB dual-source behavior, GNSS 25 Hz throughput, RF coexistence, and complete parked current require prototype measurement.

## Risks

- A pulse profile more severe than the provisional clamp can invalidate the TVS, 60 V MOSFETs, regulator headroom, fuse, and filter together.
- The 0.5 mA parked stretch target has only 0.129 mA calculated margin before complete-netlist leakage and temperature measurements.
- NEO-M9N-00B, the prototype U.FL, ESP32 module, and provisional connectors may not meet the eventual automotive environmental/lifecycle requirement.
- AUX5 reaches 1.955 A only under the constructed full simultaneous envelope, leaving little nominal margin on a 2 A converter; load policy and thermal testing are required.
- Shared SPI/display cabling and the GNSS RF section create signal-integrity and EMC risks that component selection alone cannot close.

## GPIO / peripheral changes

| Resource | Previous Task 2 allocation | Frozen Task 4 allocation | Reason |
|---|---|---|---|
| GPIO11 | spare/status candidate | microSD CS | Removes a removable-card load from strap GPIO45 |
| GPIO17 | power-hold legacy concept | onboard buzzer PWM | New architecture no longer uses the legacy hold circuit |
| GPIO18 | spare | TCA6408A interrupt | Supports low-speed rail/status expansion |
| GPIO42 | buzzer proposal | reserved JTAG MTMS test pad | Preserves debug and eliminates the conflict |
| GPIO45 | microSD CS | reserved strap/test pad | Deterministic reset/boot behavior |
| TCA6408A P0–P7 | none | GNSS_EN, SD_EN, DISP3_EN, DISP5_EN, SHIFT5_EN, SD_CD_N, STATUS_LED_N, reserved | Preserves direct MCU pins while hardware pull-downs keep rails off at reset |

All other assignments are listed in `docs/interfaces.md` and `docs/schematic-architecture.md`.

## Power / CAN / RF impact

- **Automotive power:** frozen controller, MOSFET, fuse, two regulator families, load switches, USB isolation, state sequencing, and a revised 0.371 mA parked envelope. The input TVS/filter remain provisional pending the pulse profile.
- **CAN:** frozen TCAN3404DRQ1, ESDCAN04-2BWY, DNP ACT45B footprint, and DNP/OFF split termination. Only one Classical CAN channel exists.
- **GNSS/RF:** frozen NEO-M9N-00B prototype architecture, U.FL-R-SMT-1(10), ESDAXLC6-1BT2Y, 50 Ω path, switched power, and the u-blox bias-T starting topology. No RF layout was performed.
- **USB:** frozen USBLC6-2SC6Y, TPS2553QDBVRQ1 with 43.2 kΩ ILIM, PMEG6030EP-Q reverse isolation, and ≤500 mA USB-only configured load. The receptacle is mechanically provisional.
- **ESP32 boot/strapping:** GPIO45 is no longer microSD CS; GPIO0/3/45/46 remain protected strap resources. GPIO42 is reserved for MTMS. PSRAM ECC is mandatory for the +85 °C documented operating extension.

## Datasheets / primary sources consulted

- Espressif ESP32-S3-WROOM-1/1U data sheet v1.8 and ESP32-S3 Hardware Design Guidelines.
- TI TCAN3404-Q1, TCAN1043A-Q1, LMQ664x0-Q1, LM74502H-Q1, TPS22919-Q1, TCA6408A-Q1, TPS2553-Q1, TPS1H100-Q1, and automotive AHCT buffer documentation.
- NXP TJA1044 and Infineon TLE9251VLE official data/product lifecycle information.
- ST ESDCANxx-2BWY, LDP01-28AY, ESDAXLC6-1BT2Y, and USBLC6-2SC6Y data sheets.
- TDK ACT45B, Diodes Incorporated DMT6007LFGQ/2N7002KQ/BAS21WQ, Littelfuse 437A, and Nexperia PMEG6030EP-Q data sheets.
- u-blox NEO-M9N-00B data sheet R08 and Integration Manual R10; Hirose U.FL specification.
- GCT USB4105 and Molex 104031-0811 product specifications.
- Manufacturer lifecycle pages and distributor stock/price pages captured on 2026-08-14; exact links and quantities are recorded in `docs/component-freeze.md`.

## Recommended next step

Complete the input-protection design gate by approving one written 12 V passenger-vehicle crank, jump-start, reverse-battery, ISO 7637-2, load-dump, ISO 10605, and temperature profile and closing the LDP01-28AY/filter calculations against it.

## STOP condition

Task 4 is complete. Stop here: do not begin schematic capture, KiCad edits, PCB layout, firmware, CAD, manufacturing outputs, procurement, or copied third-party implementation until the review gate is accepted and a separate task authorizes the next stage.
