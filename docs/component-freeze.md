# Telemetry v1 component freeze

Status: Task 5A.1 conditional pre-schematic re-freeze, 2026-08-17. This document freezes component identity and electrical architecture for conditional prototype capture only. It does not authorize PCB layout, procurement, production, or an automotive/standards-compliance claim.

## Task 5A.1 conditional re-freeze

The earlier Task 5A input-power block is superseded by the coordinated selection below. The authoritative primary-source evidence, equations, exact filter population, source-mux implementation, availability snapshots, and laboratory gates are in [`task5a1-power-architecture.md`](task5a1-power-architecture.md).

- The vehicle main-current path is `VBAT_OBD_RAW` -> `0437002.WRA` -> `VBAT_FUSED_CLAMPED` -> `LM74720QDRRRQ1` + two back-to-back `STL125N10F8AG` -> the damped filter controlled by the Task 5A.1 architecture document. At `VBAT_FUSED_CLAMPED`, the common-anode TVS pair is a shunt branch to `POWER_GND`: `SM15T47AY` cathode to fused VBAT, both anodes at `TVS_MID`, and `SM15T33AY` cathode to ground.
- The selected LM74720-Q1 path provides true reverse-current blocking while enabled.
- Native OV uses two 249 kΩ top resistors in series and 28.0 kΩ bottom, all 0.1%. With the explicit ±1 µA leakage model, rising/open is 20.690–25.531 V and falling/reconnect is 18.815–23.366 V.
- The former cutoff-by-20 V and protected-node-≤24 V guarantees are deliberately relaxed. Downstream compatibility and dynamic overshoot are prototype gates.
- The corrected maximum simultaneous case is `3.3 V×0.750 A + 5 V×1.16001 A = 8.27505 W`. A separate 3.3 V-display case loads the MC3 output to 1.050 A/3.465 W and produces 6.765 W total rail power; 1.313 A is sizing margin only. AUX5 MAX is prohibited below 10 V; the 2 A fuse is retained only with enforced low-voltage shedding, temperature validation, and complete fault/pulse validation.
- `LMQ66420MC3RXBRQ1` remains frozen for the vehicle 3.3 V rail. `LMQ66420MC5RXBRQ1` is conditional for AUX5: STREET 0.56001 A, TRACK 0.92001 A, and MAX 1.16001 A for ≤10 s and ≤25% rolling 60 s.
- USB is converted independently to 3.3 V with `TPS62162QDSGRQ1` and muxed with `VEH_3V3` by `TPS2116DRLR`. CAN VCC and AUX5 are vehicle-only. The former `PMEG6030EP-Q` raw source-OR is superseded.

## Status and evidence convention

- `FROZEN`: use this exact orderable part in the first derivative schematic unless review evidence reopens the decision.
- `CONDITIONAL PROTOTYPE FREEZE`: capture the exact part/value only with the named operating policy and validation gates; this is not production approval.
- `PROVISIONAL`: the candidate is electrically plausible, but a named missing input prevents a defensible freeze.
- `PROPOSED CHANGE`: supersedes an earlier freeze only after the reason and affected documents are explicit.
- `REOPENED`: later calculation invalidated or left a required guarantee unresolved; do not capture the affected circuit until the named gate is closed.
- `REJECTED`: evaluated but not selected for the stated v1 architecture.
- `DNP`: footprint is retained but the production-default population is absent.
- Numerical statements use `VERIFIED_DATASHEET`, `CALCULATED`, `DESIGN_REQUIREMENT`, and `ASSUMPTION` as defined in `CODEX.md`.

The freeze is intentionally limited to Telemetry v1: one OBD/Classical-CAN interface, ESP32-S3, onboard GNSS with an external active antenna, microSD, USB-C, BLE, an interchangeable external display, external shift-light, onboard audible alarm, MODE, and debug. TPMS, tire temperature, IMU, analog sensor hubs, external sensor networks, a second CAN channel, and unrelated expansion hardware are excluded.

## Major component table

| Function | Exact part | Manufacturer | Qualification | Package | Status | Reason | Alternatives considered | Confidence |
|---|---|---|---|---|---|---|---|---|
| MCU/radio/module | `ESP32-S3-WROOM-1-N16R8` | Espressif | Not AEC-qualified | 41-pad module, PCB antenna | `FROZEN` | 16 MB Quad-SPI flash, 8 MB Octal-SPI PSRAM, BLE/Wi-Fi, native USB and adequate GPIO. PSRAM ECC is mandatory to extend the documented R8 ambient maximum from +65 °C to +85 °C; usable PSRAM becomes 7.5 MB (`8 MB × 15/16`, `CALCULATED`). | `ESP32-S3-WROOM-1-N16R2`: native −40…+85 °C but only 2 MB PSRAM. `WROOM-1U`: conflicts with the desired integrated BLE/Wi-Fi antenna architecture. | High, with thermal validation risk |
| CAN transceiver | `TCAN3404DRQ1` | Texas Instruments | AEC-Q100 | SOIC-8 | `FROZEN` | Single 3.3 V rail, standby WUP on RXD, 17 µA maximum standby, high-Z unpowered bus, ±58 V bus fault standoff, ±30 V common-mode, and SOIC prototype assembly. Protocol use remains Classical CAN. | NXP `TJA1044VT/3Z`; Infineon `TLE9251VLEXUMA1`; TI `TCAN1043A-Q1`. See comparison below. | High |
| CAN-line TVS | `ESDCAN04-2BWY` | STMicroelectronics | AEC-Q101 | SOT323-3L | `FROZEN` | Dual CAN-line protection; 25.5 V stand-off, 27.5 V typical breakdown, 43 V maximum clamp at 3 A, 19 pF maximum line capacitance, 0.05 µA maximum leakage at 25 °C. Compatible with v1 Classical CAN bandwidth. | Lower-capacitance ESDCAN02/03 variants; TI ESD2CAN24-Q1. | High for schematic; validate full interface pulses |
| Optional CAN CMC | `ACT45B-510-2P-TL003` | TDK | AEC-Q200 | 1812, four pad | `FROZEN footprint`, `DNP` | Production-status 51 µH common-mode choke, 200 mA, 50 V, 1 Ω maximum per winding, −40…+150 °C. Default 0 Ω bypasses avoid adding unmeasured impedance; populate only after EMC/SI testing. | No choke; ACT45B-101. | High |
| Vehicle 3.3 V buck | `LMQ66420MC3RXBRQ1` | Texas Instruments | AEC-Q100 | 14-pin 2.6 mm VQFN wettable flank | `FROZEN silicon`; exact population conditional | Fixed 3.3 V, 2 A, 3.6–36 V startup and 3–36 V after startup, with a 42 V absolute transient limit; 1.5 µA typical no-load IQ, PGOOD, spread spectrum, 2.2 MHz. Output is 0.750 A/2.475 W in the maximum simultaneous case, while the separate 1.050 A/3.465 W 3.3 V-display case controls thermal validation; 1.313 A is sizing margin only. Shared population: `XGL5030-222MEC`; 2 × `CGA6P1X7R1N106K250AC` CIN; 4 × `CGA6P3X7R1E226M250AB` COUT plus one DNP; 2 × `CGA3E1X7R1C105K080AC` CVCC; CBOOT DNP. Effective capacitance, thermal and stability proof remain gates. | `LM53602-Q1`; `LMQ66430MC3`. | High part; medium thermal/EMI |
| AUX5 buck | `LMQ66420MC5RXBRQ1` | Texas Instruments | AEC-Q100 | 14-pin 2.6 mm VQFN wettable flank | `CONDITIONAL PROTOTYPE FREEZE` | Fixed 5 V and normally disabled. STREET=0.56001 A, TRACK=0.92001 A, MAX=1.16001 A for ≤10 s and ≤25% rolling 60 s. Uses the same exact inductor/CIN/COUT/CVCC/CBOOT population listed for MC3; effective capacitance, efficiency, enclosure thermal behavior, and firmware enforcement require proof. | `LMQ66430MC5` or another lower-loss/higher-current part if validation fails. | Conditional; no 2 A continuous claim |
| 3.3/5 V branch switches | `TPS22919QDCKRQ1` | Texas Instruments | AEC-Q100 | SC70-6 | `FROZEN` | 1.6–5.5 V, 1.5 A, 90 mΩ typical, controlled rise, QOD, short and thermal protection, 2 nA typical off current. Populate four for GNSS_3V3, SD_3V3, DISPLAY_3V3, and DISPLAY_5V. | AP22919QDW-7; discrete MOSFET switches. | High electrically; distributor stock must be rechecked |
| Low-speed enable expansion | `TCA6408AQPWRQ1` | Texas Instruments | AEC-Q100 | TSSOP-16 | `FROZEN` | Eight I²C GPIO, reset/POR, all ports input/no-glitch after reset. External pull-downs keep every rail off before configuration and preserve GPIO42 for JTAG. | Direct GPIO allocation; TCA9539-Q1. | High |
| Vehicle-path controller | `LM74720QDRRRQ1` | Texas Instruments | AEC-Q100 | Wettable-flank WSON | `CONDITIONAL PROTOTYPE FREEZE` | 27 µA typical/35 µA maximum operating current, back-to-back N-FET drive, native OV, and true RCB while enabled. Use 2×249 kΩ top + 28.0 kΩ bottom, all 0.1%; controller+divider allocation is 60 µA at 12 V. Exact support is `CGA5H2X7R2A224K115AE` 220 nF/100 V A-to-ground; VS tied to C through `CRCW06030000Z0EA`; `CGA6N3X7R2A225K230AE` 2.2 µF/100 V C/VS-to-ground; `CGA6M3X7R1H225K200AE` 2.2 µF/50 V CAP-to-C/VS; `BZT52H-C18-Q`; `LPS3015-104MRC` 100 µH; `CRCW0603330RFKEA` 330 Ω RPD; and `CRCW0603100RFKEA` 100 Ω plus `CGA2B3X7R1H103K050BE` 10 nF slew parts. The post-Q2 filter shunts are separate. | `LM74502QDDFRQ1` superseded because it lacks enabled RCB; LTC4368 and higher-IQ controllers rejected for the stated envelope/budget. | High public-data fit; effective-C, pulse/layout and dynamic proof required |
| Vehicle path MOSFETs | `STL125N10F8AG`, quantity 2 | STMicroelectronics | AEC-Q101 | PowerFLAT 5×6 | `CONDITIONAL PROTOTYPE FREEZE` | 100 V, 4.6 mΩ maximum at 10 V gate, ±20 V VGS, 175 °C. The 74.731 V held-output negative screen leaves 25.269 V before layout overshoot. | 80 V parts rejected for margin; Infineon IAUA210N10S5N024 is a larger/costlier 100 V alternate. | Conditional on VGS/hot RDS(on), SOA, and pulse measurement |
| OBD input fuse | `0437002.WRA` | Littelfuse | AEC-Q200 | 1206 ceramic fast-acting | `CONDITIONAL PROTOTYPE FREEZE` | 2 A/63 V. MAX is prohibited below 10 V: authoritative 10 V MAX is 0.9064 A nominal/0.9545 A sensitivity. Below 9 V for 100 ms the permitted steady state is CORE/SHED, 0.1211 A nominal at 6 V. The 6 V TRACK 1.0535/1.1102 A result is transient arithmetic only. | `0437003.WRA` only after downstream fault/harness re-coordination; PPTC rejected. | Conditional |
| Positive input TVS | `SM15T47AY` | STMicroelectronics | AEC-Q101 | SMC | `CONDITIONAL PROTOTYPE FREEZE` | `VRM=40.2 V`, `VBR=44.7–49.4 V`, `VC=64.5 V at 23.2 A`; cathode to fused VBAT and anode to common midpoint. | Single bidirectional TVS and lower-standoff positive legs superseded. | Conditional on paired pulse/layout test |
| Negative input TVS | `SM15T33AY` | STMicroelectronics | AEC-Q101 | SMC | `CONDITIONAL PROTOTYPE FREEZE` | `VRM=28.2 V`, `VBR=31.4–34.7 V`, `VC=45.7 V at 33 A`; anode to common midpoint and cathode to ground. | 18–30 V legs increase pulse/fuse current; a single bidirectional device was rejected. | Conditional on forward-drop/clamp/pulse test |
| Input EMI filter | Exact Task 5A.1 damped post-switch network | See authoritative architecture document | Automotive/AEC-Q200 where applicable | — | `CONDITIONAL PROTOTYPE FREEZE` | Exact values/order codes, leakage, damping, DC-bias, inrush, impedance/stability and placement are controlled only by `task5a1-power-architecture.md`. | Do not reuse the superseded Task 4.7 envelope or invent substitutions. | Conditional bench/EMI gate |
| GNSS module | `NEO-M9N-00B` | u-blox | Professional grade, **not automotive grade** | 24-pad LCC, 12.2 × 16.0 mm | `FROZEN` | Current product, four concurrent GNSS, 2.7–3.6 V, UART, integrated SAW/LNA and antenna control; matches the project’s 20–25 Hz target subject to message/throughput validation. | u-blox automotive variants do not preserve the exact agreed M9N feature/firmware target without a new product review. | High function; medium product-grade risk |
| GNSS RF connector | `U.FL-R-SMT-1(10)` | Hirose | Commercial; Hirose requests consultation for high-reliability automotive use | U.FL SMT | `FROZEN for prototype` | Genuine 50 Ω U.FL, 6 GHz, −40…+90 °C, current/active and widely stocked. External U.FL-to-SMA pigtail remains allowed. | TE/Linx MHF-compatible receptacles; board SMA. | High prototype, medium production |
| GNSS RF ESD | `AQ3118E-01ETG` | Littelfuse | AEC-Q101, PPAP capable | SOD882/0402 | `PROPOSED CHANGE`, approved for Task 5 | Active product; bidirectional 18 V stand-off, 0.3 pF typical, 1 nA typical leakage at 18 V, ISO 10605 330 pF/330 Ω characterization. Place at U.FL with shortest ground return. | Earlier ST `ESDAXLC6-1BT2Y` is NRND; its planned replacement is not yet committed to production. | High part; RF loss/C/N0 must be measured |
| microSD socket | `104031-0811` candidate | Molex | Commercial, −25…+85 °C | SMT push-pull with detect | `PROVISIONAL` | Active and readily stocked, 0.5 A/contact, shield and card detect. It fails a possible −40 °C product target and final card access/enclosure is unknown, so its footprint is not frozen. | Higher-temperature Amphenol socket; upstream MR01A-01211. | Low mechanical/environmental |
| USB-C receptacle | `USB4105-GF-A-120` candidate | GCT | Commercial, −40…+85 °C | Hybrid SMT/through-hole USB-C | `PROVISIONAL` | USB 2.0-only 16-contact architecture, 20,000 cycles and good stock. Final shell, retention and enclosure datum are not frozen. | GCT USB4800 family; Molex USB-C receptacles. | Medium |
| USB ESD | `USBLC6-2SC6Y` | STMicroelectronics | AEC-Q101 | SOT23-6 | `FROZEN` | Active/volume production, USB 2.0 high-speed, 2.5 pF, 10 nA leakage, coordinated D+/D− and VBUS protection. | HSP051/HSP181 families. | High |
| USB input current limiter | `TPS2553QDBVRQ1` + `CRCW060360K4FKEA` | Texas Instruments / Vishay | AEC-Q100 limiter | SOT-23-6 / 0603 | `CONDITIONAL PROTOTYPE FREEZE` | 60.4 kΩ ±1% gives a 387.2–491.3 mA fault/current-limit population band, not a load contract. Prototype startup source/cable must advertise and sustain ≥500 mA at 4.75 V. Post-ramp `USB_ENUM` is ≤100 mA `MAIN_3V3` (about 82 mA VBUS); configured ceilings remain 350 mA VBUS/400 mA MAIN_3V3; MAX prohibited. This is not generic legacy USB 2.0 pre-enumeration compliance. | PPTC; TPS25221. | Conditional source/cable/startup test |
| USB 3.3 V buck | `TPS62162QDSGRQ1` | TI | AEC-Q100 | WSON | `CONDITIONAL PROTOTYPE FREEZE` | Exact conditional population: `CGA2B3X7R1H104M050BB` 100 nF; `CGA6P1X7R1E106M250AC` 10 µF/25 V CIN; `XFL3012-222MEC` 2.2 µH; `CGA6M3X7R1C106K200AB` 10 µF/16 V COUT. Effective capacitance, current-limited startup, dropout and heat remain validation gates. | Raw Schottky source OR is superseded. | Conditional inrush/capacitance/thermal gate |
| 3.3 V source mux | `TPS2116DRLR` + `EEEFK0J101AV` | Texas Instruments / Panasonic | Catalog/not AEC-qualified mux; AEC-Q200 capacitor | SOT-5X3 / SMD aluminum electrolytic | `CONDITIONAL PROTOTYPE FREEZE` | `VEH_3V3` to VIN1, `USB_3V3` to VIN2, VOUT to `MAIN_3V3`; vehicle priority and reverse isolation keep the sources separated. Populate 100 µF/6.3 V directly at VOUT. CAN VCC and AUX5 do not use mux output. | Diode OR; discrete ideal-diode pair. | Conditional transition/reverse-current/environmental test |
| Shift-light high-side switch | `TPS1H100BQPWPRQ1` | Texas Instruments | AEC-Q100 | HTSSOP-14 | `FROZEN` | 5–40 V catalog range, 4 A output class, adjustable 0.5–7 A current limit, diagnostics, short/thermal protection; configure nominal limit for the 1 A branch during schematic calculation. | TPS22919 lacks a precise 1 A fault limit; discrete switch/fuse. | High part, medium current-limit thermal design |
| Shift-light level shifter/buffer | `CAHCT1G126QDCKRQ1` | Texas Instruments | Automotive-rated logic | SC70-5 | `FROZEN` | 3–5.5 V, TTL-compatible input accepts ESP32 3.3 V, active-high OE produces safe high-Z/off until SHIFT5_EN, 8 mA output. Add connector-side series damping and ESD. | SN74AHCT1G125-Q1; discrete transistor. | High |
| Shift-light connector ESD | `PESD2USB5UVT-Q` candidate | Nexperia | AEC-Q101 | SOT23 | `PROVISIONAL` | Two 5 V lines can protect `SHIFT5` and `SHIFT_DATA_5V` at the connector; exact clamp/hot-plug/harness fit still needs review. | Single-line automotive TVS parts; larger rail TVS plus data ESD. | Medium architecture, low exact part |
| Display power/backlight interface | `TPS22919QDCKRQ1` switches plus `2N7002KQ-7` open-drain BL sink | TI / Diodes | AEC-Q100 / AEC-Q101 | SC70-6 / SOT-23 | `FROZEN electrical contract` | Separately switched 3.3 V and 5 V; open-drain BL control avoids driving backlight current from ESP32. Display module must contain its own backlight current regulator if raw LED drive is required. | Direct GPIO; fixed display-specific driver. | High contract, provisional mechanical connector |
| Sound amplifier | `TPA2005D1TDGNRQ1` | Texas Instruments | AEC-Q100 grade 2 | 8-pin MSOP PowerPAD | `FROZEN direction`; speaker `PROVISIONAL` | Active −40…+105 °C order code; 2.5–5.5 V mono filter-free class-D, differential BTL, 1.4 W into 8 Ω at 5 V/10% THD, 0.5 µA shutdown, short-circuit and thermal protection. Retains GPIO17. Exact 8 Ω, ≥1 W speaker/acoustics require testing. | Active piezo/magnetic buzzer; passive piezo; other amplifier. | High amplifier evidence, low acoustic selection |
| RGB status driver | `LP5814DRLR` | Texas Instruments | Not AEC-qualified | SOT-5X3-8 | `FROZEN architecture`; LED `PROVISIONAL` | Four I²C current-sink channels, 0.1–51 mA/channel, −40…+125 °C and 0.3 µA maximum shutdown. One common-anode RGB LED; P6 is `STATUS_DRV_EN`, hardware default off. | Direct GPIO; resistor-coded LED; external smart pixel. | Medium; board-level environmental/optical review required |

## MCU decision details

`VERIFIED_DATASHEET`: the N16R8 module has 16 MB Quad-SPI flash, 8 MB Octal-SPI PSRAM and a PCB antenna. The base ambient range is −40…+65 °C. Espressif states that enabling PSRAM ECC extends the R8 maximum ambient to +85 °C while reducing usable PSRAM by 1/16. Therefore PSRAM ECC is a hardware/firmware release requirement, not an optional optimization.

The module’s antenna end shall be placed at a board edge and the exact Espressif land-pattern keep-out in module-data-sheet Figure 10-1 plus the Hardware Design Guidelines shall be copied into the derivative board constraints. No copper, trace, component or enclosure metal may enter that controlled region without an RF review. GPIO19 is USB D− and GPIO20 USB D+. GPIO45 remains a strap pin even though the in-package-PSRAM eFuse fixes VDD_SPI; it is reserved rather than used for a removable-card select signal.

Native USB, TWAI, two UARTs, shared SPI and I²C remain available under the final map in `interfaces.md`. BLE is internal to the module. The module is not automotive-qualified; full-board temperature/RF testing remains mandatory.

## CAN comparison

| Exact part | Supplies / logic | Standby and wake | Bus/environment | Lifecycle/package | Disposition |
|---|---|---|---|---|---|
| `TCAN3404DRQ1` | Single 3.3 V | 17 µA max standby; WUP drives RXD; shutdown exists but cannot wake | ±58 V bus fault, ±30 V common mode, ±8 kV ISO 10605 bus ESD, high-Z unpowered; AEC-Q100 | TI active; SOIC-8 | **Freeze**: lowest-complexity rail-on CAN wake and direct ESP32 interface. |
| `TJA1044VT/3Z` | 5 V VCC plus 3.3 V VIO | Standby/WUP | Automotive high-speed CAN, passive unpowered behavior | NXP current product; SO8 | Reject for v1: requires an always-available 5 V CAN supply or added sequencing. |
| `TLE9251VLEXUMA1` | 5 V VCC plus VIO | <15 µA standby; bus WUP | Automotive CAN/CAN FD | Infineon marks **not for new design**, despite planned availability to 2039; TSON-8 | Reject lifecycle status and prototype package. |
| `TCAN1043A-Q1` | Battery VSUP, 5 V VCC and VIO | True sleep/INH/WAKE | Automotive system-basis features | TI active; 14 pins | Retain only as fallback if measured rail-on parked current fails. |

Passive/listen-only is enforced both by ESP32 TWAI listen-only mode and a transmit-policy invariant; no hardware transceiver alone proves the software state. Diagnostic transmission uses the same physical transceiver through the single bounded scheduler. OBD-II, ISO-TP and UDS are protocol layers and do not change the physical part.

## Input-protection calculation and conditional freeze

`CALCULATED` worst named rail power, before efficiency:

```text
PMAIN = 3.3 V × 0.750 A    = 2.47500 W
PAUX  = 5.0 V × 1.16001 A = 5.80005 W
PPEAK = 8.27505 W

simple IIN(MAX at 10 V) = 8.27505 W / (10 V × 0.80)
                         = 1.034 A
```

The 0.750 A `MAIN` value is the complete vehicle-source 3.3 V conversion bucket in the maximum simultaneous 5 V-display case and includes direct vehicle-only TCAN current. A separate 3.3 V-display case loads the MC3 output to 1.050 A/3.465 W and produces 6.765 W total rail power; 1.313 A is sizing margin, not an operating current. The 1.050 A case controls MC3 thermal validation. Physical `MAIN_3V3` is after TPS2116 and excludes CAN; a USB-only budget must be built from its actual core loads.

The authoritative mode/efficiency model gives 0.9064 A nominal and 0.9545 A with sensitivity for MAX at 10 V. MAX is prohibited below 10 V. Below 9 V for 100 ms, the permitted steady state is CORE/SHED, 0.1211 A nominal at 6 V. The 6 V TRACK result, 1.0535 A nominal/1.1102 A sensitivity, is a pre-shed transient screen only. The simple 1.034 A figure is a separate 80% `ASSUMPTION` envelope. This conditional policy does not close time-current/inrush, harness fault clearing, interrupt-rating, ageing, or transient-pulse validation.

The conditionally re-frozen order is:

```text
OBD16 -> 0437002.WRA -> VBAT_FUSED_CLAMPED
                              +-> LM74720QDRRRQ1 + 2× back-to-back STL125N10F8AG
                              |   -> exact Task 5A.1 damped filter -> FILTERED_VEHICLE
                              +-> SM15T47AY cathode
                                  SM15T47AY anode = TVS_MID = SM15T33AY anode
                                  SM15T33AY cathode -> POWER_GND
```

Native OV uses 2×249 kΩ top and 28.0 kΩ bottom, all 0.1%. The ±1 µA modeled envelope is 20.690–25.531 V for the rising OV/PD-low command and 18.815–23.366 V for falling reconnect eligibility. The design therefore remains on at 18 V, commands opening before +26 V, and permits reconnect when the source returns to 18 V. Completed switching, protected-node peak, inrush and recovery time remain dynamic gates. It does not retain the old cutoff-by-20 V or protected-node-≤24 V guarantees. Severe unsuppressed load dump remains outside v1.

### Superseded Task 4.7 input history — not for capture

The former `0437002A` spelling, `SM8SF24CA-Q`, `LM74502QDDFRQ1`, two `DMT6007LFGQ-7`, 12.458 W peak, 6 V USB crossover, and 18–20 V precision window are historical only. They are neither active selections nor current Task 5A.1 blockers.

## Regulator implementation envelope

For both 2.2 MHz fixed-output LMQ66420-Q1 candidates, TI Table 8-5 gives `2.2 µH`, `2 × 22 µF` nominal output, at least `40 µF` effective output after bias/temperature, `4.7 µF` input and `1 µF` VCC as typical 12 V starting values (`VERIFIED_DATASHEET`). Any later schematic must recalculate:

- inductor peak/saturation current over approved minimum and maximum VIN;
- capacitor effective capacitance, ripple current and voltage derating;
- IC, inductor and capacitor loss at both the 0.750 A simultaneous case and the separate 1.050 A thermal operating case for `VEH_3V3`, and at each 0.56001/0.92001/1.16001 A AUX5 policy point;
- junction temperature with the actual PCB copper and enclosure ambient;
- startup/inrush for every load combination; and
- switching-frequency/EMI interaction with GNSS, CAN, USB, SD and display clocks.

The MC3 silicon remains frozen for `VEH_3V3`; TPS2116 mux output is `MAIN_3V3`. MC5 is conditional for AUX5 only under the named STREET/TRACK/MAX policy and is not approved as a 2 A continuous source. Exact capture values and validation conditions must remain consistent with `task5a1-power-architecture.md`.

## Lifecycle, availability and prototype suitability

Manufacturer/lifecycle evidence was refreshed on 2026-08-17 for the Task 5A.1 power selection. TI lists LM74720-Q1, both LMQ66420-Q1 variants, TPS2553-Q1, TPS62162-Q1, TPS2116, TCAN3404-Q1, TPS22919-Q1, TPS1H100-Q1, TCA6408A-Q1 and the AHCT buffer as active. ST lists the SM15T-Y family and STL125N10F8AG as active; Littelfuse lists the 437A fuse family as active. TPS2116 is catalog/not AEC-qualified, so environmental qualification is a named prototype/product gate. The other Task 4 component lifecycle conclusions remain unchanged.

Prototype assembly is easiest for SOIC/TSSOP/SOT parts. LM74720's wettable-flank WSON supports side-joint inspection, while the LMQ/TPS62162 QFNs/WSON, PowerFLAT 5×6 FETs, two SMC TVSs and NEO-M9N LCC require controlled stencil/reflow and appropriate inspection. Confirm PowerFLAT voiding/land pattern and keep the TVS high-current loop out of logic/RF ground paths. The U.FL has only 30 specified mating cycles and remains a service/RF connector. Recheck all stock immediately before BOM release.

## Superseded Task 4.7 preliminary cost snapshot — historical only

The following 2026-08-14 snapshot contains the superseded controller, FET, TVS, and USB source-OR and must not be used as an active BOM or subtotal. Task 5A.1 availability and indicative pricing are controlled by `task5a1-power-architecture.md`.

| Major item / quantity per board | Prototype | 10 units | 100 units | Pricing note |
|---|---:|---:|---:|---|
| ESP32-S3-WROOM-1-N16R8 ×1 | 6.76 | 5.85 | 5.11 | DigiKey stock 6,022 |
| TCAN3404DRQ1 ×1 | 1.96 | 1.45 | 1.17 | DigiKey stock 2,350 |
| NEO-M9N-00B ×1 | 14.69 | 14.69 | 14.69 | DigiKey had no cut-tape quantity breaks; stock 6,155 |
| LMQ66420-Q1 ×2 | 8.60 | 8.60 | 8.60 | $4.30 each cut tape; reel price is lower but MOQ 3,000 |
| LM74502QDDFRQ1 ×1 | 2.42 | 2.42 | 2.42 | `ASSUMPTION`: prior H-family price placeholder; recheck exact non-H stock/price before BOM |
| DMT6007LFGQ ×2 | 2.00 | 1.60 | 1.20 | `ASSUMPTION` envelope pending distributor quote |
| TPS22919-Q1 ×4 | 1.00 | 0.69 | 0.52 | DigiKey $0.25/$0.172/$0.1304 each |
| TCA6408A-Q1 ×1 | 1.84 | 1.35 | 1.10 | Converted/rounded regional DigiKey tiers; inventory >2,000 |
| TPS2553-Q1 + PMEG6030EP-Q | 1.50 | 1.20 | 0.95 | `ASSUMPTION` envelope pending distributor quote |
| TPS1H100BQPWPRQ1 ×1 | 2.38 | 1.77 | 1.44 | DigiKey published USD regional tiers |
| CAN TVS + DNP CMC | 1.98 | 1.71 | 1.49 | CMC $1.53/$1.258/$1.034; TVS conservative $0.45 |
| USBLC6 + GNSS RF ESD | 1.10 | 0.80 | 0.60 | Distributor/ST budget envelope |
| U.FL-R-SMT-1(10) ×1 | 1.48 | 1.25 | 1.07 | DigiKey stock >270,000 |
| USB4105-GF-A-120 ×1 | 0.80 | 0.68 | 0.57 | DigiKey stock about 69,000 |
| Molex 104031-0811 ×1 | 2.04 | 1.73 | 1.47 | Provisional socket; DigiKey stock >17,000 |
| Fuse + input TVS | 4.32 | 3.00 | 2.25 | LDP01 direct $3.82/$2.49/$1.85 plus fuse envelope |
| Shift buffer + approved sound/status drivers | 4.50 | 3.50 | 2.50 | `ASSUMPTION` envelope pending exact packages/quotes |
| **Preliminary major-electronics subtotal** | **59.4** | **52.3** | **47.2** | Rounded; uncertainty roughly ±20% before full BOM |

This table is retained only to preserve Task 4.7 history; its subtotal is invalid for Task 5A.1 purchasing.

## Primary sources

- Espressif, [ESP32-S3-WROOM-1/1U data sheet v1.8](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf), Tables 1-1 and 3-1, §§2.4 and 3.3, Figures 3-1 and 10-1; and ESP32-S3 Hardware Design Guidelines.
- TI, [TCAN3404-Q1 data sheet SLLSFQ6A](https://www.ti.com/lit/ds/symlink/tcan3404-q1.pdf), Tables 7-1/7-2 and §§8–9; NXP, [TJA1044 data sheet](https://www.nxp.com/docs/en/data-sheet/TJA1044.pdf); Infineon TLE9251VLE official product page/data sheet; TI TCAN1043A-Q1 data sheet.
- ST, [ESDCANxx-2BWY data sheet DS12789](https://www.st.com/resource/en/datasheet/esdcan04-2bwy.pdf), Tables 1–2; TDK, [ACT45B data sheet](https://www.tdk-electronics.tdk.com/inf/30/ds/act45b.pdf).
- TI, [LM74720-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lm74720-q1.pdf); [LMQ664x0-Q1 data sheet SNVSBV1E](https://www.ti.com/lit/ds/symlink/lmq66420-q1.pdf), Tables 4 and 8-5; TPS2553-Q1, TPS62162-Q1, TPS2116, TPS22919-Q1, TPS1H100-Q1 and TCA6408A-Q1 official data sheets/product pages.
- ST, [SM15T-Y data sheet](https://www.st.com/resource/en/datasheet/sm15t36cay.pdf) and [STL125N10F8AG product/data-sheet page](https://www.st.com/en/power-transistors/stl125n10f8ag.html); Littelfuse [437A data sheet](https://www.littelfuse.com/assetdocs/littelfuse-fuse-437a-datasheet?assetguid=82c80a59-a4b9-4748-920b-3e2b65b813a9); Diodes 2N7002KQ data sheet.
- u-blox, [NEO-M9N-00B data sheet](https://content.u-blox.com/sites/default/files/NEO-M9N-00B_DataSheet_UBX-19014285.pdf), and [NEO-M9N Integration Manual R10](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf), §§4.5–4.7.
- Hirose U.FL-R-SMT-1(10) specification; Littelfuse AQ3118E-01ETG and ST USBLC6-2SC6Y data sheets; GCT USB4105 product specification; Molex 104031-0811 product specification.
- Current distributor availability/pricing evidence for the Task 5A.1 input, USB, and filter components is recorded in `task5a1-power-architecture.md`; recheck it before procurement.

- Worldsemi, [WS2812 family](https://world-semi.com/ws2812-family/) and WS2812B-2020 v1.3 data sheet; TI, [TPA2005D1-Q1](https://www.ti.com/product/TPA2005D1) and [LP5814](https://www.ti.com/product/LP5814).

## Remaining conditional gates

1. Capture the exact Task 5A.1 front-end, filter, USB buck/mux, and policies without substitution; then validate OV/recovery, both fast-pulse references, −14 V/60 s, fuse survival/fault clearing, reverse-current blocking, dynamic overshoot, source transitions, and downstream absolute-maximum margin before any production freeze.
2. Complete MC3 passive/loss calculations and prove MC5 STREET/TRACK/MAX thermal behavior plus the ≤10 s/≤25% rolling-60-s firmware limit.
3. Select the final display, shift-light and OBD harness connector families from mechanical/current/environment requirements.
4. Select and qualify the exact speaker, acoustic geometry and RGB LED; TPA2005D1TDGNRQ1 and LP5814DRLR directions are approved.
5. Select the exact V6-class shift-light pixel/order code and validate current, timing, EMC, cable and optics.
6. Replace or approve the provisional microSD socket for the final temperature/access requirement.
