# ChatGPT engineering review packet

## Task completed

Completed Task 5A.1 — Automotive Power Architecture Correction and Re-Freeze.
Recovered and preserved the valid work present after the editor restart, then
finished the cross-document audit, USB power correction, and final handoff.

The result is conditionally re-frozen for a later prototype schematic-capture
task. No KiCad schematic or PCB, firmware, CAD, manufacturing output,
procurement action, physical test, or compliance claim was created.

## Files changed

- `docs/architecture.md`
- `docs/automotive-electrical-profile.md`
- `docs/chatgpt-review.md`
- `docs/component-freeze.md`
- `docs/hardware-spec.md`
- `docs/input-protection-architecture.md`
- `docs/open-source-dependencies.md`
- `docs/power-budget.md`
- `docs/power-wake-review.md`
- `docs/power.md`
- `docs/requirements.md`
- `docs/rejsacan-analysis.md`
- `docs/schematic-architecture.md`
- `docs/system-validation-plan.md`
- `docs/task5a-power-calculations.md`
- `docs/task5a1-power-architecture.md`
- `docs/telemetry-v1-change-list.md`
- `docs/user-io-configuration.md`

`CODEX.md` and all hardware, firmware, PCB, CAD, and manufacturing files were
unchanged.

## Engineering findings

- Task 5A blocker evidence remains preserved in
  `docs/task5a-power-calculations.md` and is explicitly historical.
- The former 12.458 W total incorrectly combined mutually exclusive display
  rails and sizing margins. The corrected maximum simultaneous output is 8.27505 W:
  2.47500 W from the complete MC3/vehicle 3.3 V bucket plus 5.80005 W AUX5.
- A separate 3.3 V-display case loads MC3 to 1.050 A/3.465 W; 1.313 A is only
  its sizing value. This case, not the 0.750 A simultaneous bucket alone,
  controls MC3 thermal validation.
- Both MC3 and MC5 now have the same exact conditional capture population:
  `XGL5030-222MEC`; two `CGA6P1X7R1N106K250AC` CIN; four
  `CGA6P3X7R1E226M250AB` COUT plus one DNP; two
  `CGA3E1X7R1C105K080AC` CVCC; and CBOOT DNP. Effective capacitance,
  stability and hardware thermal proof remain release gates.
- Corrected AUX5 contracts are 0.56001 A STREET continuous, 0.92001 A TRACK
  continuous, and 1.16001 A MAX for at most 10 s and 25% of a rolling 60 s.
- The permitted low-voltage policy prohibits MAX/Wi-Fi/full-white below
  10.0 V, requests bounded SD flush below 9.5 V, enters CORE/SHED below
  9.0 V for 100 ms, and recovers optional loads above 10.0 V for 2 s.
- The 2 A `0437002.WRA` fuse is the lowest conditionally defensible rating
  after load correction. Fuse time-current, hot derating, interrupt, inrush,
  TVS-short clearing, and harness/copper coordination remain physical gates.
- LM74720 OV uses 249 kΩ + 249 kΩ over 28.0 kΩ, all 0.1%. With resistor
  tolerance and the documented ±1 µA node-leakage envelope, the rising
  OV/PD-low command is 20.690–25.531 V and falling reconnect eligibility is
  18.815–23.366 V. Completed isolation/recovery and downstream peak remain
  dynamic gates.
- The exact LM74720 support population is `CGA5H2X7R2A224K115AE` 220 nF/
  100 V from A to ground; `CRCW06030000Z0EA` tying VS to C;
  `CGA6N3X7R2A225K230AE` 2.2 µF/100 V from C/VS to ground;
  `CGA6M3X7R1H225K200AE` 2.2 µF/50 V from CAP to C/VS;
  `BZT52H-C18-Q`; `LPS3015-104MRC` 100 µH;
  `CRCW0603330RFKEA` 330 Ω in series with PD; and
  `CRCW0603100RFKEA`/`CGA2B3X7R1H103K050BE` 100 Ω/10 nF for slew control.
  The post-Q2 filter shunts are separate. Effective capacitance, resistor
  pulse stress and the calculated 0.846–1.443 A turn-on inrush range remain
  model/bench gates.
- The common-anode shunt pair `SM15T47AY`/`SM15T33AY` stays below intended
  avalanche at +26 V and the defined +38 V suppressed source. The
  −100 V/10 Ω/2 ms linear screen is 6.223–6.479 A and 0.456–0.470 J in the
  pair; electrothermal and layout behavior are unverified.
- With a protected node held at 25.531 V and a 49.2 V negative-clamp
  engineering envelope, LM74720 C-to-A/off-FET stress is 74.731 V. Margins
  before overshoot are 10.269 V to the controller's 85 V absolute limit and
  25.269 V to the 100 V MOSFET rating.
- TI specifies at most 1.5 µs from OV detection to PD low, but the selected
  FET has only a typical gate-charge value, so completed disconnect has no
  data-sheet-backed maximum. The documented +38 V filter sensitivity reaches
  41.639 V, only 0.361 V below the buck's 42 V absolute limit, and the +50 V
  illustrative dynamic screen drives the filter inductor beyond its
  saturation rating. These are warnings and model/bench gates, not passes.
- The selected filter uses exact shunt parts, `XEL4030V-222MEC`,
  four populated direct MLCCs plus one DNP footprint, and a
  `WSL2512R3900FEA`/`EEH-ZC1H121P` series-RC damping branch. Nominal
  resonance/Q screens are not bounds; the tolerance corner is
  21.54–39.13 kHz and Q=0.69–1.37.
- The complete conservative parked subtotal is 204.013 µA at 12 V; the
  doubled paper envelope is 408.026 µA (0.408 mA). The doubled
  6/8/10/12/14.4/18 V model is
  0.476/0.438/0.419/0.408/0.402/0.400 mA. It passes both paper targets but
  is not a hot-board guarantee.
- TPS2553 with 60.4 kΩ ±1% has a calculated 387.2–491.3 mA fault/current-limit
  population band, not an operating contract. The prototype startup source
  and cable must advertise and sustain at least 500 mA at 4.75 V. Post-ramp
  `USB_ENUM` is at most 100 mA total on `MAIN_3V3`, approximately 82 mA at
  VBUS; it is not an inrush ceiling. Configured limits remain 350 mA VBUS
  steady and 400 mA `MAIN_3V3`, with MAX prohibited. This does not establish
  generic legacy USB 2.0 pre-enumeration compliance.
- The exact conditional USB population is `CRCW060360K4FKEA` RILIM;
  `CGA2B3X7R1H104M050BB` 100 nF; `CGA6P1X7R1E106M250AC` 10 µF/25 V CIN;
  `XFL3012-222MEC`; `CGA6M3X7R1C106K200AB` 10 µF/16 V COUT; and
  `EEEFK0J101AV` 100 µF/6.3 V at TPS2116 VOUT. Effective capacitance,
  current-limited startup, mux hold-up/clamp and source handoff remain gates.

## Decisions made

- Selected complete Architecture B: 2 A fuse; asymmetric ST TVS shunt;
  LM74720 true reverse-current-blocking controller; two 100 V ST MOSFETs;
  damped post-switch filter; separate vehicle and USB 3.3 V conversion; and
  TPS2116 regulated-rail selection.
- Rejected Architecture A because retained LM74502 has no enabled reverse
  current blocking. Its screened major-power BOM is about $34.55, its
  support-adjusted comparison is about $35.55, and its parked envelope is
  about 0.468 mA.
- Rejected Architecture C because LTC4368 increases current, cost, and
  negative-clamp/fuse coordination burden. Its screened major-power BOM is
  about $41.01, its support-adjusted comparison is about $42.01, and its
  parked envelope is about 0.757 mA.
- Selected Architecture B's traceable major-power screen is $35.43, its
  support-adjusted comparison is about $38.38, and its parked envelope is
  0.408 mA. The support-adjusted A/B/C screens include explicit controller-
  specific allowances but still exclude common LMQ/USB passives, connectors,
  PCB and assembly. Allowances carry at least ±$1 planning uncertainty, so
  cost did not decide A versus B; enabled reverse-current behavior did.
- Froze `LMQ66420MC3RXBRQ1` for the vehicle 3.3 V source.
- Conditionally retained `LMQ66420MC5RXBRQ1` for the corrected AUX5 modes;
  the same-family 3 A part does not reduce switch resistance or heat.
- Conditionally retained `TPS2553QDBVRQ1`, `TPS62162QDSGRQ1`, and
  `TPS2116DRLR` with the exact passive population listed above.
  `TPS2116DRLR` remains a catalog/non-AEC risk.
- Superseded LM74502, DMT6007, SM8SF24CA-Q, PMEG6030EP-Q, the 20 V cutoff,
  the 24 V protected-node cap, and the 12.458 W load total.

## Assumptions

- MAIN efficiency by 6/8/10/12/14.4/18 V is
  90/91/92/92/91/90%; AUX is 88/90/91/92/91.5/90%.
- Separate hot-regulator screens assume MC3 efficiency of 93/92/91% and MC5
  efficiency of 92/91/90% at 12/14.4/18 V.
- USB screening uses 4.75 V at the connector, TPS2553 maximum 135 mΩ
  resistance, and 85% TPS62162 efficiency.
- Thermal calculations use manufacturer JEDEC comparison metrics and 85 °C
  ambient; they are not PCB thermal models. The selected layout starts from
  TI's 2 oz outer/1 oz inner four-layer stack.
- MOSFET conduction uses a 1.7× hot-resistance screen; copper/inductor hot
  screening uses a 1.393 resistance factor. Neither substitutes for measured
  junction, copper or enclosure temperature.
- TVS pulse results use endpoint-linear interpolation, an assumed forward
  drop where stated, rectangular energy, and the documented source models.
- Fast-positive sensitivity uses the explicitly stated idealized timing,
  second-order and lossless LC models; it is not a voltage bound.
- OV analysis includes a conservative ±1 µA total pin/PCB leakage envelope.
- Capacitor screens use the documented tolerance, temperature, DC-bias and
  ageing reserves; vendor characteristic curves remain reference-only.
  Filter ESR at resonance and converter negative-input impedance are modeled,
  not guaranteed production values.
- The turn-on inrush sensitivity assumes a linear PD current-source ramp into
  the stated maximum capacitor population. It is not a MOSFET SOA pass.
- Parked rail-load conversion uses 60% low-load efficiency and a 100%
  uncertainty/temperature allowance.
- Architecture cost uses the stated quantity-one observations plus $1.00/
  $2.00 controller-support allowances and excludes common passives, PCB and
  assembly; it is a comparison model rather than a quotation.
- Component stock and prices are 2026-08-17 observations and are volatile.

## Uncertainties / unresolved questions

- Exact +38 V duration, impedance, repetition, and purchased OEM/standard
  pulse set.
- TVS dynamic clamp/forward voltage, pulse temperature/repetition, fixture
  inductance, and fuse survival/clearing.
- LM74720/FET VDS, VGS, C-to-A, timing, SOA, inrush, reverse handoff, and
  output-precharge behavior including overshoot; exact completed-off time is
  unknown because maximum gate charge and the assembled gate network are not
  bounded by the public switching table.
- Downstream survival through 25.531 V plus measured dynamic overshoot.
- Exact effective filter capacitance, resistor pulse capability, populated
  impedance, converter interaction, and conducted/radiated EMI.
- USB effective capacitance under bias/temperature/ageing, current-limited
  startup sequencing, TPS2116 clamp/hold-up, actual source/Type-C contract,
  generic legacy-host behavior, and hot 105 °C behavior. Exact prototype
  capacitor/inductor/resistor order codes are no longer open.
- LMQ effective input/output capacitance, loss, stability, load steps and PCB/
  enclosure temperature using the exact shared MC3/MC5 population.
- Measured MAIN/AUX efficiency, dropout, load steps, PCB temperature, and
  whether a 105 °C enclosure requirement is necessary.
- Complete-board parked current versus voltage, temperature, contamination,
  wake duty, and post-drive shutdown.
- Exact VBAT ADC tolerances, crank waveforms, and SD-flush time/energy.

## Risks

- The 10.269 V controller stress margin can be consumed by TVS/layout
  overshoot if the pulse model or placement is wrong.
- The selected 2 A fuse may nuisance-open or fail to coordinate if real
  inrush, temperature, repetition, or prospective fault current exceeds the
  paper model.
- The filter can resonate with converter negative input impedance or overheat
  its damping resistor if the provisional population is captured without
  impedance/pulse testing.
- TPS2116 is not automotive-qualified and is specified only through 105 °C
  for the limits used.
- USB startup can enter current limiting before firmware can enforce a load
  policy.
- The 0.408 mA parked result has not been proven at temperature on a complete
  populated board.
- Switching-current paths can desense GNSS or disturb CAN/USB without the
  required placement, grounding, and EMI validation.

## GPIO / peripheral changes

| Item | Task 5A.1 result |
|---|---|
| ESP32 GPIO allocation | No GPIO allocation changed |
| CAN peripheral | No CAN/TWAI mapping changed; TCAN VCC becomes explicitly vehicle-only |
| USB peripheral | Native USB pins unchanged; only the documented power-source partition changed |
| GNSS / SD / display / outputs | Existing mappings unchanged; their power-state contracts were corrected |
| Boot/strapping | No change; existing strap-pin restrictions remain |

## Power / CAN / RF impact

- Automotive power changed materially to the coordinated fuse/TVS/controller/
  MOSFET/filter architecture and corrected load/parked/low-voltage contracts.
- USB now powers only `USB_3V3 -> TPS2116 -> MAIN_3V3`. It cannot energize
  OBD pin 16, CAN VCC, AUX5, or external 5 V outputs.
- CAN topology and protocol scope did not change. TCAN VCC is physically on
  `VEH_3V3`; USB-only operation must leave bus pins passive and unpowered.
- GNSS/RF component selections did not change. Both converter switch nodes,
  inductors, high-di/dt entry loop, and surge return require separation from
  GNSS/ESP antenna paths and the mandatory EMI/GNSS/power placement review.
- ESP32 USB pins and boot/strapping assignments did not change.

## Datasheets / primary sources consulted

- TI `LM74720-Q1` data sheet and EVM guide.
- TDK exact automotive support-capacitor data for
  `CGA5H2X7R2A224K115AE`, `CGA6N3X7R2A225K230AE`,
  `CGA6M3X7R1H225K200AE`, and `CGA2B3X7R1H103K050BE`.
- Nexperia `BZT52H-Q` automotive zener data and Vishay D/CRCW e3
  automotive resistor data for the exact 0 Ω, 100 Ω, 330 Ω and 60.4 kΩ
  support values.
- ST `SM15T-Y` automotive TVS family data and `STL125N10F8AG` data sheet.
- Littelfuse `437A` fuse data sheet.
- TI `LMQ66420-Q1` data sheet.
- TI `TPS2553-Q1`, `TPS62162-Q1`, and `TPS2116` data sheets.
- Coilcraft `XEL4030V`, `XGL5030-222MEC`, `XFL3012-222MEC`, and
  `LPS3015-104MRC` manufacturer data.
- TDK CGA automotive MLCC product data and characteristic sheets for the
  exact LMQ and USB populations.
- Panasonic `EEH-ZC` hybrid-capacitor data and FK-series
  `EEEFK0J101AV` product data.
- Vishay `WSL` pulse-resistor family data/tool guidance.
- TI TIDA-00699/TIDA-01167 and SNVA538/SLVAFE0/SNVA801.
- Analog Devices `LTC4368`, ST `STPM801`, NXP `FS27`, `FS23` and `UJA1163A`,
  Infineon `IAUA210N10S5N024`, `TLE9471` and `TLE9261`, and Diodes
  `DMT10H020SDGQ-13` primary product data.
- Repository RejsaCAN schematic/BOM/source and Task 5A blocker calculations.

## Recommended next step

Perform an independent Task 5A.1 power-design review, including model review
and approval of the bench validation matrix, before any KiCad capture.

## STOP condition

Task 5A.1 ends after this documentation re-freeze and review packet. Stop
before KiCad capture, firmware implementation, PCB layout, CAD, manufacturing
generation, purchasing, or physical testing.
