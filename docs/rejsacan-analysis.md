# RejsaCAN v3.x analysis for telemetry-v1

## Scope, revision, and evidence policy

This analysis uses **RejsaCAN v3.4**, the newest v3.x design present in this checkout. It is a one-sheet EasyEDA design dated 2023-08-25. No upstream hardware or manufacturing file has been edited. Audit note (2026-08-14): reference design values and GPIO/component mappings below are direct repository evidence, not derivative numerical decisions. External electrical figures are `VERIFIED_DATASHEET`; arithmetic is `CALCULATED`; preliminary choices are superseded where `power-budget.md` or `power-wake-review.md` gives a classified decision.

Local evidence:

- **Schematic:** [`RejsaCAN v3.4 - Schematic.png`](../Schematics/RejsaCAN%20v3.x%20%28ESP32-S3%20based%20board%29/RejsaCAN%20v3.4%20-%20Schematic.png), sheet 1/1. The JSON source carries the same title/revision/connectivity in [`RejsaCAN v3.4 - Schematic.json`](../Schematics/RejsaCAN%20v3.x%20%28ESP32-S3%20based%20board%29/RejsaCAN%20v3.4%20-%20Schematic.json).
- **BOM:** [`RejsaCAN v3.4 - BOM.csv`](../Schematics/RejsaCAN%20v3.x%20%28ESP32-S3%20based%20board%29/RejsaCAN%20v3.4%20-%20BOM.csv), entries U1–U5 and associated passives/protection.
- **Pinout:** [`RejsaCAN v3.4 - Pinout.h`](../Schematics/RejsaCAN%20v3.x%20%28ESP32-S3%20based%20board%29/RejsaCAN%20v3.4%20-%20Pinout.h).
- **Upstream description:** [repository `README.md`](../README.md), especially CAN interface, Auto shutdown, Good to haves, and v3.1→v3.2 Changes.
- **Examples:** [`getall_s3.ino`](../Code%20Examples/RejsaCAN%20v3.x%20-%20ESP32-S3%20-%20listen%20to%20all%20CAN%20broadcasts%20printed%20over%20USB/getall_s3.ino), [`s3-sdcard.ino`](../Code%20Examples/RejsaCAN%20v3.x%20-%20ESP32-S3%20-%20SPI%20settings%20for%20SD%20card%20reader/s3-sdcard.ino), [`board v3.x.ino`](../Code%20Examples/Test%20AUTO-OFF%20keeping%20board%20on%20after%20engine%20stops/board%20v3.x.ino), and [`simpleshiftlight_s3_collin.ino`](../Code%20Examples/Simple%20shift%20light/simpleshiftlight_s3_collin.ino).
- **Fabrication geometry only:** [`RejsaCAN v3.4 - Gerber.zip`](../Schematics/RejsaCAN%20v3.x%20%28ESP32-S3%20based%20board%29/RejsaCAN%20v3.4%20-%20Gerber.zip), `Gerber_BoardOutlineLayer.GKO` header and outline coordinates.

External authoritative cross-checks are linked where they prevent misinterpretation, but local files remain the evidence for what was actually designed/populated.

## Architecture summary

RejsaCAN v3.4 is a compact ESP32-S3 module board with native TWAI connected to a single 3.3 V CAN transceiver, selectable 120 Ω termination, direct USB-C, SPI microSD, voltage-derived auto-shutdown, vehicle-voltage sensing, and a GPIO-controlled 3.3 V load switch. A 5–24 V claim appears in the repository README. The board can either remove its main 3.3 V rail in hardware or remain powered while the ESP32 sleeps and watches voltage/CAN activity.

## Required findings

### 1–2. ESP32-S3 variant and memory

| Item | Finding | Evidence / confidence |
|---|---|---|
| Module | ESP32-S3-WROOM-1-N16R8, PCB-antenna WROOM-1 variant | Schematic sheet 1, U1 value; BOM entry 29/U1. **High confidence.** |
| Flash | 16 MB Quad-SPI flash | `N16` in populated U1 part number, cross-checked with the [Espressif module data sheet](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf). **High confidence.** |
| PSRAM | 8 MB Octal-SPI PSRAM | `R8` in populated U1 part number and Espressif variant table. **High confidence.** |
| Temperature caveat | Current Espressif table lists N16R8 ambient range −40 to +65 °C unless PSRAM ECC provisions improve it as documented | `VERIFIED_DATASHEET`, Espressif module data sheet v1.8, Table 1-1. This remains a derivative part-selection review item. |
| Relevant resources | Dual-core ESP32-S3, BLE/Wi-Fi, native USB, TWAI, UART/SPI/I2C through GPIO matrix, RTC GPIO wake features | U1 identity plus [ESP32-S3 data sheet](https://documentation.espressif.com/esp32_s3_datasheet_en.pdf). Peripheral count/limits must be checked against the exact silicon/SDK revision during firmware design. |

### 3–5. CAN controller, transceiver, and termination

| Item | Finding | Evidence / confidence |
|---|---|---|
| Controller architecture | ESP32-S3 internal TWAI controller; no external CAN controller | Schematic: U1 GPIO14 `CAN_TX` to U2 D, GPIO13 `CAN_RX` from U2 R. `getall_s3.ino` includes `driver/twai.h` and instantiates TWAI on pins 14/13. **High confidence.** |
| Transceiver | TI SN65HVD230DR, 3.3 V Classical CAN transceiver | Schematic U2; BOM entry 30/U2. **High confidence.** It is not CAN FD and the BOM does not specify the automotive-qualified SN65HVD230Q. |
| Mode control | U2 RS connects to GPIO38 `CAN_RS` through R3 = 10 kΩ; low = high speed, high = low-current standby, high impedance uses resistor slope control | Schematic U2/R3 and pinout; examples drive GPIO38. TI's [SN65HVD230 data sheet](https://www.ti.com/lit/ds/symlink/sn65hvd230.pdf) confirms the receiver remains active in standby. **High confidence.** |
| Termination | R2 = 120 Ω and a two-pad `TERMINATION JUMPER` form the termination network between CAN-H and CAN-L | Schematic U2/R2/jumper; BOM entry 19/R2; repository README says termination is present by default, can be disabled by cutting a PCB trace, and can then be made selectable with a header/jumper. **High confidence on value/intent; the exact copper cut and post-cut jumper path should be confirmed from PCB source or an actual board.** |
| CAN-line protection | No dedicated CAN-line TVS or common-mode choke is visible in the BOM/schematic | Schematic CAN block and complete BOM. **High confidence for v3.4 source.** Suitability is not established. |

For an OBD node on a vehicle bus, an enabled additional 120 Ω terminator is usually undesirable; the derivative must define default-open behavior after CAN-network review rather than silently copying the default.

### 6–10. Automotive input power, range, protection, shutdown, and wake

| Item | Finding | Evidence / confidence |
|---|---|---|
| Input architecture | POWER pin 1 passes through series D6 DSS34 and F1 MF-MSMF110/16-2 to net VCC; D4 SMF30A shunts VCC to ground; VCC feeds U4 buck and sensing | Schematic sheet 1 power area; BOM entries 9, 11, 13, 32. **High confidence topology; diode polarity should be rechecked in editable CAD during derivative capture.** |
| Stated input range | 5–24 V | Repository README opening paragraph. **This is an upstream operating claim, not a demonstrated transient rating.** |
| Regulator | U4 LMR14006XDDCR asynchronous buck, nominal 3.3 V via R9 = 33 kΩ and R10 = 10 kΩ; L1 = 10 µH, D5 SS34 | Repository values: schematic U4/L1/D5/R9/R10 and BOM entries 15, 18, 21, 32. `VERIFIED_DATASHEET`: TI lists U4 as 4–40 V, up to 600 mA. This is insufficient for the derivative's calculated 1.050 A peak. |
| Input protection present | Series Schottky D6, resettable PTC F1, 30 V TVS D4, input/output capacitors; USB data ESD diodes D8/D10/D11 | Schematic and BOM entries 2–12/13. **High confidence that parts exist; automotive adequacy unproven.** |
| Not evidenced | No reverse-battery test, load-dump calculation, ISO pulse results, jump-start rating, input filter/EMC evidence, or automotive qualification claim for U4 | Repository search and source set. **Unknown/unsupported.** |
| Threshold shutdown | U3 ME2807A33M3G detector, Q1 SS8050, R4 = 91 kΩ, R5 = 33 kΩ and D1/D3-associated hysteresis/control drive U4 SHDN | Repository schematic/BOM source values. README values of about 13.7 V on/13.0 V off are an upstream claim, not a verified derivative threshold. U3 primary data and calculation remain missing. |
| MCU hold | GPIO17 `FORCE_ON` feeds U4 shutdown network through D2; pulling high keeps board powered | Schematic D2/FORCE_ON; pinout; `board v3.x.ino`; README Auto shutdown. **High confidence.** |
| Digital/analog sense | GPIO8 `SENSE_V_DIG` observes threshold state; GPIO9 `SENSE_V_ANA` receives VCC divider R18 120 kΩ/R6 33 kΩ | Schematic U1/lower-left, pinout, BOM R6/R18, shutdown example. **High confidence.** |
| CAN wake while rail on | CAN_RX GPIO13 is RTC-capable and can observe U2 RXD; U2 receiver remains active in standby | Schematic, pinout/README, TI transceiver data sheet. **Supported in principle; exact ESP sleep configuration/current must be tested.** |
| CAN wake from hardware-off | Not present: U2 and U1 share 3V3, which disappears when U4 shuts down | Schematic power connectivity. **High confidence.** Cold CAN wake would require an always-powered wake circuit or different power partition. |

The power section requires specialist automotive review. Existing parts/topology must not be described as automotive-safe based solely on the repository.

### 11–13. Switched power, microSD, and USB

| Item | Finding | Evidence / confidence |
|---|---|---|
| Switched power | U5 MT9700 is powered from 3V3, enabled by GPIO21 `HI_DRIVER`, and outputs `3V3_SWITCHED`; R16 = 12 kΩ is attached to U5 SET | Repository schematic/BOM source values. README “few hundred mA/max 500 mA” wording is unsupported for derivative use and is superseded by named, calculated domains in `power-budget.md`. |
| microSD | Socket MR01A-01211 in SPI mode: GPIO39 SCLK→CLK, GPIO40 MOSI→CMD, GPIO41 MISO←DAT0, GPIO45 `SD_CARD`→DAT3/CS; R17 = 10 kΩ pulls DAT2; DAT1 and card-detect are grounded in the drawn circuit | Schematic SD-CARD block; BOM entries 18/28; pinout; `s3-sdcard.ino`. **High confidence.** GPIO45 is a strap pin and needs reset-loading validation. |
| USB data | U1 GPIO19 = USB D− and GPIO20 = USB D+ connect to USB-C mirrored D−/D+ contacts; D8/D10/D11 provide 5 V ESD devices to ground | Schematic U1/USBC; BOM D8/D10/D11 and entry 34. **High confidence.** |
| USB-C role/power | USB-C receptacle uses R11/R12 = 5.1 kΩ CC pull-downs (device/sink). VBUS is named USB5V and feeds the power/enable network through D7 | Schematic USB block and lower power block; BOM entries 11/22/34. **High confidence connectivity.** Dual-supply/back-feed and ESD/layout compliance are not established. |
| Development/recovery | PROG button pulls GPIO0 low and RESET button acts on EN/reset network; USB is the native ESP32-S3 interface | Schematic U1/buttons and repository README Extra UART/JTAG sections. **High confidence.** |

### 14–18. GPIO and serial-bus resources

#### Fixed/occupied in v3.4

| GPIO | Function | Evidence / constraint |
|---:|---|---|
| 0 | PROG button / boot strap | Schematic, pinout; do not load casually |
| 1/2 | I2C SDA/SCL breakout | Schematic I2C connector, pinout; available if bus-shared |
| 4/5 | board-version straps/identification | Schematic U1 labels and README Board version; exact v3.4 encoding is not documented |
| 8 | SENSE_V_DIG | Schematic/pinout |
| 9 | SENSE_V_ANA | Schematic/pinout |
| 10/11 | blue/yellow LEDs | Schematic LEDs; pinout says no breakout pad |
| 13/14 | CAN RX/TX | Schematic/pinout |
| 17 | FORCE_ON | Schematic/pinout |
| 19/20 | native USB D−/D+ | Schematic; Espressif data sheet |
| 21 | high-driver/load-switch enable | Schematic/pinout |
| 38 | CAN RS | Schematic/pinout |
| 39/40/41 | SD SPI SCLK/MOSI/MISO and SPI breakout; JTAG MTCK/MTDO/MTDI | Schematic/pinout |
| 42 | rear JTAG MTMS pad | Pinout |
| 43/44 | rear TXD0/RXD0 pads | Schematic labels and pinout; exact GPIO numbers follow ESP32-S3 module pin mapping |
| 45 | microSD CS and strap | Schematic/pinout; Espressif data sheet |

#### Available or shareable

| GPIO | Access/status | Caveat |
|---:|---|---|
| 6, 7, 15, 16 | Rear solder pads, no default function | No header clearance because module is opposite; pinout |
| 12, 47, 48 | Main breakout connector, no default function | GPIO47/48 are 3.3 V on this N16R8 (not R8V) module; verify exact module population |
| 1, 2 | Main I2C connector | Shareable I2C rather than truly unused |
| 39, 40, 41 | Main SPI connector | Shareable with SD; also optional external JTAG conflict |
| 42 | Rear pad | Optional JTAG conflict |
| 43, 44 | Rear UART0 pads | Preserve for debug if possible |
| 10, 11 | Could be reclaimed only by modifying LED circuit | No existing pad |
| 0, 3, 45, 46 | **Not general allocation candidates without strap review** | ESP32-S3 strapping pins |

UART signals can be routed through the ESP32-S3 GPIO matrix. v3.4 explicitly exposes TXD0/RXD0 rear pads; GPIO15/16 are practical free pins for a derivative GNSS UART. SPI is already on 39/40/41 and can be shared with separate CS lines. I2C is already broken out on 1/2. Exact controller instance assignment is a firmware decision; the source does not reserve a specific I2C or secondary-UART peripheral number.

### 19–21. Rails, connectors, and dimensions

| Item | Finding | Evidence / confidence |
|---|---|---|
| Rails | `VCC` protected/ORed input domain, main `3V3`, `3V3_SWITCHED`, `USB5V`, and GND | Schematic net labels. **High confidence.** No regulated 5 V rail is generated. |
| Main connectors | POWER 1×2, CAN 1×2, I2C 1×4, SPI 1×3, GPIO 1×4, USB-C, microSD; additional rear solder pads and PROG/RESET buttons | Schematic/pinout/README. Connector physical orientation must be verified against placement before harness release. |
| Reference size | Board-outline Gerber header states 1.24 in × 1.95 in; outline coordinates agree, approximately 31.50 mm × 49.53 mm, with an antenna-end notch | `Gerber_BoardOutlineLayer.GKO` inside Gerber archive. README rounds this to 3×5 cm. **High confidence for fabricated outline.** |

### 22. Conflicts and capacity for planned additions

| Addition | Conflict/risk found | Preliminary resolution |
|---|---|---|
| NEO-M9N | Needs two GPIO, meaningful current, quiet supply, RF area/keep-out, U.FL and active-antenna bias; reference outline has little spare RF area | Use GPIO15/16 UART; create separate GNSS/RF region and preferably switched GNSS rail; board will likely grow |
| 20–25 Hz GNSS | UART throughput depends on enabled messages/baud; RF performance depends on antenna/layout/noise | Use UBX configuration with calculated throughput; follow official integration manual and validate coexistence |
| External SPI display | Shares 39/40/41 with SD; needs CS/DC/reset/backlight and potentially substantial power/current | Separate CS (47), DC (48), reset (12), backlight driver (7); serialize SPI and validate cable integrity |
| Shift light | Reference 3V3_SWITCHED is a load-switch output, not a universal automotive external driver; combined load budget unknown | GPIO6 into dedicated protected driver sized after module specification |
| Buzzer | No existing buzzer circuit; direct GPIO unsuitable for unspecified load | Resolved for v1: GPIO17 drives the TPA2005D1TDGNRQ1 input network; GPIO42 remains JTAG MTMS |
| MODE button | GPIO0 is already PROG strap and unsuitable for normal mode key | Reclaim GPIO10 from blue LED in derivative; keep boot strap untouched |
| microSD | GPIO45 CS is a strap pin; display sharing adds bus/cable load | Resolved for v1: SD CS moves to GPIO11; GPIO45 stays unloaded |
| Low power | Reference has CAN wake only while main rail is retained; full hardware-off is voltage/USB wake only | Resolved for the conditional v1 architecture: hybrid rail-on ESP32/TCAN standby with switched peripherals; Task 5A.1 paper envelope is 408.026 µA (0.408 mA) at 12 V and 0.476/0.438/0.419/0.408/0.402/0.400 mA at 6/8/10/12/14.4/18 V |
| Peak power | U4 is 600 mA class and U5 is undocumented here; new loads exceed its class | Resolved conditional silicon/population: LMQ66420MC3RXBRQ1 with exact `XGL5030-222MEC`, CIN/COUT/CVCC and CBOOT-DNP population; the 1.050 A operating-peak/thermal case and 1.313 A sizing result require a ≥2.0 A class. Effective capacitance, stability, thermal and EMI validation remain open |
| USB/JTAG | USB consumes 19/20; SPI overlaps JTAG 39–41 | Preserve USB; reserve GPIO42 MTMS test pad and treat external JTAG as optional |

## Frozen allocation result

The recommended table is maintained in [`interfaces.md`](interfaces.md). In compact form:

- CAN: 13/14/38.
- GNSS UART: 15/16.
- Shared SD/display SPI: 39/40/41; SD CS 11; display CS/DC/reset 47/48/12; backlight 7.
- I2C: 1/2.
- Shift-light: 6 through driver; sound: 17 through TPA2005D1TDGNRQ1 input network.
- MODE: 10; expander interrupt: 18; preserve UART0 43/44 and JTAG MTMS 42.
- Power/monitor: 21, 5, 8, 9 remain assigned.

## Unresolved engineering questions

1. Can the calculated 408.026 µA (0.408 mA) 12 V parked envelope, its 6–18 V model, and the <0.50 mA stretch limit be verified on the complete board over voltage/temperature and wake duty cycle?
2. What exact 12 V passenger-vehicle transient, EMC, ESD, temperature, vibration, and compliance standards/classes apply? 24 V support is not required.
3. Can optional split 120 Ω DNP/OFF termination and a DNP choke be laid out without harmful stubs/parasitics?
4. What are the exact display module voltage, peak/backlight current, cable length, connector, and MISO behavior?
5. What are the shift-light electrical load/fault cases and the exact speaker, acoustic load, and maximum operating duty?
6. Is NEO-M9N availability/lifecycle and qualification acceptable, and which active antenna is selected? V_BCKP is switched off in v1.
7. Which GNSS messages fit the assumed 200-byte epoch at the selected 230,400 bit/s while running the verified up-to-25 Hz constellation configuration?
8. Do the frozen LMQ66420 variants satisfy the transient, effective-capacitance, thermal, stability and EMI requirements with the exact Task 5A.1 `XGL5030-222MEC`/TDK population?
9. Does the GPIO11 SD_CS implementation remain inactive and non-back-powering through every reset/power state?
10. Are GPIO4/5 still needed for board revision identification in the derivative?
11. Is external four-wire JTAG required concurrently with the display/sound allocation, or is USB-JTAG sufficient?
12. What enclosure and OBD/display/GNSS cable geometry determines PCB outline and RF/EMC constraints?

## Authoritative documents required before schematic design

- [u-blox NEO-M9N integration manual, UBX-19014286](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf) and current NEO-M9N data sheet/product summary; selected active-antenna and U.FL vendor data sheets.
- [Espressif ESP32-S3-WROOM-1/1U data sheet](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf), [ESP32-S3 data sheet](https://documentation.espressif.com/esp32_s3_datasheet_en.pdf), hardware design guidelines, and module land-pattern/antenna guidance.
- [TI SN65HVD230 data sheet](https://www.ti.com/lit/ds/symlink/sn65hvd230.pdf), plus the selected automotive-qualified CAN-transceiver/protection data sheets if U2 changes.
- [TI LMR14006 data sheet](https://www.ti.com/lit/ds/symlink/lmr14006.pdf); complete, authoritative ME2807A33M3G and MT9700 data sheets. These latter two are not included in the repository and their official sources remain to be confirmed.
- Manufacturer data sheets for SMF30A, MF-MSMF110/16-2, every series/TVS/ESD diode, inductor, capacitors (including bias/temperature derating), microSD socket, USB-C connector, and all proposed load switches/drivers.
- Exact GC9A01/ST7789/display-module data sheets; panel-controller data sheet alone is insufficient for module power/backlight/connector details.
- SD Association simplified physical-layer specification and selected microSD card electrical/inrush guidance.
- USB Type-C and USB 2.0 requirements applicable to a USB device, plus Espressif native-USB layout guidance.
- SAE J1962 connector/pin requirements; ISO 11898-2 CAN physical-layer requirements.
- [ISO 7637-2](https://www.iso.org/standard/50925.html), [ISO 16750-2:2023](https://www.iso.org/standard/76119.html), ISO 10605, CISPR 25, and UNECE R10 or the exact regional/customer alternatives selected by the compliance plan.

## Stop point

No schematic, PCB, BOM, Gerber, placement, or manufacturing file has been created or modified. Review and resolve the requirements and high-risk questions above before derivative schematic capture begins.
