# Telemetry v1 power, wake and electrical-architecture review

Status: Task 5A.1 conditional power-architecture selection, 2026-08-17. This is a documented prototype/schematic-capture starting point, not a released schematic, footprint/layout approval, manufacturing package or compliance claim. Detailed calculations and validation gates are controlled in [`task5a1-power-architecture.md`](task5a1-power-architecture.md); historical Task 4.7 selections below are explicitly marked superseded.

## Recommendation in one view

Use the selected **hybrid rail-on architecture with source isolation after conversion**. The vehicle path is `0437002.WRA` fuse -> `VBAT_FUSED_CLAMPED` -> `LM74720QDRRRQ1` with two back-to-back `STL125N10F8AG` 100 V N-FETs -> damped input filter -> shared `FILTERED_VEHICLE`. The common-anode anti-series `SM15T47AY` + `SM15T33AY` network is a shunt from `VBAT_FUSED_CLAMPED` to `POWER_GND`, not a series element. `FILTERED_VEHICLE` feeds both `LMQ66420MC3RXBRQ1` -> `VEH_3V3` and normally-off `LMQ66420MC5RXBRQ1` -> AUX5. USB-C uses sink CC terminations and USBLC6 protection on D+/D−; VBUS passes `TPS2553QDBVRQ1` and fixed-3.3 V `TPS62162QDSGRQ1` to `USB_3V3`. `TPS2116DRLR` selects `VEH_3V3` on VIN1 or `USB_3V3` on VIN2 and produces core rail `MAIN_3V3`.

The ESP32-S3 core can therefore run from either source, while `TCAN3404DRQ1` VCC and the AUX5 converter remain directly vehicle-only. GNSS, SD, display and AUX5 loads remain independently disabled by default. This partition prevents USB from accidentally powering the CAN transceiver or external 5 V loads and removes vehicle under-voltage thresholds from the USB crossover decision.

`CALCULATED`: the corrected Task 5A.1 parked-current tree has a 204.013 µA conservative subtotal and a doubled 408.026 µA (0.408 mA) design envelope at 12 V. The <1.0 mA release requirement and <0.50 mA room-temperature stretch target pass on paper by 0.592 mA and 0.092 mA, respectively. The doubled 6/8/10/12/14.4/18 V model is 0.476/0.438/0.419/0.408/0.402/0.400 mA. This remains a component-limit calculation, not an accepted board measurement; voltage, temperature, production spread and wake-duty measurements are still mandatory.

## Selected conditional power-domain block diagram

```text
OBD pin 16
    │
    └─ 0437002.WRA ─ VBAT_FUSED_CLAMPED ─ LM74720-Q1 + 2× back-to-back STL125N10F8AG
                              │                              │
                              └─ common-anode TVS shunt      └─ post-disconnect damped filter ─ FILTERED_VEHICLE
                                 SM15T47AY + SM15T33AY
                                 to POWER_GND
                                                               │
                                                               ├─ LMQ66420 MAIN buck ─ VEH_3V3 ─┬─ TPS2116 VIN1
                                                               │                                 └─ TCAN3404 VCC
                                                               └─ vehicle-only LMQ66420 AUX5
                                                                                 ├─ DISPLAY_5V
                                                                                 ├─ SHIFT5
                                                                                 └─ SOUNDER5

USB-C D+/D− ─ USBLC6 D+/D− pass-through (VBUS shunt/reference)
USB-C CC1/CC2 ─ independent Type-C Rd
USB-C VBUS ─ TPS2553-Q1 ─ TPS62162-Q1 ─ USB_3V3 ─ TPS2116 VIN2
                                                                                           │
                                                                           TPS2116 ─ MAIN_3V3
                                                                                           │
                                               ┌───────────────────────────────────────────┼─────────────┐
                                               │                                           │             │
                                         ESP32-S3 N16R8                           TCA6408A/LP5814   switched 3.3 V
                                         PARKED: deep sleep                                         branches

OBD pins 6/14 ─ CAN TVS ─ optional DNP common-mode choke ─ TCAN3404-Q1
OBD pins 4/5  ─ separate conductors, one entry join into continuous POWER_GND
```

With vehicle power present in `PARKED/SLEEP`, only the vehicle protection/controller tree, vehicle MAIN buck, `VEH_3V3`, TPS2116, ESP32 deep-sleep domain, TCAN3404 standby domain and required wake/sense nets remain energized. All switched peripherals, AUX5 and visible indicators are off. With USB only, TPS2116 powers `MAIN_3V3`, but the vehicle-only CAN and AUX5 domains remain unpowered. TPS2116 reverse-current blocking and the LM74720 vehicle reverse-current block are the architectural no-backfeed barriers; their behavior, including power-up/down overlap and abnormal connections, remains a four-state bench acceptance gate.

## Sleep/wake alternatives

| Option | Parked topology | Wake coverage | Current/complexity | Finding |
|---|---|---|---|---|
| A — rail-on deep sleep | ESP32 and CAN powered; peripherals off | CAN RX, timer, MODE, voltage, USB | Lowest additional parts; current depends on complete module rail | Viable and reliable; essentially the core of the recommendation |
| B — hardware-off plus always-on wake | Main rails off; battery CAN wake transceiver/controller remains | CAN/WAKE/voltage can assert regulator; timer/button need extra AON logic | Lowest main leakage but needs TCAN1043A-Q1-class 5 V/VIO/INH architecture or another AON controller and more sequencing | Not justified for v1 while the doubled rail-on envelope is 0.408 mA |
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
- `USB_DEBUG`: native USB and the `MAIN_3V3` core are enabled. USB alone never powers TCAN3404, AUX5 or any external 5 V output. Optional switched 3.3 V loads are default-off and may run only after firmware has established the applicable USB configured-current limit and can shed loads before the total reaches it.
- Low vehicle voltage does not select USB through an input-controller UV threshold. Vehicle MAIN-buck PGOOD controls the TPS2116 preference: loss of valid `VEH_3V3` selects `USB_3V3` when USB exists; without USB, the core resets or turns off cleanly. The later firmware contract prohibits MAX/full-white diagnostics and Wi-Fi below 10.0 V; below 9.5 V it starts a bounded SD flush and no new optional work; below 9.0 V for 100 ms it enters `LOW-VOLTAGE SHED` with AUX5/display/shift/sound/Wi-Fi off and no new SD writes; it exits only above 10.0 V stable for 2 s. These values remain prototype requirements pending ADC/front-end tolerance, real crank data and measured flush time.
- Wake/activity thresholds, debounce times, quiet timeout and diagnostic grace period remain unresolved policy values to be derived from logged vehicle behavior. The numeric low-voltage values above are explicit Task 5A.1 prototype requirements, not measured production limits.

## Automotive input review and requirements

### RejsaCAN v3.4 evidence

The repository single-sheet v3.4 schematic shows DSS34 D6 in series, MF-MSMF110/16-2 F1, SMF30A D4 from VCC to ground, and LMR14006X U4. That is evidence of a fuse/series-diode/TVS/buck concept, not proof of a coordinated automotive protection network. The PTC's `/16` voltage class, 30 V TVS clamping behavior and 40 V buck limit need full pulse/energy coordination; the reference circuit is not carried forward unchanged.

### Task 5A.1 condition-to-element selection

The former Task 4.7 `SM8SF24CA-Q` / `LM74502QDDFRQ1` / 60 V `DMT6007LFGQ-7` front end and `PMEG6030EP-Q` source-OR are **superseded historical candidates**. They must not be copied into Task 5A.1 capture. The updated conditional selection is:

| Electrical condition | Selected function / starting element | Remaining verification |
|---|---|---|
| Board/harness hard short | `0437002.WRA` 2 A fuse plus downstream protected branches | Fuse time-current/inrush/hot derating, copper/harness coordination and TVS-short clearing |
| Positive and negative transients | Common-anode anti-series unidirectional `SM15T47AY` + `SM15T33AY`; 47 V-leg cathode to fused VBAT, 33 V-leg cathode to `POWER_GND` | Polarity/orientation review, purchased pulse matrix, hot/repetitive clamp, leakage and fuse-energy tests |
| Reverse battery / off-state isolation | `LM74720QDRRRQ1` with two back-to-back `STL125N10F8AG` 100 V N-FETs | −14 V/60 s and fast-negative tests; FET VDS/VGS/SOA, recovery and reverse-current measurement |
| Suppressed load dump / OV | TVS network plus LM74720 OV/reverse-blocking stage | +38 V source test, protected-node waveform, automatic recovery and device-temperature margins |
| Fast positive transients / converter interaction | Short entry loop and selected post-disconnect damped filter | Source-impedance sweep, converter negative-impedance interaction, ringing, conducted/radiated EMI and thermal test |
| Cranking/brownout | Vehicle MAIN-buck PGOOD drives TPS2116 preference; staged reset/load shedding | Real crank waveforms, no reboot loop, USB handover, CAN/SD recovery and AUX5 low-voltage cutoff |
| ESD/conducted emissions | Interface-local ESD and controlled returns/filtering | ISO 10605/CISPR 25/applicable UNECE R10 plan; no compliance claim without testing |

Task 5A.1 retains a 6–18 V source envelope, but below 10 V only the bounded low-voltage policy applies and below 9 V for 100 ms the permitted steady state is CORE/SHED. Full-feature operation is not a 6 V requirement. No survival or compliance claim follows from a paper screen. Severe unsuppressed load dump remains outside the v1 guaranteed envelope unless a later purchased-standard/OEM requirement and surge-stopper review explicitly adds it. See [`task5a1-power-architecture.md`](task5a1-power-architecture.md), [`automotive-electrical-profile.md`](automotive-electrical-profile.md), and [`input-protection-architecture.md`](input-protection-architecture.md).

### Selected post-disconnect filter starting population

The conditional capture population, after the LM74720 disconnect FETs and before the vehicle MAIN/AUX converters, is:

```text
VBAT_SWITCHED -> CGA2B3X7R1H104M050BB 100 nF/50 V shunt
              -> CGA5L3X7R1H105K160AB 1 µF/50 V shunt
              -> XEL4030V-222MEC 2.2 µH -> FILTERED_VEHICLE
                                                                      |-> 4 × CGA6P3X7R1H475K250AB 4.7 µF direct
                                                                      |   + a fifth identical DNP footprint
                                                                      +-> WSL2512R3900FEA 0.39 Ω
                                                                          in series with EEH-ZC1H121P 120 µF
                                                                          to POWER_GND
```

The four direct MLCCs have a required aggregate effective-capacitance acceptance window of 9.4–20.68 µF after the specified bias, temperature, tolerance and ageing treatment. The fifth footprint is DNP and is not counted. The 120 µF hybrid capacitor has `Cd,min = 96 µF`, so `Cd,min / Cf,max = 96 / 20.68 = 4.64`. With nominal 2.2 µH, nominal 0.39 Ω and 28 mΩ maximum ESR, the simple pre-lab screen is `f0 = 23.6–35.0 kHz` and `Q = 0.78–1.16`. Those are nominal screens, not bounds. Including ±20% L, ±1% Rd and 0–28 mΩ ESR gives the full corner screen `f0 = 21.54–39.13 kHz` and `Q = 0.69–1.37` (`CALCULATED`).

Those ratios are capture checks, not a stability or EMI proof. At 25.531 V, an ideal hard step screens to about 1.67 kW initial resistor power and 46.9 mJ stored energy. The acceptance gate must therefore pass the exact WSL2512 point through the manufacturer pulse tool/model and test, and measure populated impedance over voltage/temperature, source and harness impedance, converter operating modes and load steps; it must also confirm MLCC effective capacitance, hybrid leakage/ESR, hot-plug/inrush, ringing and conducted/radiated emissions. The 0.39 Ω/120 µF branch is a selected starting value and may be tuned only from recorded lab evidence.

### Corrected load and low-voltage contracts

The maximum named simultaneous case uses the 5 V display: the historical vehicle-source “MAIN” bucket is 0.750 A at 3.3 V and AUX5 is 1.16001 A at 5 V, for 8.27505 W total. The alternative 3.3 V-display capacity case makes the vehicle-source MAIN bucket 1.050 A (1.313 A with a 25% sizing margin) and totals 6.765 W; it is never added to the 5 V-display case. That historical MAIN bucket includes the direct `VEH_3V3` TCAN branch even though physical `MAIN_3V3` is after TPS2116 and excludes TCAN. A USB-only budget must therefore be rebuilt from actual core loads rather than copied from the vehicle bucket.

AUX5 contracts are STREET 0.56001 A continuous, TRACK 0.92001 A continuous, and MAX 1.16001 A for no more than 10 s and 25% of any rolling 60 s. The 1.450 A value is sizing margin, not an operating load. `LOW-VOLTAGE SHED` requires AUX5 = 0 A. These are firmware-controlled prototype contracts; efficiency, load-step, dropout and board/junction temperature still require measurement.

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
| OBD present, USB absent | Vehicle MAIN buck produces `VEH_3V3`; valid PGOOD selects TPS2116 VIN1, which powers `MAIN_3V3`. TCAN3404 and state-selected vehicle peripherals may operate. TPS2116 isolates VIN2, so USB VBUS is not driven. |
| OBD absent, USB present | TPS2553 and TPS62162 produce `USB_3V3`; TPS2116 VIN2 powers `MAIN_3V3`. TCAN3404 VCC and AUX5 stay unpowered because they are outside the mux on the vehicle-only side. Optional 3.3 V loads are default-off pending the USB budget. |
| OBD and USB present | Valid vehicle-buck PGOOD selects VIN1; VIN2 remains energized but reverse-current-blocked. Loss of PGOOD selects VIN2 independently of raw-vehicle UV thresholds. TCAN3404/AUX5 follow the vehicle rail and turn off during an invalid-vehicle interval. Handover droop and reverse-current peaks must be measured. |
| Neither present | All rails off. |

Preserve ESP32-S3 native USB on GPIO19/20 and use `USBLC6-2SC6Y` connector-local D+/D− ESD plus independent Type-C sink CC pull-downs. The power chain is `VBUS -> TPS2553QDBVRQ1 -> TPS62162QDSGRQ1 -> USB_3V3 -> TPS2116DRLR VIN2`; there is no Schottky path to the vehicle input. Populate TPS2553 `RILIM` with 60.4 kΩ ±1% (`CRCW060360K4FKEA`); TI's equations screen a 387.2–491.3 mA fault/current-limit population band, not a load contract. The prototype startup source and cable must advertise and sustain at least 500 mA at a 4.75 V connector voltage. After ramp and before configuration, `USB_ENUM` is at most 100 mA total on `MAIN_3V3`, approximately 82 mA steady at VBUS under the documented screen; it is not an inrush ceiling. Configured use remains provisionally limited to 350 mA VBUS steady and 400 mA `MAIN_3V3`, with MAX prohibited. These requirements do not establish generic legacy USB 2.0 pre-enumeration compliance.

TPS2116 starting control is `MODE = VEH_3V3`, with PR1 pulled up to `VEH_3V3` through 1 MΩ, down to ground through 2 MΩ, and held low by the vehicle MAIN-buck PGOOD open-drain output while that rail is invalid. This gives vehicle priority only after a valid vehicle 3.3 V rail and leaves USB selection independent of the LM74720 OV function. The values, thresholds, ramp sequence, brownout chatter and handover timing are conditional until a prototype passes all four source states, slow ramps and repeated connect/disconnect tests.

TPS2116 provides the source-mux reverse-current barrier and LM74720 provides the vehicle-input reverse-current barrier. TCAN3404 VCC and AUX5 never connect to `MAIN_3V3`; firmware must keep TXD/STB benign when VCC is absent, and unpowered injection must be measured. USB presence is never permission to transmit CAN. The exact conditional TPS62162 population is `CGA2B3X7R1H104M050BB` 100 nF at TPS2553 IN, `CGA6P1X7R1E106M250AC` 10 µF/25 V at TPS2553 OUT/TPS62162 VIN, `XFL3012-222MEC` 2.2 µH, and `CGA6M3X7R1C106K200AB` 10 µF/16 V at TPS62162 OUT/TPS2116 VIN2. Populate `EEEFK0J101AV` 100 µF/6.3 V directly at TPS2116 VOUT. Effective capacitance, enable/PG sequencing, hold-up/clamp behavior, current-limited startup and load remain validation gates; GCT USB4105 remains provisional.

## Decision table

| Item | RejsaCAN v3.4 | Telemetry v1 proposal | Reason | Confidence |
|---|---|---|---|---|
| Input protection | DSS34 + MF-MSMF110/16-2 + SMF30A | `0437002.WRA`; common-anode `SM15T47AY` + `SM15T33AY`; `LM74720QDRRRQ1`; two `STL125N10F8AG`; selected damped filter | Coordinated positive/negative clamp screen, reverse isolation and post-disconnect damping | Conditional capture selection; full pulse/SOA/thermal/EMI validation pending |
| Vehicle MAIN regulator | LMR14006X, 600 mA class | `LMQ66420MC3RXBRQ1` -> `VEH_3V3`; shared exact `XGL5030-222MEC`/TDK CGA population | Vehicle core/CAN supply and low-IQ parked operation | Exact capture population selected; effective-capacitance/thermal/low-voltage tests pending |
| USB regulator and source selection | Schottky from VBUS into common input | `TPS2553QDBVRQ1` -> `TPS62162QDSGRQ1` -> `USB_3V3`; `TPS2116DRLR` mux after conversion | True source separation without coupling USB choice to vehicle UV thresholds | Conditional control network; four-state, handover and catalog-part environment tests pending |
| Secondary regulator(s) | MT9700 switched 3.3 V load switch | TPS22919QDCKRQ1 switched rails plus vehicle-only LMQ66420 AUX5 | Isolation, inrush/fault control and 5 V external compatibility | Silicon selected; managed-load/thermal policy pending proof |
| Always-on rail | Common 3.3 V only while buck held | `MAIN_3V3` core from TPS2116; vehicle parked source is `VEH_3V3`; peripherals independently off | CAN/timer/button wake with 0.408 mA doubled parked envelope | Paper pass pending measurement |
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
| USB power isolation | Schottky from VBUS into VCC/enable path | Separate USB buck plus TPS2116 RCB mux; TCAN3404/AUX5 vehicle-only | Supports all four cases with defined barriers and no intended DC backfeed path | Conditional pending four-state/abnormal bench proof |

## Unresolved items and validation gates

- Validate the selected SM15T47AY/SM15T33AY orientation and pulse sharing, LM74720 OV/reverse-current behavior, 100 V FET VDS/VGS/SOA, fuse clearing/inrush and every purchased-standard/OEM transient before release.
- Validate the complete post-disconnect filter impedance and damping over MLCC bias/temperature/ageing, hybrid-capacitor ESR/leakage, source/harness impedance, converter mode/load, resistor pulse heating and conducted/radiated EMI. Tune 0.39 Ω/120 µF only from recorded evidence.
- Validate the shared exact LMQ population (`XGL5030-222MEC`; 2 × `CGA6P1X7R1N106K250AC` CIN; 4 × `CGA6P3X7R1E226M250AB` COUT plus one DNP; 2 × `CGA3E1X7R1C105K080AC` CVCC; CBOOT DNP) for effective capacitance, loss, stability, thermal/EMI, crank and transient headroom; prove low-voltage AUX5 shedding and clean vehicle-only reset before schematic release.
- Confirm GPIO13 CAN wake, GPIO8 vehicle-activity wake, and all power-off signal isolation in an ESP32-S3 prototype.
- Select the exact active antenna, production card/socket, physical display connector/module adapter, shift pixel, speaker and status LED; replace remaining envelopes with maximum data and measurements.
- Validate GNSS 25 Hz configuration/message set and interference with the buck, ESP RF, SPI and external cables.
- Finalize connector families/pin numbering, cable construction, USB-shell population and environmental ratings; the one-plane ground architecture is frozen.
- Bench all four USB/vehicle source combinations, slow ramps, brownouts and abnormal connections. Confirm the ≥500 mA/4.75 V prototype startup-source contract, post-ramp ≤100 mA `USB_ENUM` load, PGOOD/PR1/MODE sequencing, acceptable `MAIN_3V3` droop, no sustained or unsafe crossover current into USB VBUS or OBD battery, no TCAN/AUX5 phantom power, and configured-current load shedding. This gate does not claim generic legacy USB 2.0 pre-enumeration compliance.
- Qualify the catalog TPS2116 temperature/lifetime/environment fit for this product or select a proven qualified alternative before production release.

## Primary sources

- Repository: `Schematics/RejsaCAN v3.x (ESP32-S3 based board)/RejsaCAN v3.4 - Schematic.json` and matching single-sheet PNG; U2, U3, U4, D4, D6, F1 and related nets/components.
- TI, [SN65HVD230 Data Sheet SLLS500K](https://www.ti.com/lit/ds/symlink/sn65hvd230.pdf).
- TI, [TCAN3404-Q1/TCAN3403-Q1 Data Sheet SLLSFQ6A](https://www.ti.com/lit/ds/symlink/tcan3404-q1.pdf), Tables 7-1/7-2 and §§8–9.
- TI, [TCAN1043A-Q1 Data Sheet SLLSF27](https://www.ti.com/lit/ds/symlink/tcan1043a-q1.pdf), supply-current tables and mode descriptions.
- STMicroelectronics, [ESDCAN04-2BWY product information](https://www.st.com/en/protections-and-emi-filters/esdcan04-2bwy.html).
- STMicroelectronics, [SM15T47AY](https://www.st.com/en/protections-and-emi-filters/sm15t47ay.html) and [SM15T33AY](https://www.st.com/en/protections-and-emi-filters/sm15t33ay.html); TI, [LM74720-Q1](https://www.ti.com/product/LM74720-Q1), [TPS2116](https://www.ti.com/product/TPS2116), [TPS2553-Q1](https://www.ti.com/product/TPS2553-Q1), and [TPS62162-Q1](https://www.ti.com/product/TPS62162-Q1). The former Bourns SM8SF-Q and TI LM74502-Q1 sources are retained only in historical Task 4.7 records.
- STMicroelectronics, [LDP01-xxAY automotive load-dump TVS data sheet](https://www.st.com/resource/en/datasheet/ldp01-28ay.pdf), rejected unidirectional pre-controller candidate.
- STMicroelectronics, [TN1517 TVS FAQ](https://www.st.com/resource/en/technical_note/tn1517-faq-tvs-stmicroelectronics.pdf).
- Coilcraft, [XEL4030V high-voltage inductor family](https://www.coilcraft.com/en-us/products/power/high-voltage-inductors/xel/xel4030v/); Vishay, [WSL Power Metal Strip resistor family](https://www.vishay.com/en/product/30100/).
- ISO, [ISO 7637-2:2011](https://www.iso.org/standard/50925.html), [ISO 16750-2:2023](https://www.iso.org/standard/76119.html), and [ISO 10605:2023](https://www.iso.org/standard/79094.html) scope pages; IEC [CISPR 25:2021](https://webstore.iec.ch/en/publication/64645).
- Espressif, [ESP32-S3-WROOM-1/1U Data Sheet v1.8](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf).
- u-blox, [NEO-M9N-00B Data Sheet R08](https://content.u-blox.com/sites/default/files/NEO-M9N-00B_DataSheet_UBX-19014285.pdf) and [Integration Manual R10](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf).
- GalaxyCore, [GC9A01A Data Sheet v1.0 preliminary](https://buydisplay.com/download/ic/GC9A01A.pdf).
- Worldsemi, [WS2812B-2020 Data Sheet v1.3](https://cdn-shop.adafruit.com/product-files/4684/4684_WS2812B-2020_V1.3_EN.pdf).
