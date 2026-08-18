# AutoTelemetry v1 automotive electrical profile

Status: Task 5A.1 corrected requirement profile, 2026-08-17. The selected
implementation and calculations are in
[`task5a1-power-architecture.md`](task5a1-power-architecture.md). This profile
is not evidence of ISO, CISPR, UNECE, SAE, OEM, or vehicle compliance.

## Evidence labels

- `VERIFIED_DATASHEET`: a value published for the named component under the cited conditions.
- `CALCULATED`: a reproducible result derived in this document.
- `DESIGN_REQUIREMENT`: a release requirement for AutoTelemetry v1.
- `DESIGN_TARGET`: a preferred behavior that may be relaxed only by documented review.
- `ASSUMPTION`: an input that must be replaced by a selected-part value or measurement.
- `STANDARD_REFERENCE`: a public standards scope or a manufacturer reference-design test condition; not a copied standard requirement and not automatically an AutoTelemetry limit.
- `UNVERIFIED_TEST_REQUIREMENT`: a future bench/laboratory test whose severity, setup, acceptance class, or result is not yet approved.

## Scope and load-dump decision

AutoTelemetry v1 is a permanently or long-duration connected **12 V passenger-car** OBD device. OBD pin 16 supplies battery power; pins 4 and 5 supply ground; pins 6 and 14 carry Classical CAN where supported. Continuous 24 V commercial-vehicle use is out of scope.

The selected load-dump envelope is **option C: a defined suppressed passenger-car environment plus strong fast-transient protection**:

- survive and automatically recover from a source clamped to 38 V (`DESIGN_REQUIREMENT`, derived from the public TIDA-00699 reference condition);
- disconnect downstream electronics during sustained overvoltage, jump start, and load dump rather than promise uninterrupted operation;
- protect the raw front end with an asymmetric anti-series automotive TVS
  pair, a true reverse-current-blocking controller and 100 V back-to-back
  MOSFETs; validate the coordinated network physically;
- do **not** claim survival of every severe unsuppressed load-dump corner. TI TIDA-01167 demonstrates that unsuppressed protection is a distinct, larger surge-stopper architecture and warns that non-OEM alternators can change the environment (`STANDARD_REFERENCE`).

This is appropriate for a compact universal aftermarket OBD device because it gives explicit protection against the modern suppressed environment without hiding the size, energy, cost, and thermal consequences of a full unsuppressed requirement. Vehicles with an unknown or unsuppressed charging system are outside the v1 guaranteed envelope until a laboratory profile is approved and passed.

## OBD pin 16 voltage envelope

| Condition at OBD pin 16 | Frozen value or range | Classification | Required behavior |
|---|---:|---|---|
| Nominal system | 12 V class | `DESIGN_REQUIREMENT` | Passenger vehicles only |
| Engine-off nominal reference | 12.6 V | `ASSUMPTION` | Budget and bench reference, not a battery-health claim |
| Expected engine-off operating band | 11.0–13.0 V | `DESIGN_TARGET` | Normal operation |
| Expected charging/smart-alternator band | 11.0–16.0 V | `DESIGN_TARGET` | Normal operation; actual vehicles vary |
| Core-survival input range | 6.0–18.0 V | `DESIGN_REQUIREMENT` | Preserve core/CAN/wake where regulation permits; full peripheral load is not required at 6 V |
| Preferred full-feature range | 10.0–18.0 V | `DESIGN_TARGET` | State-selected features subject to thermal and duty contracts |
| MAX-load inhibit | below 10.0 V | `DESIGN_REQUIREMENT` | No Wi-Fi/full-white/MAX diagnostic state |
| SD-flush request | below 9.5 V | `DESIGN_REQUIREMENT` | Begin a bounded flush; stop starting optional work |
| LOW-VOLTAGE SHED entry | below 9.0 V for 100 ms | `DESIGN_REQUIREMENT` | AUX5/display/shift/sound/Wi-Fi off; logging stops after bounded flush |
| LOW-VOLTAGE SHED exit | above 10.0 V for 2 s | `DESIGN_REQUIREMENT` | Requalify source before optional rails return |
| Supply interruption | 0 V, indefinite | `DESIGN_REQUIREMENT` | No damage; clean restart when a valid source returns |
| Controlled-reset region | below converter regulation / approved brownout threshold | `DESIGN_REQUIREMENT` | Reset is acceptable; no oscillation, uncontrolled CAN TX or avoidable SD corruption |
| Maximum continuous operating voltage | 18.0 V | `DESIGN_REQUIREMENT` | No OV trip through this point |
| OV command, rising | 20.690–25.531 V modeled | `CALCULATED` | OV/PD-low command before +26 V with 0.1% divider and ±1 µA node-leakage envelope; completed opening and downstream peak require capture |
| OV recovery, falling | 18.815–23.366 V modeled | `CALCULATED` | Returning to 18 V makes every modeled unit reconnect-eligible; completion, inrush and ring time require capture |
| Protected-node design envelope | 25.531 V static threshold plus measured overshoot | `DESIGN_REQUIREMENT` | Supersedes the old 24 V cap; every downstream input must tolerate the measured result |
| Jump start | +26 V for 60 s | `STANDARD_REFERENCE` from TIDA-00699 | Survive, disconnect, and recover automatically; operation is not required |
| Reverse battery | −14 V for 60 s | `STANDARD_REFERENCE` from TIDA-00699/TIDA-01167 | No damage and no input-fuse opening solely from correct reverse-polarity application |
| Suppressed load dump | source up to +38 V | `STANDARD_REFERENCE` from TIDA-00699, adopted as a `DESIGN_REQUIREMENT` | Downstream disconnect; no damage; automatic recovery |
| Public severe unsuppressed reference | source 79–101 V, 0.5–4 Ω, 40–400 ms | `STANDARD_REFERENCE` from TIDA-01167 | **Not a v1 guaranteed envelope**; use only to bound risk and future test planning |
| Fast positive transient reference | +50 V, 2 Ω, 0.05 ms | `STANDARD_REFERENCE` from TIDA-01167 | `UNVERIFIED_TEST_REQUIREMENT`; no damage and automatic recovery |
| Fast negative transient reference | −100 V, 10 Ω, 2 ms | `STANDARD_REFERENCE` from TIDA-01167 | `UNVERIFIED_TEST_REQUIREMENT`; no damage and automatic recovery |
| ESD and coupled transients | interface-specific | `UNVERIFIED_TEST_REQUIREMENT` | Test after connector/enclosure/harness freeze |

The public reference-design pulse values are not reproduced from a purchased standard and are not claimed to be the only valid ISO levels. OEM requirements, repetition counts, source impedance, harness inductance, pulse spacing, ambient temperature, and functional-status classification still require an approved laboratory plan.

## Functional behavior by voltage

| Region | MAIN_3V3 / MCU | CAN | GNSS / SD | Display / shift / sound | Recovery |
|---|---|---|---|---|---|
| 10–18 V qualified | Normal within mode contract | State-controlled RX/TX | State-controlled | State-controlled | Normal |
| 9.5–10 V | Core active | Receive/safe by default | Start no optional work | MAX/Wi-Fi/full-white prohibited | Return after >10 V/2 s |
| 9.0–9.5 V | Core active where regulation permits | Receive/safe by default | Bounded flush, then logging off | AUX optional work stopping | Enter/exit with defined timers |
| 6–9 V | CORE/SHED only where regulation permits | Passive/default-safe | GNSS only if budget permits; SD off after flush | AUX5/display/shift/sound/Wi-Fi off | Controlled recovery; no full-load promise |
| Below regulation/brownout | Controlled loss/reset | Unpowered/high impedance; no intentional transmit | Off; filesystem recovery after abrupt interruption | Off | Restart only after source qualification |
| 18–25.531 V possible OV band | Vehicle path may remain on until its tolerance-bounded trip | No new transmit during fault handling | Optional loads off | Off | Automatic after source returns to 18 V |
| 25.531–38 V overvoltage/load dump | OV commands opening; steady/post-response intent is open, while fast-edge output peak/timing remains unbounded pending capture | Unpowered unless USB independently supplies core; bus passive | Off | Off | Reconnect eligible after return to 18 V; completion/inrush/ring require capture |
| Reverse battery / negative pulse | Vehicle path open | Unpowered/passive | Off | Off | Automatic after valid polarity returns |

USB may keep MAIN_3V3 alive while the vehicle source is invalid. Firmware must still treat the OBD path as absent and must not transmit merely because USB preserved the processor.

## Cranking and brownout choice

AutoTelemetry v1 selects **option B: controlled reset with fast automatic recovery**. MAIN_3V3 may ride through a mild sag above UVLO, but uninterrupted MCU/GNSS/SD/display operation is not a requirement. AUX5, display, shift-light, and sound are disabled first. This avoids a boost or large energy-storage stage and removes repeated half-powered peripheral states.

The hold-up cost shows why full ride-through is rejected. For a capacitor feeding a constant-power load:

```text
C = 2 × P × t / [efficiency × (Vstart² − Vend²)]
```

Using `Vstart = 12 V`, `Vend = 4.5 V`, `t = 100 ms`, and `efficiency = 0.80` (`ASSUMPTION`):

```text
0.33 W essential load: C = 2×0.33×0.1 / [0.8×(12²−4.5²)]
                           = 667 µF                     [CALCULATED]

4.0 W active load:       C = 2×4×0.1 / [0.8×(12²−4.5²)]
                           = 8,081 µF                   [CALCULATED]
```

These idealized values exclude ESR, capacitance tolerance/temperature, aging,
converter current limit, and extra reserve. They would also increase inrush
and fuse/MOSFET stress. The selected 120 µF nominal series-R damping branch
and direct MLCC bank are for filter damping and load-step support, not a crank
ride-through guarantee.

Brownout acceptance requires:

- measured PGOOD, VBAT-state and brownout hysteresis so the system does not
  chatter across low-voltage state boundaries;
- ESP32 brownout/reset enabled and validated;
- no new SD transaction once low-voltage shutdown begins, while abrupt interruption remains a filesystem fault case;
- CAN TX default-recessive and transceiver standby by hardware during reset;
- optional rails default off until MAIN_3V3 is stable and firmware requalifies the source;
- bounded reboot/backoff behavior under a repetitive crank waveform.

## Ground disturbance and grounding requirement

OBD pins 4 and 5 are both connected. Each receives its own harness conductor and reaches one robust connector-entry join into the continuous `POWER_GND` plane; neither is daisy-chained through a sensitive return. The PCB must not create a separate “quiet ground” island that reconnects through an uncontrolled trace.

Ground offset, cable resistance, starter current, and USB-connected equipment can disturb this reference. Future tests must inject approved common-mode/ground-offset conditions without using the AutoTelemetry board as a return path for unrelated vehicle fault current. Connector, copper, and harness current ratings must support the product fault strategy.

## External-interface ESD classification

| Interface | Class | Frozen protection intent |
|---|---|---|
| OBD pin 16 | `CABLE_EXTERNAL` | Fuse, asymmetric anti-series TVSs, LM74720 reverse/OV disconnect, filtered entry |
| OBD CAN-H / CAN-L | `CABLE_EXTERNAL` | Connector-local ESDCAN04-2BWY, short return, optional CMC footprint DNP |
| USB-C | `CABLE_EXTERNAL` | USBLC6-2SC6Y at receptacle; shell network remains an EMC tuning item |
| GNSS U.FL and antenna cable | `CABLE_EXTERNAL` | Connector-local ≤0.5 pF RF ESD, shortest RF-ground return, protected bias-T |
| Display harness | `CABLE_EXTERNAL` | Connector ESD, current-limited switched rails, source damping, return pins |
| Shift-light harness | `CABLE_EXTERNAL` | Current-limited high-side switch, data buffer/series resistor, connector ESD |
| MODE | `ENCLOSURE_INTERNAL` | Non-conductive actuator over internal switch; GPIO series/RC network only. Reclassify if metal is externally touchable |
| BOOT and RESET | `ENCLOSURE_INTERNAL` | Recessed service access; avoid TVS leakage/capacitance that alters GPIO0/EN behavior. Reclassify if externally exposed |

No interface earns a standards pass from the protector data sheet alone. The final enclosure, harness, discharge point, ground plane, and powered/unpowered state determine the system result.

## Standards and regulatory map

| Reference | Relevance | Historical Task 4.7 use (context only) | Remaining question |
|---|---|---|---|
| ISO 16750-2:2023 | Electrical loads for road-vehicle equipment; ISO notes harness/connection impedance changes stress | Environment taxonomy and future electrical-load plan | Exact mounting class, severities, durations, functional status, and OEM overlay |
| ISO 7637-2:2011 | Bench methods for conducted transients on 12/24 V supply lines and functional-performance classification | Supply-line transient planning | Purchased-standard test matrix, repetition, coupling network, and acceptance class |
| ISO 10605:2023 | Bench and vehicle ESD methods covering assembly, service, and occupants | External-interface ESD planning | Final contact/air points, DUT states, levels, and acceptance criteria |
| CISPR 25:2021 | 150 kHz–5.925 GHz emissions methods/limits protecting onboard receivers including GNSS, Wi-Fi, and Bluetooth | Input-filter/layout and future emissions plan | Product class, limits, chamber/LISN method, harness, and operating modes |
| UN Regulation No. 10 | Vehicle and electrical/electronic subassembly EMC approval context | Regulatory applicability review only | Intended markets, ESA category, approval route, and applicable amendment series |
| SAE J1962 | Diagnostic connector location, geometry, terminals, and electrical interface context | OBD connector/pin context | SAE explicitly excludes long-term-retention needs; permanent-install retention/strain relief needs separate design evidence |

## Designed, tested, and compliance states

**Designed in Task 5A.1:** corrected voltage states, controlled-reset and
load-shed behavior, suppressed-event boundary, coordinated asymmetric TVSs,
LM74720/100 V FET disconnect, regulated-rail USB isolation, component screens,
ground intent and future test points.

**Not yet tested:** parked/active current, UV/OV thresholds, crank recovery, reverse polarity, source crossover, TVS clamp/energy, positive/negative transients, ESD, conducted/radiated emissions and immunity, RF desense, thermal behavior, cable faults, or vehicle coexistence.

**Not established:** ISO/CISPR/UNECE/SAE/OEM compliance, regulatory approval, compatibility with every passenger car, or survival of the severe unsuppressed load-dump envelope.

## Primary sources

- ISO, [ISO 16750-2:2023 scope](https://www.iso.org/standard/76119.html), electrical loads and harness/connection impedance context.
- ISO, [ISO 7637-2:2011 scope](https://www.iso.org/standard/50925.html), supply-line transient bench methods and functional-performance classification.
- ISO, [ISO 10605:2023 scope](https://www.iso.org/standard/79094.html), module and vehicle ESD methods.
- IEC, [CISPR 25:2021 scope](https://webstore.iec.ch/en/publication/64645), onboard receiver protection and 150 kHz–5.925 GHz emissions measurement.
- UNECE, [UN Regulation No. 10 document index](https://unece.org/transport/vehicle-regulations-wp29/standards/addenda-1958-agreement-regulations-0-20), electromagnetic compatibility.
- SAE International, [SAE J1962 diagnostic connector scope](https://saemobilus.sae.org/standards/j1962_199802-diagnostic-connector), including its long-term-retention exclusion.
- Texas Instruments, [TIDA-00699](https://www.ti.com/tool/TIDA-00699) and [design guide TIDUB49](https://www.ti.com/lit/pdf/TIDUB49), 10–15 W suppressed-load-dump/cold-crank/reverse-battery reference front end.
- Texas Instruments, [TIDA-01167](https://www.ti.com/tool/TIDA-01167) and [design guide TIDUC41A](https://www.ti.com/lit/pdf/TIDUC41), unsuppressed-load-dump protection reference and public test conditions.
- Analog Devices, [low-quiescent-current automotive surge stopper](https://www.analog.com/en/resources/technical-articles/low-quiescent-current-surge-stopper-robust-automotive-supply-protection.html), suppressed/unsuppressed load-dump and reverse-battery architecture context.
