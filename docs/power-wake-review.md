# Telemetry v1 power, wake and electrical-architecture review

Status: reconciled to the Task 4.7 automotive-electrical/protection freeze, 2026-08-17. Values use the classification in [`power-budget.md`](power-budget.md). No schematic, footprint, layout or compliance approval is implied.

## Recommendation in one view

Use the frozen **hybrid rail-on architecture**. A protected source-OR feeds an LMQ66420MC3RXBRQ1 3.3 V buck. The ESP32-S3 and TCAN3404DRQ1 remain powered in deep-sleep/standby; GNSS, SD, display and the LMQ66420-based AUX5 converter are independently disabled. This supports CAN, timer, MODE and voltage wake without a separate wake MCU. USB powers the development domain through TPS2553QDBVRQ1 current limiting and PMEG6030EP-Q reverse isolation.

`CALCULATED`: complete parked envelope is ≤0.424 mA at 12 V with a 100% allowance; see `power-budget.md`. The <1.0 mA release requirement and <0.50 mA room-temperature stretch target pass on paper by 0.576 mA and 0.076 mA, respectively; neither is accepted until populated-board testing.

## Frozen power-domain block diagram

```text
OBD pin 16
    │
    ├─ 0437002A ─ SM8SF24CA-Q ─ LM74502 + 2×DMT6007LFGQ ─ damped filter ─ VEHICLE_PROTECTED
    │                                                        │
    │                                                        ├─ gated vehicle sense
    │                                                        │      (PARKED: on, ≤15 µA)
    │                                                        │
USB-C VBUS ─ USBLC6 ─ TPS2553-Q1 ─ PMEG6030EP-Q ──────────────┤ source OR
                                                             │
                                                    LMQ66420 MAIN_3V3 buck
                                                             │
                         ┌───────────────────────────────────┼──────────────────┐
                         │                                   │                  │
                  ESP32-S3 N16R8                      TCAN3404-Q1       switch: SD_3V3
                  PARKED: deep sleep                  PARKED: standby          │
                         │                                   │               microSD
                         │                                   └─ CANH/L protection
                         ├─ TPS22919: GNSS_3V3
                         │       └─ NEO-M9N + protected active-antenna bias
                         │          V_BCKP tied to switched GNSS rail (v1)
                         ├─ TPS22919: DISPLAY_3V3 ─ external display connector
                         └─ enable: LMQ66420 AUX5 2 A ─ protected branches
                                                   ├─ DISPLAY_5V option
                                                   ├─ SHIFT_5V
                                                   └─ buzzer driver option

OBD pins 6/14 ─ CAN TVS ─ optional DNP common-mode choke ─ TCAN3404-Q1
OBD pins 4/5  ─ separate conductors, one entry join into continuous POWER_GND
```

Only the input protection, low-IQ buck, vehicle sense, ESP32, TCAN3404 and wake nets remain energized in `PARKED/SLEEP`. USB ESD/CC remain electrically present but must not create a vehicle back-feed.

## Sleep/wake alternatives

| Option | Parked topology | Wake coverage | Current/complexity | Finding |
|---|---|---|---|---|
| A — rail-on deep sleep | ESP32 and CAN powered; peripherals off | CAN RX, timer, MODE, voltage, USB | Lowest additional parts; current depends on complete module rail | Viable and reliable; essentially the core of the recommendation |
| B — hardware-off plus always-on wake | Main rails off; battery CAN wake transceiver/controller remains | CAN/WAKE/voltage can assert regulator; timer/button need extra AON logic | Lowest main leakage but needs TCAN1043A-Q1-class 5 V/VIO/INH architecture or another AON controller and more sequencing | Not justified for v1 while the rail-on estimate is 0.424 mA |
| C — hybrid | ESP32/CAN rail-on; GNSS/SD/display/AUX5 independently off | Same as A, with clean peripheral isolation | More load switches/nets; best observability and fault containment | **Recommended** |

`VERIFIED_DATASHEET`: TCAN3404-Q1 standby monitors the bus for a wake-up pattern and drives RXD low after a valid WUP; standby current is at most 17 µA at 150 °C. `VERIFIED_DATASHEET`: TCAN1043A-Q1 provides VSUP, WAKE and INH and specifies 18 µA typical/30 µA maximum VSUP sleep current, enabling a true hardware-off system, but it also requires 5 V VCC and additional sequencing. The latter remains a future alternative if measured v1 parked current cannot meet the target.

### Wake-source behavior

| Source | Conceptual mechanism | Important qualification |
|---|---|---|
| CAN | TCAN3404DRQ1 standby WUP → RXD/CAN_RX on GPIO13 | Verify deep-sleep wake behavior on the exact ESP32-S3 module and bench-test real vehicle traffic. A WUP indicates activity, not permission to transmit. |
| Vehicle voltage | Low-IQ comparator/detector plus gated ADC measurement | A hint only; smart charging invalidates voltage-only ignition inference. Thresholds/hysteresis remain `TBD` pending vehicle measurements. |
| Timer | ESP32 RTC timer | Used for bounded health checks; duty cycle must be included in average parked current. |
| MODE button | Normally-open button to non-strap wake GPIO10 | Debounce and leakage network must meet the parked allocation. |
| USB | VBUS presence forces/requests development power and enters `USB_DEBUG` | Source isolation must prevent VBUS reaching OBD pin 16 or AUX5 outputs. |

## Conceptual vehicle-state machine

```text
                         USB present
                 ┌──────────────────────┐
                 ▼                      │
PARKED ─event─> WAKE ─qualified──────> ACTIVE
  ▲              │  │                    │
  │              │  └─not qualified─────┤ timeout
  │              │                       ▼
  │              └────────────── SHUTDOWN_PENDING
  │                                      │ flush log, save GNSS state,
  │                                      │ disable outputs/peripherals
  └──────────────── SLEEP <──────────────┘
                         │
                         └─ CAN WUP / timer / MODE / voltage → WAKE

Any powered state ─ USB-only or explicit debug policy ─> USB_DEBUG
USB_DEBUG ─ OBD activity qualified ─> ACTIVE
USB_DEBUG ─ USB removed, no OBD ─> SHUTDOWN_PENDING/SLEEP or off
```

State policy (`DESIGN_REQUIREMENT`):

- `PARKED/SLEEP`: CAN standby and ESP deep sleep; all peripheral rails and indicators off.
- `WAKE`: debounce/classify event without immediately transmitting. CAN traffic plus valid frames is stronger evidence than voltage alone.
- `ACTIVE`: entered by sustained/recognized CAN activity, recent diagnostic exchange, an explicit MODE action, or a combined voltage-and-activity policy. Diagnostic transmission is rate-limited and independently enabled; passive/listen-only remains a distinct mode.
- `SHUTDOWN_PENDING`: stop new transactions, make shift-light/buzzer safe, flush/close SD, optionally save GNSS state to host nonvolatile storage, then disable peripheral rails. Repeated CAN activity returns to `ACTIVE`.
- `USB_DEBUG`: native USB enabled. USB alone may run ESP32, SD for servicing and optionally GNSS within the declared USB current; it does not energize external 5 V loads or the vehicle-side input.
- All thresholds, debounce times, quiet timeout and diagnostic grace period are unresolved policy values, to be derived from logged vehicle behavior. No numeric timing is invented here.

## Automotive input review and requirements

### RejsaCAN v3.4 evidence

The repository single-sheet v3.4 schematic shows DSS34 D6 in series, MF-MSMF110/16-2 F1, SMF30A D4 from VCC to ground, and LMR14006X U4. That is evidence of a fuse/series-diode/TVS/buck concept, not proof of a coordinated automotive protection network. The PTC's `/16` voltage class, 30 V TVS clamping behavior and 40 V buck limit need full pulse/energy coordination; the reference circuit is not carried forward unchanged.

### Task 4.7 condition-to-element freeze

| Electrical condition | Frozen element/function | Remaining verification |
|---|---|---|
| Board/harness hard short | 2 A/63 V `0437002A`; downstream branch switches | Fuse time-current/inrush/hot derating, copper/harness coordination and TVS-short clearing |
| Reverse battery / negative pulse | Bidirectional SM8SF24CA-Q plus LM74502-Q1/back-to-back FET disconnect | −14 V/60 s and approved fast-negative tests; no normal fuse opening; VDS/VGS/SOA and recovery |
| Suppressed load dump / OV | 24 V TVS, 18 V nominal OV disconnect, raw 60/65 V devices | +38 V source test, hot/repetitive clamp, `VEHICLE_PROTECTED` ≤24 V and automatic recovery |
| Severe unsuppressed load dump | Outside v1 guaranteed envelope | Purchased-standard/OEM requirement and a larger surge-stopper review if later required |
| Fast positive transients | TVS, short entry loop and damped post-switch C-L-C | Approved pulse matrix, simultaneous raw/switched/protected voltage/current and ringing |
| Cranking/brownout | 6 V nominal UV disconnect, controlled reset, staged restart | Threshold tolerance, crank waveforms, no reboot loop, CAN/SD/peripheral recovery |
| ESD/conducted emissions | Interface-local ESD and controlled returns/filtering | ISO 10605/CISPR 25/applicable UNECE R10 plan; no compliance claim without testing |

`SM8SF24CA-Q` is frozen: bidirectional AEC-Q101, 24 V stand-off, 26.7–29.5 V breakdown, 38.9 V maximum clamp at 180 A, 7 kW 10/1000 µs and 10 µA maximum leakage at 24 V/25 °C (`VERIFIED_DATASHEET`). Its bidirectional construction avoids forward conduction during sustained reverse battery. The +50 V/2 Ω public fast-pulse reference produces a conservative `(50−38.9)/2 = 5.55 A` source-current screen; the controller and MOSFET margins to 38.9 V are 26.1 V and 21.1 V (`CALCULATED`). These are screening calculations, not a measured clamp result.

Task 4.7 selects 6–18 V operation, +26 V/60 s survival by disconnect, −14 V/60 s survival, +38 V suppressed-load-dump-source survival by disconnect, and controlled reset below UVLO. Severe public unsuppressed references remain outside the v1 guarantee. Full evidence and standards boundaries are in [`automotive-electrical-profile.md`](automotive-electrical-profile.md) and [`input-protection-architecture.md`](input-protection-architecture.md).

## CAN physical layer

### Transceiver comparison

| Part | Qualification / supply | Low-power and wake | Bus robustness | Active supply | Assessment |
|---|---|---|---|---|---|
| TI SN65HVD230DR (reference U2) | Standard catalog, 3.0–3.6 V, Classical CAN | 370 µA typical standby; receiver remains active | −4 to +16 V common mode; ±36 V bus-pin protection stated | 17 mA recessive typical; 48 mA dominant typical with 60 Ω | Functional reference but not automotive-qualified, materially higher standby current and less bus-fault margin. Do not retain. |
| TI TCAN3404-Q1 | AEC-Q100, 3.3 V single supply, Classical CAN/CAN FD | 10 µA typical, 17 µA maximum standby; WUP on RXD; shutdown available but cannot wake | ±58 V bus standoff, ±30 V common mode, ±8 kV ISO 10605 bus ESD; high-Z unpowered bus pins | 8.2 mA recessive max; 55 mA dominant max at 60 Ω; 130 mA fault limit | **Recommended** for rail-on/hybrid topology: simple 3.3 V interface and low standby. |
| TI TCAN3403-Q1 | AEC-Q100, VCC plus VIO, CAN FD | Similar WUP; split VCC/VIO permits VCC switching | Same family bus ratings | Standby combines VCC and VIO currents | Useful if physical layer supply is switched separately; unnecessary complexity for v1. |
| TI TCAN1043A-Q1 | AEC-Q100, battery VSUP + 5 V VCC + VIO; INH/WAKE | True sleep/cold wake; VSUP 18 µA typical/30 µA max | ±58 V CAN pins and automotive system features | Up to 60 mA dominant at 60 Ω | Best option for hardware-off architecture, but requires 5 V and more sequencing/parts. Reserve as fallback. |

Values are `VERIFIED_DATASHEET` from TI SLLS500K, SLLSFQ6A and SLLSF27. Exact temperature/condition rows remain controlling.

Package/lifecycle comparison (`VERIFIED_DATASHEET`/official product status): SN65HVD230DR is the reference SOIC-8 catalog device. TCAN3403/3404-Q1 are active/production, −40 to +150 °C orderable families. TCAN1043A-Q1 is active/production with more pins and supplies. TCAN3404DRQ1 SOIC-8 is frozen; Task 4 availability and price evidence is recorded in `component-freeze.md`.

### Recommended CAN network interface

- Use frozen TCAN3404DRQ1 SOIC-8 with TXD, RXD and STB; default STB high during reset. Firmware modes are passive/listen-only, active raw CAN, and diagnostic transmission. ISO-TP/UDS/OBD are protocol layers, not electrical modes.
- Use connector-local ESDCAN04-2BWY. `VERIFIED_DATASHEET`: maximum capacitance is 19 pF, typical breakdown is 27.5 V, clamping is 43 V at 3 A, and leakage is 0.05 µA under the specified condition. System pulse/ESD validation remains mandatory.
- Provide an ACT45B-510-2P-TL003 footprint, DNP with 0 Ω bypasses by default. Populate only if emissions/immunity testing justifies it.
- Provide optional split 120 Ω termination as two 60.4 Ω-class resistors plus center capacitor **DNP/OFF by default**. `VERIFIED_DATASHEET`: TI states a high-speed CAN bus is terminated by 120 Ω at each physical end and split termination may filter common-mode noise. `CALCULATED`: adding 120 Ω in parallel with the already terminated vehicle's effective 60 Ω produces `60 || 120 = 40 Ω`, an improper extra load. Populate only for isolated bench use where this node is an actual endpoint.
- Keep the OBD branch short, route CANH/L as a pair, put TVS/filtering at entry, avoid long test-point stubs and verify unpowered leakage. The recommended transceiver specifies high-impedance bus pins when unpowered.
- `VERIFIED_DATASHEET`: TCAN3404-Q1 bus loading is at least 13 kΩ single-ended and 25 kΩ differential, with at most 40 pF to ground and 20 pF differential; unpowered bus leakage is at most 5 µA under the stated test. With the optional terminator DNP, this is a high-impedance stub rather than a third terminator. Add TVS/choke parasitics to the final signal-integrity budget.
- A valid WUP wakes the processor; it does not cause automatic diagnostic traffic. Default product behavior after wake is passive observation until the vehicle profile/policy explicitly allows transmission.

## GNSS electrical architecture

`VERIFIED_DATASHEET` from u-blox NEO-M9N-00B R08: VCC is 2.7–3.6 V, V_BCKP is 1.65–3.6 V, peak acquisition current is 100 mA, continuous four-GNSS tracking is 36 mA at the documented 1 Hz condition, and backup current is 45 µA typical at 3 V/25 °C. VCC_RF is approximately VCC−0.1 V and can source 50 mA. UART supports 4,800–921,600 bit/s, defaults to 38,400 8N1 and has no hardware flow control. PIO levels reference VCC.

- Switch a ≥200 mA GNSS_3V3 branch with local decoupling. The integration manual says startup charging makes VCC dynamic and series resistance above 0.2 Ω is prohibited (`VERIFIED_DATASHEET`). Check switch/drop/filter resistance accordingly.
- Use U.FL and a 50 Ω RF path, but do not route it yet. Feed the selected active antenna through a current-limited/short-protected, filtered bias-T based on the integration manual and antenna data sheet. The manual example uses 100 nF X7R DC block and 27 nH choke with >500 Ω impedance at GNSS frequencies and >300 mA rating; these are reference values, not an approved schematic.
- Add low-capacitance RF ESD protection only after its capacitance/insertion loss is checked. Locate ESD at the connector and keep switching nodes/cables away from RF.
- Isolate UART/PIO when GNSS VCC is off; u-blox warns these pins must be high impedance with V_BCKP present to avoid back-power.

### UART throughput

The exact enabled UBX message set remains unresolved, so use an explicit envelope:

```text
Message envelope = 200 bytes/navigation epoch                  [ASSUMPTION]
Raw serial rate at 25 Hz, 8N1 = 200 × 25 × 10 = 50,000 bit/s  [CALCULATED]
Required rate at ≤50% occupancy = 50,000 / 0.50 = 100,000 bit/s
Selected UART rate = 230,400 bit/s                              [DESIGN_REQUIREMENT]
Occupancy = 50,000 / 230,400 = 21.7%                            [CALCULATED]
```

Configure UBX binary output only and validate the complete message list, parser latency and error recovery. The 230,400 rate is supported by the module and has >2× capacity over the 100 kbit/s requirement. `VERIFIED_DATASHEET`: u-blox Integration Manual R10 Appendix A states up to 25 Hz for all supported constellation combinations; the exact enabled messages and hardware performance still require validation.

### Backup comparison and choice

| Option | Parked effect | Benefits | Risks | Decision |
|---|---:|---|---|---|
| No dedicated backup; V_BCKP follows GNSS rail | 0 µA | Simplest, no back-power/storage aging | Cold start unless state restored | **Recommended for v1**, plus host-side UBX save/restore investigation |
| Always-on regulated backup | 45 µA typical module plus regulator loss | RTC/BBR retention | Material persistent load, rail sequencing | Not needed for first revision |
| Supercapacitor | Time-limited | No replaceable cell | Leakage, temperature/lifetime, charge control and uncertain hold time | Reject until a start-time requirement justifies it |
| Rechargeable cell | Long retention | Good restart continuity | Charger, safety, temperature, lifetime and shipping complexity | Reject for v1 |
| Primary cell | Longest retention | Independent of vehicle battery | Service, shipping, depletion and reverse-current isolation | Reject for v1 |

The integration manual documents UBX-UPD-SOS save/restore using host storage. That is the preferred path to investigate; if it is not reliable enough, a DNP-isolated V_BCKP option may be added at schematic review. Cold/warm/hot time-to-fix values depend on retained data, signal conditions and configuration and are not used as product guarantees.

Startup implication: removing VCC and V_BCKP loses RTC/backup RAM and normally produces a cold start; retained valid time/orbit data enables warm/hot behavior, while host-restored data produces an aided start. `VERIFIED_DATASHEET`: the NEO-M9N product summary quotes typical default-mode 24 s cold, 2 s aided and 2 s hot acquisition under its footnoted test conditions. These are comparison values, not acceptance limits; enclosure antenna performance and parked duration can materially change time to fix.

## Universal display connector

Proposed logical contract (final connector family/pin numbering is unresolved):

| Signal | Electrical contract |
|---|---|
| GND ×2 | Adjacent returns for power and SPI |
| DISP_3V3 | Switched, current-limited, ≥400 mA branch capacity |
| DISP_5V | Optional switched AUX5 branch, ≥600 mA; keyed/configured so incompatible power is not applied |
| SCLK, MOSI, MISO | 3.3 V SPI; MISO optional and must release when CS inactive |
| CS, DC, RESET_N | 3.3 V logic, defined inactive state while module is off/reset |
| BL_PWM/EN | 3.3 V logic control; backlight power is not sourced by the GPIO |
| SDA, SCL | Optional 3.3 V I²C for touch/ID; pull-up ownership documented |
| INT/TE | Optional 3.3 V input for touch interrupt or tearing-effect indication |

Do not assume a module accepts both power rails or that “5 V module” logic is 3.3 V safe. A keyed cable plus module-specific adapter/pin map is preferred. Add connector-side ESD, series-damping footprints on high-edge-rate SPI and local bulk capacitance at the display. Default cable requirement is ≤200 mm (`DESIGN_REQUIREMENT`) until eye/edge measurements prove longer; choose an initial SPI clock ≤20 MHz over the cable (`DESIGN_REQUIREMENT`) and pair clocks with ground in the harness.

For RGB565 full-frame writes:

```text
Frame = 240 × 240 × 16 = 921,600 bit = 115,200 bytes       [CALCULATED]
20 MHz ideal frame rate = 20,000,000 / 921,600 = 21.7 fps  [CALCULATED]
At 75% bus efficiency = 16.3 fps                            [ASSUMPTION + CALCULATED]
40 MHz ideal / 75% = 32.6 fps                               [CALCULATED]
```

`DESIGN_REQUIREMENT`: guarantee 15 fps full-frame at 20 MHz assuming ≥75% transfer efficiency, and use dirty rectangles for higher apparent rates. GC9A01A supports serial interfaces, but the exact module's maximum SPI rate, cable and shared-SD integrity must be verified before approving 40 MHz. SD and display share only SCLK/MOSI/MISO; each has independent CS, serialized transactions and power-off isolation.

## Shift-light interface

| Alternative | Power/data | Advantages | Disadvantages |
|---|---|---|---|
| Individual LEDs | Multiple constant-current channels and wires | Deterministic EMI/current, precise current | Connector/pin/driver count and cabling |
| WS2812/SK6812-class strip | Switched 5 V, one data line | Simplest 8–10 LED hardware, easy patterns | Product variants, single-ended edge over cable, EMI and 3.3 V logic compatibility |
| Intelligent external module | Protected power plus CAN/UART/differential control | Best long-cable robustness and independent mechanics | More cost/protocol/software and another bus node |

**v1 recommendation:** a protected, current-limited 5 V/1 A connector and buffered addressable-LED data, intended for a short local cable. Use a 5 V AHCT-class buffer or an equivalent translator whose 3.3 V input-high guarantee and 5 V output are verified; add default-low bias, source-series damping, connector ESD and local strip bulk capacitance. The WS2812B-2020 minimum VIH of 2.7 V at 5 V means direct 3.3 V can work for that exact part, but interchangeability/cable noise margin justifies translation. Connector current rating ≥1.5 A (`DESIGN_REQUIREMENT`, 50% above the 1 A branch limit). For cables beyond 0.5 m or an electrically noisy mounting, require an intelligent/differential external module (`DESIGN_REQUIREMENT`); exact threshold requires EMC testing.

## Sound architecture

Use approved-direction `TPA2005D1TDGNRQ1` on the switched `SOUNDER5`/AUX5 domain, driving a provisional onboard differential 8 Ω, ≥1 W speaker. GPIO17 supplies the conditioned waveform. Neither BTL terminal is ground; the PowerPAD, local decoupling, 300 mA branch envelope, maximum-load thermal result and RF/EMI effect require validation. AUX5 is off parked.

## USB power interaction

| Case | Required result |
|---|---|
| OBD present, USB absent | Vehicle path powers MAIN_3V3; peripherals follow state policy; USB VBUS pin is not driven. |
| OBD absent, USB present | Reverse-blocked USB path powers MAIN_3V3 for ESP32 native USB, CAN logic, SD servicing and optionally GNSS within source budget. AUX5/external loads default off. CAN bus pins remain high impedance if no OBD harness. |
| OBD and USB present | Vehicle normally supplies the buck and PMEG blocks current to VBUS. Tolerance-bounded vehicle UVLO must open above the USB-derived crossover during an OBD sag; transient reverse current into pin 16 must be measured. |
| Neither present | All rails off. |

Preserve ESP32-S3 native USB on GPIO19/20 and use USBLC6-2SC6Y connector ESD, Type-C sink CC pull-downs, TPS2553QDBVRQ1 current limiting, and PMEG6030EP-Q reverse isolation. With 43.2 kΩ ILIM, the TI table equation gives 604.6 mA nominal and approximately 544.3–673.1 mA bounds (`CALCULATED`); firmware/configuration still limits USB-only load to 500 mA and keeps AUX5 exports off. GCT USB4105 is provisional pending mechanical confirmation.

## Decision table

| Item | RejsaCAN v3.4 | Telemetry v1 proposal | Reason | Confidence |
|---|---|---|---|---|
| Input protection | DSS34 + MF-MSMF110/16-2 + SMF30A | 0437002A, SM8SF24CA-Q, LM74502QDDFRQ1, 2×DMT6007LFGQ-7, damped C-L-C | Bidirectional clamp, calculable inrush, UV/OV disconnect and defined suppressed envelope | Architecture/active parts frozen; exact passives and physical validation pending |
| Main regulator | LMR14006X, 600 mA class | LMQ66420MC3RXBRQ1, 2 A | 1.05 A calculated 3.3 V peak envelope and parked IQ target | Silicon frozen; passives/thermal pending |
| Secondary regulator(s) | MT9700 switched 3.3 V load switch | TPS22919QDCKRQ1 ×4 plus LMQ66420 AUX5 | Isolation, inrush/fault control and 5 V external compatibility | Silicon frozen |
| Always-on rail | Common 3.3 V only while buck held | MAIN_3V3 always-on in parked sleep; peripherals independently off | CAN/timer/button wake with 0.424 mA bounded input | High pending measurement |
| CAN transceiver | SN65HVD230DR | TCAN3404-Q1 | AEC-Q100, much lower standby, WUP, wider bus fault/common mode | High |
| CAN protection | No dedicated CAN TVS/choke evident | ESDCAN04-2BWY; ACT45B-510-2P-TL003 DNP footprint | Connector ESD/transient robustness and test flexibility | Frozen |
| CAN termination | 120 Ω cut/jumper concept | Optional split 120 Ω, DNP/OFF default | OBD node joins already terminated vehicle; extra 120 Ω yields 40 Ω effective | High |
| Sleep strategy | Buck hold or hardware off | ESP deep sleep + CAN standby; all peripherals off | Reliable wake and low measured-risk complexity | High |
| CAN wake | Only while common 3.3 V remains | TCAN3404 WUP on RXD to ESP wake input | Direct low-IQ wake path | High transceiver; medium GPIO until bench test |
| GNSS main power | None | Independent ≥200 mA GNSS_3V3 switch and protected antenna bias | Peak/inrush, RF noise and parked isolation | High |
| GNSS backup | None | V_BCKP follows switched VCC; host save/restore investigated | Zero parked backup current and no storage lifecycle | Medium |
| Display power | Generic 3V3_SWITCHED | 3.3 V/400 mA and optional 5 V/600 mA switched contracts | Module independence; controller current is not backlight current | Medium until module selection |
| Shift-light power | None | Ten-pixel, 0.50 A qualified load; retained protected 5 V/1 A fault envelope plus buffered data | Supports the frozen addressable load without GPIO power | Medium pending exact pixel/connector |
| Sounder | None | Approved-direction TPA2005D1TDGNRQ1, 300 mA branch and provisional 8 Ω speaker | Programmable volume/tone and protected BTL drive | High amplifier evidence; acoustic/thermal test pending |
| USB power isolation | Schottky from VBUS into VCC/enable path | Reverse-blocked source OR before main buck; AUX5 off on USB-only | Supports all four cases without vehicle/host back-feed | High architecture; low exact part |

## Unresolved items and validation gates

- Complete UV/OV divider tolerances, `Cdvdt` inrush, fuse/SOA and damped-filter calculations, then validate the frozen electrical profile physically.
- Complete LMQ66420-Q1 inductor/capacitor, loss, stability, thermal/EMI, crank and transient-headroom calculations before schematic release.
- Confirm GPIO13 CAN wake, GPIO8 vehicle-activity wake, and all power-off signal isolation in an ESP32-S3 prototype.
- Select the exact active antenna, production card/socket, physical display connector/module adapter, shift pixel, speaker and status LED; replace remaining envelopes with maximum data and measurements.
- Validate GNSS 25 Hz configuration/message set and interference with the buck, ESP RF, SPI and external cables.
- Finalize connector families/pin numbering, cable construction, USB-shell population and environmental ratings; the one-plane ground architecture is frozen.
- Bench all USB source combinations and abnormal connections; confirm no reverse current into USB VBUS or OBD battery.

## Primary sources

- Repository: `Schematics/RejsaCAN v3.x (ESP32-S3 based board)/RejsaCAN v3.4 - Schematic.json` and matching single-sheet PNG; U2, U3, U4, D4, D6, F1 and related nets/components.
- TI, [SN65HVD230 Data Sheet SLLS500K](https://www.ti.com/lit/ds/symlink/sn65hvd230.pdf).
- TI, [TCAN3404-Q1/TCAN3403-Q1 Data Sheet SLLSFQ6A](https://www.ti.com/lit/ds/symlink/tcan3404-q1.pdf), Tables 7-1/7-2 and §§8–9.
- TI, [TCAN1043A-Q1 Data Sheet SLLSF27](https://www.ti.com/lit/ds/symlink/tcan1043a-q1.pdf), supply-current tables and mode descriptions.
- STMicroelectronics, [ESDCAN04-2BWY product information](https://www.st.com/en/protections-and-emi-filters/esdcan04-2bwy.html).
- Bourns, [SM8SF-Q automotive TVS data sheet](https://www.bourns.com/docs/Product-Datasheets/SM8SF-Q.pdf); TI, [LM74502-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lm74502-q1.pdf).
- STMicroelectronics, [LDP01-xxAY automotive load-dump TVS data sheet](https://www.st.com/resource/en/datasheet/ldp01-28ay.pdf), rejected unidirectional pre-controller candidate.
- STMicroelectronics, [TN1517 TVS FAQ](https://www.st.com/resource/en/technical_note/tn1517-faq-tvs-stmicroelectronics.pdf).
- ISO, [ISO 7637-2:2011](https://www.iso.org/standard/50925.html), [ISO 16750-2:2023](https://www.iso.org/standard/76119.html), and [ISO 10605:2023](https://www.iso.org/standard/79094.html) scope pages; IEC [CISPR 25:2021](https://webstore.iec.ch/en/publication/64645).
- Espressif, [ESP32-S3-WROOM-1/1U Data Sheet v1.8](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf).
- u-blox, [NEO-M9N-00B Data Sheet R08](https://content.u-blox.com/sites/default/files/NEO-M9N-00B_DataSheet_UBX-19014285.pdf) and [Integration Manual R10](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf).
- GalaxyCore, [GC9A01A Data Sheet v1.0 preliminary](https://buydisplay.com/download/ic/GC9A01A.pdf).
- Worldsemi, [WS2812B-2020 Data Sheet v1.3](https://cdn-shop.adafruit.com/product-files/4684/4684_WS2812B-2020_V1.3_EN.pdf).
