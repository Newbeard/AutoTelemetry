# AutoTelemetry v1 automotive electrical profile

Status: Task 4.7 pre-schematic requirement freeze, 2026-08-17. This profile defines design and future validation inputs for a 12 V passenger-vehicle OBD product. It is not evidence of ISO, CISPR, UNECE, SAE, OEM, or vehicle compliance.

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
- protect the raw 60/65 V front end with a bidirectional high-energy TVS and validate the coordinated clamp physically;
- do **not** claim survival of every severe unsuppressed load-dump corner. TI TIDA-01167 demonstrates that unsuppressed protection is a distinct, larger surge-stopper architecture and warns that non-OEM alternators can change the environment (`STANDARD_REFERENCE`).

This is appropriate for a compact universal aftermarket OBD device because it gives explicit protection against the modern suppressed environment without hiding the size, energy, cost, and thermal consequences of a full unsuppressed requirement. Vehicles with an unknown or unsuppressed charging system are outside the v1 guaranteed envelope until a laboratory profile is approved and passed.

## OBD pin 16 voltage envelope

| Condition at OBD pin 16 | Frozen value or range | Classification | Required behavior |
|---|---:|---|---|
| Nominal system | 12 V class | `DESIGN_REQUIREMENT` | Passenger vehicles only |
| Engine-off nominal reference | 12.6 V | `ASSUMPTION` | Budget and bench reference, not a battery-health claim |
| Expected engine-off operating band | 11.0–13.0 V | `DESIGN_TARGET` | Normal operation |
| Expected charging/smart-alternator band | 11.0–16.0 V | `DESIGN_TARGET` | Normal operation; actual vehicles vary |
| Full functional input range | 6.0–18.0 V | `DESIGN_REQUIREMENT` | MAIN_3V3 and state-selected peripherals may operate; AUX5 only when its regulation/headroom and thermal conditions are valid |
| Preferred normal range | 9.0–16.0 V | `DESIGN_TARGET` | All active features without crank-specific shedding |
| UV disconnect, falling | 6.0 V nominal | `DESIGN_REQUIREMENT` | Vehicle path turns off; exact divider must guarantee cutoff above the highest USB-derived `SYS_IN` crossover |
| UV reconnect, rising | about 6.58 V typical | `CALCULATED` | `6.0 V × 1.25 V / 1.14 V`; exact limits include comparator and resistor tolerances |
| Supply interruption | 0 V, indefinite | `DESIGN_REQUIREMENT` | No damage; clean restart when a valid source returns |
| Controlled-reset region | below the tolerance-bounded UV threshold | `DESIGN_REQUIREMENT` | Reset/power loss is acceptable; no reboot oscillation, uncontrolled CAN transmission, or SD corruption |
| Maximum continuous operating voltage | 18.0 V | `DESIGN_REQUIREMENT` | Overvoltage disconnect begins above the tolerance-bounded limit |
| OV disconnect, nominal | 18.0 V | `DESIGN_REQUIREMENT` | Exact divider must produce worst-case steady cutoff no higher than 20.0 V |
| Protected downstream maximum | 24.0 V | `DESIGN_REQUIREMENT` | Maximum allowed at `VEHICLE_PROTECTED` during the approved transient matrix; must be measured |
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
| 9–16 V preferred | Normal | State-controlled RX/TX | State-controlled | State-controlled | Normal |
| 6–9 V valid input | May remain active | Default to receive/safe state | Stop new SD writes before an anticipated shutdown; peripheral resets are allowed | AUX5 loads disabled unless regulation is proven | Return to normal after stable voltage |
| Below UV threshold | Controlled loss/reset | Transceiver becomes unpowered/high impedance; no intentional transmit | Off; filesystem recovery required after an abrupt case | Off | Restart only after UV hysteresis and source qualification |
| 18–20 V OV transition | Disconnecting | No new transmit | Off | Off | Automatic after hysteresis |
| 20–38 V overvoltage/load dump | Vehicle path open | Unpowered unless USB independently supplies logic; bus pins remain passive | Off | Off | Automatic after source returns to range |
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

These idealized values exclude ESR, capacitance tolerance/temperature, aging, converter current limit, and extra reserve. They would also increase inrush and fuse/MOSFET stress. Therefore the post-protection 47–100 µF bulk envelope is for switching/load-step support and brief glitches, not a crank ride-through guarantee.

Brownout acceptance requires:

- hardware UV hysteresis so the source path does not chatter;
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
| OBD pin 16 | `CABLE_EXTERNAL` | Fuse, bidirectional load-dump TVS, reverse/OV disconnect, filtered entry |
| OBD CAN-H / CAN-L | `CABLE_EXTERNAL` | Connector-local ESDCAN04-2BWY, short return, optional CMC footprint DNP |
| USB-C | `CABLE_EXTERNAL` | USBLC6-2SC6Y at receptacle; shell network remains an EMC tuning item |
| GNSS U.FL and antenna cable | `CABLE_EXTERNAL` | Connector-local ≤0.5 pF RF ESD, shortest RF-ground return, protected bias-T |
| Display harness | `CABLE_EXTERNAL` | Connector ESD, current-limited switched rails, source damping, return pins |
| Shift-light harness | `CABLE_EXTERNAL` | Current-limited high-side switch, data buffer/series resistor, connector ESD |
| MODE | `ENCLOSURE_INTERNAL` | Non-conductive actuator over internal switch; GPIO series/RC network only. Reclassify if metal is externally touchable |
| BOOT and RESET | `ENCLOSURE_INTERNAL` | Recessed service access; avoid TVS leakage/capacitance that alters GPIO0/EN behavior. Reclassify if externally exposed |

No interface earns a standards pass from the protector data sheet alone. The final enclosure, harness, discharge point, ground plane, and powered/unpowered state determine the system result.

## Standards and regulatory map

| Reference | Relevance | Task 4.7 use | Remaining question |
|---|---|---|---|
| ISO 16750-2:2023 | Electrical loads for road-vehicle equipment; ISO notes harness/connection impedance changes stress | Environment taxonomy and future electrical-load plan | Exact mounting class, severities, durations, functional status, and OEM overlay |
| ISO 7637-2:2011 | Bench methods for conducted transients on 12/24 V supply lines and functional-performance classification | Supply-line transient planning | Purchased-standard test matrix, repetition, coupling network, and acceptance class |
| ISO 10605:2023 | Bench and vehicle ESD methods covering assembly, service, and occupants | External-interface ESD planning | Final contact/air points, DUT states, levels, and acceptance criteria |
| CISPR 25:2021 | 150 kHz–5.925 GHz emissions methods/limits protecting onboard receivers including GNSS, Wi-Fi, and Bluetooth | Input-filter/layout and future emissions plan | Product class, limits, chamber/LISN method, harness, and operating modes |
| UN Regulation No. 10 | Vehicle and electrical/electronic subassembly EMC approval context | Regulatory applicability review only | Intended markets, ESA category, approval route, and applicable amendment series |
| SAE J1962 | Diagnostic connector location, geometry, terminals, and electrical interface context | OBD connector/pin context | SAE explicitly excludes long-term-retention needs; permanent-install retention/strain relief needs separate design evidence |

## Designed, tested, and compliance states

**Designed in Task 4.7:** the voltage bands, controlled-reset behavior, suppressed-load-dump envelope, reverse/OV disconnect architecture, component rating margins, ground intent, interface protection intent, and future test points.

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

