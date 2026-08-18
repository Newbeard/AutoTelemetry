# AutoTelemetry v1 input and interface protection architecture

Status: Task 5A.1 conditional pre-schematic re-freeze, 2026-08-17. This document controls prototype schematic intent only. It is not a tested design, production release, or automotive/standards-compliance claim. The reproducible source data, equations, filter population, source-mux details, tolerances, availability snapshots, and laboratory gates are authoritative in [`task5a1-power-architecture.md`](task5a1-power-architecture.md).

The earlier Task 5A `BLOCKED` status is superseded for the coordinated vehicle-input and source-selection architecture below. Capture remains conditional on copying the exact Task 5A.1 values and preserving its validation gates; PCB and manufacturing release are not authorized.

## Task 5A.1 conditionally re-frozen vehicle-input chain

```text
OBD16 / VBAT_OBD_RAW
  -> 0437002.WRA, 2 A / 63 V fixed fuse
  -> VBAT_FUSED_CLAMPED
       +-> SM15T47AY cathode at fused VBAT, anode at TVS_MID
       +-> SM15T33AY anode at TVS_MID, cathode at POWER_GND
           (common-anode anti-series pair)
  -> LM74720QDRRRQ1 + 2 × STL125N10F8AG back-to-back N-MOSFETs
       OV top = 249 kΩ + 249 kΩ series, bottom = 28.0 kΩ, all 0.1%
  -> VBAT_SWITCHED
  -> damped post-switch filter exactly as controlled by task5a1-power-architecture.md
  -> FILTERED_VEHICLE
       +-> LMQ66420MC3RXBRQ1 -> VEH_3V3
       +-> conditionally frozen LMQ66420MC5RXBRQ1 -> AUX5
```

The LM74720-Q1 actively drives both N-MOSFETs and provides true reverse-current blocking while enabled. The 100 V FET class is required by the negative-pulse/held-output stress screen. The TVS pair and fuse must be at the connector-entry power region with the shortest practical high-current loop to `POWER_GND`; protected loads must not branch from the connector side of that loop.

## Reverse-polarity architecture comparison

| Architecture | Normal-path behavior | Parked/USB implications | Disposition |
|---|---|---|---|
| Series Schottky | Zero controller IQ but material voltage and thermal loss | Blocks reverse polarity but does not provide the selected OV disconnect/RCB behavior | `REJECTED` for the vehicle path |
| P-channel MOSFET | Lower loss than a diode but higher resistance than the selected N-FET pair | Still needs qualified negative-gate and OV control | `REJECTED` |
| LM74720-Q1 plus back-to-back 100 V N-FETs | Low conduction loss, programmable native OV, reverse-polarity protection, and true RCB while enabled | Prevents a powered downstream rail from driving OBD16; controller plus OV divider receives a 60 µA design allocation at 12 V | **`CONDITIONAL PROTOTYPE FREEZE`** |
| Full surge-stopper/eFuse front end | Can address a different unsuppressed envelope but adds sense/pass-element SOA, loss, IQ, and thermal work | Does not remove the need for an upstream physical fuse | `REJECTED` for the stated v1 pulse envelope |

### Controller and OV threshold

`LM74720QDRRRQ1` is AEC-Q100 qualified and replaces the earlier controller. TI specifies 27 µA typical and 35 µA maximum operating current; the selected divider draws

```text
IDIV(12 V) = 12 V / (249 kΩ + 249 kΩ + 28.0 kΩ)
            = 22.814 µA                         [CALCULATED]
```

The controller-plus-divider result is 49.814 µA typical and 57.814 µA maximum at 12 V; use 60 µA as the `DESIGN_REQUIREMENT` allocation before any separate vehicle-present monitor.

With all three divider resistors at 0.1% and an explicit ±1 µA OV-node/PCB-leakage model, Task 5A.1 calculates:

| Transition | Modeled input range | Design consequence |
|---|---:|---|
| OV rising / PD-low command | 20.690–25.531 V | Remains on through 18 V and commands opening before the defined +26 V/60 s event reaches 26 V; completed isolation/output peak is dynamic |
| OV falling / reconnect eligibility | 18.815–23.366 V | Returning to 18 V permits reconnect; completion, inrush and ring time require capture |

This intentionally relaxes the former guarantees of cutoff by 20 V and a protected node no higher than 24 V. Every downstream raw-input rating and dynamic overshoot must instead be reviewed against the 25.531 V modeled cutoff endpoint plus measured parasitics. No separate 18–20 V precision supervisor is retained. A separate raw-VBAT monitor, if fitted, is only a vehicle-present/CAN-gating function and is not part of protection.

## Fuse and load policy

`0437002.WRA` is conditionally retained: Littelfuse 437A, AEC-Q200, 2 A, 63 V, fast acting, 1206. The corrected simultaneous named rail peak is:

```text
PMAIN = 3.3 V × 0.750 A   = 2.47500 W
PAUX  = 5.0 V × 1.16001 A = 5.80005 W
PPEAK = 8.27505 W                         [CALCULATED]

simple IIN(MAX at 10 V) = PPEAK / (10 V × 0.80)
                         = 1.034 A        [80% efficiency ASSUMPTION]
```

The 0.750 A `MAIN` quantity is the complete vehicle-source 3.3 V conversion bucket in the maximum simultaneous 5 V-display case and includes the direct vehicle-only TCAN branch. A separate 3.3 V-display case loads the MC3 output to 1.050 A/3.465 W and produces 6.765 W total rail power; 1.313 A is sizing margin, not an operating current. The 1.050 A case controls MC3 thermal validation. Physical `MAIN_3V3` is downstream of TPS2116 and excludes CAN; do not copy the whole vehicle bucket into a USB-only budget.

The Task 5A.1 mode/efficiency model gives 0.9064 A nominal and 0.9545 A with its sensitivity allowance for MAX at 10 V; the simple 1.034 A envelope above is more conservative. MAX is prohibited below 10 V. Below 9.0 V for 100 ms the system enters LOW-VOLTAGE SHED/CORE, whose nominal 6 V input is 0.1211 A. The 6 V TRACK values, 1.0535 A nominal and 1.1102 A with sensitivity, are arithmetic/pre-shed transient screens only, not a permitted steady mode. All remain below Littelfuse's 1.60 A 25 °C guidance and 1.36 A 75 °C example on paper. Measured efficiency, temperature, timers, and load-shedding behavior remain gates; the fuse is not a normal electronic current limiter.

The −100 V/10 Ω/2 ms reference screen gives approximately 0.0774–0.0840 A²s for the selected TVS pair, versus the fuse data sheet's 0.144 A²s nominal melting value. That comparison is encouraging but does not guarantee fuse survival across tolerance, temperature, repetition, waveform, or ageing. Time-current/inrush, PCB/harness fault clearing, interrupt applicability, and prototype pulse tests remain mandatory. A PPTC, fusible resistor, or eFuse does not replace this physical fuse.

## Coordinated TVS pair and transient intent

| Part and role | Qualification/package | Manufacturer data used by Task 5A.1 | Conditional decision |
|---|---|---|---|
| `SM15T47AY`, positive leg | AEC-Q101, SMC | `VRM=40.2 V`; `VBR=44.7–49.4 V`; `VC=64.5 V at 23.2 A`; 1.5 kW 10/1000 µs | Cathode to fused VBAT; stays below avalanche for normal 18 V, +26 V/60 s, and defined +38 V source |
| `SM15T33AY`, negative leg | AEC-Q101, SMC | `VRM=28.2 V`; `VBR=31.4–34.7 V`; `VC=45.7 V at 33 A`; 1.5 kW 10/1000 µs | Cathode to ground; avoids sustained conduction at −14 V/60 s and reduces negative-pulse/fuse current |

Both devices specify 0.2 µA maximum leakage at 25 °C and 1 µA maximum at 85 °C. They are used in the exact common-anode orientation shown above; two independently oriented unidirectional TVSs are not interchangeable with one bidirectional symbol.

| Input condition | Public-data/calculation conclusion | Evidence still required |
|---|---|---|
| 12/14.4/18 V | Both TVSs remain below standoff; LM74720 path is enabled and reverse-current blocking is active | Loss, temperature, inrush, and RCB measurement |
| +26 V/60 s | OV commands PD low by the modeled 25.531 V endpoint; the positive TVS remains below its 40.2 V standoff | Completed isolation, threshold distribution, downstream overshoot, and 60 s recovery test |
| Defined suppressed +38 V source | Positive TVS remains below 40.2 V standoff; the steady/post-response intent is an open path, but a fast edge crosses OV and gate delay | Exact duration/source/repetition and A/C/PD/VDS/VGS/output measurement |
| +50 V/2 Ω fast reference | Source itself bounds the raw node to 50 V; Task 5A.1's piecewise TVS screen is conditional | Dynamic clamp, trace inductance, energy, and temperature |
| −100 V/10 Ω/2 ms fast reference | Piecewise screen gives about 6.223–6.479 A and 0.0774–0.0840 A²s | Actual forward drop, overshoot, pulse sharing, fuse, controller, and FET waveforms |
| −14 V/60 s reverse | 28.2 V negative-leg standoff prevents sustained TVS avalanche; LM74720/FETs block | Hot/cold reverse-current and 60 s recovery test |

ST publishes only typical forward-voltage curves for this use. Task 5A.1 therefore treats 45.7 V plus 3.5 V as a deliberately conservative 49.2 V engineering stress envelope, not a guaranteed pair clamp. With the downstream node held at the allowed 25.531 V endpoint, the corresponding screen is 74.731 V across the controller C–A/FET isolation path: 10.269 V to the LM74720 85 V C–A absolute maximum and 25.269 V to a 100 V FET, before layout overshoot. These are prototype-screen margins, not compliance proof.

## OV recovery, low voltage, and protected maximum

- No static OV trip through 18 V and reconnect eligibility after +26 V are guaranteed only within the stated resistor and ±1 µA leakage model; completion time is not.
- Low-voltage operation uses controlled reset plus the fuse/load-shedding policy; it is no longer coupled to a USB/source-crossover threshold.
- USB/vehicle selection occurs after independent 3.3 V conversion, so a raw-input UV divider is not used to decide whether USB may supply the core.
- After OV/reverse events, reconnect is automatic only after hysteresis and source stability. Reconnection never authorizes CAN transmission.
- The former `≤20 V` cutoff and `≤24 V` protected-node limits are superseded. Prototype tests must establish actual `FILTERED_VEHICLE` overshoot and downstream margin.

## Input filter gate

The damped filter remains after the disconnect MOSFETs so stored energy cannot discharge directly into OBD16 and OV turn-off isolates the bulk. Exact L/C/R values, order codes, leakage, DC-bias derating, inrush, impedance/stability analysis, and layout constraints are controlled only by [`task5a1-power-architecture.md`](task5a1-power-architecture.md). Do not copy the superseded Task 4.7 2.2–4.7 µH/47–100 µF envelope or invent alternate values in schematic capture.

## Ground architecture

- OBD4 and OBD5 arrive on separate rated conductors and join once at the connector-entry `POWER_GND` region.
- Use one continuous ground plane. `POWER_GND`, digital return, CAN transceiver ground, GNSS/RF reference, SD ground, and USB signal ground are the same DC net; layout controls return paths without split-plane slots.
- TVS, input capacitors, and fuse-fault returns use the entry/high-current region and do not flow under GNSS RF or ESP antenna keep-outs.
- CAN TVS returns at connector entry; CANH/L remain a short paired stub.
- GNSS ESD returns immediately to the continuous RF ground beside U.FL. The bias supply return does not share a narrow path with the RF ESD impulse.
- Display and SHIFT5 each receive dedicated connector ground contacts/wires rated for their loads. Their high-current returns rejoin near the power-entry/power-conversion region, away from GNSS.
- TPA2005D1-Q1 is BTL: neither speaker terminal is ground. Its PowerPAD and supply-decoupling return use local copper tied to the plane.
- USB GND joins board ground directly. The USB shell gets a connector-local configurable 0 Ω/R-C/capacitive footprint; final population follows ESD/EMC testing. There is no isolated chassis net in v1.
- Remove the misleading `CAN_QUIET_GND` concept: optional split-termination capacitance returns to the same plane through the shortest local path.

## Vehicle and USB source coexistence

```text
USB-C D+/D- -> USBLC6-2SC6Y I/O pass-through -> ESP32-S3 USB D+/D-
USB-C VBUS -> USB_VBUS_RAW
                  +-> USBLC6-2SC6Y VBUS shunt/reference pin
                  +-> TPS2553QDBVRQ1 -> USB5_PROTECTED
                      -> TPS62162QDSGRQ1 fixed 3.3 V -> USB_3V3 -> TPS2116DRLR VIN2
USB-C CC1/CC2 -> separate USB Type-C sink terminations

FILTERED_VEHICLE -> LMQ66420MC3RXBRQ1 -> VEH_3V3 -> TPS2116DRLR VIN1
                                                 -> TPS2116 VOUT -> MAIN_3V3

FILTERED_VEHICLE -> conditionally frozen LMQ66420MC5RXBRQ1 -> AUX5
VEH_3V3 -----------------------------------------------------> CAN VCC
```

| OBD | USB | Required result |
|---|---|---|
| OFF | OFF | All rails off; only passive ESD/CC structures present |
| ON | OFF | Vehicle supplies `VEH_3V3`, CAN VCC, MAIN_3V3, and policy-permitted AUX5 loads |
| OFF | ON | TPS2553/TPS62162 supplies only `USB_3V3` and the muxed MAIN_3V3/core; CAN VCC and AUX5 remain off and the LM74720/FET path blocks USB-to-OBD current |
| ON | ON | TPS2116 selects the vehicle-priority input and isolates the two regulated 3.3 V sources; verify transition droop and reverse current in both directions |

`TPS2553QDBVRQ1` uses `CRCW060360K4FKEA`, 60.4 kΩ ±1%, for `RILIM`. TI Section 8.5 equations at 59.796–61.004 kΩ screen a 387.2–491.3 mA fault/current-limit population band, not an operating contract. The prototype startup source/cable must advertise and sustain at least 500 mA at 4.75 V. After ramp, `USB_ENUM` is at most 100 mA total on `MAIN_3V3` (about 82 mA steady at VBUS under the documented screen); configured operation is provisionally capped at 350 mA VBUS steady and 400 mA `MAIN_3V3`, with MAX prohibited. The 100 mA contract is not an inrush ceiling and does not establish generic legacy USB 2.0 pre-enumeration compliance. Verify source/cable droop, current-limited startup and every load state. The former `PMEG6030EP-Q` raw source-OR is superseded.

USB-only behavior:

- MAIN_3V3 powers the core development domain; optional GNSS/SD operation must remain inside the verified TPS2553/TPS62162 and USB configured-current budget.
- TCAN3404-Q1 VCC is vehicle-only. Its bus pins must remain high impedance with OBD absent, and the board must remain non-transmitting.
- No AUX5, DISPLAY_5V, SHIFT5, or sound load is enabled.
- With both sources, USB data remains usable. Source presence never authorizes CAN transmission.

## Always-connected component voltage margins

`Protected max` values are design-node limits to be proven on the prototype.

| Component / node | Rating or modeled limit | Task 5A.1 stress screen | Static margin before overshoot | Status |
|---|---:|---:|---:|---|
| `0437002.WRA` fuse | 63 V rating; nominal melt `I²t=0.144 A²s` | Negative screen `0.0774–0.0840 A²s` | No voltage margin claimed: interrupt applicability depends on the prospective DC fault circuit | `CONDITIONAL PROTOTYPE FREEZE`; temperature/fault/pulse tests required |
| `LM74720QDRRRQ1` A pin | recommended −60…+65 V | +50 V source; −49.2 V engineering envelope | +15.0/−10.8 V to recommended endpoints | `CONDITIONAL PROTOTYPE FREEZE` |
| `LM74720QDRRRQ1` C–A | 85 V absolute maximum | 74.731 V with output held at 25.531 V | 10.269 V | Lab overshoot gate |
| `STL125N10F8AG` pair | 100 V VDS | 74.731 V held-output screen | 25.269 V | `CONDITIONAL PROTOTYPE FREEZE`; measure each FET |
| `LMQ66420MC3RXBRQ1` VIN/EN | 42 V absolute, 36 V recommended | 25.531 V modeled OV-open endpoint plus dynamic overshoot | 16.469 V absolute / 10.469 V recommended before overshoot | `FROZEN silicon`; dynamic proof required |
| `LMQ66420MC5RXBRQ1` VIN/EN | 42 V absolute, 36 V recommended | 25.531 V modeled OV-open endpoint plus dynamic overshoot | 16.469 V absolute / 10.469 V recommended before overshoot | `CONDITIONAL PROTOTYPE FREEZE` |
| `TPS22919-Q1` | 6 V absolute | 5.5 V rail maximum | 0.5 V | `FROZEN`; regulate AUX5 tolerance accordingly |
| `TPA2005D1TDGNRQ1` active supply | 6 V absolute | 5.5 V rail maximum | 0.5 V | `FROZEN direction`; use T-suffix −40…+105 °C order code |
| `LP5814DRLR` | 6 V absolute | 3.6 V MAIN_3V3 maximum | 2.4 V | `APPROVED`; not AEC-qualified |

The margin table does not replace pin-specific injection-current, transient duration, temperature, repetitive-stress, layout-inductance, or SOA checks. Exact filter capacitor ratings and stress are controlled by `task5a1-power-architecture.md`.

## CAN protection freeze

- `TCAN3404DRQ1` remains frozen: AEC-Q100, 3.3 V, 17 µA maximum standby, WUP indication on RXD, ±58 V bus-fault standoff, ±30 V common-mode, and high-impedance unpowered bus pins (`VERIFIED_DATASHEET`).
- `ESDCAN04-2BWY` remains frozen at the OBD connector: AEC-Q101, 19 pF maximum per line, 25.5 V stand-off, 43 V maximum clamp at 3 A, and 0.05 µA maximum leakage at 25 °C (`VERIFIED_DATASHEET`).
- `ACT45B-510-2P-TL003` footprint remains frozen and **DNP by default**, with 0 Ω bypasses. Populate only if EMC/SI tests justify it.
- Optional split 120 Ω termination remains **DNP/OFF by default**. An OBD stub must not become a third terminator; bench population is allowed only when the board is a physical endpoint.
- Protection precedes the choke footprint. Keep CANH/L paired and short, avoid large test-pad stubs, and validate ESD, fast transients, ground offset, dominant fault, unpowered loading, wake, listen-only, and bounded diagnostic transmit.

## GNSS antenna protection freeze

`PROPOSED CHANGE`: replace NRND `ESDAXLC6-1BT2Y` with active `AQ3118E-01ETG` from Littelfuse.

The selected part is AEC-Q101/PPAP-capable, bidirectional, 18 V stand-off, 0.3 pF typical, 1 nA typical leakage at 18 V, and is characterized for ISO 10605 330 pF/330 Ω at ±25 kV contact and ±28 kV air (`VERIFIED_DATASHEET`). It is placed immediately at the U.FL center contact with the shortest possible ground return. Its insertion loss and complete active-antenna RF path still require VNA/C/N0 measurement; do not route RF before layout review.

The u-blox R10 active-antenna bias-T topology remains frozen: controlled 50 Ω path, 100 nF supply filter/DC element and 27 nH choke meeting the published RF impedance/current guidance. The exact antenna is `PROVISIONAL`. The short-current limiter must be fed from switched `GNSS_3V3`, not treated as an unlimited VCC_RF output, and must keep the module supply within its allowed dynamic impedance. The current-limit topology/value is reopened for Task 5 because the previous 22 Ω/136 mA calculation is not, by itself, proof of VCC_RF or antenna-short safety.

## Shift-light protection freeze

Preserve:

```text
GPIO6 -> CAHCT1G126QDCKRQ1 -> 33 Ω starting series footprint
      -> connector-side ESD -> SHIFT_DATA_5V

AUX5 -> TPS1H100BQPWPRQ1 -> SHIFT5
```

- `CAHCT1G126QDCKRQ1` remains frozen; its OE defaults disabled and only asserts with the safe SHIFT5 policy.
- `TPS1H100BQPWPRQ1` remains frozen; qualify 0.50 A continuous and retain a 1 A protected fault envelope. Exact current-limit resistor and short-circuit SOA are Task 5 calculations.
- Add connector-local two-line 5 V ESD protection for `SHIFT5` and `SHIFT_DATA_5V`; `PESD2USB5UVT-Q` is the `PROVISIONAL` AEC-Q101 candidate pending exact connector/harness and clamp review.
- Use local connector bulk/100 nF, ≤0.5 m cable, ≥26 AWG power/ground, ≥28 AWG data, and ≥1.5 A connector rating.
- Test output short to ground/battery-like miswire only with an approved current-limited bench setup. ESD, hot plug, reversed connector, data-to-power short, sustained output short, and real ten-pixel load are separate cases.

## Sound, status, and service inputs

- `TPA2005D1TDGNRQ1` is an `APPROVED DIRECTION` and exact high-temperature MSOP-PowerPAD order code. It is AEC-Q100 grade 2, 2.5–5.5 V operating, 1.4 W typical into 8 Ω at 5 V/10% THD, 2.8 mA quiescent, 0.5 µA shutdown, and has short/thermal protection. The 8 Ω, ≥1 W speaker remains `PROVISIONAL`. AUX5 is off parked, so amplifier shutdown leakage does not add to vehicle parked current.
- `LP5814DRLR` is `APPROVED`: 2.5–5.5 V, −40…+125 °C, 0.3 µA maximum shutdown at 3.6 V, I2C current/PWM control. It is catalog/not AEC-qualified; qualify it at board level. `STATUS_DRV_EN` has a hardware default off and the LED stays off parked. Exact common-anode RGB LED/optics are `PROVISIONAL`.
- MODE remains GPIO10 with the Task 4.6 short/long-press contract. It is a non-strap input behind a non-conductive actuator. GPIO0 BOOT and EN RESET remain recessed service inputs. If any metal actuator becomes touchable, reclassify the interface and add a low-leakage protector without changing strap/reset timing.
- Do not place a large capacitor or leaky clamp directly on GPIO0 or EN. Verify all three inputs during powered and unpowered ESD tests.

## Complete parked-current revalidation

The complete Task 5A.1 parked tree, including the selected filter and regulated-source mux, is controlled by [`task5a1-power-architecture.md`](task5a1-power-architecture.md). The obsolete Task 4.7 subtotal must not be carried into capture. The directly reproducible vehicle-front-end contributions at 12 V are:

| Always-connected vehicle-front-end path | Contribution | Basis |
|---|---:|---|
| LM74720-Q1 controller | 27 µA typical / 35 µA maximum | `VERIFIED_DATASHEET` operating current |
| OV divider | 22.814 µA | `12 V / 526 kΩ`, `CALCULATED` |
| Controller + divider | 49.814 µA typical / 57.814 µA maximum | `CALCULATED`; use 60 µA design allocation |
| SM15T47AY + SM15T33AY | 0.4 µA maximum at 25 °C; 2 µA maximum at 85 °C | Sum of the two data-sheet leakage limits |

A complete conservative 12 V subtotal is 204.013 µA; applying the 100% uncertainty/temperature allowance gives a 408.026 µA (0.408 mA) paper envelope. The doubled 6/8/10/12/14.4/18 V model is 0.476/0.438/0.419/0.408/0.402/0.400 mA. These are calculation results, not measured board limits.

A TPS3899 raw-VBAT monitor, if retained, is accounted separately as a vehicle-present/CAN-gate function; it does not set protection thresholds. With USB absent, the TPS62162 USB buck is not connected to raw vehicle input. Final acceptance still requires complete assembled-board current measurement over voltage and temperature, including filter leakage, converter/mux states, ESP32/PSRAM, CAN standby, pull networks, contamination, and source transitions.

## Pre-schematic thermal sanity check

Assumptions: 85 °C hot enclosure ambient; 50 °C/W effective buck junction-to-ambient with the required multilayer copper/vias; maximum simultaneous rail allocations; efficiency values are placeholders to be replaced by schematic loss calculations.

| Item | Calculation / concern | Result and requirement |
|---|---|---|
| Back-to-back STL125N10F8AG pair | Hot screen `2×4.6 mΩ×1.7=15.64 mΩ`; permitted conservative MAX at 10 V is 0.9545 A | Approximately 14.3 mW (`CALCULATED`); 6 V TRACK is only a ≤100 ms pre-shed arithmetic screen. Verify actual VGS, RDS(on), copper, temperature, and transient stress |
| SM15T47AY + SM15T33AY | No normal avalanche at 12–18 V; transient-only energy | Large thermal/ground copper; keep away from GNSS/ESP antennas; test hot/cold pulses, no compliance claim |
| MAIN vehicle buck | `3.3 V×0.750 A=2.475 W` in the maximum simultaneous 5 V-display case; separate 3.3 V-display case is `3.3 V×1.050 A=3.465 W` | `LMQ66420MC3RXBRQ1` remains frozen; 1.050 A controls thermal validation, while 1.313 A is sizing margin only. Shared exact population: `XGL5030-222MEC`; 2 × `CGA6P1X7R1N106K250AC` CIN; 4 × `CGA6P3X7R1E226M250AB` COUT plus one DNP; 2 × `CGA3E1X7R1C105K080AC` CVCC; CBOOT DNP. Validate losses, effective capacitance, copper and temperature. |
| AUX5 buck | STREET `0.56001 A`; TRACK `0.92001 A`; MAX `1.16001 A` | `LMQ66420MC5RXBRQ1` is conditional and uses the same exact shared population: MAX is limited to ≤10 s and ≤25% rolling 60 s; validate efficiency, effective capacitance, hot enclosure and each duty state. |
| TPS22919-Q1 | Use conservative 200 mΩ hot bound: at 0.6 A, `I²R=72 mW`; 0.4 A gives 32 mW | Provide local copper; verify voltage drop/thermal/current-limit and QOD behavior per branch |
| TPS1H100-Q1 | 0.50 A qualified load is modest, but a short forces linear/current-limit operation | Calculate RCL, timer/SOA, copper and repetitive fault behavior; locate away from GNSS RF |
| USB buck/mux | TPS2553-limited VBUS -> exact TPS62162 population -> TPS2116 with `EEEFK0J101AV` 100 µF/6.3 V at VOUT | Verify the ≥500 mA/4.75 V prototype source contract, post-ramp ≤100 mA `USB_ENUM`, current-limited startup/dropout, transition droop, reverse blocking and package temperatures; do not claim generic legacy USB 2.0 pre-enumeration compliance. |
| TPA2005D1-Q1 | `1 W/0.75−1 W = 0.333 W` loss at assumed 75% efficiency | PowerPAD soldered to ground copper/vias; keep BTL traces compact and away from GNSS; speaker opening/acoustics deferred |
| Fuse | MAX at 10 V is 0.9064 A nominal/0.9545 A sensitivity; below 9 V for 100 ms the permitted steady state is CORE/SHED, 0.1211 A nominal at 6 V | Conditionally retain 2 A; 6 V TRACK 1.0535/1.1102 A is transient arithmetic only. Validate shedding, insertion, inrush, pulses, ageing, and downstream fault clearing |

This is not a thermal simulation. Actual efficiency curves, switching frequency, inductor/core/copper loss, board stackup, enclosure, airflow, duty cycle, and simultaneous-load policy determine the result.

## Frozen net names

| Domain | Frozen names |
|---|---|
| Raw vehicle | `VBAT_OBD_RAW`, `POWER_GND`, `VBAT_FUSED_CLAMPED`, `TVS_MID`, `VBAT_SWITCHED`, `FILTERED_VEHICLE`, `VEHICLE_PRESENT`, `PWR_FAULT_N` |
| Source conversion/mux | `USB_VBUS_RAW`, `USB5_PROTECTED`, `USB_3V3`, `VEH_3V3`, `MAIN_3V3`, `AUX5`, `MAIN_PGOOD`, `AUX5_EN`, `AUX5_PGOOD` |
| Switched rails | `GNSS_3V3`, `SD_3V3`, `DISPLAY_3V3`, `DISPLAY_5V`, `SHIFT5`, `SOUNDER5` |
| CAN | `CANH_OBD`, `CANL_OBD`, `CANH_PHY`, `CANL_PHY`, `CAN_RX`, `CAN_TX`, `CAN_STB` |
| GNSS | `GNSS_RF`, `ANT_BIAS`, `GNSS_RX`, `GNSS_TX`, `GNSS_TIMEPULSE`, `GNSS_RESET_N` |
| External/control | `SHIFT_DATA_5V`, `SHIFT5_EN`, `SHIFT_FAULT_N`, `MODE_N`, `STATUS_DRV_EN`, `USB_PRESENT` |

There is no separate `AON_3V3`: muxed `MAIN_3V3` is the always-on core rail. `VEH_3V3` and `USB_3V3` remain distinct mux inputs; CAN VCC and AUX5 are vehicle-only.

## First-prototype test points

| Test point | Required | Constraint |
|---|---|---|
| `VBAT_OBD_RAW` | Yes | Before fuse; guarded/labelled for controlled bench use |
| `VBAT_FUSED_CLAMPED` | Yes | Measure TVS clamp; high-voltage spacing |
| `VBAT_SWITCHED` | Yes | Verify MOSFET turn-on/off and inrush |
| `FILTERED_VEHICLE` | Yes | Measure post-filter overshoot, ripple, and disconnect behavior |
| `POWER_GND` | Multiple compact points | One near power entry and one low-current instrument point |
| `VEH_3V3`, `USB_3V3`, `MAIN_3V3`, `AUX5` | Yes | Rail/mux ripple, transition, thermal, and load-step probes |
| `MAIN_PGOOD`, `AUX5_EN`, `AUX5_PGOOD` | Yes | Small logic pads |
| `CANH_OBD`, `CANL_OBD` | Yes | Compact paired pads; no long stubs |
| `CAN_RX`, `CAN_TX`, `CAN_STB` | Yes | Logic verification/listen-only evidence |
| `GNSS_3V3`, `GNSS_TX`, `GNSS_RX` | Yes | Keep away from RF path |
| `GNSS_TIMEPULSE`, `GNSS_RESET_N` | Yes | Small pads, no RF stub |
| `SHIFT5`, `SHIFT_DATA_5V`, `SHIFT_FAULT_N` | Yes | Load/fault/hot-plug test access |
| `USB_VBUS_RAW`, `USB5_PROTECTED`, `USB_PRESENT` | Yes | Verify all four source states; do not add USB D+/D− stubs |
| Individual load-switch outputs | Measurement links preferred | 0 Ω/current-shunt options may replace large test points |

Do not add a GNSS RF test pad. Use the U.FL/VNA fixture and an approved RF coupon/fixture if needed.

## Prototype validation plan

Bench work uses current-limited supplies, appropriate transient/ESD equipment, differential probes, fusing, thermal monitoring, and an approved safety procedure. Ordinary bench tests do not establish standards compliance.

1. Measure parked current at 12 V/25 °C, then over the approved parked voltage and temperature range; separate controller/divider, TVSs, vehicle buck, mux, ESP, CAN, expander, filter, sensing, and leakage paths.
2. Measure active current for named typical and maximum-load states; verify fuse and source copper temperature.
3. Sweep low input and OV slowly and dynamically; record load shedding, reset, 20.690–25.531 V turn-off, 18.815–23.366 V reconnect, hysteresis, chatter, `FILTERED_VEHICLE` overshoot, and repeated recovery.
4. Apply approved warm/cold-crank waveforms; verify controlled reset, no reboot loop, SD recovery, CAN passive state, and staged peripheral restart.
5. Perform a current-limited −14 V reverse-polarity test and approved fast positive/negative transient tests; confirm no fuse opening from normal reverse connection and no damage.
6. Measure the TVS current/clamp at hot and cold starts under the selected pulse matrix; inspect component and copper temperature. Stop before unapproved destructive severity.
7. Load-step and impedance-test the damped input filter and both converters; check resonance, conducted noise, startup/inrush, PGOOD, ripple, and stability.
8. Thermally soak maximum credible MAIN_3V3/AUX5 load overlap, including the bounded AUX5 MAX duty; inspect both bucks, mux, inductors, MOSFETs, fuse, TVSs, load switches, shift switch, USB path, and audio amplifier.
9. Verify CAN RX/TX, hardware/software listen-only, standby wake, bounded diagnostics, bus faults, termination OFF, CMC bypass/population comparison, and unpowered loading.
10. Test OBD-only, USB-only, neither, and simultaneous sources through TPS2116 transitions; measure rail droop and reverse current into OBD, each 3.3 V source, and VBUS.
11. Verify GNSS startup, 25 Hz UART load, active-antenna voltage/current/short, RF insertion loss/C/N0, ESD-device effect, and converter/display/shift/audio desense.
12. Test SD insertion/write/load steps and controlled/abrupt brownout corruption recovery.
13. Test a 0.5 A electronic SHIFT5 load, short/hot-plug/fault reporting, then the real ten-pixel cable at full configured brightness; inspect data integrity and EMI.
14. Test maximum audio load, BTL waveform, PowerPAD temperature, audible acceptance, and interference with GNSS/CAN/SD.
15. Verify LP5814 default-off, MODE behavior, BOOT/RESET straps, and zero visible LED in PARKED/SLEEP.
16. Execute the approved ISO 7637-2/ISO 16750-2/ISO 10605/CISPR 25 and applicable UNECE R10 plan at a qualified facility if the product/compliance plan requires it.

## Reference-only comparison

The repository-owned RejsaCAN v3.4 schematic, BOM and example firmware were reviewed as `REFERENCE_ONLY` secondary evidence. No external design file or legacy input-protection circuit was copied as a validated solution; the legacy vehicle-input architecture was rejected against the v1 envelope above.

### Superseded Task 4.7 choices — historical only

The former `SM8SF24CA-Q`/single-bidirectional-TVS, `LM74502QDDFRQ1`, two `DMT6007LFGQ-7`, and `PMEG6030EP-Q` raw source-OR selections are superseded. They are not approved alternates or active blockers for Task 5A.1 capture.

## Primary sources

- Texas Instruments, [LM74720-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lm74720-q1.pdf), ratings, reverse-current blocking, OV thresholds, application circuit, and layout.
- STMicroelectronics, [SM15T-Y automotive TVS data sheet](https://www.st.com/resource/en/datasheet/sm15t36cay.pdf), `SM15T47AY` and `SM15T33AY` ratings, electrical table, leakage, and typical forward curve.
- STMicroelectronics, [STL125N10F8AG product page](https://www.st.com/en/power-transistors/stl125n10f8ag.html) and data sheet, 100 V/RDS(on)/gate/thermal/package limits.
- Littelfuse, [437A data sheet](https://www.littelfuse.com/assetdocs/littelfuse-fuse-437a-datasheet?assetguid=82c80a59-a4b9-4748-920b-3e2b65b813a9), `0437002.WRA` ratings, derating guidance, and nominal melting `I²t`.
- Texas Instruments, [TIDA-00699](https://www.ti.com/tool/TIDA-00699), suppressed-load-dump/cold-crank/EMI reference; [TIDA-01167](https://www.ti.com/tool/TIDA-01167), unsuppressed-load-dump reference.
- Texas Instruments, [LMQ66420-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lmq66420-q1.pdf), recommended/absolute VIN, IQ, thermal and passive guidance.
- Texas Instruments, TPS2553-Q1, TPS62162-Q1, TPS2116, TCAN3404-Q1, TPS22919-Q1, TPS1H100-Q1, TCA6408A-Q1, TPA2005D1-Q1 and LP5814 data sheets.
- STMicroelectronics, ESDCAN04-2BWY and USBLC6-2SC6Y data sheets.
- Littelfuse, [AQ3118E-01ETG data sheet](https://www.littelfuse.com/assetdocs/littelfuse_tvs_diode_array_aq3118e-01etg_datasheet.pdf), automotive RF ESD protection.
- u-blox, [NEO-M9N Integration Manual R10](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf), active-antenna bias-T and RF layout guidance.
- ISO/IEC/UNECE/SAE sources are catalogued in [`automotive-electrical-profile.md`](automotive-electrical-profile.md).
