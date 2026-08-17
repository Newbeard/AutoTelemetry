# Telemetry v1 component freeze

Status: Task 4.6 pre-schematic research gate, 2026-08-17. This document freezes component identity and electrical architecture only. It does not authorize schematic capture, PCB layout, procurement, or an automotive-compliance claim.

## Status and evidence convention

- `FROZEN`: use this exact orderable part in the first derivative schematic unless review evidence reopens the decision.
- `PROVISIONAL`: the candidate is electrically plausible, but a named missing input prevents a defensible freeze.
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
| AUX5 buck | `LMQ66420MC5RXBRQ1` | Texas Instruments | AEC-Q100 | 14-pin 2.6 mm VQFN wettable flank | `FROZEN silicon` | Same family as MAIN_3V3, fixed 5 V option, 2 A; normally disabled. Meets 1.625 A calculated peak-with-margin envelope, but simultaneous full-load operation is a validation corner. | `LMQ66430MC5`: greater margin at BOM/area cost. | Medium; thermal validation required |
| 3.3/5 V branch switches | `TPS22919QDCKRQ1` | Texas Instruments | AEC-Q100 | SC70-6 | `FROZEN` | 1.6–5.5 V, 1.5 A, 90 mΩ typical, controlled rise, QOD, short and thermal protection, 2 nA typical off current. Populate four for GNSS_3V3, SD_3V3, DISPLAY_3V3, and DISPLAY_5V. | AP22919QDW-7; discrete MOSFET switches. | High electrically; distributor stock must be rechecked |
| Low-speed enable expansion | `TCA6408AQPWRQ1` | Texas Instruments | AEC-Q100 | TSSOP-16 | `FROZEN` | Eight I²C GPIO, reset/POR, all ports input/no-glitch after reset. External pull-downs keep every rail off before configuration and preserve GPIO42 for JTAG. | Direct GPIO allocation; TCA9539-Q1. | High |
| Vehicle-path controller | `LM74502HQDDFRQ1` | Texas Instruments | AEC-Q100 grade 1 | 8-pin thin SOT-23 | `FROZEN` | 3.2–65 V, −65 V reverse withstand, 110 µA maximum operating current, programmable UV/OV and fast 11 mA gate drive for back-to-back N-MOSFETs. It does **not** provide reverse-current blocking while on; USB isolation is separate. | LM74700-Q1 lacks OV disconnect; passive diode wastes voltage/power. | High architecture |
| Vehicle path MOSFETs | `DMT6007LFGQ-7`, quantity 2 | Diodes Incorporated | AEC-Q101 | PowerDI3333-8 | `FROZEN` | 60 V, 8.5 mΩ maximum at 4.5 V gate, low loss and back-to-back load disconnect. | 80 V FETs; single-FET reverse protection. | Medium; SOA/transient test profile remains a gate |
| OBD input fuse | `0437002A` / packing suffix `WRA` | Littelfuse | AEC-Q200 | 1206 ceramic fast-acting | `FROZEN` | 2 A, 63 V fixed fuse. A fixed fuse provides a deterministic open fault instead of a high-temperature PPTC hold-current ambiguity. | RejsaCAN `MF-MSMF110/16-2`; automotive PPTC. | Medium; inrush/time-current check required |
| Input load-dump TVS | `LDP01-28AY` | STMicroelectronics | AEC-Q101 | D2PAK | `PROVISIONAL` | 24 V stand-off, 26.7 V minimum breakdown, 40 V clamp at 120 A for 10/1000 µs, 5 kW class, 1 µA leakage at 25 °C. It protects the 60/65 V front end, but final energy adequacy depends on the still-unapproved vehicle pulse/source-impedance profile. | `LDP01-26AY`; SM6F24AY; higher-standoff parts. | Medium electrical, low pulse-profile confidence |
| Input EMI filter | exact inductor and capacitors TBD | — | Automotive grade required where available | — | `PROVISIONAL` | Damped post-protection filter is required, but impedance, current, saturation, capacitor voltage/temperature derating and conducted-EMI target are not yet known. | Ferrite/LC/π alternatives. | Low until pulse/EMI profile |
| GNSS module | `NEO-M9N-00B` | u-blox | Professional grade, **not automotive grade** | 24-pad LCC, 12.2 × 16.0 mm | `FROZEN` | Current product, four concurrent GNSS, 2.7–3.6 V, UART, integrated SAW/LNA and antenna control; matches the project’s 20–25 Hz target subject to message/throughput validation. | u-blox automotive variants do not preserve the exact agreed M9N feature/firmware target without a new product review. | High function; medium product-grade risk |
| GNSS RF connector | `U.FL-R-SMT-1(10)` | Hirose | Commercial; Hirose requests consultation for high-reliability automotive use | U.FL SMT | `FROZEN for prototype` | Genuine 50 Ω U.FL, 6 GHz, −40…+90 °C, current/active and widely stocked. External U.FL-to-SMA pigtail remains allowed. | TE/Linx MHF-compatible receptacles; board SMA. | High prototype, medium production |
| GNSS RF ESD | `ESDAXLC6-1BT2Y` | STMicroelectronics | AEC-Q101 | 0402 | `FROZEN` | 0.4 pF ultra-low capacitance, RF-front-end application, ISO 10605 characterization. Place at U.FL with shortest ground return. | HSP061 family; non-automotive RF ESD diodes. | High part; RF loss must be measured |
| microSD socket | `104031-0811` candidate | Molex | Commercial, −25…+85 °C | SMT push-pull with detect | `PROVISIONAL` | Active and readily stocked, 0.5 A/contact, shield and card detect. It fails a possible −40 °C product target and final card access/enclosure is unknown, so its footprint is not frozen. | Higher-temperature Amphenol socket; upstream MR01A-01211. | Low mechanical/environmental |
| USB-C receptacle | `USB4105-GF-A-120` candidate | GCT | Commercial, −40…+85 °C | Hybrid SMT/through-hole USB-C | `PROVISIONAL` | USB 2.0-only 16-contact architecture, 20,000 cycles and good stock. Final shell, retention and enclosure datum are not frozen. | GCT USB4800 family; Molex USB-C receptacles. | Medium |
| USB ESD | `USBLC6-2SC6Y` | STMicroelectronics | AEC-Q101 | SOT23-6 | `FROZEN` | Active/volume production, USB 2.0 high-speed, 2.5 pF, 10 nA leakage, coordinated D+/D− and VBUS protection. | HSP051/HSP181 families. | High |
| USB input current limiter | `TPS2553QDBVRQ1` | Texas Instruments | AEC-Q100 | SOT-23-6 | `FROZEN` | USB power switch with adjustable current limit, soft start and reverse-voltage protection. Use 43.2 kΩ 1% for 604.6 mA nominal, 544.3–673.1 mA data-sheet range (`VERIFIED_DATASHEET`); firmware/configured USB-only load remains ≤500 mA. | PPTC; TPS25221. | High |
| USB-to-SYS reverse diode | `PMEG6030EP-Q` | Nexperia | AEC-Q101 | CFP5/SOD128 | `FROZEN` | 60 V, 3 A Schottky blocks protected vehicle voltage from USB VBUS even if SYS_IN is above 5 V. | Active ideal diode; lower-voltage Schottky. | High |
| Shift-light high-side switch | `TPS1H100BQPWPRQ1` | Texas Instruments | AEC-Q100 | HTSSOP-14 | `FROZEN` | 5–40 V catalog range, 4 A output class, adjustable 0.5–7 A current limit, diagnostics, short/thermal protection; configure nominal limit for the 1 A branch during schematic calculation. | TPS22919 lacks a precise 1 A fault limit; discrete switch/fuse. | High part, medium current-limit thermal design |
| Shift-light level shifter/buffer | `CAHCT1G126QDCKRQ1` | Texas Instruments | Automotive-rated logic | SC70-5 | `FROZEN` | 3–5.5 V, TTL-compatible input accepts ESP32 3.3 V, active-high OE produces safe high-Z/off until SHIFT5_EN, 8 mA output. Add connector-side series damping and ESD. | SN74AHCT1G125-Q1; discrete transistor. | High |
| Display power/backlight interface | `TPS22919QDCKRQ1` switches plus `2N7002KQ-7` open-drain BL sink | TI / Diodes | AEC-Q100 / AEC-Q101 | SC70-6 / SOT-23 | `FROZEN electrical contract` | Separately switched 3.3 V and 5 V; open-drain BL control avoids driving backlight current from ESP32. Display module must contain its own backlight current regulator if raw LED drive is required. | Direct GPIO; fixed display-specific driver. | High contract, provisional mechanical connector |
| Sound amplifier | `TPA2005D1-Q1` | Texas Instruments | AEC-Q100 | 8-pin MSOP PowerPAD | `PROPOSED CHANGE`; speaker `PROVISIONAL` | 2.5–5.5 V mono filter-free class-D, differential BTL, 1.4 W into 8 Ω at 5 V/10% THD, shutdown, short-circuit and thermal protection. Replaces the frozen low-side buzzer MOSFET/clamp proposal while retaining GPIO17. Exact 8 Ω, ≥1 W speaker and acoustic geometry require testing. | Active piezo/magnetic buzzer; passive piezo; speaker plus other amplifier. | High amplifier evidence, low acoustic selection |
| RGB status driver | `LP5814DRLR` | Texas Instruments | Not AEC-qualified | SOT-5X3-8 | `PROPOSED CHANGE`; LED `PROVISIONAL` | Four I²C current-sink channels, 0.1–51 mA/channel, 8-bit current/PWM, 23 kHz PWM, 0.1 µA shutdown typical. Drives one common-anode RGB LED without three MCU GPIOs; P6 becomes `STATUS_DRV_EN`. | Direct GPIO; resistor-coded LED; external smart pixel. | Medium pending qualification/optical review |

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

## Input-protection calculations and unresolved freeze

`CALCULATED` worst named rail power, before efficiency:

```text
PLOAD = 3.3 V × 1.313 A + 5.0 V × 1.625 A
      = 4.333 W + 8.125 W
      = 12.458 W

IIN(12 V, 80% assumed efficiency) = 12.458 W / (12 V × 0.80)
                                    = 1.298 A

2 A fuse current margin = (2.000 - 1.298) / 1.298 = 54.1%
```

The 80% simultaneous-envelope efficiency is an `ASSUMPTION`; normal operation will be lower than this constructed simultaneous peak. The fuse’s time-current curve, ambient derating, input capacitor inrush and harness fault clearing still require schematic review. The 2 A result is a design starting point, not proof that the fuse coordinates with a vehicle harness.

Vehicle path order is frozen as:

```text
OBD16 -> 0437002A fuse -> LDP01-28AY TVS to POWER_GND
      -> LM74502H-Q1 + back-to-back DMT6007LFGQ MOSFETs
      -> damped input EMI filter -> VEHICLE_PROTECTED -> SYS_IN
```

The LDP01-28AY and exact EMI passives remain `PROVISIONAL` until the project approves minimum crank, jump-start duration, positive/negative ISO 7637-2 pulses, load-dump source impedance/energy, maximum ambient and desired failure behavior. The selected 60 V MOSFETs and 65 V controller exceed the TVS’s documented 45 V maximum 8/20 µs clamp point, but this comparison does not establish energy survival.

The LM74502H UV threshold shall be calculated after the crank requirement is approved; the OV target is initially below the buck’s 36 V operating maximum with tolerance included. Brownout uses PGOOD/ESP brownout indication, stops new writes, closes the log within measured hold-up time, and never transmits merely because voltage changed. No hold-up capacitor value is frozen without a measured SD close/flush time.

## Regulator implementation envelope

For both 2.2 MHz fixed-output LMQ66420-Q1 devices, TI Table 8-5 gives `2.2 µH`, `2 × 22 µF` nominal output, at least `40 µF` effective output after bias/temperature, `4.7 µF` input and `1 µF` VCC as typical 12 V starting values (`VERIFIED_DATASHEET`). The schematic must recalculate:

- inductor peak/saturation current over approved minimum and maximum VIN;
- capacitor effective capacitance, ripple current and voltage derating;
- IC, inductor and capacitor loss at 1.313 A MAIN_3V3 and 1.625 A AUX5 peaks;
- junction temperature with the actual PCB copper and enclosure ambient;
- startup/inrush for every load combination; and
- switching-frequency/EMI interaction with GNSS, CAN, USB, SD and display clocks.

The silicon is frozen; those passives and thermal conclusions are not.

## Lifecycle, availability and prototype suitability

Manufacturer status was checked on 2026-08-14 for the Task 4 parts and on 2026-08-17 for TPA2005D1-Q1 and LP5814. TI lists both proposed additions as active. TI marks TCAN3404-Q1, both LMQ66420-Q1 variants, LM74502H-Q1, TPS22919-Q1, TPS2553-Q1, TPS1H100-Q1, TCA6408A-Q1 and the AHCT buffer active. ST marks USBLC6-2SC6Y and LDP01-28AY active; TDK lists ACT45B-510 as production; Nexperia lists PMEG6030EP-Q as production; u-blox presents NEO-M9N-00B as a current variant. Infineon’s rejected TLE9251VLE is specifically “not for new design.”

Prototype assembly is easiest for SOIC/TSSOP/SOT parts. The 2.6 mm LMQ VQFN, PowerDI3333 MOSFET and NEO-M9N LCC require stencil/reflow and inspection but are realistic for professional prototype assembly. The U.FL has only 30 specified mating cycles and is a service/RF connector, not a daily user connector. `TPS22919QDCKRQ1` showed conflicting DigiKey regional snapshots (one out of stock, another large inventory), so stock must be checked immediately before BOM release.

## Preliminary major-electronics cost envelope

Prices are USD per board, distributor web pricing observed 2026-08-14, excluding VAT/tariffs/shipping, passives, PCB, assembly, enclosure, harness, display, antenna/pigtail and microSD card. Quantities without a published break use the next available/conservative single-unit price. These are estimates, not a purchasing authorization.

| Major item / quantity per board | Prototype | 10 units | 100 units | Pricing note |
|---|---:|---:|---:|---|
| ESP32-S3-WROOM-1-N16R8 ×1 | 6.76 | 5.85 | 5.11 | DigiKey stock 6,022 |
| TCAN3404DRQ1 ×1 | 1.96 | 1.45 | 1.17 | DigiKey stock 2,350 |
| NEO-M9N-00B ×1 | 14.69 | 14.69 | 14.69 | DigiKey had no cut-tape quantity breaks; stock 6,155 |
| LMQ66420-Q1 ×2 | 8.60 | 8.60 | 8.60 | $4.30 each cut tape; reel price is lower but MOQ 3,000 |
| LM74502HQDDFRQ1 ×1 | 2.42 | 2.42 | 2.42 | DigiKey stock 8,744; only single/reel result captured |
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
| Shift buffer + proposed sound/status drivers | 4.50 | 3.50 | 2.50 | `ASSUMPTION` envelope pending exact packages/quotes |
| **Preliminary major-electronics subtotal** | **59.4** | **52.3** | **47.2** | Rounded; uncertainty roughly ±20% before full BOM |

The cost risk is dominated by GNSS, the two bucks and the ESP32 module. The subtotal includes provisional input/mechanical parts to avoid understating the prototype, but it is not a final BOM cost.

## Primary sources

- Espressif, [ESP32-S3-WROOM-1/1U data sheet v1.8](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf), Tables 1-1 and 3-1, §§2.4 and 3.3, Figures 3-1 and 10-1; and ESP32-S3 Hardware Design Guidelines.
- TI, [TCAN3404-Q1 data sheet SLLSFQ6A](https://www.ti.com/lit/ds/symlink/tcan3404-q1.pdf), Tables 7-1/7-2 and §§8–9; NXP, [TJA1044 data sheet](https://www.nxp.com/docs/en/data-sheet/TJA1044.pdf); Infineon TLE9251VLE official product page/data sheet; TI TCAN1043A-Q1 data sheet.
- ST, [ESDCANxx-2BWY data sheet DS12789](https://www.st.com/resource/en/datasheet/esdcan04-2bwy.pdf), Tables 1–2; TDK, [ACT45B data sheet](https://www.tdk-electronics.tdk.com/inf/30/ds/act45b.pdf).
- TI, [LMQ664x0-Q1 data sheet SNVSBV1E](https://www.ti.com/lit/ds/symlink/lmq66420-q1.pdf), Tables 4 and 8-5; LM74502H-Q1, TPS22919-Q1, TPS2553-Q1, TPS1H100-Q1 and TCA6408A-Q1 official data sheets/product pages.
- Diodes Incorporated, DMT6007LFGQ and 2N7002KQ data sheets; Littelfuse 437A data sheet; ST [LDP01-xxAY data sheet](https://www.st.com/resource/en/datasheet/ldp01-28ay.pdf); Nexperia PMEG6030EP-Q data sheet.
- u-blox, [NEO-M9N-00B data sheet](https://content.u-blox.com/sites/default/files/NEO-M9N-00B_DataSheet_UBX-19014285.pdf), and [NEO-M9N Integration Manual R10](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf), §§4.5–4.7.
- Hirose U.FL-R-SMT-1(10) specification; ST ESDAXLC6-1BT2Y and USBLC6-2SC6Y data sheets; GCT USB4105 product specification; Molex 104031-0811 product specification.
- Distributor stock/prices: DigiKey product pages for ESP32-S3-WROOM-1-N16R8, TCAN3404DRQ1, NEO-M9N-00B, LMQ66420MC3RXBRQ1, LM74502HQDDFRQ1, TPS22919QDCKRQ1, TCA6408AQPWRQ1, TPS1H100BQPWPRQ1, ACT45B-510-2P-TL003, U.FL-R-SMT-1(10), USB4105-GF-A-120 and 104031-0811; ST eStore for LDP01-28AY.

- Worldsemi, [WS2812 family](https://world-semi.com/ws2812-family/) and WS2812B-2020 v1.3 data sheet; TI, [TPA2005D1-Q1](https://www.ti.com/product/TPA2005D1) and [LP5814](https://www.ti.com/product/LP5814).

## Remaining component blockers

1. Approve the 12 V passenger-vehicle electrical test profile; then validate/freeze the input TVS and EMI components.
2. Complete regulator magnetics, capacitor, loss and enclosure thermal calculations.
3. Select the final display, shift-light and OBD harness connector families from mechanical/current/environment requirements.
4. Approve or reject the proposed TPA2005D1-Q1 and LP5814 changes; then select and qualify the exact speaker, acoustic geometry and RGB LED.
5. Select the exact V6-class shift-light pixel/order code and validate current, timing, EMC, cable and optics.
5. Replace or approve the provisional microSD socket for the final temperature/access requirement.

