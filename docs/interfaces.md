# Telemetry v1 interfaces and preliminary pin allocation

## Proposed ESP32-S3 allocation

This allocation is direct repository evidence from the RejsaCAN v3.4 single-sheet schematic and `RejsaCAN v3.4 - Pinout.h`; it is not a calculated electrical value. It is preliminary: rows marked **modify** require a new derivative schematic and are not instructions to alter the reference files. No GPIO assignment changed during the power/wake review.

| Function | ESP32-S3 GPIO | Direction | Reference status / conflict | Proposal |
|---|---:|---|---|---|
| CAN RX | 13 | In | Used by U2 R; wake-capable RTC GPIO | Keep |
| CAN TX | 14 | Out | Used by U2 D | Keep |
| CAN standby/slope | 38 | Out/Hi-Z | Used by U2 RS through R3 10 kΩ | Keep; define modes in firmware |
| GNSS UART RX (ESP RX / M9N TX) | 15 | In | Rear breakout, otherwise unused | Add |
| GNSS UART TX (ESP TX / M9N RX) | 16 | Out | Rear breakout, otherwise unused | Add |
| Shared SPI SCLK | 39 | Out | SD CLK; JTAG MTCK | Keep/share with display; JTAG conflict documented |
| Shared SPI MOSI | 40 | Out | SD CMD; JTAG MTDO | Keep/share with display |
| Shared SPI MISO | 41 | In | SD DAT0; JTAG MTDI | Keep/share/optionally expose to display |
| microSD CS | 45 | Out | SD DAT3; **strapping pin** | Keep initially; validate reset loading and prefer moving in a future pin optimization if feasible |
| Display CS | 47 | Out | Front connector, unused | Add |
| Display DC | 48 | Out | Front connector, unused | Add |
| Display reset | 12 | Out | Front connector, unused | Add |
| Display backlight PWM/enable | 7 | Out | Rear breakout, unused | Add through a driver, not directly to load |
| Shift-light control | 6 | Out | Rear breakout, unused | Add through dedicated driver |
| Buzzer control | 42 | Out | Rear breakout/JTAG MTMS | Add through driver; loses optional external JTAG while used |
| I2C SDA | 1 | I/O | Existing I2C connector | Keep |
| I2C SCL | 2 | I/O | Existing I2C connector | Keep |
| MODE button | 10 | In | Currently BLUE LED, no pad | **Modify:** reclaim LED net and add benign pull-up/button-to-ground network |
| Peripheral power enable | 21 | Out | Controls reference U5/3V3_SWITCHED | Keep as domain enable initially; load/current architecture must be redesigned for total demand |
| Hardware hold | 17 | Out | FORCE_ON | Keep |
| Threshold sense | 8 | In | SENSE_V_DIG | Keep |
| Vehicle voltage ADC | 9 | In | SENSE_V_ANA | Keep; recalibrate divider and ADC protection |
| UART0 TX debug | 43 | Out | Rear TXD0 pad | Preserve |
| UART0 RX debug | 44 | In | Rear RXD0 pad | Preserve |
| Spare GPIO | 11 | I/O | Currently YELLOW LED, no pad | **Modify:** reclaim/expose if status LED is not retained |

GPIO4/GPIO5 are tied by the reference schematic to board-version identification and should be reviewed before reuse. GPIO0, GPIO3, GPIO45, and GPIO46 are strapping pins; GPIO19/20 are native USB; GPIO39–42 overlap external JTAG; GPIO43/44 are UART0. GPIO0 and GPIO3 remain untouched in this proposal.

## Proposed external connectors

Connector families, pin numbers, keying, retention, current ratings, and ESD parts are TBD. The names below define logical contracts only.

### OBD-II harness

| OBD pin | Signal | Notes |
|---:|---|---|
| 16 | VBAT | Permanent vehicle supply into protection |
| 4, 5 | GND | Define chassis/signal-ground strategy and wire allocation |
| 6 | CAN-H | Short branch; protection close to entry |
| 14 | CAN-L | Route as pair with CAN-H |

### DISPLAY

| Signal | Purpose |
|---|---|
| GND ×2 | Power and signal returns; connector current rating applies per pin |
| DISP_3V3 | Switched/current-limited ≥400 mA (`DESIGN_REQUIREMENT`) |
| DISP_5V | Optional switched/current-limited ≥600 mA (`DESIGN_REQUIREMENT`); never raw USB VBUS |
| SCLK, MOSI | Shared SPI bus |
| MISO | Optional for modules that read data; must tri-state when CS inactive |
| DISP_CS | Per-display selection |
| DISP_DC | Command/data selection |
| DISP_RST_N | Hardware reset |
| DISP_BL | PWM/enable into display/backlight driver contract |
| SDA, SCL | Optional 3.3 V I2C for touch/module identification |
| INT/TE | Optional 3.3 V touch interrupt or tearing-effect input |

Do not assume different marketplace GC9A01/ST7789/AMOLED modules share voltage levels, pin order, regulator/backlight topology, or reset polarity. Initial cable length is ≤200 mm and SPI clock ≤20 MHz (`DESIGN_REQUIREMENT`) until signal-integrity measurements justify more. At RGB565, one 240×240 frame is 921,600 bits (`CALCULATED`); 20 MHz at assumed 75% transfer efficiency is 16.3 frame/s (`CALCULATED`).

### SHIFT_LIGHT

Expose GND, switched/current-limited 5 V up to 1 A, and buffered 5 V logic data (`DESIGN_REQUIREMENT`). Connector rating is ≥1.5 A (`DESIGN_REQUIREMENT`). Default state is off/low during reset and sleep. Require an intelligent/differential module for cables longer than 0.5 m until EMC testing supports another limit.

### BUZZER (if external)

Expose GND and driven output, plus a defined supply if required. Connector must not invite connection of an inductive load without the required clamp.

### I2C expansion

Expose GND, 3.3 V (with current limit/budget), SDA, and SCL. Put one intentional set of pull-ups on the system and document allowable external pull-up range/cable length.

## Reference v3.4 connectors

From the single-sheet schematic:

| Connector | Pins/signals |
|---|---|
| POWER, 1×2 | pin 1 vehicle power into protection; pin 2 GND |
| CAN, 1×2 | pin 1 CAN-H; pin 2 CAN-L |
| I2C, 1×4 | GND, 3V3, SCL/GPIO2, SDA/GPIO1 |
| SPI, 1×3 | MOSI/GPIO40, SCLK/GPIO39, MISO/GPIO41 |
| GPIO, 1×4 | 3V3_SWITCHED, GPIO47, GPIO48, GPIO12 |
| USB-C | VBUS, USB D+/D−, two 5.1 kΩ CC pull-downs, shell/GND |

The schematic symbol establishes connectivity, but physical header pin-one orientation must be confirmed from the placement/Gerber files before creating harness drawings.

## Bus-level rules

- Every shared-SPI device gets a unique CS and must release MISO when not selected.
- Keep all device CS lines inactive through reset; GPIO45 strap behavior needs explicit oscilloscope validation with cards inserted and absent.
- Avoid using GPIO19/20 for anything except USB.
- MODE must not use the existing PROG/GPIO0 button because a held button changes boot behavior.
- CAN-H/L test points must be compact and not create long stubs.
- UART naming on connectors should include endpoint perspective (`ESP_RX/M9N_TX`, `ESP_TX/M9N_RX`).

## Internal product contracts

| Producer | Contract | Consumers |
|---|---|---|
| CAN/TWAI | Timestamped raw frames plus bus/mode health; transmission only through diagnostic policy | profile decoder, OBD/ISO-TP scheduler, optional raw logger |
| Vehicle decoders and GNSS | Canonical samples with source, time, validity, quality, and profile metadata | normalized telemetry core |
| Normalized telemetry core | Latest-value lookup and bounded event subscriptions with deterministic arbitration | RaceChrono, device protocol, display, shift-light, alarms, logger, optional lap engine |
| Configuration service | Validated versioned snapshots and migrations | all configurable modules |
| Health supervision | Module state, counters, timeouts, queue pressure, and restart/escalation events | device status, logger, first-party protocol |

RaceChrono and display adapters are never internal data buses. Consumers may subscribe at different rates, but a slow consumer cannot block acquisition or alter canonical values.

## Protocol and mechanical boundaries

- The first-party API has transport-independent service/schema versions with BLE, Wi-Fi, and USB bindings; see [`device-protocol-architecture.md`](device-protocol-architecture.md).
- Vehicle profiles are canonical repository data under `profiles/`; firmware deployment artifacts remain derived and version-linked.
- The core enclosure, display enclosure, and vehicle mount are separate interfaces. PCB geometry and electrical cable limits remain authoritative inputs to CAD; see [`enclosure-architecture.md`](enclosure-architecture.md).
- No GPIO or peripheral allocation is changed by these software/mechanical contracts.
