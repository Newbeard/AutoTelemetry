# Telemetry v1 component freeze

Status: Task 4.7 pre-schematic electrical/protection gate, 2026-08-17. This document freezes component identity and electrical architecture only. It does not authorize schematic capture, PCB layout, procurement, or an automotive-compliance claim.

## Task 5A superseding gate

Status: **BLOCKED**. The Task 4.7 table and rationale below are retained as historical context, but the following later findings supersede their `FROZEN` or approved wording. See [`task5a-power-calculations.md`](task5a-power-calculations.md) for the primary-source equations and calculations. No affected value or substitute part is approved for schematic capture.

- `LM74502QDDFRQ1` threshold implementation is `REOPENED`: its published UV/OV spread cannot guarantee both the 6 V/USB crossover and 18–20 V OV windows. Its operating quiescent current is 45 µA typical/65 µA maximum; the historical 110 µA figure is a conservative `DESIGN_REQUIREMENT` allocation, not a `VERIFIED_DATASHEET` maximum.
- `SM8SF24CA-Q` is `REOPENED`: its 24 V working standoff does not cover the required +26 V/60 s jump start, and a higher-standoff TVS must be coordinated with the 60 V MOSFETs and the defined +38 V pulse.
- `0437002A` is `REOPENED` with the full-load policy: Littelfuse's continuous-use guidance gives 1.60 A at 25 °C and 1.36 A in its 75 °C example; the 12.458 W envelope exceeds those limits below 9.73 V and 11.45 V respectively. Define low-voltage/hot load shedding or reopen the fuse rating and fault coordination.
- `DMT6007LFGQ-7` is `REOPENED` for negative-pulse coordination. Normal conduction loss remains acceptable, but a held 24 V output at the TVS's published 38.9 V clamp point can impose about 62.2 V across a 60 V MOSFET; clamp voltage, overshoot and pulse precondition remain unresolved.
- The damped post-switch filter topology remains a starting direction, but its exact implementation is `REOPENED`. The conditional 100 µF hybrid damping capacitor adds up to 50 µA leakage and raises the doubled parked envelope from 425.08 µA to 525.08 µA, above the 0.5 mA stretch target.
- `LMQ66420MC5RXBRQ1` for AUX5 is `REOPENED`: the 2 A electrical rating does not establish guaranteed continuous 2 A operation at 85 °C. The 1.30 A named load is conditional on measured efficiency and achieved `RθJA ≤50 °C/W`; 1.625 A must be managed/short-duration unless further evidence closes the thermal gate.

## Status and evidence convention

- `FROZEN`: use this exact orderable part in the first derivative schematic unless review evidence reopens the decision.
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
| MAIN_3V3 buck | `LMQ66420MC3RXBRQ1` | Texas Instruments | AEC-Q100 | 14-pin 2.6 mm VQFN wettable flank | `FROZEN silicon` | Fixed 3.3 V option, 2 A, 3–36 V operating/42 V transient, 1.5 µA typical no-load IQ, PGOOD, spread spectrum, 2.2 MHz. Meets 1.313 A calculated peak-with-margin requirement. Magnetics/capacitors remain schematic calculations. | `LM53602-Q1`: 23 µA typical IQ; `LMQ66430MC3`: more current but no budget need. | High part; medium thermal/EMI |
| AUX5 buck | `LMQ66420MC5RXBRQ1` | Texas Instruments | AEC-Q100 | 14-pin 2.6 mm VQFN wettable flank | `REOPENED` — Task 5A thermal gate | Fixed 5 V, 2 A electrical class and normally disabled. At 85 °C, 1.30 A is conditional on measured efficiency and `RθJA ≤50 °C/W`; 1.625 A is managed/short-duration pending proof, and 2 A continuous is not defensible from public data. | `LMQ66430MC5` or another lower-loss/higher-current part; alternatively approve a managed-load requirement. | Blocked continuous-load definition and thermal proof |
| 3.3/5 V branch switches | `TPS22919QDCKRQ1` | Texas Instruments | AEC-Q100 | SC70-6 | `FROZEN` | 1.6–5.5 V, 1.5 A, 90 mΩ typical, controlled rise, QOD, short and thermal protection, 2 nA typical off current. Populate four for GNSS_3V3, SD_3V3, DISPLAY_3V3, and DISPLAY_5V. | AP22919QDW-7; discrete MOSFET switches. | High electrically; distributor stock must be rechecked |
| Low-speed enable expansion | `TCA6408AQPWRQ1` | Texas Instruments | AEC-Q100 | TSSOP-16 | `FROZEN` | Eight I²C GPIO, reset/POR, all ports input/no-glitch after reset. External pull-downs keep every rail off before configuration and preserve GPIO42 for JTAG. | Direct GPIO allocation; TCA9539-Q1. | High |
| Vehicle-path controller | `LM74502QDDFRQ1` | Texas Instruments | AEC-Q100 grade 1 | 8-pin thin SOT-23 | `REOPENED` — Task 5A `BLOCKED` | TI characterizes 4–60 V operation and specifies 45 µA typical/65 µA maximum operating IQ; 110 µA is only a conservative `DESIGN_REQUIREMENT` allocation. The non-H variant supports external `Cdvdt`, but published UV/OV spread cannot guarantee the required source-crossover and OV windows. It does **not** reverse-block while on. | Earlier `LM74502HQDDFRQ1` remains rejected; use a precision supervisor, a tighter reverse-blocking controller, or changed voltage/crossover requirements. | Blocked threshold implementation; conditional inrush direction only |
| Vehicle path MOSFETs | `DMT6007LFGQ-7`, quantity 2 | Diodes Incorporated | AEC-Q101 | PowerDI3333-8 | `REOPENED` — Task 5A | 60 V, 8.5 mΩ maximum at 4.5 V gate; normal conduction loss is acceptable. Negative-pulse stress can reach about 62.2 V for a held 24 V output at the published 38.9 V clamp point, exceeding the rating. | 80 V FETs; single-FET reverse protection. | Blocked negative-pulse coordination; hot SOA still unproven |
| OBD input fuse | `0437002A` / packing suffix `WRA` | Littelfuse | AEC-Q200 | 1206 ceramic fast-acting | `REOPENED` — Task 5A | 2 A, 63 V fixed fuse. Continuous-use guidance is 1.60 A at 25 °C and 1.36 A in Littelfuse's 75 °C example; full load requires low-voltage/hot load shedding or a reopened fuse and fault design. | RejsaCAN `MF-MSMF110/16-2`; automotive PPTC; higher-rated fuse with complete downstream coordination. | Blocked full-load/thermal policy; fault current unresolved |
| Input load-dump TVS | `SM8SF24CA-Q` | Bourns | AEC-Q101 | 8.1 × 10.5 × 1.3 mm DFN | `REOPENED` — Task 5A | Bidirectional polarity remains required for −14 V reverse battery, but 24 V working standoff does not cover +26 V/60 s. A higher-standoff TVS must be coordinated with the 60 V MOSFETs and the defined +38 V pulse. | Vishay `SM8S24CA` credible larger alternate; higher-standoff SM8SF variants; ST `LDP01-28AY` remains rejected at this position because it is unidirectional. | Blocked jump-start and TVS/FET coordination |
| Input EMI filter | 2.2–4.7 µH, ≥3 A/≥4 A saturation post-switch damped C-L-C envelope | exact order codes TBD | Automotive grade/AEC-Q200 where applicable | — | Topology retained; damping implementation `REOPENED` | The post-switch damped topology remains a starting point, but exact impedance, bias derating, inrush and converter interaction remain unproved. The conditional 100 µF hybrid capacitor adds up to 50 µA leakage and raises the doubled parked envelope to 525.08 µA. | Lower-leakage damping network; approved parked-current relaxation; ferrite-only option only if EMI testing proves sufficient. | Blocked exact passives/leakage; topology direction retained |
| GNSS module | `NEO-M9N-00B` | u-blox | Professional grade, **not automotive grade** | 24-pad LCC, 12.2 × 16.0 mm | `FROZEN` | Current product, four concurrent GNSS, 2.7–3.6 V, UART, integrated SAW/LNA and antenna control; matches the project’s 20–25 Hz target subject to message/throughput validation. | u-blox automotive variants do not preserve the exact agreed M9N feature/firmware target without a new product review. | High function; medium product-grade risk |
| GNSS RF connector | `U.FL-R-SMT-1(10)` | Hirose | Commercial; Hirose requests consultation for high-reliability automotive use | U.FL SMT | `FROZEN for prototype` | Genuine 50 Ω U.FL, 6 GHz, −40…+90 °C, current/active and widely stocked. External U.FL-to-SMA pigtail remains allowed. | TE/Linx MHF-compatible receptacles; board SMA. | High prototype, medium production |
| GNSS RF ESD | `AQ3118E-01ETG` | Littelfuse | AEC-Q101, PPAP capable | SOD882/0402 | `PROPOSED CHANGE`, approved for Task 5 | Active product; bidirectional 18 V stand-off, 0.3 pF typical, 1 nA typical leakage at 18 V, ISO 10605 330 pF/330 Ω characterization. Place at U.FL with shortest ground return. | Earlier ST `ESDAXLC6-1BT2Y` is NRND; its planned replacement is not yet committed to production. | High part; RF loss/C/N0 must be measured |
| microSD socket | `104031-0811` candidate | Molex | Commercial, −25…+85 °C | SMT push-pull with detect | `PROVISIONAL` | Active and readily stocked, 0.5 A/contact, shield and card detect. It fails a possible −40 °C product target and final card access/enclosure is unknown, so its footprint is not frozen. | Higher-temperature Amphenol socket; upstream MR01A-01211. | Low mechanical/environmental |
| USB-C receptacle | `USB4105-GF-A-120` candidate | GCT | Commercial, −40…+85 °C | Hybrid SMT/through-hole USB-C | `PROVISIONAL` | USB 2.0-only 16-contact architecture, 20,000 cycles and good stock. Final shell, retention and enclosure datum are not frozen. | GCT USB4800 family; Molex USB-C receptacles. | Medium |
| USB ESD | `USBLC6-2SC6Y` | STMicroelectronics | AEC-Q101 | SOT23-6 | `FROZEN` | Active/volume production, USB 2.0 high-speed, 2.5 pF, 10 nA leakage, coordinated D+/D− and VBUS protection. | HSP051/HSP181 families. | High |
| USB input current limiter | `TPS2553QDBVRQ1` | Texas Instruments | AEC-Q100 | SOT-23-6 | `FROZEN` | USB power switch with adjustable current limit, soft start and reverse-voltage protection. Use 43.2 kΩ 1% for 604.6 mA nominal, 544.3–673.1 mA data-sheet range (`VERIFIED_DATASHEET`); firmware/configured USB-only load remains ≤500 mA. | PPTC; TPS25221. | High |
| USB-to-SYS reverse diode | `PMEG6030EP-Q` | Nexperia | AEC-Q101 | CFP5/SOD128 | `FROZEN` | 60 V, 3 A Schottky blocks protected vehicle voltage from USB VBUS even if SYS_IN is above 5 V. | Active ideal diode; lower-voltage Schottky. | High |
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

## Input-protection calculation and freeze

`CALCULATED` worst named rail power, before efficiency:

```text
PLOAD = 3.3 V × 1.313 A + 5.0 V × 1.625 A
      = 4.333 W + 8.125 W
      = 12.458 W

IIN(12 V, 80% assumed efficiency) = 12.458 W / (12 V × 0.80)
                                    = 1.298 A

2 A fuse current margin = (2.000 - 1.298) / 1.298 = 54.1%
```

The 80% simultaneous-envelope efficiency is an `ASSUMPTION`; normal operation will be lower than this constructed simultaneous peak. Task 5A supersedes the simple 12 V nameplate-margin framing: the full envelope draws 2.595 A at 6 V and 1.730 A at 9 V. The fuse/load-shedding policy, time-current/inrush behavior, harness fault clearing and applicable interrupt rating are `REOPENED`.

The Task 4.7 vehicle-path order, retained as historical architecture context, was:

```text
OBD16 -> 0437002A fuse -> SM8SF24CA-Q bidirectional TVS to POWER_GND
      -> LM74502-Q1 + back-to-back DMT6007LFGQ MOSFETs
      -> damped post-switch C-L-C filter -> VEHICLE_PROTECTED -> SYS_IN
```

The Task 4.7 envelope was 6–18 V operation, +26 V/60 s jump-start survival by disconnect, −14 V/60 s reverse survival, and +38 V suppressed-load-dump source survival by disconnect. Task 5A found that the TVS remains connected ahead of the disconnect and its 24 V working standoff does not cover +26 V/60 s. TVS/FET coordination is therefore `REOPENED`. `VEHICLE_PROTECTED` must still remain ≤24 V in any approved test matrix. Severe unsuppressed public reference conditions remain outside the v1 guarantee.

The 6.0 V UV and 18.0 V OV values were Task 4.7 nominal targets, not guaranteed thresholds. Task 5A proved that LM74502-Q1 tolerances cannot simultaneously guarantee the required 6 V/USB source crossover or operation through 18 V with cutoff by 20 V. The threshold implementation is `REOPENED`; conditional `Cdvdt` and filter calculations do not authorize capture. Brownout still uses controlled reset and full crank ride-through is not required.

## Regulator implementation envelope

For both 2.2 MHz fixed-output LMQ66420-Q1 candidates, TI Table 8-5 gives `2.2 µH`, `2 × 22 µF` nominal output, at least `40 µF` effective output after bias/temperature, `4.7 µF` input and `1 µF` VCC as typical 12 V starting values (`VERIFIED_DATASHEET`). Any later schematic must recalculate:

- inductor peak/saturation current over approved minimum and maximum VIN;
- capacitor effective capacitance, ripple current and voltage derating;
- IC, inductor and capacitor loss at 1.313 A MAIN_3V3 and 1.625 A AUX5 peaks;
- junction temperature with the actual PCB copper and enclosure ambient;
- startup/inrush for every load combination; and
- switching-frequency/EMI interaction with GNSS, CAN, USB, SD and display clocks.

The MAIN_3V3 MC3 silicon remains frozen. The AUX5 MC5 selection and its continuous-load requirement are `REOPENED`; neither candidate has approved exact passives or final thermal conclusions.

## Lifecycle, availability and prototype suitability

Manufacturer status was checked on 2026-08-14 for the Task 4 parts and on 2026-08-17 for TPA2005D1-Q1 and LP5814. TI lists the approved TPA2005D1TDGNRQ1 and LP5814 additions as active. TI marks TCAN3404-Q1, both LMQ66420-Q1 variants, LM74502-Q1, TPS22919-Q1, TPS2553-Q1, TPS1H100-Q1, TCA6408A-Q1 and the AHCT buffer active. Bourns lists SM8SF-Q, Littelfuse lists AQ3118E-01ETG active, ST marks USBLC6-2SC6Y active and the rejected LDP01-28AY candidate active; TDK lists ACT45B-510 as production; Nexperia lists PMEG6030EP-Q as production; u-blox presents NEO-M9N-00B as a current variant. Infineon’s rejected TLE9251VLE is specifically “not for new design.”

Prototype assembly is easiest for SOIC/TSSOP/SOT parts. The 2.6 mm LMQ VQFN, PowerDI3333 MOSFET and NEO-M9N LCC require stencil/reflow and inspection but are realistic for professional prototype assembly. The U.FL has only 30 specified mating cycles and is a service/RF connector, not a daily user connector. `TPS22919QDCKRQ1` showed conflicting DigiKey regional snapshots (one out of stock, another large inventory), so stock must be checked immediately before BOM release.

## Preliminary major-electronics cost envelope

Prices are USD per board, distributor web pricing observed 2026-08-14, excluding VAT/tariffs/shipping, passives, PCB, assembly, enclosure, harness, display, antenna/pigtail and microSD card. Quantities without a published break use the next available/conservative single-unit price. These are estimates, not a purchasing authorization.

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

The cost risk is dominated by GNSS, the two bucks and the ESP32 module. The subtotal includes provisional input/mechanical parts to avoid understating the prototype, but it is not a final BOM cost.

## Primary sources

- Espressif, [ESP32-S3-WROOM-1/1U data sheet v1.8](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf), Tables 1-1 and 3-1, §§2.4 and 3.3, Figures 3-1 and 10-1; and ESP32-S3 Hardware Design Guidelines.
- TI, [TCAN3404-Q1 data sheet SLLSFQ6A](https://www.ti.com/lit/ds/symlink/tcan3404-q1.pdf), Tables 7-1/7-2 and §§8–9; NXP, [TJA1044 data sheet](https://www.nxp.com/docs/en/data-sheet/TJA1044.pdf); Infineon TLE9251VLE official product page/data sheet; TI TCAN1043A-Q1 data sheet.
- ST, [ESDCANxx-2BWY data sheet DS12789](https://www.st.com/resource/en/datasheet/esdcan04-2bwy.pdf), Tables 1–2; TDK, [ACT45B data sheet](https://www.tdk-electronics.tdk.com/inf/30/ds/act45b.pdf).
- TI, [LMQ664x0-Q1 data sheet SNVSBV1E](https://www.ti.com/lit/ds/symlink/lmq66420-q1.pdf), Tables 4 and 8-5; LM74502-Q1, TPS22919-Q1, TPS2553-Q1, TPS1H100-Q1 and TCA6408A-Q1 official data sheets/product pages.
- Diodes Incorporated, DMT6007LFGQ and 2N7002KQ data sheets; Littelfuse 437A data sheet; ST [LDP01-xxAY data sheet](https://www.st.com/resource/en/datasheet/ldp01-28ay.pdf); Nexperia PMEG6030EP-Q data sheet.
- u-blox, [NEO-M9N-00B data sheet](https://content.u-blox.com/sites/default/files/NEO-M9N-00B_DataSheet_UBX-19014285.pdf), and [NEO-M9N Integration Manual R10](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf), §§4.5–4.7.
- Hirose U.FL-R-SMT-1(10) specification; Littelfuse AQ3118E-01ETG and ST USBLC6-2SC6Y data sheets; GCT USB4105 product specification; Molex 104031-0811 product specification.
- Distributor stock/prices: DigiKey product pages for ESP32-S3-WROOM-1-N16R8, TCAN3404DRQ1, NEO-M9N-00B, LMQ66420MC3RXBRQ1, LM74502QDDFRQ1, TPS22919QDCKRQ1, TCA6408AQPWRQ1, TPS1H100BQPWPRQ1, ACT45B-510-2P-TL003, U.FL-R-SMT-1(10), USB4105-GF-A-120 and 104031-0811; Bourns SM8SF-Q and Littelfuse AQ3118E-01ETG official product data; ST LDP01-28AY retained as rejected-candidate evidence.

- Worldsemi, [WS2812 family](https://world-semi.com/ws2812-family/) and WS2812B-2020 v1.3 data sheet; TI, [TPA2005D1-Q1](https://www.ti.com/product/TPA2005D1) and [LP5814](https://www.ti.com/product/LP5814).

## Remaining component blockers

1. Resolve the Task 5A vehicle-input threshold, TVS/FET, fuse/load-policy, AUX5-load and filter-leakage decision set before schematic capture.
2. Complete MAIN regulator magnetics/capacitor/loss calculations and select or qualify the AUX5 regulator against the enclosure thermal requirement.
3. Select the final display, shift-light and OBD harness connector families from mechanical/current/environment requirements.
4. Select and qualify the exact speaker, acoustic geometry and RGB LED; TPA2005D1TDGNRQ1 and LP5814DRLR directions are approved.
5. Select the exact V6-class shift-light pixel/order code and validate current, timing, EMC, cable and optics.
6. Replace or approve the provisional microSD socket for the final temperature/access requirement.
