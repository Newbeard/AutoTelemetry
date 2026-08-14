# Telemetry v1 interfaces and frozen pin allocation

## ESP32-S3 allocation

This Task 4 allocation supersedes the preliminary Task 2 table. It is based on the RejsaCAN v3.4 single-sheet schematic, `RejsaCAN v3.4 - Pinout.h`, and the ESP32-S3-WROOM-1 v1.8 pin/strap tables. It defines the future derivative schematic and does not modify the reference files.

| Function | ESP32-S3 GPIO | Direction | Reference status / conflict | Status |
|---|---:|---|---|---|
| CAN RX/WUP | 13 | In | Used by U2 R; RTC GPIO | `FROZEN` |
| CAN TX | 14 | Out | Used by U2 D | `FROZEN` |
| CAN standby | 38 | Out | Used by U2 RS | `FROZEN`; pull-up makes standby default |
| GNSS UART RX (ESP RX / M9N TX) | 15 | In | Rear breakout | `FROZEN` |
| GNSS UART TX (ESP TX / M9N RX) | 16 | Out | Rear breakout | `FROZEN` |
| Shared SPI SCLK/MOSI/MISO | 39/40/41 | Out/Out/In | SD bus; JTAG MTCK/MTDO/MTDI | `FROZEN`; shared SD/display |
| microSD CS | 11 | Out | Reclaims YELLOW LED | `FROZEN`; moved from GPIO45 |
| Display CS/DC/reset | 47/48/12 | Out | Reference breakouts | `FROZEN` |
| Display backlight PWM | 7 | Out | Rear breakout | `FROZEN`; through open-drain driver |
| Shift-light data | 6 | Out | Rear breakout | `FROZEN`; through 5 V buffer |
| Buzzer PWM | 17 | Out | Reference FORCE_ON is replaced by rail-on architecture | `FROZEN`; MOSFET driver |
| I2C SDA/SCL | 1/2 | I/O | Existing I2C connector | `FROZEN`; also TCA6408A-Q1 |
| MODE button | 10 | In | Reclaims BLUE LED | `FROZEN`; non-strap RTC GPIO |
| AUX5 enable | 21 | Out | Replaces generic 3V3_SWITCHED control | `FROZEN`; pull-down default off |
| USB present | 4 | In | Reclaims board-version input | `FROZEN`; high-impedance VBUS sense |
| Power fault/status | 5 | In | Reclaims board-version input | `PROVISIONAL`; exact supervisor TBD |
| Expander interrupt | 18 | In | Previously unused | `FROZEN` |
| Threshold/activity sense | 8 | In | SENSE_V_DIG concept | `FROZEN`; detector circuit provisional |
| Protected vehicle ADC | 9 | In | SENSE_V_ANA concept | `FROZEN`; gated divider provisional |
| UART0 TX debug | 43 | Out | Rear TXD0 pad | Preserve |
| UART0 RX debug | 44 | In | Rear RXD0 pad | Preserve |
| Native USB D−/D+ | 19/20 | I/O | Native USB | `FROZEN` |
| JTAG MTMS | 42 | I/O | Rear breakout | `RESERVED`; no buzzer conflict |
| Strap/test | 45 | — | SD DAT3 in reference; VDD_SPI strap | `RESERVED`; no removable-card load |
| Strap pins | 0/3/46 | — | Boot/configuration straps | `RESERVED` except GPIO0 PROG |

TCA6408A-Q1 P0…P7 are frozen as GNSS_EN, SD_EN, DISP3_EN, DISP5_EN, SHIFT5_EN, SD_CD_N, STATUS_LED_N and reserved. External pull-downs keep all rail enables off while the expander powers up as inputs. GPIO39–41 intentionally overlap external JTAG; native USB Serial/JTAG is primary, and GPIO42 remains a test pad.

## Proposed external connectors

OBD/display/shift connector mechanics remain provisional. Electrical pin contracts, current limits and state behavior are frozen in [`schematic-architecture.md`](schematic-architecture.md).

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
- Keep all device CS lines inactive through reset. GPIO45 is reserved; microSD CS is GPIO11.
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
