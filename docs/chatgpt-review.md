# ChatGPT engineering review packet

## Task completed

Completed Task 4.7 — Automotive Electrical Environment and Input Protection Freeze. Defined the 12 V passenger-car electrical envelope, selected the suppressed-load-dump design boundary, froze the vehicle input/reverse/OV/TVS/fuse/filter/ground/source-OR architecture, revalidated CAN/GNSS/USB/external-interface protection, recalculated parked current and thermal screening, froze schematic net names/test points, and created the future prototype validation plan. No KiCad, PCB, firmware, CAD, or manufacturing file was created or modified, and Task 5 was not started.

## Files changed

- Created `docs/automotive-electrical-profile.md`.
- Created `docs/input-protection-architecture.md`.
- Updated `docs/component-freeze.md`.
- Updated `docs/power-budget.md`.
- Updated `docs/power-wake-review.md`.
- Updated `docs/power.md`.
- Updated `docs/schematic-architecture.md`.
- Updated `docs/requirements.md`.
- Updated `docs/hardware-spec.md`.
- Updated `docs/interfaces.md`.
- Updated `docs/test-architecture.md`.
- Updated `docs/telemetry-v1-change-list.md`.
- Updated `docs/rejsacan-analysis.md`.
- Updated `docs/user-io-configuration.md`.
- Updated `docs/chatgpt-review.md`.

## Engineering findings

- ISO 16750-2:2023 covers electrical loads and notes that harness/connection impedance changes stress; ISO 7637-2:2011 defines conducted supply-line transient methods and functional-performance classification; ISO 10605:2023 covers module/vehicle ESD. CISPR 25:2021 is an emissions standard protecting onboard receivers, while UNECE R10 applicability/approval remains a product-market question. SAE J1962 explicitly does not cover long-term connector retention.
- The v1 envelope is 6–18 V operation, +26 V for 60 s jump-start survival by disconnect, −14 V for 60 s reverse survival, and a +38 V suppressed-load-dump source condition survived by disconnect. `VEHICLE_PROTECTED` must measure ≤24 V. Severe unsuppressed load dump is not guaranteed.
- TIDA-01167's public severe reference illustrates the excluded risk. At a hypothetical 101 V/0.5 Ω source clamped to 38.9 V, current is 124.2 A and instantaneous TVS power is 4.83 kW (`CALCULATED`); headline pulse power alone cannot establish 400 ms hot/repetitive survival.
- `PROPOSED CHANGE`, approved for Task 5: use `LM74502QDDFRQ1` instead of `LM74502HQDDFRQ1`. TI supports external `Cdvdt` inrush control on the 60 µA non-H variant; both variants retain the 2 A gate sink used for turn-off.
- Freeze `SM8SF24CA-Q`: bidirectional, AEC-Q101, 24 V VRWM, 26.7–29.5 V breakdown, 38.9 V maximum clamp at 180 A and 7 kW 10/1000 µs. Bidirectionality avoids forward-biasing the main clamp during sustained reverse battery. ST `LDP01-28AY` is rejected at this pre-controller position because it is unidirectional.
- The +50 V/2 Ω public fast-positive reference gives a conservative 5.55 A source-current screening value at 38.9 V clamp; margins to the 65 V controller and 60 V MOSFET are 26.1 V and 21.1 V. These are calculations, not measured clamp results.
- Freeze `0437002A` 2 A/63 V input fuse. The 12.458 W maximum named rail output at assumed 80% conversion requires 1.298 A from 12 V, leaving 54.1% arithmetic current margin to 2 A. Inrush/time-current/hot coordination remains open.
- Freeze post-switch damped C-L-C topology with 100 V raw ceramics, 50 V post-switch capacitance, a 2.2–4.7 µH/≥3 A/≥4 A-saturation inductor envelope and an R-C damping option. Exact parts require impedance/stability/inrush calculation.
- Select controlled reset/automatic recovery for crank. Ideal 100 ms hold-up from 12 V to 4.5 V needs about 667 µF even for 0.33 W and 8,081 µF for 4 W at assumed 80% efficiency; full ride-through is not justified.
- OBD4 and OBD5 join once at connector entry into one continuous `POWER_GND`; return paths are controlled by placement, not split ground nets. USB ground is common; speaker output is BTL.
- LM74502 does not reverse-block while enabled. Therefore its tolerance-bounded UVLO must open above the highest USB-derived `SYS_IN` crossover, and reverse current during slow/fast OBD sag is a prototype acceptance test.
- CAN remains `TCAN3404DRQ1` plus `ESDCAN04-2BWY`, ACT45B-510 footprint DNP/0 Ω bypass default, and split 120 Ω DNP/OFF default. No CAN component or termination change was required.
- `PROPOSED CHANGE`, approved for Task 5: replace NRND GNSS protector `ESDAXLC6-1BT2Y` with active `AQ3118E-01ETG` (AEC-Q101/PPAP, bidirectional 18 V, 0.3 pF typical). The prior 22 Ω antenna-short limiter is reopened because the calculation did not prove VCC_RF/module/antenna hot-short safety.
- Approve `TPA2005D1TDGNRQ1` direction and `LP5814DRLR` architecture. Speaker and RGB LED remain provisional. LP5814 is not AEC-qualified and requires board-level qualification.
- Revised parked-current subtotal is 211.54 µA; with a 100% allowance the envelope is 423.08 µA = 0.424 mA at 12 V. This passes <1.0 mA by 0.576 mA and the <0.50 mA room target by 0.076 mA on paper, not by measurement.
- Thermal screening at 85 °C ambient and assumed 50 °C/W gives about 123.2 °C MAIN_3V3 junction at its envelope and 140.4 °C AUX5 junction at its envelope. AUX5 has only 9.6 °C to the 150 °C limit and is the principal pre-schematic thermal risk.

## Decisions made

- Limit v1 to a defined suppressed-load-dump passenger-car environment; explicitly exclude guaranteed severe unsuppressed load dump.
- Require disconnect and automatic recovery during jump/OV/load dump rather than uninterrupted operation.
- Use controlled reset below UVLO instead of full crank ride-through.
- Retain back-to-back N-MOSFET reverse/OV disconnect, but use the inrush-controllable LM74502 variant.
- Freeze a bidirectional Bourns main TVS, fixed 2 A fuse and damped post-switch filter topology.
- Freeze 6.0 V nominal UV falling, about 6.58 V typical reconnect, 18.0 V nominal OV and ≤20 V worst-case steady cutoff as Task 5 tolerance targets; require ≤24 V downstream during approved transients.
- Join OBD4/5 at one entry region into a continuous ground plane.
- Preserve USB TPS2553 plus PMEG source isolation and add a measured UVLO/source-crossover requirement.
- Preserve CAN TVS/CMC/termination defaults.
- Approve active Littelfuse GNSS RF ESD, TPA2005D1TDGNRQ1 direction and LP5814 architecture.
- Freeze Task 5 net names, hierarchical block contracts and first-prototype test points in `docs/schematic-architecture.md`.

## Assumptions

- Engine-off nominal reference is 12.6 V; expected engine-off and smart-charging bands are design targets, not universal vehicle guarantees.
- Rail-envelope input-current arithmetic uses 80% combined conversion efficiency.
- Parked rail-load conversion uses 60% low-load efficiency and 5 µA PMEG reverse leakage at approximately 10–12 V/25 °C.
- Hot MOSFET screening uses 1.7 times the 8.5 mΩ/part 25 °C RDS(on) maximum.
- Thermal screening uses 85 °C ambient, 50 °C/W effective buck thermal resistance, 85% MAIN efficiency and 88% AUX5 efficiency.
- Hold-up screening uses 80% conversion efficiency and constant power.
- Public TIDA reference-design test conditions are treated as `STANDARD_REFERENCE`, not copied ISO requirements or guaranteed field events.

## Uncertainties / unresolved questions

- Purchased-standard/OEM pulse severities, repetition counts, source networks, acceptance classes and applicable regulatory route.
- Exact LM74502 UV/OV divider values/tolerances, `Cdvdt` inrush network, fuse time-current behavior and MOSFET SOA.
- Exact filter inductor/capacitors/damping, effective capacitance, impedance interaction, CISPR population and transient ringing.
- Physical TVS clamp/energy/temperature behavior and proof that `VEHICLE_PROTECTED` remains ≤24 V.
- Maximum hot PMEG reverse leakage, guaranteed buck maximum IQ, and complete assembled-board parked current.
- Actual regulator/inductor loss, PCB copper/thermal-via result and allowed simultaneous AUX5 load policy.
- Exact active GNSS antenna and passive/active short-current limiter; RF insertion loss/C/N0 with AQ3118E-01ETG.
- Exact display/shift connector ESD parts, connector families, cable construction and USB-shell population.
- Exact speaker/acoustics and RGB LED/optics.
- ESD, transient, EMC, RF, thermal and vehicle test results; no compliance result exists.

## Risks

- A vehicle with a severe unsuppressed charging-system load dump is outside the v1 guarantee and can exceed the selected energy envelope.
- AUX5 simultaneous maximum load may approach the regulator junction limit in a hot enclosure.
- PMEG Schottky leakage can consume the small <0.50 mA parked-current stretch margin at high temperature.
- Source-crossover tolerance or turn-off delay could briefly backfeed a sagging OBD source while USB is present.
- An undamped or incorrectly populated input filter can ring or destabilize the bucks.
- LP5814, ESP32-S3-WROOM and NEO-M9N are not automotive-qualified even though surrounding protection parts may be.
- GNSS antenna protection/short limiting can degrade sensitivity or damage the supply if the final antenna and layout are not jointly validated.

## GPIO / peripheral changes

| Resource | Task 4.6 assignment | Task 4.7 result |
|---|---|---|
| ESP32 GPIOs | Frozen map, including GPIO10 MODE, GPIO17 sound, GPIO0 BOOT and EN RESET | No GPIO-number change |
| TCA6408A P6 | `STATUS_DRV_EN` proposed | Same function, now approved with LP5814 architecture |
| Sound peripheral | TPA2005D1-Q1 proposed on GPIO17 | Exact TPA2005D1TDGNRQ1 direction approved; GPIO unchanged |
| GNSS RF ESD | ESDAXLC6-1BT2Y | AQ3118E-01ETG proposed change approved; no MCU peripheral change |

## Power / CAN / RF impact

- Automotive power: major requirement/protection freeze. Main TVS and controller variant change; operating/OV/UV/load-dump/crank/ground/filter/source-crossover contracts are now explicit. The parked estimate rises from 0.371 to 0.424 mA.
- CAN: no transceiver, GPIO, termination or default population change. ESDCAN04 stays fitted, CMC and termination stay DNP. Ground/ESD placement and future validation are more explicit.
- GNSS/RF: main module/topology unchanged; RF ESD changes to AQ3118E-01ETG and antenna-short limiter returns to provisional. No RF routing was performed.
- USB: TPS2553/PMEG/USBLC6 architecture unchanged; UVLO coordination and the simultaneous-source sag test are new requirements.
- ESP32 boot/strapping: no allocation change. MODE remains GPIO10; BOOT GPIO0 and RESET EN remain internal/recessed and must not receive protection that alters timing/leakage.

## Datasheets / primary sources consulted

- ISO 16750-2:2023, ISO 7637-2:2011 and ISO 10605:2023 official scope pages.
- IEC CISPR 25:2021 official scope; UNECE UN Regulation No. 10 official index; SAE J1962 official scope.
- TI LM74502-Q1/LM74502H-Q1, LMQ66420-Q1, TCAN3404-Q1, TPS22919-Q1, TPS2553-Q1, TPS1H100-Q1, TCA6408A-Q1, TPA2005D1-Q1 and LP5814 data sheets/product data.
- TI TIDA-00699 suppressed-load-dump/cold-crank reference and TIDA-01167 unsuppressed-load-dump reference.
- Analog Devices automotive low-IQ surge-stopper application article.
- Bourns SM8SF-Q, Vishay SM8S and ST LDP01-28AY TVS data sheets.
- Diodes Incorporated DMT6007LFGQ and Littelfuse 437A/0437002A data.
- Nexperia PMEG6030EP-Q and CAHCT1G126-Q1 manufacturer data.
- ST ESDCAN04-2BWY/USBLC6-2SC6Y and Littelfuse AQ3118E-01ETG data.
- u-blox NEO-M9N data sheet and Integration Manual R10.
- Repository RejsaCAN v3.4 schematic/BOM evidence as `REFERENCE_ONLY`; its legacy protection circuit was not copied as a validated solution.

## Recommended next step

Begin Task 5 with the vehicle-input and main-power schematic sheets, completing the explicitly frozen UV/OV-tolerance, `Cdvdt`-inrush, fuse/SOA and damped-filter calculations before capturing their exact passive values.

## STOP condition

Task 4.7 is complete. Stop before Task 5 schematic capture, PCB work, firmware implementation, CAD, manufacturing generation, procurement, or electrical testing.
