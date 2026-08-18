# Telemetry v1 schematic architecture

Status: Task 5A.1 conditional pre-schematic power-architecture selection, 2026-08-17. This document is a controlled prototype/capture starting point only. No KiCad project, schematic sheet, ERC run, PCB placement/layout, manufacturing package or compliance approval is authorized by this amendment. Detailed calculations and bench gates are in [`task5a1-power-architecture.md`](task5a1-power-architecture.md); the earlier Task 5A conflict record remains in [`task5a-power-calculations.md`](task5a-power-calculations.md).

## Sheet hierarchy and controlled net names

| Sheet | Functional ownership | Principal ports/nets |
|---:|---|---|
| 1 | OBD and vehicle entry | `VBAT_OBD_RAW`, `POWER_GND`, `CANH_OBD`, `CANL_OBD` |
| 2 | Vehicle protection/source isolation | `VBAT_FUSED_CLAMPED`, `VBAT_SWITCHED`, `FILTERED_VEHICLE`, `VEHICLE_PRESENT`, `PWR_FAULT_N` |
| 3 | Vehicle/USB conversion and core-source mux | `FILTERED_VEHICLE`, `VEH_3V3`, `USB5_PROTECTED`, `USB_3V3`, `MAIN_3V3`, `AUX5`, PGOOD and enable nets |
| 4 | ESP32-S3, reset, straps and low-speed expander | all `GPIOxx`, `I2C_SDA/SCL`, `EXP_Px` |
| 5 | CAN physical layer | `VEH_3V3`, `CANH_OBD`, `CANL_OBD`, `CAN_RX`, `CAN_TX`, `CAN_STB` |
| 6 | GNSS digital/power | `GNSS_3V3`, `GNSS_RX/TX`, reset/timepulse test nets |
| 7 | GNSS RF/active antenna | `GNSS_RF`, `ANT_BIAS`, U.FL and RF ground |
| 8 | microSD/shared SPI | `SD_3V3`, `SPI_SCLK/MOSI/MISO`, `SD_CS`, `SD_CD_N` |
| 9 | USB-C and isolated USB source | `USB_VBUS_RAW`, `USB5_PROTECTED`, `USB_3V3`, `USB_D_N/P`, `USB_PRESENT` |
| 10 | Display interface | `DISPLAY_3V3`, `DISPLAY_5V`, shared SPI and display controls |
| 11 | Shift-light output | `SHIFT5`, `SHIFT_DATA_5V`, fault/status |
| 12 | Sounder, MODE and status | `SOUND_PWM`, `MODE_N`, `STATUS_DRV_EN` |
| 13 | Debug and test points | UART0, power, CAN logic/bus and GNSS digital test points |

No Telemetry v1 sheet adds TPMS, tire temperature, IMU, analog sensor hubs, an external sensor bus or a second CAN channel.

### Hierarchical block contracts

| Block | Inputs | Outputs / rails | Control / frozen parts | Constraints, test points and dependencies |
|---|---|---|---|---|
| OBD entry | OBD16, OBD4/5, OBD6/14 | `VBAT_OBD_RAW`, `POWER_GND`, `CANH_OBD`, `CANL_OBD` | Connector/harness provisional | Pins 4/5 join once at entry; TP raw battery/ground; connector mechanical freeze required |
| Vehicle protection | `VBAT_OBD_RAW` | `VBAT_FUSED_CLAMPED`, `VBAT_SWITCHED`, `FILTERED_VEHICLE`, `VEHICLE_PRESENT`, `PWR_FAULT_N` | Fuse; anti-series `SM15T47AY` + `SM15T33AY`; `LM74720QDRRRQ1`; two 100 V N-FETs; selected post-disconnect filter | Conditional capture selection only; fuse/TVS/FET/SOA, filter impedance, transient and EMI tests remain gates |
| Power conversion/mux | `FILTERED_VEHICLE`, `USB5_PROTECTED` | `VEH_3V3`, `USB_3V3`, `MAIN_3V3`, `AUX5`, PGOOD | Vehicle MAIN `LMQ66420MC3RXBRQ1`; USB `TPS62162QDSGRQ1`; core mux `TPS2116DRLR`; vehicle-only AUX LMQ66420 | `FILTERED_VEHICLE` feeds both LMQ converters; MAIN core may use either 3.3 V source; CAN/AUX5 remain vehicle-only; TP all rails/PGOOD/MODE/PR1; low-voltage, thermal and stability gates |
| MCU/expander | `MAIN_3V3`, reset/USB/CAN/GNSS/SPI inputs | GPIO controls, I2C, reset defaults | ESP32-S3-WROOM-1-N16R8, TCA6408A-Q1 | GPIO0/3/45/46 strap rules; all power enables default off; logic test pads only |
| CAN PHY | `CANH/L_OBD`, `CAN_TX`, `CAN_STB`, `VEH_3V3` | `CANH/L_PHY`, `CAN_RX/WUP` | TCAN3404DRQ1, ESDCAN04-2BWY, ACT45B-510 DNP, split 120 Ω DNP | Vehicle-only VCC; USB never powers CAN; product termination OFF; compact CANH/L and logic TPs; unpowered-I/O validation required |
| GNSS digital/power | MAIN_3V3, UART/control | `GNSS_3V3`, `GNSS_RX/TX`, reset/timepulse | NEO-M9N-00B, TPS22919-Q1 | Off parked; TP rail/UART/reset/timepulse; active-antenna and RF-sheet dependency |
| GNSS RF | `GNSS_3V3`, module RF | U.FL `GNSS_RF`, `ANT_BIAS` | AQ3118E-01ETG; u-blox bias-T topology; limiter/antenna provisional | 50 Ω, ≤0.5 pF ESD, no RF TP/stub; layout/VNA/C/N0 review required |
| microSD | MAIN_3V3, shared SPI, GPIO11 CS | `SD_3V3`, card detect/data | TPS22919-Q1; socket/card provisional | Power-off isolation and SD corruption recovery; TP rail/current link; mechanical access dependency |
| USB-C/source | VBUS, D+/D−, CC1/2, vehicle PGOOD | `USB5_PROTECTED`, `USB_3V3`, `USB_PRESENT`, `MAIN_3V3` | USBLC6-2SC6Y, TPS2553QDBVRQ1, TPS62162QDSGRQ1, TPS2116DRLR | RILIM 60.4 kΩ ±1% starting value; no CAN/AUX5 power; four-state/no-backfeed/handover test; TP VBUS/rails/MODE/PR1/present, no data stubs |
| Display | shared SPI/I2C, MAIN/AUX rails | DISPLAY_3V3/5V and controls | 2×TPS22919-Q1, 2N7002KQ; connector/ESD provisional | Only compatible rail enabled; cable ≤200 mm; rail/control TPs or current links |
| Shift light | AUX5, GPIO6, enable | `SHIFT5`, `SHIFT_DATA_5V`, fault | TPS1H100B-Q1, CAHCT1G126-Q1; ESD provisional | 0.50 A qualified/1 A fault, ≤0.5 m; TP power/data/fault; exact RCL/SOA gate |
| Sound/MODE/status | AUX5, GPIO17/10, I2C/P6 | BTL speaker, `MODE_N`, RGB sinks | TPA2005D1TDGNRQ1, LP5814DRLR; speaker/LED provisional | BTL outputs never grounded; MODE non-strap/internal; PowerPAD, status-enable and acoustic/optical test dependencies |
| Debug/test | Named sheet ports | Compact pads/current links | No active architecture | Test access must not add CAN/USB/RF/SPI stubs or increase board size without value |

## Top-level power architecture

The selected Task 5A.1 conditional topology is:

```text
OBD16 / VBAT_OBD_RAW
  -> fuse
  -> VBAT_FUSED_CLAMPED + anti-series SM15T47AY / SM15T33AY to POWER_GND
  -> LM74720-Q1 + back-to-back 100 V N-FETs
  -> VBAT_SWITCHED -> selected damped filter -> FILTERED_VEHICLE
                                                    |
                                                    +-> LMQ66420MC3 -> VEH_3V3 -> TPS2116 VIN1
                                                    |                       +-> TCAN3404 VCC
                                                    +-> vehicle-only AUX5 buck -> DISPLAY_5V / SHIFT5 / SOUNDER5

USB-C D+/D− -> USBLC6 D+/D− pass-through (VBUS shunt/reference)
USB-C CC1/CC2 -> independent Type-C Rd
USB-C VBUS -> TPS2553-Q1 -> TPS62162-Q1 -> USB_3V3 -> TPS2116 VIN2

TPS2116 OUT -> MAIN_3V3 -> ESP32-S3 + TCA6408A-Q1 + LP5814 + switched 3.3 V branches
```

The former Task 4.7 `SM8SF24CA-Q`, `LM74502QDDFRQ1`, 60 V `DMT6007LFGQ-7` and `PMEG6030EP-Q` common-input source-OR are **superseded historical candidates** and must not be copied into capture. Task 5A.1 moves source selection to the regulated 3.3 V rails. TPS2116 provides reverse-current blocking between them; LM74720 and its back-to-back FETs provide the vehicle-side reverse-current barrier. TCAN3404 VCC and AUX5 remain outside the mux on vehicle-only supplies.

TPS2116's conditional priority network starts with MODE tied to `VEH_3V3`, PR1 pulled up to `VEH_3V3` by 1 MΩ and down by 2 MΩ, and the vehicle MAIN-buck PGOOD open-drain output pulling PR1 low until valid. This must be tested through both supply ramp orders, chatter, brownout and repeated handover; it is not a timing guarantee.

### Ground and input-filter intent

OBD4 and OBD5 arrive on separate conductors and join once at the connector-entry region into one continuous `POWER_GND` plane. TVS/fuse/input-capacitor current returns stay in that region. CAN, USB, GNSS, SD, display, shift and audio use the same DC ground net with layout-managed return paths; no split ground island is permitted. USB shell population is a connector-local EMC option. Speaker outputs are BTL and never ground.

The selected post-disconnect starting network is `CGA2B3X7R1H104M050BB` 100 nF/50 V plus `CGA5L3X7R1H105K160AB` 1 µF/50 V shunts, `XEL4030V-222MEC` 2.2 µH ±20% series, and four direct `CGA6P3X7R1H475K250AB` 4.7 µF MLCCs plus a fifth DNP footprint. The required aggregate effective acceptance is 9.4–20.68 µF. In parallel, `WSL2512R3900FEA` 0.39 Ω ±1% is in series with `EEH-ZC1H121P` 120 µF to ground. `Cd,min = 96 µF` is 4.64× `Cf,max`. The nominal-L/maximum-ESR Q screen is 0.78–1.16; the full L/R/ESR corner screen is `f0 = 21.54–39.13 kHz` and `Q = 0.69–1.37`. These are capture screens, not stability/EMI proof. At 25.531 V, the ideal hard-step screen is about 1.67 kW initial resistor power and 46.9 mJ stored energy; the exact WSL2512 point must pass the manufacturer pulse tool/model and test. Populated impedance, effective capacitance, ESR/leakage, hot-plug, source/harness interaction, converter negative impedance and emissions remain mandatory tests. See [`task5a1-power-architecture.md`](task5a1-power-architecture.md).

## Rail contract

| Rail | Nominal | Continuous design allocation | Peak design allocation | State/control | Required schematic provisions |
|---|---:|---:|---:|---|---|
| `FILTERED_VEHICLE` | source-dependent | 0.5326 A TRACK sensitivity at 12 V | 0.7892 A managed-MAX sensitivity at 12 V; inrush TBD | Vehicle only, after LM74720/filter | Feeds both vehicle bucks; MAX ≤10 s/25% rolling duty; separate MC3 1.050 A operating-peak/thermal case; 1.313 A is sizing-only, not a load; compact TP/current link; no raw external export |
| `VEH_3V3` | 3.3 V | 0.2982 A TRACK named continuous bucket | 1.050 A separate 3.3 V-display operating-peak/thermal bucket | Vehicle MAIN LMQ output; TCAN VCC direct; TPS2116 VIN1 | The bucket includes direct vehicle-only TCAN current plus core/peripheral current that reaches `MAIN_3V3` through TPS2116; 1.313 A is a 25%-margin capacity-sizing result, not a continuous or operating-peak load. Capture the exact shared LMQ population and validate effective capacitance, loss and thermal result. |
| `USB_3V3` | 3.3 V | post-ramp `USB_ENUM` ≤100 mA `MAIN_3V3`; configured `MAIN_3V3` ≤400 mA | TPS2553 fault/current-limit band 387.2–491.3 mA; configured VBUS ≤350 mA steady; MAX prohibited | TPS2553 -> exact TPS62162 population -> TPS2116 VIN2 | Prototype startup source/cable ≥500 mA at 4.75 V; optional loads default-off; 100 mA is not an inrush ceiling or generic legacy USB 2.0 compliance claim; validate current-limited startup/handoff. |
| `MAIN_3V3` | 3.3 V | Vehicle: 0.290 A TRACK physical downstream load; USB: core-only configured budget | Vehicle: 0.995 A through the mux in the separate 1.050 A vehicle-buck operating-peak/thermal case; USB: source-policy limited | TPS2116 output; present when either selected 3.3 V source is valid | ESP32/TCA6408A/LP5814 and switched 3.3 V branches only; TCAN VCC is not on this physical rail; 1.313 A remains sizing-only |
| `GNSS_3V3` | 3.3 V | 70 mA design continuous | 200 mA branch requirement | TPS22919, `EXP_P0/GNSS_EN`; off parked | 100 nF at every module VCC pin group plus local bulk per u-blox reference; measurement link |
| `SD_3V3` | 3.3 V | card-dependent | 250 mA requirement | TPS22919, `EXP_P1/SD_EN`; off parked | local 100 nF plus ≥10 µF starting bulk; final value from card inrush measurement |
| `DISPLAY_3V3` | 3.3 V | 300 mA initial module contract | 400 mA maximum | TPS22919, `EXP_P2/DISP3_EN`; off parked | connector-side 100 nF + ≥22 µF starting bulk; display requires local decoupling |
| `AUX5` | 5.0 V | STREET 0.56001 A; TRACK 0.92001 A continuous | MAX 1.16001 A for ≤10 s and ≤25% of any rolling 60 s; 1.450 A sizing margin is not a load | Vehicle-only LMQ66420MC5 from `FILTERED_VEHICLE`; EN from GPIO21; off parked/USB-only/low-voltage shed | MAX/full-white prohibited below 10.0 V; AUX5 off below the qualified shed threshold; efficiency, dropout and 85 °C PCB thermal results remain prototype gates |
| `DISPLAY_5V` | 5.0 V | 500 mA | 600 mA maximum | TPS22919, `EXP_P3/DISP5_EN`; only after AUX5 PGOOD | connector-side bulk; never enable with DISPLAY_3V3 for an incompatible module |
| `SHIFT5` | 5.0 V | ≤500 mA qualified | 1 A protected fault envelope | TPS1H100-Q1, `EXP_P4/SHIFT5_EN`; only after AUX5 PGOOD | current-limit resistor calculation, fault sense, connector ESD and bulk |
| `SOUNDER5` | 5.0 V | ≤300 mA envelope | 8 Ω, ≥1 W speaker provisional | AUX5 plus approved-direction TPA2005D1TDGNRQ1, GPIO17 waveform/control; off parked | BTL output; input reconstruction/coupling, PowerPAD copper and local decoupling |

Loads shall not rely on an ESP32 GPIO for power. All switch-enable nets have hardware pull-downs so reset, an unconfigured TCA6408A or a broken I²C bus leaves external rails off.

## Hardware power states and sequencing

| State | Always powered / enabled | Disabled | Wake/exit path |
|---|---|---|---|
| `UNPOWERED` | ESD/CC/passive source components only | All rails | OBD or USB insertion |
| `PARKED` | vehicle protection/filter, vehicle MAIN buck, `VEH_3V3`, TPS2116/`MAIN_3V3`, ESP32 deep-sleep domain, vehicle-only TCAN3404 standby, TCA6408A and required sense | GNSS_3V3, SD_3V3, DISPLAY_3V3, AUX5, DISPLAY_5V, SHIFT5, buzzer, status LED | CAN WUP via RXD/GPIO13, MODE/GPIO10, RTC timer, qualified vehicle activity, USB presence/GPIO4 |
| `WAKE` | `MAIN_3V3`; vehicle-only CAN receiver when `VEH_3V3` is valid; MCU validates cause | All optional rails remain off initially | Qualify activity then ACTIVE, otherwise return PARKED |
| `ACTIVE` | `MAIN_3V3`; selected GNSS/SD/display rails; vehicle-only CAN; AUX5 only when required and valid | Unused outputs | inactivity/policy -> SHUTDOWN_PENDING |
| `SHUTDOWN_PENDING` | only rails needed to finish bounded storage/protocol shutdown | new logging/output activity prohibited | complete within measured time -> PARKED; new qualified event -> ACTIVE |
| `USB_DEBUG` | `USB_3V3` -> TPS2116 -> `MAIN_3V3`, ESP32/native USB; optional switched 3.3 V loads only after a valid USB budget | TCAN3404, AUX5, DISPLAY_5V, SHIFT5 and all vehicle-only outputs | USB removal -> UNPOWERED/PARKED according to OBD; valid vehicle PGOOD -> vehicle-priority policy |

Vehicle startup is `FILTERED_VEHICLE -> vehicle MAIN buck -> VEH_3V3/PGOOD -> TPS2116 VIN1 -> MAIN_3V3 -> ESP reset release -> expander initialization -> optional 3.3 V branches`. USB startup is `VBUS -> TPS2553 -> TPS62162 -> USB_3V3 -> TPS2116 VIN2 -> MAIN_3V3`; its optional loads remain default-off. For 5 V loads, firmware first proves a valid vehicle input, asserts GPIO21, waits for AUX5 PGOOD, and then enables only the required branch. Loss of vehicle MAIN PGOOD selects USB for the core when available, but powers CAN and AUX5 down; without USB the core resets/off cleanly. Shutdown reverses the branch sequence, with a bounded SD flush. Hardware defaults alone must never cause CAN transmission.

The capture contract reserves the Task 5A.1 low-voltage policy: prohibit MAX/full-white diagnostics and Wi-Fi below 10.0 V; below 9.5 V start a bounded SD flush and no new optional work; below 9.0 V for 100 ms enter `LOW-VOLTAGE SHED` with AUX5/display/shift/sound/Wi-Fi off and no new SD writes; exit only above 10.0 V stable for 2 s. These thresholds/timers remain firmware design requirements pending ADC/front-end tolerance, crank data and measured flush time. A controlled reset with passive CAN defaults is required when the physical rails can no longer regulate.

### Parked-current consequence

The corrected Task 5A.1 tree gives a 204.013 µA conservative subtotal and a doubled 408.026 µA (0.408 mA) parked-current envelope at 12 V (`CALCULATED`). This paper result passes <1.0 mA by 0.592 mA and the <0.50 mA room-temperature stretch target by 0.092 mA; the doubled 6/8/10/12/14.4/18 V model is 0.476/0.438/0.419/0.408/0.402/0.400 mA. For continuity with the prior budget, the historical “MAIN” bucket still includes TCAN3404 current even though TCAN VCC is physically on direct vehicle-only `VEH_3V3`, while the ESP32 core is on muxed `MAIN_3V3`. Neither the bucket nor the doubled envelope is accepted until complete-board voltage/temperature/production-spread and wake-duty measurements are recorded. See [`task5a1-power-architecture.md`](task5a1-power-architecture.md) and [power-budget.md](power-budget.md).

## CAN sheet

```text
OBD6 CANH_OBD ----+---- ESDCAN04 ----+---- CMC footprint ---- CANH_PHY ---- TCAN3404 pin CANH
                  |                  |      (0R bypass default)
OBD14 CANL_OBD ---+---- ESDCAN04 ----+---- CMC footprint ---- CANL_PHY ---- TCAN3404 pin CANL
                                                        |
                                                        +-- 60.4R 1% -- TERM_MID -- 60.4R 1% --+
                                                                            |                    |
                                                                          4.7nF                 (across pair)
                                                                            |
                                                                      POWER_GND
```

The split-termination branch is isolated by two normally-open solder bridges or zero-ohm DNP positions. Both 60.4 Ω resistors and the 4.7 nF capacitor are DNP in the product BOM. Default measured DC resistance added by the node is therefore open/high impedance, not 120 Ω. Bench termination is enabled only by populating the split network and both documented service bridges. An added 120 Ω across a normally terminated 60 Ω vehicle bus would produce `60 || 120 = 40 Ω` (`CALCULATED`), so product termination stays OFF.

The ESDCAN04 sits closest to OBD entry with a short, dedicated ground return. The CMC footprint follows it; two 0 Ω bypasses are populated by default and ACT45B-510 is DNP until EMC tests justify population. CANH/L remain a paired short stub. No large test-pad stubs are allowed.

TCAN3404 VCC is connected directly to vehicle-only `VEH_3V3`, not muxed `MAIN_3V3`, with at least 100 nF at the pin and a 4.7–10 µF local bulk footprint. USB-only operation therefore cannot power the CAN transceiver. TXD is GPIO14, RXD/WUP is RTC-capable GPIO13 and STB is GPIO38; their unpowered leakage/phantom-power behavior must be verified. A hardware pull-up makes standby the reset default when VCC exists. LISTEN_ONLY additionally configures TWAI to non-transmitting listen-only; DIAGNOSTIC_POLLING is the only production state allowed to request bounded transmission.

## GNSS digital and RF sheets

NEO-M9N-00B VCC uses switched GNSS_3V3. V_BCKP follows GNSS_3V3 through a 0 Ω link; no parked backup source is populated. This intentionally trades warm/hot retention for zero GNSS parked current. ESP GPIO15 receives M9N TX and GPIO16 drives M9N RX. The operational UART target is 230,400 bit/s; the module default is 38,400 and must be changed during initialization. TIMEPULSE and RESET_N receive labelled test pads; they are not assigned scarce direct GPIO in v1. Reset follows the official pull/control reference.

RF implementation follows u-blox Integration Manual R10 Figure 33, not an invented matching network:

```text
NEO-M9N RF_IN (internally DC blocked / 50 ohm)
       |
       +---------------- controlled 50 ohm GNSS_RF ---------------- U.FL center
                                                                     |
GNSS_3V3/VCC_RF -- 100 nF supply filter -- [DNP LIMITER_SELECT] -- 27 nH --+
                                              passive/active TBD     bias-T

U.FL center -> AQ3118E-01ETG -> shortest RF-ground return
```

The exact drawing shall preserve the manual’s component orientation/topology. u-blox recommends C1 100 nF X7R 16 V, L1 27 nH with >500 Ω impedance at GNSS frequency and >300 mA current rating, and requires current limiting. `PROPOSED CHANGE`: the earlier 22 Ω/136 mA passive calculation is reopened because it did not by itself prove VCC_RF, module-supply impedance, antenna voltage or hot-short safety. Task 5 must choose a switched-GNSS_3V3-fed passive or active limiter against the exact antenna.

The supported design target remains 50 Ω at GNSS L-band, 2.7–3.3 V at the selected active antenna after cable/limiter drop, 5–20 mA normal antenna current, and coverage of 1559–1606 MHz within the NEO-M9N gain/noise limits. `AQ3118E-01ETG` is frozen at U.FL: active product, AEC-Q101/PPAP, bidirectional 18 V, 0.3 pF typical. The exact antenna and short limiter remain provisional.

Do not route RF in this task. Future placement keeps the U.FL-to-module path short, controlled at 50 Ω, over continuous RF ground and away from ESP antenna, buck switch nodes, CAN edges, SD/SPI and display cables. The RF ESD return must not share a long digital path.

## microSD sheet and GPIO45 resolution

microSD remains 4-wire SPI on shared `SPI_SCLK/GPIO39`, `SPI_MOSI/GPIO40`, and `SPI_MISO/GPIO41`. `SD_CS` moves from GPIO45 to GPIO11. GPIO45 is reserved with no removable-card loading. This removes the known boot/reset dependency even though Espressif documents that PSRAM-module eFuses fix VDD_SPI.

SD_3V3 is independently switched. SD_CS has a pull-up to SD_3V3 and a MAIN_3V3-side default that does not phantom-power the card; series isolation/firmware high-Z must be checked when SD power is off. Card detect is `EXP_P5/SD_CD_N`, referenced to always-on MAIN_3V3 through a benign pull-up. Provide connector-local ESD only if the card is user-accessible through the enclosure; exact ESD/socket wait for the mechanical freeze. The card, display and MCU share SPI only through independent CS lines, a bus arbiter and verified MISO high-Z behavior.

## Display electrical contract

The contract is a 14-position keyed/locking candidate interface; mechanical family is provisional. Candidate families for mechanical review are Molex Micro-Lock Plus 2.0 mm, JST GH 1.25 mm and Hirose DF13 1.25 mm. The chosen connector must carry the declared rail current with contact derating, provide positive retention, tolerate the enclosure environment and avoid ambiguous 3.3/5 V mating.

| Pin contract | Direction at main unit | Electrical rule |
|---|---|---|
| `GND_PWR`, `GND_SIG` | — | two returns; do not depend on shield for DC current |
| `DISPLAY_3V3` | Out | switched, protected, 400 mA maximum |
| `DISPLAY_5V` | Out | separately switched, protected, 600 mA maximum; incompatible modules must be keyed/configured |
| `SPI_SCLK` | Out | 3.3 V; ≤20 MHz initial cable contract |
| `SPI_MOSI` | Out | 3.3 V |
| `SPI_MISO` | In | optional; module must tri-state when CS inactive/off |
| `DISP_CS` | Out | GPIO47; inactive through reset |
| `DISP_DC` | Out | GPIO48 |
| `DISP_RST_N` | Out | GPIO12 |
| `DISP_BL` | Open-drain/PWM | GPIO7 through 2N7002KQ; module owns raw-LED current regulation |
| `I2C_SDA`, `I2C_SCL` | I/O | optional 3.3 V touch/ID; one controlled pull-up set |
| `DISP_INT_TE` | In | optional; routed to expansion/test option, not a frozen direct MCU GPIO |

Maximum recommended cable length is 200 mm (`DESIGN_REQUIREMENT`) pending signal-integrity and EMC testing. Series-source resistor footprints are required on SCLK/MOSI/CS/DC and populated values are selected by measurement. Connector-side ESD is required for externally accessible signals; exact array waits for connector/ground geometry. The initial GC9A01 module is a software fixture, not a permanent electrical assumption.

## Shift-light sheet

The external contract is `SHIFT5`, `GND`, and `SHIFT_DATA_5V`, with connector current rating ≥1.5 A. TPS1H100-Q1 supplies a protected/current-limited 5 V branch up to 1 A; it is off in PARKED, reset and USB-only. The current-limit resistor and thermal SOA are calculated in the derivative schematic. CAHCT1G126-Q1 translates GPIO6 to 5 V and its active-high OE is the same safe enable policy as SHIFT5. Add a 33 Ω starting series-damping footprint at the buffer and connector-side two-line 5 V ESD; `PESD2USB5UVT-Q` is provisional pending the connector/harness clamp review.

Exactly ten V6-class WS2812-compatible pixels are the frozen functional load. `CALCULATED` all-white load is 360.010 mA; with 25% margin it is 450 mA, rounded to a 500 mA qualified load. The separate TPS1H100-Q1 fault/protection envelope remains 1 A. Start with 33 Ω data damping (22–47 Ω measurement range), local 100 nF per pixel, ≥26 AWG power/ground (24 AWG preferred), ≥28 AWG data, and ≤0.5 m cable with ground adjacent/twisted to data. Exact pixel, ESD array and connector remain provisional.

## Sounder, MODE and indicators

`APPROVED DIRECTION`: use TPA2005D1TDGNRQ1 on `SOUNDER5`, driving one onboard differential 8 Ω, ≥1 W speaker. GPIO17 remains the waveform source through an input reconstruction/coupling network. Neither BTL speaker terminal is ground. The 300 mA branch derives from 1 W/(5 V×0.75 assumed efficiency)+2.8 mA = 269.5 mA rounded up. Exact speaker, acoustic port/back volume and in-cabin acceptance require measurement; no maximum-SPL claim is frozen.

MODE is a normally-open switch from GPIO10 to ground with a 47 kΩ starting pull-up to MAIN_3V3 and 1 nF DNP debounce capacitor. `CALCULATED` pressed current is `3.3 V / 47 kΩ = 70.2 µA`; this is momentary and not parked average. Debounce is primarily firmware; capacitor population must not violate wake timing. GPIO10 is not a strap pin.

`APPROVED`: LP5814DRLR on MAIN_3V3 drives one common-anode RGB status LED over shared I²C, with `EXP_P6/STATUS_DRV_EN` keeping it off by default and in PARKED. Exact LED/resistors/optics remain provisional. No always-on power LED is permitted. PROG/GPIO0 and RESET/EN buttons remain available for recovery and are distinct from MODE.

## USB-C sheet

USB-C is a USB 2.0 sink/device only. CC1 and CC2 each use an independent 5.1 kΩ 1% Rd to ground. D− is GPIO19 and D+ GPIO20. USBLC6-2SC6Y sits at the connector. Provide matched 22 Ω series-resistor starting footprints close to the ESP32 and DNP shunt-cap footprints; final values follow Espressif guidance and signal-integrity validation. Shield-to-chassis/ground strategy remains an EMC schematic-review item, not an arbitrary direct connection.

Power cases:

| Vehicle | USB | Hardware result |
|---|---|---|
| absent | absent | All rails off/passive |
| present | absent | Vehicle LMQ produces `VEH_3V3`; PGOOD releases PR1 and TPS2116 VIN1 powers `MAIN_3V3`. CAN and qualified vehicle-only loads are available; VIN2/VBUS is not driven |
| absent | present | TPS2553/TPS62162 produce `USB_3V3`; TPS2116 VIN2 powers only the development/core `MAIN_3V3` domain. TCAN VCC and AUX5 remain physically off; no intended OBD backfeed path exists |
| present | present | Valid vehicle rail has priority at VIN1; VIN2 is reverse-current-blocked and remains available for data/handoff. Loss of vehicle PGOOD selects VIN2; CAN/AUX5 power down with the vehicle rail |

GPIO4 senses `USB_PRESENT` through a high-value divider/clamp that meets USB VBUS leakage and ESP input limits. USB source presence never authorizes CAN transmission. TCAN3404 VCC is `VEH_3V3`, not `MAIN_3V3`; TXD/STB defaults and unpowered injection must prevent phantom power and be measured.

`TPS2553QDBVRQ1` uses `CRCW060360K4FKEA`, 60.4 kΩ ±1%, for RILIM; 387.2–491.3 mA is its calculated fault/current-limit population band, not a load contract. The prototype startup source/cable must advertise and sustain at least 500 mA at 4.75 V. After ramp, `USB_ENUM` is at most 100 mA total on `MAIN_3V3` (about 82 mA steady at VBUS); it is not an inrush ceiling. Configured operation is provisionally capped at 350 mA VBUS steady and 400 mA `MAIN_3V3`, with MAX prohibited. This does not establish generic legacy USB 2.0 pre-enumeration compliance. The exact conditional TPS62162 population is `CGA2B3X7R1H104M050BB` 100 nF, `CGA6P1X7R1E106M250AC` 10 µF/25 V CIN, `XFL3012-222MEC` 2.2 µH, and `CGA6M3X7R1C106K200AB` 10 µF/16 V COUT. Populate `EEEFK0J101AV` 100 µF/6.3 V at TPS2116 VOUT. Effective capacitance, current-limited startup, MODE/PR1/PGOOD, 105 °C environment, ramp, brownout, handover and reverse-leakage validation remain open gates.

## Final GPIO allocation

| GPIO / resource | Function | Classification | Reset/strap rule |
|---:|---|---|---|
| 0 | PROG/BOOT button | `FROZEN` | strap; no other load |
| 1 / 2 | I2C SDA / SCL | `FROZEN` | MAIN_3V3 pull-ups; shared expander/external contract |
| 3 | reserved | `RESERVED` | strap/JTAG-source selection; no external load |
| 4 | USB_PRESENT | `FROZEN` | high-impedance divider, RTC-capable input |
| 5 | PWR_FAULT_N / protected-source status | `PROVISIONAL` | exact supervisor output depends on input schematic |
| 6 | SHIFT_DATA | `FROZEN` | buffer disabled until SHIFT5 enable |
| 7 | DISPLAY_BL_PWM | `FROZEN` | open-drain driver default off |
| 8 | VEHICLE_ACTIVITY | `FROZEN` | qualified threshold/wake input; exact detector provisional |
| 9 | PROTECTED_VBAT_ADC | `FROZEN` | gated/high-value divider; recalibrate |
| 10 | MODE_N | `FROZEN` | non-strap RTC input, pull-up/button-to-ground |
| 11 | SD_CS | `FROZEN` | moved from GPIO45; inactive through reset |
| 12 | DISP_RST_N | `FROZEN` | display off/reset parked |
| 13 / 14 | CAN_RX/WUP / CAN_TX | `FROZEN` | RX RTC-wake capable; TX safe/recessive policy |
| 15 / 16 | GNSS_RX / GNSS_TX | `FROZEN` | endpoint naming: ESP RX/M9N TX and reverse |
| 17 | SOUND_PWM | `FROZEN GPIO`, approved amplifier function | safe low/default; waveform into amplifier input network |
| 18 | EXP_INT_N | `FROZEN` | TCA6408A interrupt/wake/status input |
| 19 / 20 | USB D− / D+ | `FROZEN` | native USB only |
| 21 | AUX5_EN | `FROZEN` | direct safety-critical domain enable, pull-down |
| 38 | CAN_STB | `FROZEN` | pull-up gives standby at reset |
| 39 / 40 / 41 | SPI SCLK / MOSI / MISO | `FROZEN` | shared SD/display; overlaps JTAG TCK/TDO/TDI |
| 42 | JTAG MTMS test pad | `RESERVED` | no buzzer allocation; optional external JTAG retained |
| 43 / 44 | UART0 TX / RX debug pads | `FROZEN` | test/debug only |
| 45 | reserved strap/test pad | `RESERVED` | no SD or removable-card loading |
| 46 | reserved strap | `RESERVED` | input-only/boot strap; no external load |
| 47 / 48 | DISP_CS / DISP_DC | `FROZEN` | 3.3 V on N16R8; inactive/default states defined |

TCA6408A allocation: P0 GNSS_EN, P1 SD_EN, P2 DISP3_EN, P3 DISP5_EN, P4 SHIFT5_EN, P5 SD_CD_N, P6 STATUS_DRV_EN, P7 reserved. Every output-enable target has an independent pull-down; because the expander powers up with ports as inputs, rails remain disabled until software deliberately configures them.

GPIO39–42 external JTAG conflicts are explicit: shared SPI consumes 39–41 during normal operation, while GPIO42 is preserved as a test pad. Native USB Serial/JTAG is primary debug. This is an accepted multiplexing limitation, not an unresolved collision.

## Test/debug contract

Freeze compact points for `VBAT_OBD_RAW`, `VBAT_FUSED_CLAMPED`, `VBAT_SWITCHED`, `FILTERED_VEHICLE`, entry/instrument `POWER_GND`, `VEH_3V3`, `USB5_PROTECTED`, `USB_3V3`, `MAIN_3V3`, `AUX5`, vehicle MAIN PGOOD, TPS2116 MODE/PR1, `AUX5_EN/PGOOD`, CANH/L and CAN RX/TX/STB, GNSS_3V3/RX/TX/TIMEPULSE/RESET, SHIFT5/data/fault, USB VBUS/present, UART0, and expander reset/interrupt. Use measurement links instead of large pads for individual switched branches where practical.

Do not add USB D+/D−, SPI-clock or GNSS-RF stubs. CAN pads remain compact and paired. No RF test pad is allowed; use U.FL and an approved RF fixture. Full constraints are in [`input-protection-architecture.md`](input-protection-architecture.md).

## Firmware-architecture compatibility check

The one high-impedance Classical-CAN PHY supports raw CAN, Generic OBD-II, future ISO-TP/UDS, LISTEN_ONLY and bounded DIAGNOSTIC_POLLING. Independent GNSS UART/power supports streaming. ESP32 BLE supports RaceChrono and a future first-party protocol without changing the internal data model. Shared SPI with independent selection/power supports interchangeable displays and logging; display, SD and GNSS faults can be power-isolated. Direct shift-light data and approved autonomous sound/status drivers support headless operation. Native USB and UART0 preserve update/debug. No external display is required for headless operation.

## Conditional capture and release gates

- Review and then bench the exact `0437002.WRA`, common-anode `SM15T47AY`/`SM15T33AY`, LM74720 and two `STL125N10F8AG` implementation for fuse clearing/inrush, TVS pulse sharing/orientation, VDS/VGS/SOA, reverse current, hot/repetitive transients and every purchased-standard/OEM pulse. The 47 V-leg cathode goes to fused VBAT and the 33 V-leg cathode to `POWER_GND`.
- Verify the LM74720 native-OV network and downstream survival against the Task 5A.1 modeled 25.531 V endpoint plus measured overshoot; the former cutoff-by-20 V and protected-node-≤24 V limits are superseded.
- Measure the complete `FILTERED_VEHICLE` impedance/damping network over voltage, temperature, ageing, harness/source impedance, converter modes and loads; verify effective capacitance, hybrid leakage/ESR, resistor pulse heating, inrush/ringing and conducted/radiated/GNSS coexistence before tuning.
- Bench TPS2553/TPS62162/TPS2116 in all four source states, both ramp orders, brownout and abnormal connections. Prove current-limit tolerance, MODE/PR1/PGOOD thresholds, acceptable `MAIN_3V3` droop, no sustained or unsafe reverse current, no CAN/AUX phantom power and the 105 °C catalog-part limitation.
- Qualify the exact shared vehicle MAIN/AUX LMQ population (`XGL5030-222MEC`; 2 × `CGA6P1X7R1N106K250AC` CIN; 4 × `CGA6P3X7R1E226M250AB` COUT plus one DNP; 2 × `CGA3E1X7R1C105K080AC` CVCC; CBOOT DNP), load-step/dropout, PCB junction temperature and the STREET/TRACK/MAX duty contracts at the specified voltage/ambient points. Validate the 10.0/9.5/9.0 V load-shed policy and controlled reset with real crank and SD-flush measurements.
- Final OBD, display and shift-light connector mechanical/current/environment selection.
- Final microSD socket temperature/access choice; exact speaker, RGB LED and acoustic/optical geometry.
- Exact vehicle-activity detector, protected ADC divider and PWR_FAULT_N circuit.
- ESD arrays for display/shift connector selected after connector pinout and return geometry.

See [`task5a1-power-architecture.md`](task5a1-power-architecture.md) for the selected calculations and [`task5a-power-calculations.md`](task5a-power-calculations.md) for the superseded conflict record. These conditional gates do not reopen the unrelated system partition, CAN termination default, GPIO45 decision or downstream functional-sheet allocations unless recorded evidence forces an approved change.
