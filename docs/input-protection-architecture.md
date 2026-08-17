# AutoTelemetry v1 input and interface protection architecture

Status: Task 4.7 pre-schematic freeze, 2026-08-17. This is the controlled circuit intent for Task 5, not a schematic, layout, tested design, or standards-compliance claim. Exact resistor/capacitor/inductor order codes remain Task 5 calculations where explicitly marked `PROVISIONAL`.

Superseding status: Task 5A is **BLOCKED**. The Task 4.7 architecture remains below as historical context, but its vehicle-input capture authorization is withdrawn. The reopened threshold, TVS, fuse/load, MOSFET negative-pulse and filter-leakage gates are controlled by [`task5a-power-calculations.md`](task5a-power-calculations.md); no affected value or substitute part is approved for capture.

## Task 4.7 vehicle-input chain

```text
OBD16 / VBAT_OBD_RAW
  -> 0437002A, 2 A / 63 V fixed fuse
  -> VBAT_FUSED_CLAMPED
       +-> SM8SF24CA-Q bidirectional TVS -> POWER_GND
       +-> 100 nF + 1 µF, 100 V ceramic starting envelope -> POWER_GND
  -> LM74502QDDFRQ1 + 2 × DMT6007LFGQ-7 back-to-back N-MOSFETs
       UV falling target 6.0 V; OV nominal target 18.0 V
  -> VBAT_SWITCHED
  -> damped C-L-C / pi filter
  -> VEHICLE_PROTECTED
  -> SYS_IN source-OR node
  -> LMQ66420-Q1 MAIN_3V3 and switched AUX5 converters
```

The chain order records the Task 4.7 architecture, but Task 5A `REOPENED` the TVS, threshold implementation, fuse/full-load policy, MOSFET negative-pulse coordination and exact damping network. Conditional inrush calculations do not close those gates.

## Reverse-polarity architecture comparison

| Architecture | Loss at 1.298 A reference input | Parked/behavior | Automotive and USB implications | Disposition |
|---|---:|---|---|---|
| Series Schottky | `1.298 A × 0.5 V = 0.649 W` (`ASSUMPTION` + `CALCULATED`) | Zero control IQ, but material crank headroom and heat loss | Simple reverse block; high-current diode and thermal copper required | `REJECTED` for main vehicle path |
| P-channel MOSFET | For an assumed 20 mΩ, `I²R = 33.7 mW` | Low static current | Higher RDS(on), negative-gate/transient control and OV disconnect still required | `REJECTED` versus N-FET controller |
| N-channel MOSFET plus ideal-diode/reverse controller | Selected pair hot estimate 48.7 mW; calculation below | LM74502 operating IQ is 45 µA typical/65 µA maximum; 110 µA is a conservative design allocation | Low loss, −65 V absolute controller withstand, programmable UV/OV, back-to-back disconnect; does not reverse-block while enabled | `REOPENED` threshold and negative-pulse coordination |
| Full surge-stopper/eFuse-style unsuppressed front end | Application dependent | Added controller/sense/pass element and SOA/thermal burden | Can support a wider unsuppressed envelope, but materially larger and must be designed around pass-FET SOA | `REJECTED` for v1 selected envelope; future revision option |

### Controller variant change

Task 4.7 approved replacing `LM74502HQDDFRQ1` with `LM74502QDDFRQ1`; Task 5A retains that comparison as historical rationale but `REOPENED` the controller/threshold implementation.

Both variants are AEC-Q100 grade 1. TI characterizes operation from 4–60 V, recommends −60…+60 V at the pins and specifies ±65 V absolute maximum. Operating quiescent current is 45 µA typical/65 µA maximum; 110 µA remains only a conservative `DESIGN_REQUIREMENT` budget allocation. The non-H device uses a 60 µA typical gate-source current and TI supports an external `Cdvdt` network; this variant rationale remains historical, but Task 5A reopened the controller/threshold implementation.

The `C_dvdt` value is not guessed. Task 5 shall use TI equation 2:

```text
C_dvdt = I_GATE × C_OUT / I_INRUSH
```

with the maximum effective post-switch capacitance, desired inrush below the fuse/time-current and MOSFET SOA limits, controller/source-current tolerance, and an isolation resistor.

## Fuse and fault strategy

`0437002A` (packing suffix `WRA`) is `REOPENED` with the load policy: Littelfuse 437A family, 2 A, 63 V, fast acting, AEC-Q200, 1206.

```text
PLOAD = 3.3 V×1.313 A + 5.0 V×1.625 A = 12.458 W
IIN at 12 V and 80% assumed efficiency = 12.458/(12×0.80) = 1.298 A
current margin to 2 A = (2.000−1.298)/1.298 = 54.1%
```

The 54.1% comparison is historical nameplate arithmetic, not a continuous-use result. Littelfuse recommends at most 80% of rating continuously (1.60 A at 25 °C); its 75 °C example gives 1.36 A. The full envelope draws 2.595 A at 6 V and 1.730 A at 9 V, requiring load shedding below 9.73 V at 25 °C and 11.45 V at 75 °C, or a reopened fuse and downstream-fault design.

The fuse protects the PCB/harness segment from a persistent hard fault even though the vehicle OBD circuit is upstream fused. It is not the normal branch limiter. TPS22919-Q1 switches protect the four small peripheral branches; TPS1H100B-Q1 protects/current-limits SHIFT5; TPS2553-Q1 limits USB input. A PPTC is rejected because its hot-cabin hold/trip behavior and reset ambiguity are less deterministic. A fusible resistor is rejected because it adds normal loss without replacing TVS energy coordination. A monolithic input eFuse is not added because reverse/OV disconnect is already provided and no selected device has been shown to meet this exact energy/current/IQ envelope.

Open items before schematic release are the 437A time-current/inrush curve at temperature, copper fault-current rating, and proof that a normal insertion/load step does not fatigue or open the fuse.

## Main TVS critical gate

### Candidate comparison

| Candidate | Polarity / qualification | VRWM / VBR | Published clamp and pulse rating | Leakage/package | Decision |
|---|---|---|---|---|---|
| `SM8SF24CA-Q` Bourns | Bidirectional, AEC-Q101 | 24 V; 26.7–29.5 V at 5 mA | 38.9 V maximum at 180 A; 7,000 W at 10/1000 µs | 10 µA max at 24 V/25 °C; 8.1×10.5×1.3 mm DFN | `REOPENED`; correct polarity for −14 V, but 24 V standoff does not cover +26 V/60 s |
| `SM8S24CA` Vishay | Bidirectional, AEC-Q101 series | 24 V; 26.7–29.5 V | 38.9 V at about 169.5 A; 6,600 W at 10/1000 µs | DO-218AB | Credible alternate; larger through-lead/power package |
| `LDP01-28AY` ST | Unidirectional, AEC-Q101 | 24 V; 26.7–29.5 V | 40 V at 120 A, 10/1000 µs; 45 V at 1,250 A, 8/20 µs; 5 kW class | 1 µA max at 25 °C; D2PAK | `REJECTED` at the pre-controller position because sustained reverse battery forward-biases a unidirectional clamp and can open the fuse |
| `SM8SF33CA-Q` Bourns | Bidirectional, AEC-Q101 | 33 V; 36.7–40.6 V | 53.3 V at 131 A; 7,000 W at 10/1000 µs | 10 µA max; DFN | `REJECTED`; inadequate margin to 60 V MOSFETs and no need for 33 V standoff |

The Task 4.7 `FROZEN` decision for `SM8SF24CA-Q` is superseded and `REOPENED`: the TVS is ahead of the disconnect, its 24 V working standoff does not cover +26 V/60 s, and its 26.7 V minimum breakdown leaves no data-sheet guarantee of non-avalanche operation at that condition. A higher-standoff choice must be coordinated with the 60 V MOSFETs and the exact +38 V pulse.

Any selected pre-controller TVS must mount immediately after the input fuse and beside the OBD ground-entry region. Its loop to `POWER_GND` must be short and wide; protected circuitry must not branch from the connector side of that loop.

### Transient-to-rating coordination

For the public TIDA-01167 positive fast-transient reference of +50 V with 2 Ω source resistance:

```text
conservative TVS bound = 38.9 V at the data-sheet rated current
source current at that bound = (50−38.9)/2 = 5.55 A
controller positive margin = 65−38.9 = 26.1 V
MOSFET VDS margin = 60−38.9 = 21.1 V
```

For the −100 V, 10 Ω negative reference, treating the bidirectional clamp symmetrically and conservatively:

```text
source current magnitude = (100−38.9)/10 = 6.11 A
controller reverse margin = 65−38.9 = 26.1 V
```

These are historical voltage-stress screens, not current upper bounds or measured operating points: using the maximum clamp specified at 180 A in the source-resistance equation produces a lower, not upper, current estimate. Task 5A's two-point linear screen is still only an `ASSUMPTION`; the physical test must measure TVS current, `VBAT_FUSED_CLAMPED`, `VBAT_SWITCHED`, and `VEHICLE_PROTECTED` simultaneously.

For the required +38 V suppressed-load-dump source, duration, source resistance, repetition and temperature remain undefined; the earlier 27 V/22 V static margin statement does not close TVS/FET coordination. The downstream node must remain ≤24 V (`DESIGN_REQUIREMENT`), giving each 42 V-absolute-maximum buck at least 18 V margin.

The severe public unsuppressed corner demonstrates why it is excluded. If a 101 V source with 0.5 Ω were hypothetically clamped at 38.9 V:

```text
I = (101−38.9)/0.5 = 124.2 A
instantaneous TVS power = 38.9×124.2 = 4.83 kW
rectangular 400 ms energy bound = 4.83 kW×0.4 s = 1.93 kJ
```

The real exponential pulse is not rectangular, and the Bourns data sheet contains load-dump graphs, but a 7 kW 10/1000-µs headline does not prove 400 ms, hot-start, repetitive, PCB-mounted survival. This remains outside the v1 guarantee unless a purchased-standard/OEM test profile and thermal calculation approve it.

## UV, OV, source crossover, and protected maximum

- The Task 4.7 values `UV falling nominal = 6.0 V` and `OV nominal = 18.0 V` are historical targets, not guaranteed thresholds.
- Task 5A proved no UV-divider overlap: guaranteed turn-on by 6.0 V requires divider scale `K ≤ 4.545`, while turn-off above even 5.0 V USB requires `K > 4.869`; EN sink-current behavior makes the high-value-divider uncertainty worse.
- Task 5A also proved no OV-divider overlap: guaranteed operation through 18 V requires `K ≥ 15.451`, while guaranteed cutoff by 20 V requires `K ≤ 15.004`.
- Because LM74502-Q1 does not reverse-block while enabled, the threshold/controller implementation is `REOPENED`; resistor selection alone cannot close either window.
- `VEHICLE_PROTECTED ≤24.0 V` for every approved transient is a measured `DESIGN_REQUIREMENT`, not a current claim.
- After UV/OV, reconnect is automatic only after hysteresis and source stability. Firmware must not turn the event into permission to transmit.

Exact divider values must balance comparator bias/error, parked current, resistor voltage rating, and input leakage. Their combined 12 V draw plus the vehicle detector/ADC network is limited to 25 µA at the OBD input (`DESIGN_REQUIREMENT`).

## Input filter gate

The filter is after the disconnect MOSFETs so it cannot dump stored energy directly into the raw vehicle pin and so OV turn-off isolates its bulk.

```text
VBAT_SWITCHED
  -> 100 nF + 1 µF, 50 V X7R local high-frequency shunt
  -> 2.2–4.7 µH series inductor, ≥3 A continuous, ≥4 A saturation, low DCR
  -> 2 × 4.7 µF, 50 V X7R plus 47–100 µF, 50 V bulk
  -> local 4.7 µF buck input capacitors and 100 nF bypasses
  -> VEHICLE_PROTECTED / SYS_IN
```

The topology remains a starting direction; exact L/C/R order codes and the damping implementation are `REOPENED`. The conditional 100 µF hybrid damping capacitor has up to 50 µA leakage and raises the doubled parked envelope to 525.08 µA, so it is not approved while the 0.5 mA stretch target remains. Any replacement still requires source/filter impedance, negative-incremental-impedance, bias, tolerance, temperature, harness, inrush and EMI validation.

The raw-node ceramics are 100 V rated because they sit before disconnect. Post-switch capacitors are 50 V rated against the 24 V protected maximum. Effective capacitance after DC bias, ripple current, bulk ESR, inductor saturation, inrush, and the LMQ66420 input-capacitance requirements remain part of the schematic calculation. CISPR 25 testing, not visual resemblance to a reference filter, decides final population.

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
USB_VBUS_RAW -> USBLC6-2SC6Y -> TPS2553QDBVRQ1 -> USB5_PROTECTED
               -> PMEG6030EP-Q -> SYS_IN

VEHICLE_PROTECTED -------------------------------> SYS_IN
```

| OBD | USB | Required result |
|---|---|---|
| OFF | OFF | All rails off; only passive ESD/CC structures present |
| ON | OFF | Vehicle supplies MAIN_3V3 and permitted peripherals; USB VBUS is not driven |
| OFF | ON | TPS2553/PMEG path supplies MAIN_3V3 for development; AUX5 and all external 5 V outputs locked off; vehicle MOSFETs remain off |
| ON | ON | Vehicle normally wins; PMEG blocks vehicle-to-host current. If OBD sags, UVLO opens the vehicle path before USB can drive it; measure the crossover and reverse current |

`PMEG6030EP-Q` remains frozen because its 60 V rating tolerates the selected `SYS_IN` envelope and its forward drop preserves USB-only headroom. Its data sheet gives 5 µA typical reverse current at 10 V/25 °C and 200 µA maximum only at 60 V/25 °C. Reverse leakage rises strongly with temperature; room/hot parked current and thermal runaway margin must be measured. A future low-leakage PN or ideal-diode replacement is required if this path prevents the parked-current target.

USB-only behavior:

- MAIN_3V3 powers ESP32, TCAN logic, and expander; optional GNSS/SD operation must remain inside the configured 500 mA VBUS budget.
- TCAN3404-Q1 bus pins are high impedance when unpowered and remain non-transmitting when powered without an OBD harness. STB defaults high.
- No AUX5, DISPLAY_5V, SHIFT5, or sound load is enabled.
- With both sources, USB data remains usable. Source presence never authorizes CAN transmission.

## Always-connected component voltage margins

`Protected max` values are design-node limits to be proven on the prototype.

| Component / node | Absolute maximum or rating | Protected max | Margin | Status |
|---|---:|---:|---:|---|
| `0437002A` fuse | 63 V | 40 V raw clamp requirement | 23 V | `REOPENED`; full-load/thermal policy and pulse/fault coordination unresolved |
| `LM74502QDDFRQ1` VS/EN/OV | ±65 V | ±40 V raw clamp requirement | 25 V magnitude | `REOPENED`; threshold implementation and physical clamp test unresolved |
| `DMT6007LFGQ-7` pair | 60 V VDS | 40 V raw clamp requirement | 20 V | `REOPENED`; negative-pulse stress can reach about 62.2 V with a held 24 V output at the published 38.9 V clamp point |
| `LMQ66420MC3RXBRQ1` VIN/EN | 42 V absolute, 36 V recommended | 24 V downstream requirement | 18 V absolute / 12 V recommended | `FROZEN silicon` |
| `LMQ66420MC5RXBRQ1` VIN/EN | 42 V absolute, 36 V recommended | 24 V downstream requirement | 18 V absolute / 12 V recommended | `REOPENED`; AUX5 continuous-load/thermal requirement unresolved |
| `PMEG6030EP-Q` reverse voltage | 60 V | 24 V at cathode with USB absent | 36 V | `FROZEN`; leakage test required |
| `TPS22919-Q1` | 6 V absolute | 5.5 V rail maximum | 0.5 V | `FROZEN`; regulate AUX5 tolerance accordingly |
| `TPA2005D1TDGNRQ1` active supply | 6 V absolute | 5.5 V rail maximum | 0.5 V | `FROZEN direction`; use T-suffix −40…+105 °C order code |
| `LP5814DRLR` | 6 V absolute | 3.6 V MAIN_3V3 maximum | 2.4 V | `APPROVED`; not AEC-qualified |
| Raw-node capacitors | 100 V | 40 V | 60 V | exact AEC-Q200 parts `PROVISIONAL` |
| Post-switch capacitors | 50 V | 24 V | 26 V | exact AEC-Q200 parts `PROVISIONAL` |

The margin table does not replace pin-specific injection-current, transient duration, temperature, or repetitive-stress checks.

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

All values are referred to a 12 V OBD input. Rail-load conversions use 60% low-load efficiency (`ASSUMPTION`).

| Always-powered path | 12 V contribution | Basis |
|---|---:|---|
| LM74502-Q1 controller | 110.0 µA | Conservative `DESIGN_REQUIREMENT` allocation; data-sheet operating IQ is 45 µA typical/65 µA maximum |
| SM8SF24CA-Q TVS | 10.0 µA | `VERIFIED_DATASHEET` maximum at 24 V/25 °C, conservatively allocated at 12 V; hot leakage unverified |
| MAIN buck own IQ | 5.0 µA | `DESIGN_REQUIREMENT`; data-sheet typical is 1.5 µA, guaranteed implementation max unresolved |
| UV/OV, vehicle detection, gated ADC sensing | 25.0 µA | `DESIGN_REQUIREMENT` combined at 12 V |
| TCAN3404 standby | 7.8 µA | `17 µA×3.3/(12×0.60)` (`CALCULATED`) |
| ESP32 module deep-sleep branch | 22.9 µA | `50 µA×3.3/(12×0.60)` (`DESIGN_REQUIREMENT` + `CALCULATED`) |
| TCA6408A-Q1 standby | 4.6 µA | `10 µA×3.3/(12×0.60)` using data-sheet maximum (`CALCULATED`) |
| LP5814 shutdown | 0.14 µA | `0.3 µA×3.3/(12×0.60)` using data-sheet maximum (`CALCULATED`) |
| Wake/button/timer logic | 4.6 µA | Existing 10 µA rail allocation converted to input |
| Disabled TPS22919 branches | 1.5 µA | `DESIGN_REQUIREMENT` total including temperature/leakage |
| USB PMEG/source-isolation path | 5.0 µA | `ASSUMPTION` based on 5 µA typical at 10 V/25 °C; must be measured hot |
| CAN/USB/other signal ESD leakage | 5.0 µA | `DESIGN_REQUIREMENT` |
| Miscellaneous leakage | 10.0 µA | `DESIGN_REQUIREMENT` |
| GNSS, SD, display, AUX5, SHIFT5, sound, visible LEDs | 0 µA intended load | Rails/outputs off; residual switch leakage accounted above |
| Disabled AUX5 buck VIN current | 1.0 µA | `VERIFIED_DATASHEET` maximum shutdown input current |
| **Corrected subtotal before optional damping capacitor** | **212.54 µA** | `CALCULATED` |
| **100% uncertainty/temperature allowance** | **212.54 µA** | `DESIGN_REQUIREMENT` margin |
| **Corrected doubled envelope before optional damping capacitor** | **425.08 µA = 0.425 mA** | `CALCULATED` |

The conditional damping capacitor adds 50 µA maximum before the same 100% allowance: `(212.54 + 50.00) × 2 = 525.08 µA`. This still passes the <1 mA release requirement by 0.475 mA, but fails the <0.5 mA stretch target by 0.025 mA.

Comparison:

- previous Task 4.6 estimate: 0.371 mA;
- Task 4.7 historical estimate: 0.424 mA; it omitted AUX5 shutdown current;
- corrected Task 5A base envelope before the optional damping capacitor: 0.425 mA;
- conditional envelope with the optional 50 µA capacitor: 0.525 mA;
- `<1.0 mA` release requirement: **PASS on calculated conditional allocation**, 0.475 mA margin; not verified;
- `<0.50 mA` room-temperature stretch target: **FAIL with the conditional capacitor**, 0.025 mA over; not verified.

Even the corrected 0.425 mA base margin is small. PMEG reverse leakage, buck maximum IQ, ESP32 module/PSRAM sleep current, TVS leakage, divider tolerances, GPIO pulls, filter leakage and temperature can consume it. The exact damping network is `REOPENED`; complete assembled-board measurement over voltage and temperature remains the only acceptance evidence.

## Pre-schematic thermal sanity check

Assumptions: 85 °C hot enclosure ambient; 50 °C/W effective buck junction-to-ambient with the required multilayer copper/vias; maximum simultaneous rail allocations; efficiency values are placeholders to be replaced by schematic loss calculations.

| Item | Calculation / concern | Result and requirement |
|---|---|---|
| Back-to-back DMT6007LFGQ pair | Hot RDS(on) assumption `2×8.5 mΩ×1.7 = 28.9 mΩ`; `1.298²×0.0289` | 48.7 mW (`CALCULATED`); low steady heat, but provide low-impedance copper and verify transient SOA |
| Input TVS | No normal avalanche at 12–18 V; transient-only energy | Large thermal/ground copper; keep away from GNSS/ESP antennas; test hot pulses, no steady compliance claim |
| MAIN_3V3 buck | `Pout=3.3×1.313=4.333 W`; assumed 85% gives `Ploss=0.765 W`; `ΔT=38.2 °C` | Approx. 123.2 °C junction at 85 °C ambient; 26.8 °C to 150 °C limit (`CALCULATED`), requiring TI copper/via guidance |
| AUX5 buck | `Pout=5×1.625=8.125 W`; assumed 88% gives `Ploss=1.108 W`; `ΔT=55.4 °C` | Approx. 140.4 °C junction; only 9.6 °C to 150 °C (`CALCULATED`). **High validation risk**; derate simultaneous loads or propose higher-current/lower-loss silicon if measurement fails |
| TPS22919-Q1 | Use conservative 200 mΩ hot bound: at 0.6 A, `I²R=72 mW`; 0.4 A gives 32 mW | Provide local copper; verify voltage drop/thermal/current-limit and QOD behavior per branch |
| TPS1H100-Q1 | 0.50 A qualified load is modest, but a short forces linear/current-limit operation | Calculate RCL, timer/SOA, copper and repetitive fault behavior; locate away from GNSS RF |
| PMEG6030EP-Q USB diode | At 0.5 A and 0.4 V maximum 25 °C forward drop, about 0.20 W | Give cathode copper; verify USB-only dropout and hot leakage |
| TPA2005D1-Q1 | `1 W/0.75−1 W = 0.333 W` loss at assumed 75% efficiency | PowerPAD soldered to ground copper/vias; keep BTL traces compact and away from GNSS; speaker opening/acoustics deferred |
| Fuse | 1.298 A at 12 V is below the 2 A nameplate but near the 1.36 A 75 °C continuous-use example; full load is 2.595 A at 6 V and 1.730 A at 9 V | `REOPENED`: define low-voltage/hot load shedding or change the fuse and re-coordinate inrush/fault behavior |

This is not a thermal simulation. Actual efficiency curves, switching frequency, inductor/core/copper loss, board stackup, enclosure, airflow, duty cycle, and simultaneous-load policy determine the result.

## Frozen net names

| Domain | Frozen names |
|---|---|
| Raw vehicle | `VBAT_OBD_RAW`, `POWER_GND`, `VBAT_FUSED_CLAMPED`, `VBAT_SWITCHED`, `VEHICLE_PROTECTED`, `VEHICLE_PRESENT`, `PWR_FAULT_N` |
| Source OR / conversion | `SYS_IN`, `USB_VBUS_RAW`, `USB5_PROTECTED`, `MAIN_3V3`, `AUX5`, `MAIN_PGOOD`, `AUX5_EN`, `AUX5_PGOOD` |
| Switched rails | `GNSS_3V3`, `SD_3V3`, `DISPLAY_3V3`, `DISPLAY_5V`, `SHIFT5`, `SOUNDER5` |
| CAN | `CANH_OBD`, `CANL_OBD`, `CANH_PHY`, `CANL_PHY`, `CAN_RX`, `CAN_TX`, `CAN_STB` |
| GNSS | `GNSS_RF`, `ANT_BIAS`, `GNSS_RX`, `GNSS_TX`, `GNSS_TIMEPULSE`, `GNSS_RESET_N` |
| External/control | `SHIFT_DATA_5V`, `SHIFT5_EN`, `SHIFT_FAULT_N`, `MODE_N`, `STATUS_DRV_EN`, `USB_PRESENT` |

There is no separate `AON_3V3`: `MAIN_3V3` is the always-on parked rail. Do not create KiCad nets in Task 4.7.

## First-prototype test points

| Test point | Required | Constraint |
|---|---|---|
| `VBAT_OBD_RAW` | Yes | Before fuse; guarded/labelled for controlled bench use |
| `VBAT_FUSED_CLAMPED` | Yes | Measure TVS clamp; high-voltage spacing |
| `VBAT_SWITCHED` | Yes | Verify MOSFET turn-on/off and inrush |
| `VEHICLE_PROTECTED` / `SYS_IN` | Yes, separate measurement points if nodes differ | Measure filter and source crossover |
| `POWER_GND` | Multiple compact points | One near power entry and one low-current instrument point |
| `MAIN_3V3`, `AUX5` | Yes | Rail ripple/thermal/load-step probes |
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

1. Measure parked current at 12 V/25 °C, then over the approved parked voltage and temperature range; separate controller, USB diode, MAIN buck, ESP, CAN, expander, and leakage paths.
2. Measure active current for named typical and maximum-load states; verify fuse and source copper temperature.
3. Sweep UV/OV slowly and dynamically; record turn-off/reconnect thresholds, hysteresis, chatter, `VEHICLE_PROTECTED` overshoot, and repeated recovery.
4. Apply approved warm/cold-crank waveforms; verify controlled reset, no reboot loop, SD recovery, CAN passive state, and staged peripheral restart.
5. Perform a current-limited −14 V reverse-polarity test and approved fast positive/negative transient tests; confirm no fuse opening from normal reverse connection and no damage.
6. Measure the TVS current/clamp at hot and cold starts under the selected pulse matrix; inspect component and copper temperature. Stop before unapproved destructive severity.
7. Load-step and impedance-test the damped input filter and both converters; check resonance, conducted noise, startup/inrush, PGOOD, ripple, and stability.
8. Thermally soak maximum credible MAIN_3V3/AUX5 load overlap; inspect buck, inductor, MOSFETs, fuse, TVS, load switches, shift switch, USB diode, and audio amplifier.
9. Verify CAN RX/TX, hardware/software listen-only, standby wake, bounded diagnostics, bus faults, termination OFF, CMC bypass/population comparison, and unpowered loading.
10. Test OBD-only, USB-only, neither, and simultaneous sources, including a slow/fast OBD sag through the USB crossover; measure reverse current into both OBD and VBUS.
11. Verify GNSS startup, 25 Hz UART load, active-antenna voltage/current/short, RF insertion loss/C/N0, ESD-device effect, and converter/display/shift/audio desense.
12. Test SD insertion/write/load steps and controlled/abrupt brownout corruption recovery.
13. Test a 0.5 A electronic SHIFT5 load, short/hot-plug/fault reporting, then the real ten-pixel cable at full configured brightness; inspect data integrity and EMI.
14. Test maximum audio load, BTL waveform, PowerPAD temperature, audible acceptance, and interference with GNSS/CAN/SD.
15. Verify LP5814 default-off, MODE behavior, BOOT/RESET straps, and zero visible LED in PARKED/SLEEP.
16. Execute the approved ISO 7637-2/ISO 16750-2/ISO 10605/CISPR 25 and applicable UNECE R10 plan at a qualified facility if the product/compliance plan requires it.

## Reference-only comparison

The repository-owned RejsaCAN v3.4 schematic, BOM and example firmware were reviewed as `REFERENCE_ONLY` secondary evidence. No external design file or legacy input-protection circuit was copied as a validated solution; the legacy vehicle-input architecture was rejected against the v1 envelope above.

## Primary sources

- Texas Instruments, [LM74502-Q1/LM74502H-Q1 data sheet SNOSDE0A](https://www.ti.com/lit/ds/symlink/lm74502-q1.pdf), §§9.3.3–9.3.5, application circuits, ratings, and layout.
- Diodes Incorporated, [DMT6007LFGQ data sheet](https://www.diodes.com/datasheet/download/DMT6007LFGQ.pdf), 60 V and 4.5/10 V RDS(on) limits.
- Littelfuse, 437A/`0437002A` product data, 2 A/63 V AEC-Q200 fuse.
- Bourns, [SM8SF-Q data sheet](https://www.bourns.com/docs/Product-Datasheets/SM8SF-Q.pdf), candidate table and load-dump graphs.
- Vishay, [SM8S data sheet](https://www.vishay.com/docs/88387/sm8s.pdf), 6.6 kW DO-218AB alternate.
- STMicroelectronics, [LDP01-28AY data sheet](https://www.st.com/resource/en/datasheet/ldp01-28ay.pdf), unidirectional 5 kW candidate and ISO 16750 load-dump graphs.
- Texas Instruments, [TIDA-00699](https://www.ti.com/tool/TIDA-00699), suppressed-load-dump/cold-crank/EMI reference; [TIDA-01167](https://www.ti.com/tool/TIDA-01167), unsuppressed-load-dump reference.
- Texas Instruments, [LMQ66420-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lmq66420-q1.pdf), recommended/absolute VIN, IQ, thermal and passive guidance.
- Texas Instruments, TCAN3404-Q1, TPS22919-Q1, TPS2553-Q1, TPS1H100-Q1, TCA6408A-Q1, TPA2005D1-Q1 and LP5814 data sheets.
- Nexperia, [PMEG6030EP-Q data sheet](https://assets.nexperia.com/documents/data-sheet/PMEG6030EP-Q.pdf), reverse leakage, drop, ratings and thermal behavior.
- STMicroelectronics, ESDCAN04-2BWY and USBLC6-2SC6Y data sheets.
- Littelfuse, [AQ3118E-01ETG data sheet](https://www.littelfuse.com/assetdocs/littelfuse_tvs_diode_array_aq3118e-01etg_datasheet.pdf), automotive RF ESD protection.
- u-blox, [NEO-M9N Integration Manual R10](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf), active-antenna bias-T and RF layout guidance.
- ISO/IEC/UNECE/SAE sources are catalogued in [`automotive-electrical-profile.md`](automotive-electrical-profile.md).
