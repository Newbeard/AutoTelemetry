# Task 5A.1 automotive power architecture correction and re-freeze

Status: **CONDITIONALLY RE-FROZEN FOR PROTOTYPE SCHEMATIC CAPTURE**, 2026-08-18.

This document supersedes the active power decisions in
[task5a-power-calculations.md](task5a-power-calculations.md) and the Task 4.7
power freeze. It resolves the contradictions that correctly stopped Task 5A.
It does not report a completed schematic, PCB, firmware implementation,
measurement, automotive qualification, or standards compliance.

The selected circuit is a defensible prototype starting point. Production
release remains blocked by the validation gates in this document.

## Evidence convention

- `VERIFIED_DATASHEET`: a primary manufacturer document states the value
  under the cited conditions.
- `CALCULATED`: the result follows from the shown inputs and equation.
- `DESIGN_REQUIREMENT`: a behavior the implementation must provide; it is
  not measured evidence.
- `ASSUMPTION`: a provisional model that must be replaced by measurement or
  a guaranteed selected-part value.

Typical values are not guaranteed maxima. Linear TVS interpolation, efficiency
models, `RθJA`, leakage allocations, source impedances, and rectangular-pulse
energy estimates are screening tools, not compliance evidence.

## Decision summary

The Task 5A.1 power path is:

```text
OBD pin 16
  -> 0437002.WRA 2 A fuse
  -> VBAT_FUSED_CLAMPED
     +-> common-anode anti-series SM15T47AY + SM15T33AY shunt -> POWER_GND
  -> LM74720QDRRRQ1 + 2 x STL125N10F8AG back-to-back N-MOSFETs
  -> post-switch damped input filter
  -> FILTERED_VEHICLE
     -> LMQ66420MC3RXBRQ1 -> VEH_3V3 -> TPS2116DRLR VIN1
     -> normally-off LMQ66420MC5RXBRQ1 -> AUX5

USB-C receptacle
  -> USBLC6-2SC6Y on D+/D- with its VBUS shunt/reference only
  -> Type-C sink termination on CC1/CC2
  -> VBUS -> TPS2553QDBVRQ1, 60.4 kΩ ±1% ILIM starting value
  -> TPS62162QDSGRQ1 fixed 3.3 V -> USB_3V3
  -> TPS2116DRLR VIN2

TPS2116 VOUT -> MAIN_3V3 core domain
VEH_3V3       -> TCAN3404-Q1 VCC and vehicle-only enables
```

This separates four functions that the old circuit coupled together:

1. the fuse and TVSs absorb or isolate defined source faults;
2. LM74720 and the 100 V FET pair provide polarity, reverse-current and
   overvoltage disconnection;
3. regulated-rail selection prevents USB-to-vehicle backfeed;
4. software-visible VBAT, PGOOD and CAN state describe the vehicle state
   without asking one imprecise UV comparator to distinguish 5 V USB from a
   6 V vehicle sag.

The old `LM74502QDDFRQ1`, `DMT6007LFGQ-7`, `SM8SF24CA-Q`, and
`PMEG6030EP-Q` selections are superseded.

### UV protection versus vehicle-state alternatives

The former 5 V USB-versus-6 V vehicle crossover is not a product
requirement. These four options were compared on the same criteria before
removing that responsibility from the protection controller:

| Option | Guaranteed threshold / hysteresis / temperature | Startup, USB and failure behavior | Current / qualification / procurement | Decision |
|---|---|---|---|---|
| A — retain LM74502 EN/UV | EN rise is 1.16–1.32 V and fall is 1.027–1.235 V over the data-sheet limits; the high-value divider also has an incompletely bounded sink-current term, so no exact 5 V/6 V window is guaranteed | One analog node still couples safety, USB crossover and vehicle state; a false decision can chatter or power the wrong source domain | 45 µA typical/65 µA maximum; AEC-Q100; TI active; about $1.75 comparison price | Rejected |
| B — separate TPS37A010122DYYRQ1 supervisor | Adjustable thresholds are ±1.5% over -40…125 °C; selectable hysteresis is 2–13% with ±1.5% hysteresis accuracy. A complete external-divider calculation was intentionally not frozen after the crossover requirement was removed | Startup is at least 2 ms after VDD reaches 2.7 V; below the 1.4 V power-on-reset point the output is undefined. Open-drain defaults therefore depend on the external pull-up/combining logic, and USB priority still remains independent | 1 µA typical/2 µA maximum supply current, plus up to 0.8 µA SENSE current below 10 V; AEC-Q100 grade 1; -40…125 °C; TI ACTIVE. Exact release price/stock remains unresolved because this option is rejected | Rejected because it adds logic for a 5 V/6 V threshold the product does not require |
| C — LTC4368HMS-2 precision integrated UV/OV/breaker | ±1.5% 0.5 V comparators over the automotive-temperature model; the screened divider gives UV fall 4.895–5.113 V and restart 5.095–5.434 V | Approximately 32 ms startup and defined breaker behavior default the external FETs off during invalid input; its -40 V input limit conflicts with the selected negative clamp, and a divider/open-pin fault can still defeat the intended state window. It must not decide USB priority | 81.2 µA typical/227 µA maximum screened supply current; automotive W model; production; about $7.57 comparison allowance | Rejected for negative-event coordination, parked current and cost |
| D — separate safety from state | LM74720 native OV controls safety; no precision analog UV crossover is claimed. Vehicle-state policy uses measured VBAT, PGOOD, CAN activity, USB presence and timers. The 9/10 V thresholds, hysteresis and temperature error remain ADC/firmware design requirements | Hardware defaults optional loads and CAN authorization safe/off until rails and state are valid. USB selection occurs only between independently regulated 3.3 V rails; loss of ADC/firmware state leaves optional loads off rather than selecting a power source | No extra supervisor current, package or procurement line; LM74720 and TPS2116 evidence is included in Architecture B | **Selected** |

Option D eliminates the UV contradiction rather than hiding it in resistor
nominals. A future production state detector may still be added only after its
threshold, hysteresis, temperature, startup and failure defaults are
tolerance-screened against the state policy; it must not become the primary
surge-protection decision.

## Requirement tree

```text
Automotive source
├─ normal operation: 12 V nominal, 14.4 V charging, operate through 18 V
├─ low voltage
│  ├─ preserve core telemetry where practical at 6 V
│  ├─ shed optional loads before fuse/converter stress
│  └─ controlled reset when regulation cannot be maintained
├─ positive events
│  ├─ +26 V for 60 s: OV/PD-low command before 26 V; no sustained TVS avalanche
│  ├─ defined +38 V suppressed event: post-response disconnect intent; no intended avalanche
│  └─ +50 V / 2 Ω / 0.05 ms reference: source-bounded pulse screening
├─ negative events
│  ├─ -14 V for 60 s: no sustained TVS avalanche or fuse opening
│  └─ -100 V / 10 Ω / 2 ms reference: clamp, FET and fuse coordination
├─ source isolation
│  ├─ vehicle-only
│  ├─ USB-only
│  ├─ both sources
│  └─ neither source
├─ conversion
│  ├─ MAIN_3V3 continuous core and managed RF/load peaks
│  └─ AUX5 street, track and managed-peak contracts
├─ parked current: <1.0 mA required; <0.50 mA stretch target
├─ thermal and EMI: hot enclosure, load-step, switch-node and GNSS coexistence
└─ manufacture: inspectable packages, controlled parts and accessible tests
```

Task 5A.1 deliberately changes two former requirements:

- `6–18 V full operation` becomes core/CAN/wake survival at the low end with
  defined load shedding; all maximum loads need not run at 6 V.
- `operate through 18 V and disconnect by 20 V while keeping the protected
  node at or below 24 V` becomes guaranteed no static OV trip through 18 V,
  reconnect eligibility once the source returns to 18 V, and an OV/PD-low
  command before 26 V. The modeled static rising threshold can reach
  25.531 V; completed isolation/reconnection time and the dynamic downstream
  peak remain model-and-capture gates.

All selected downstream input parts must therefore tolerate at least the new
25.531 V static threshold envelope plus measured switching overshoot. The
LMQ66420 inputs are rated for 36 V operation and 42 V absolute maximum, but
that fact does not close the dynamic test gate.

## Architecture alternatives

| Architecture | Protection and source selection | Strengths | Weaknesses | Decision |
|---|---|---|---|---|
| A — retained LM74502 with relaxed OV and regulated-rail mux | Selected fuse/TVSs/100 V FETs, LM74502 native OV, the same filter/converters, and TPS2116 after separate vehicle/USB conversion | Complete low-IQ architecture; removes the false 5 V/6 V UV crossover; slightly lower part cost | LM74502 does not reverse-block while enabled, so stored downstream energy can return toward a collapsing vehicle source; slower gate source and no benefit worth losing RCB | Rejected |
| B — LM74720 native OV plus regulated-rail mux | LM74720 true reverse-current blocking, 100 V FETs, asymmetric TVSs, independent vehicle and USB 3.3 V rails, TPS2116 mux | Lowest defensible always-on current, clean four-state behavior, no 5 V/6 V comparator conflict, simple relaxed OV window | Loose native OV requires the deliberate requirement change; no current breaker; TPS2116 is not automotive-qualified | **Selected conditionally** |
| C — LTC4368 surge stopper/breaker | `LTC4368HMS-2#WPBF`, 100 V FETs, 12 mΩ current shunt, 40 V positive TVS and a lower-clamp negative TVS, with the same filter/converters/mux | Accurate windows, true reverse cutoff and bidirectional current protection | 81.2 µA typical and 227 µA maximum combined supply current; the required low negative clamp leaves little margin above -14 V and raises negative pulse/fuse stress | Rejected for v1 |

The following matrix applies the same criteria to all three complete
architectures. Relative component-count and PCB-area entries are screening
results because no schematic has been captured; they are not invented placed
part counts or routed-area measurements.

| Criterion | A — LM74502 | B — LM74720 | C — LTC4368-2 | Selection implication |
|---|---|---|---|---|
| Electrical coverage | Contains the same fuse, TVS pair, FET pair, filter, converters and regulated-rail mux as B, but the controller does not block reverse current while enabled | Covers fuse, asymmetric positive/negative clamping, reverse polarity, enabled reverse-current blocking, OV disconnect, filtering, both conversions and all four source states | Covers the same functional blocks and adds bidirectional overcurrent sensing/breaker behavior | Only B closes every required v1 function without adding a breaker or retaining an enabled reverse path |
| Positive-event margin | With the same 498 kΩ/28 kΩ, 0.1% divider and ±1 µA OV-pin-current model, calculated OV rise is 21.346–25.587 V and fall is 19.434–23.498 V | Calculated OV rise is 20.690–25.531 V, so the OV/PD-low command occurs before +26 V; +38 V is below the 40.2 V TVS standoff, and the +50 V screen is source bounded. Completed opening and the downstream peak remain dynamic gates | The exact 1.96 MΩ/160 kΩ/57.6 kΩ divider calculates OV trip at 18.542–19.265 V; it is reproducible but discards the deliberate relaxed-window benefit | A and B both fit the relaxed static-command requirement; B retains reverse-current blocking, and all three still require dynamic testing |
| Negative-event and reverse margin | Shares B's 100 V FET/TVS concept, but stored output energy can flow toward a collapsing input because enabled reverse-current blocking is absent | The -100 V screen produces a 74.731 V held-output stress, leaving 10.269 V to the controller C-to-A absolute limit and 25.269 V to the FET rating; -14 V remains below TVS standoff | Reusing the selected negative clamp can hold the controller input near -49.2 V, violating its -40 V absolute limit. Substituting `SM15T18AY` leaves only 1.3 V standoff margin above -14 V and raises the modeled negative-pulse fuse integral to about 0.13 A²s | B provides the strongest documented reverse-source behavior and static negative margin |
| Converter and front-end thermal | Downstream MC3/MC5 thermal screens are identical to B; front-end hot loss was not separately recalculated | MC3 separate 1.050 A case screens to 102/105/108 °C; MC5 TRACK to 111/115/119 °C and MAX to 118/123/128 °C at 12/14.4/18 V and 85 °C ambient; FET-pair loss is 14.3 mW at the permitted 10 V MAX current | Downstream screens are identical, but the current shunt and breaker fault/SOA loss add uncalculated heat | B is the only alternative with a complete documented paper thermal screen; measurement remains mandatory |
| Component count | Same major functional blocks as B; controller-specific support-passive count is not frozen | Baseline major blocks; support population is enumerated below | Adds at least one current shunt plus Kelvin routing and a different divider/support network | A and B are comparable; C is unambiguously higher, while exact placed counts await capture |
| PCB area | Comparable to B; small controller package but no captured layout | 3 x 3 mm wettable-flank controller, two 5 x 6 mm FETs, and the same filter/converters as A | 3 x 3 mm DFN or 10-pin MSOP controller plus shunt/Kelvin area and the same large filter/converters | C has the largest protection footprint/routing burden; no numeric board area is claimed before placement |
| Manufacturability | Small SOT-23-THN controller and otherwise the same assembly issues as B | Wettable-flank WSON/VQFN and PowerFLAT packages aid optical inspection; exposed pads, surge loop, thermal vias and filter population still need assembler review | Adds precision-shunt tolerance, Kelvin routing and breaker fault-energy verification | B is preferable to C and has a documented inspectability path; A offers no compensating manufacturing benefit |
| Lifecycle and availability evidence | TI lists `LM74502QDDFRQ1` active; exact release stock and price still require refresh | Exact selected controller/converters are active; the dated snapshot below found authorized stock, while TPS2116 remains catalog/non-AEC | ADI lists the exact automotive `LTC4368HMS-2#WPBF` as production/recommended for new designs; its higher cost and current remain disadvantages | B has the strongest exact-order-code procurement evidence for the selected chain; every option needs BOM-release refresh |
| Screened major-power BOM cost | $34.55 | $35.43 | $41.01 | Controller price is not the comparison; all three include the complete major power chain |
| Support-adjusted architecture comparison | about $35.55 | about $38.38 | about $42.01 | Includes explicit controller-specific support allowances below; uncertainty is too large to use the small A/B delta as the selection reason |
| 12 V parked-current envelope | about 0.468 mA | about 0.408 mA | about 0.757 mA | All pass <1.0 mA on paper; B has the lowest envelope and the selected architecture's full itemized budget |

The quantity-one comparison is a **power-architecture comparison**, not a
complete populated-board BOM. The major-part column includes the front end,
both vehicle converter ICs, selected vehicle-filter population, USB
switch/buck IC and source mux. It excludes items common by construction to
A/B/C: the two `XGL5030-222MEC` converter inductors and their LMQ
capacitor/support networks, the USB `XFL3012-222MEC` and its exact
capacitors/bulk capacitor, connectors, PCB and assembly. Those common
exclusions do not change relative cost. Controller-specific support is not
common, so it is shown as an explicit planning allowance rather than silently
omitted. Using the dated observations and allowances below gives these
reproducible `ASSUMPTION` screens:

| Architecture | Major-part arithmetic | Major parts | Controller-specific support allowance | Support-adjusted comparison | 12 V parked envelope |
|---|---|---:|---:|---:|---:|
| A | `$35.43 - $2.63 LM74720 + $1.75 LM74502` | `$34.55` | `$1.00` for divider/gate/support passives not yet frozen | `$35.55` | `2 x (204.013 + 30) = 0.468026 mA` |
| B | itemized sum below | `$35.43` | `$0.95 LPS3015 + $2.00` for the exact caps, zener, divider and slew/PD resistors | `$38.38` | `2 x 204.013 = 0.408026 mA` |
| C | `$35.43 - $2.63 LM74720 + $7.57 LTC4368 + $0.50 shunt + $0.14 TVS allowance` | `$41.01` | `$1.00` for divider/support passives not yet frozen | `$42.01` | `2 x (375.821 + 2.888) = 0.757418 mA` |

The Architecture A parked delta replaces the selected 35 µA controller maximum
with the LM74502 65 µA maximum. Architecture C is traceable as
`204.013 - 35.000 - 22.814 + 227.000 + 5.510 = 378.709 µA`: remove
LM74720 and its divider, then add the LTC4368 multi-pin maximum and its
5.510 µA divider current. The selected architecture's existing 2 µA TVS
allocation is provisionally retained; a C-specific TVS leakage population
was not frozen because C is rejected. Doubling gives 0.757418 mA. These are
comparison estimates, not quotes or final populated-BOM totals. The support
allowances carry at least ±$1 planning uncertainty, so cost did not decide
between A and B; reverse-current behavior did.

The Architecture B screen is traceable as `$1.99 fuse + $1.96 TVSs + $2.63
controller + $6.28 FETs + $8.60 vehicle-converter ICs + $2.62
vehicle-filter XEL4030V inductor + $4.08 vehicle-filter direct MLCCs + $1.67
hybrid + $0.95 damping resistor + $1.60 USB switch + $2.04 USB buck IC +
$1.01 mux = $35.43`.

### Manufacturer and best-practice survey

| Manufacturer | Serious parts/approach reviewed | Disposition |
|---|---|---|
| Texas Instruments | LM74502, LM74720, LM7480/LM74930, TPS37-Q1, TPS48110-Q1 and TIDA-00699/TIDA-01167 | LM74720 selected; LM74502 lacks enabled RCB; LM7480/LM74930 consume too much parked current; TPS37 adds precision that the relaxed window does not require; TPS48110 is a high-current eFuse supplement, not a physical-fuse replacement; reference designs control the staged test method, not compliance |
| Analog Devices / Linear Technology | LTC4368-2 precision UV/OV and bidirectional breaker | Technically strong, but its -40 V input limit forces a lower negative clamp and its maximum quiescent current/cost are worse for v1 |
| NXP | FS27/FS23 and UJA1163-class system-basis approaches | Useful integration references; FS27 remains preproduction, FS23 access is restricted, and reviewed regulated outputs do not replace the required rail set |
| Infineon | IAUA210N10S5N024 100 V and IAUCN08S7L018 80 V automotive FETs plus TLE system-basis families | The 100 V part is a credible larger/lower-resistance fallback; the 80 V part has only 5.269 V static margin before overshoot; system-basis outputs do not fit the load tree |
| STMicroelectronics | SM15T TVSs, STL125N10F8AG and STPM801 | TVSs/FET selected; STPM801 is capable but draws 5–9 mA with drivers retained in standby, incompatible with parked operation |
| Littelfuse | 437A fuse and TPSMD asymmetric TVS pair | Fuse selected; TPSMD40A+TPSMD28A retained as a higher-power TVS fallback |
| Bourns | SM8SF bidirectional TVSs and MF-LSMF PPTC | Single 33 V TVS overlaps the +38 V event; PPTC voltage/temperature derating is unsuitable |
| Vishay | WSL pulse-application resistor family and automotive series-diode USB isolation | WSL2512 selected as the filter damping-resistor family; series-diode USB isolation rejected because worst-case forward drop leaves inadequate conversion/startup headroom |
| Diodes Incorporated | DMT6007 and automotive DMT10H020SDGQ dual MOSFETs | 60 V DMT6007 is invalid; the 100 V dual alternative is qualified but materially more resistive and had no dated authorized stock |

Common practice across the useful references is to keep surge absorption,
reverse/OV disconnection, EMI filtering, regulated source selection and state
detection as separately verified functions. No one component rating proves
the assembled path.

Higher-integration automotive system-basis chips were also screened. STPM801
consumes milliamps when the protected path is retained; NXP FS27 is
preproduction; NXP FS23 access is restricted; NXP UJA1163 and Infineon
TLE9471/TLE9261 regulated outputs are below this system's peak requirements.
They are useful architecture references, not better v1 substitutions.

### Component-candidate disposition tables

These tables record why reasonable alternatives were rejected or retained.
Open means an exact implementation was not calculated far enough to be
capture-ready; it does not mean an unverified part may be substituted.

#### Fuse strategies

| Exact candidate | Qualification / electrical fields | Coordination and thermal consequence | Procurement / manufacture | Disposition |
|---|---|---|---|---|
| 0437002.WRA | Littelfuse 437A; AEC-Q200; -55 to +150 °C; 1206 fast acting; 2 A, 63 V; 62 mΩ nominal; 0.144 A²s nominal melt; 0.20 V/0.40 W at rating | Manufacturer 80% continuous rule gives 1.60 A at 25 °C and the 75 °C example gives 1.36 A; lowest fault energy of the compared one-time fuses | Standard 1206 tape/reel; dated authorized stock about 13,773; $1.99 quantity-one screen | **Selected conditionally**; hot time-current, interrupt, inrush and failed-short-TVS clearing remain tests |
| 043702.5WRA | Same Littelfuse AEC-Q200 437A family/package; 2.5 A, 63 V; 43 mΩ nominal; 0.441 A²s nominal melt; 0.15 V/0.375 W at rating | Continuous-use screen is 2.00 A at 25 °C and 1.70 A in the 75 °C example, but nominal melting I²t is 3.06 times the selected 2 A part | Same assembly process; dated authorized stock about 1,859; $1.99 quantity-one screen | Rejected unless measured inrush requires it and all harness/TVS/fault coordination is repeated |
| 0437003.WRA | Same Littelfuse AEC-Q200 437A family/package; 3 A, 63 V; 35 mΩ nominal; 0.506 A²s nominal melt; 0.14 V/0.42 W at rating | Continuous-use screen is 2.40 A at 25 °C and 2.04 A in the 75 °C example; nominal melting I²t is 3.51 times the selected fuse | Same standard 1206 process; authorized stock observed, exact release price requires refresh | Rejected unless measured inrush opens the 2 A option and the entire fault/harness coordination is redone |
| MF-LSMF200/33X | Bourns Multifuse; AEC-Q200; 2920; 33 V maximum, 40 A interrupt rating, 2.0 A hold/4.0 A trip at 23 °C; 25–120 mΩ | Hold-current derating is about 0.90 A at 85 °C and 33 V does not span the defined positive-event set | Larger footprint; resettable behavior complicates hot/repetitive fault evidence; no procurement/cost advantage was established before electrical rejection | Rejected as the primary input fuse |
| TPS48110AQDGXRQ1 plus external shunt/FETs | TI active, AEC-Q100 grade 1; 3.5–80 V operating, 100 V absolute; 613 µA typical active, 1.6 µA shutdown; 19-pin VSSOP; -40 to +125 °C | Adjustable two-level overcurrent/4 µs short-circuit protection can supplement a fuse, but requires a shunt, external FET SOA, timer and separate upstream short-clearing proof | Production exact suffix; additional Kelvin routing, exposed fault-energy analysis and materially higher parked current; release stock/cost require refresh | Rejected for v1 and **cannot replace the physical fuse** |

An electronic limiter may supplement but cannot replace the physical fuse.
The exact TPS48110 option shows why it was not added: enabled current, shunt,
FET SOA, fault timer and upstream short-clearing create a materially different
and higher-IQ architecture.

#### Protection controllers

| Exact candidate | Qualification / range / current | Relevant protection behavior | Package / implementation burden | Disposition |
|---|---|---|---|---|
| LM74502QDDFRQ1 | TI active, AEC-Q100 grade 1; 3.2–65 V; -65 V reverse input; 45 µA typical/65 µA maximum operating | Reverse-polarity and OV/UV disconnect, but no enabled reverse-current blocking; 498 kΩ/28 kΩ at 0.1% with ±1 µA pin-current model gives OV rise 21.346–25.587 V and fall 19.434–23.498 V | 8-pin SOT-23-THN; same two-FET/filter/converter chain as B | Architecture A rejected: its exact relaxed divider works statically, but it does not close source-collapse reverse current |
| LM74720QDRRRQ1 | TI active, AEC-Q100 grade 1; 3–65 V; -60 to +65 V recommended at A; 27 µA typical/35 µA maximum operating | True enabled reverse-current blocking, separate ideal-diode/disconnect gates and native OV; no current breaker | 3 x 3 mm wettable-flank WSON; exact support network enumerated below | **Selected conditionally** as Architecture B |
| LM74800QDRRRQ1 | TI active, AEC-Q100 grade 1; 3–65 V; 397 µA typical at 12 V, 413 µA typical at 24 V and 495 µA maximum operating | Fast ideal-diode reverse blocking and OV/load-dump control | 12-pin wettable-flank WSON; materially larger parked-current allocation | Rejected for the parked-current target |
| LTC4368HMS-2#WPBF | ADI production/recommended for new designs, AEC-Q100; 2.5–60 V operation; -40 V reverse protection; 81.2 µA typical/227 µA maximum combined current for the screened multi-pin case | ±1.5% UV/OV and -3 mV reverse breaker threshold; 1.96 MΩ/160 kΩ/57.6 kΩ gives OV 18.542–19.265 V, UV fall 4.895–5.113 V and restart 5.095–5.434 V; 12 mΩ gives 3.33–5.00 A forward trip | 10-pin MSOP plus shunt/Kelvin route; selected -49.2 V clamp violates -40 V absolute, while SM15T18AY leaves only 1.3 V above -14 V and about 0.13 A²s modeled fuse stress | Architecture C rejected for current, cost, added fault path and negative-clamp coordination—not for missing exact evidence |
| TPS37A010122DYYRQ1 | TI active, AEC-Q100 grade 1; 2.7–65 V; about 1 µA typical; -40 to +125 °C | Two independent OV/UV detector channels, programmable delays and 65 V sense pins; it supervises but does not itself provide FET drive or reverse-current blocking | 14-pin SOT-23-THN plus a separate ideal-diode/FET driver and combining logic | Rejected: precision UV is unnecessary under the state-machine policy and the split implementation adds parts without closing RCB |

LM74930-Q1 was also screened at family level: its circuit breaker and surge
features are useful, but TI publishes 650 µA typical operating current and a
24-pin 4 x 4 mm VQFN. No exact orderable suffix was selected, so it is not
presented as a capture candidate.

#### Coordinated TVS networks

| Exact candidate/network | Manufacturer / qualification / package / temperature | VRWM; VBR; leakage | 10/1000 µs pulse and clamp point | Event result | Dated availability / cost; disposition |
|---|---|---|---|---|---|
| SM15T47AY + SM15T33AY | ST; active; AEC-Q101; unidirectional SMC; -55 to +150 °C | 40.2/28.2 V; 44.7–49.4/31.4–34.7 V; each 0.2 µA max at 25 °C and 1 µA max at 85 °C | 1.5 kW each; 64.5 V at 23.2 A / 45.7 V at 33 A | +26/+38 remain below positive-leg standoff; +50 screen is 0–1.507 A and 0–3.54 mJ; -100 screen is 6.223–6.479 A and 0.456–0.470 J; held-output screen 74.731 V | Authorized stock observed; about $0.98 each; **selected conditionally** in common-anode anti-series form |
| TPSMD40A + TPSMD28A | Littelfuse; AEC-Q101; unidirectional DO-214AB/SMC; -65 to +150 °C | 40/28 V; 44.4–49.1/31.1–34.4 V; 2 µA max each | 3 kW each; 64.5 V at 46.5 A / 45.4 V at 66.1 A | Same asymmetric intent and higher nominal pulse class, but every positive/negative energy, forward-drop and held-output calculation must be rerun | Authorized stock observed; about $1.44 + $1.08; retained fallback, not a drop-in substitution |
| SM8SF24CA-Q | Bourns; AEC-Q101; bidirectional 8.1 x 10.5 mm DFN; -55 to +175 °C | 24 V; 26.7–29.5 V; 10 µA max | 7 kW; 38.9 V at 180 A | +26 V for 60 s is not below standoff/breakdown; cannot satisfy the continuous positive event | Authorized stock observed; about $3.92; rejected |
| SM8SF33CA-Q | Bourns; AEC-Q101; bidirectional 8.1 x 10.5 mm DFN; -55 to +175 °C | 33 V; 36.7–40.6 V; 10 µA max | 7 kW; 53.3 V at 131 A | +38 V lies inside the breakdown tolerance band and the symmetric negative clamp reduces held-output margin | Dated stock about 3,732; about $3.92; rejected |

#### Back-to-back MOSFETs

| Exact candidate | Qualification / VDS / VGS | RDS(on) / gate charge | SOA, avalanche, thermal and package / drive | Dated availability / price; margin and disposition |
|---|---|---|---|---|
| STL125N10F8AG, quantity 2 | ST active, AEC-Q101; 100 V; ±20 V; -55 to +175 °C | 4.6 mΩ max at 10 V; 56 nC typical Qg | 140 mJ EAS at 60 A, 100% avalanche tested and published SOA; RθJA 16.2 °C/W and RθJC 1.0 °C/W under stated conditions; wettable PowerFLAT 5 x 6, compatible with LM74720 | Dated stock about 2,980; $3.14 each; 25.269 V static margin above 74.731 V; **selected conditionally**, dynamic SOA/gate proof remains |
| IAUA210N10S5N024AUMA1, quantity 2 | Infineon active/preferred, AEC-Q101; 100 V; ±20 V; -55 to +175 °C | 2.4 mΩ max at 10 V; Qg 91 nC typical/119 nC maximum | 245 mJ EAS at 105 A, 100% avalanche tested and published SOA/thermal data; sTOLL 7 x 8, compatible voltage class but larger gate charge/area | Dated stock about 3,147; about $5 each; availability planned through at least 2038; credible fallback pending pair gate/SOA/thermal proof |
| IAUCN08S7L018ATMA1, quantity 2 | Infineon active/preferred automotive, extended qualification beyond AEC-Q101; 80 V; ±16 V; -55 to +175 °C | 1.8 mΩ max at 10 V; Qg 79.9 nC typical/103.9 nC maximum | Manufacturer claims high avalanche-current/SOA ruggedness; exact pulse proof remains; 5 x 6 mm SSO8/PG-TDSON-8, compatible drive voltage only after VGS clamp review | Dated stock about 4,705; about $4.51 each; only 5.269 V static VDS margin above 74.731 V before overshoot, so rejected |
| DMT10H020SDGQ-13, quantity 1 dual | Diodes Incorporated; AEC-Q101, PPAP capable/IATF 16949; 100 V; ±20 V | 23 mΩ max per channel at 10 V; 14.5/14.3 nC typical Qg | Published SOA but no avalanche number was credited to this screen; compact PowerDI3333-8 dual; voltage/drive compatible but roughly five times selected resistance per channel | Dated authorized stock zero; about $0.598 indication; rejected for conduction loss and supply risk, not qualification |
| DMT6007LFGQ-7, quantity 2 | Diodes Incorporated automotive; 60 V class | Historical low-resistance candidate | Former PowerDI implementation; its voltage class fails before SOA/thermal comparison matters | Cannot survive the 74.731 V held-output screen even before overshoot; rejected as electrically invalid |

#### AUX5 converters

| Exact candidate | Qualification / operating fields | Thermal / parked / layout consequence | Availability / cost evidence | Disposition |
|---|---|---|---|---|
| LMQ66420MC5RXBRQ1 | TI active, AEC-Q100 grade 1; fixed 5 V, 2 A; 3.6–36 V startup, 3–36 V running; 2 µA typical no-load IQ for the 5 V option; shutdown 0.25 µA typical/1 µA max; 2.2 MHz; -40 to +150 °C junction | TRACK and MAX screens are documented above; compact wettable 2.6 x 2.6 mm 14-pin VQFN, but hot MAX margin is limited | Authorized stock observed; $4.30 screen | **Selected conditionally** under the named mode/duty policy |
| LMQ66430MC5RXBRQ1 | TI active, AEC-Q100 grade 1; fixed 5 V, 3 A; same input range, current consumption, frequency and 14-pin VQFN family | Published switch resistances/package match MC5, so the higher current limit does not reduce calculated thermal loss and raises fault energy | Mouser dated stock about 1,185 at $4.28; DigiKey zero | Rejected as a non-solution to the actual thermal risk |
| LM63635DQDRRRQ1 | TI active, AEC-Q100 grade 1; adjustable/fixed 5 V, 3.5–36 V, 3.25 A; 23 µA typical/40 µA max IQ; shutdown 5.3 µA typical/10 µA max; 250 kHz–2.2 MHz; -40 to +150 °C junction | Lower-loss-capable fallback, but materially raises parked current and requires a different inductor/compensation/layout/thermal calculation; 12-pin wettable WSON | Dated stock about 4,859; $4.56 | Open fallback only if MC5 thermal testing fails |
| LM53635LQRNLTQ1 | TI active, AEC-Q100 grade 1; fixed 5 V, 3.9–36 V, 3.5 A, 2.1 MHz; about 20 µA typical IQ for the 5 V option; shutdown 2 µA typical/5 µA max; -40 to +150 °C junction | Larger 5 x 4 mm, 22-pin wettable VQFN-HR and higher current/cost burden; no v1 rail thermal or passive calculation was completed | Dated stock about 475; $7.82 | Open fallback only if MC5 thermal testing fails |

#### Vehicle-input filter approaches

| Candidate topology / exact population | Damping and loss behavior | Manufacturability / area | Disposition |
|---|---|---|---|
| XEL4030V-222MEC + four CGA6P3X7R1H475K250AB + WSL2512R3900FEA in series with EEH-ZC1H121P | Direct-C range 9.4–20.68 µF; calculated full-corner f0 21.54–39.13 kHz and Q 0.69–1.37; damping branch avoids steady load-path loss | Largest compared population: inductor, four populated plus one DNP large MLCC footprints, SMC hybrid and 2512 resistor; all are inspectable but require pulse/void/polarity rules | **Selected prototype start**; impedance, pulse and EMI proof remain open |
| Same XEL/direct MLCC bank + WSL2512R2200FEA + EEH-ZC1H101P | Panasonic AEC-Q200 hybrid is 100 µF ±20%, 50 V, 28 mΩ max ESR, 50 µA max leakage; 0.22 Ω + ESR gives about 0.248 Ω branch resistance, which is underdamped at the screened corner; 80 µF minimum also misses `4 x 20.68 = 82.72 µF` | Same 10 x 10.2 mm polarized hybrid and 2512 process as selected, but no area benefit. Its lower leakage would slightly improve, not break, the parked-current screen; capacitance and damping are the rejection reasons. | Rejected |
| Same XEL/direct MLCC bank + WSL2512R3900FEA + EEH-ZC1V680XV | Panasonic AEC-Q200 hybrid is 68 µF ±20%, 35 V, 35 mΩ max ESR and 23.8 µA max leakage; its 54.4 µF minimum misses `4 x 20.68 = 82.72 µF`, and static voltage headroom above 25.531 V is only 9.469 V before overshoot | Smaller 6.3 x 7.7 mm vibration-resistant can, but 35 V rating and capacitance do not close the protected-node dynamic/minimum-C gates | Rejected |
| Same XEL/direct MLCC bank + WSL2512R4700FEA + UCD1J101MNL1GS | Nichicon UCD AEC-Q200; 100 µF ±20%, 63 V, -55 to +105 °C; 63 µA leakage limit; 80 µF minimum misses 82.72 µF and no usable maximum ESR at the 21.54–39.13 kHz resonance was credited | 10 x 10 mm SMD aluminum can and 2512 resistor are manufacturable, but lower temperature/lifetime class and uncertain damping add risk | Rejected |
| Plain LC/CLC, no exact population selected | Can form a high-Q resonance with converter negative incremental input impedance; no deterministic damping bound was established | Fewer parts, but apparent simplicity does not close stability | Rejected before part selection |
| Series resistor in the full load path, no exact population selected | Deterministic damping but continuous loss and low-voltage drop scale with vehicle load | Simple placement but thermally inefficient | Rejected before part selection |
| Ferrite-only, no exact population selected | Useful for high-frequency tuning but no deterministic low-frequency damping | Smallest likely area; bias/frequency impedance still requires measurement | Not the primary filter; a DNP tuning footprint may remain |

#### USB source isolation

| Exact candidate / approach | Four-state isolation and voltage behavior | Qualification / implementation | Disposition |
|---|---|---|---|
| TPS2553QDBVRQ1 -> TPS62162QDSGRQ1 -> TPS2116DRLR | Converts VBUS independently, then muxes USB_3V3 with VEH_3V3; CAN and AUX5 remain vehicle-only; reverse-current blocking and break-before-make support all four states | Limiter and buck are automotive-qualified; mux is catalog/non-AEC and specified here only through 105 °C; startup capacitance, current-limit and handoff gates remain | **Selected conditionally** |
| PMEG6030EP-Q raw-source Schottky OR before one vehicle converter | 60 V/3 A part provides passive directionality, but diode drop reduces USB headroom and the common raw rail recreates the 5 V USB versus 6 V vehicle threshold conflict; hot reverse leakage remains | AEC-Q101/CFP5, simple assembly, but no controlled priority or regulated-domain isolation | Superseded/rejected |
| PMEG6030EP-Q passive OR after separate 3.3 V converters | Avoids the raw crossover but loses regulated voltage in the diode, provides no clean vehicle priority, and adds load-dependent droop | Simple and qualified diode; thermal/drop proof would still be required | Rejected versus the low-loss mux |
| Two LM66100QDCKRQ1 ideal-diode stages after separate converters | Low-loss 3.3 V ORing with reverse-current blocking, but the simple dual circuit has no deterministic vehicle priority and TI notes that both paths can turn off when equal | AEC-Q100; 140 mΩ maximum is specified at 3.6 V through 125 °C; 192 °C/W comparison metric and up to 8 µA reverse leakage | Rejected: it does not close priority, loss or parked current with fewer gates than TPS2116 |
| Discrete ideal-diode pair, exact controller/FETs not selected | Could provide low-drop regulated-rail ORing, but no exact reverse leakage, switchover, priority or 105 °C implementation was demonstrated | More passives/routing and no captured automotive-qualified complete solution | Rejected before part selection; not an approved substitution |

## Corrected load model

The historical 12.458 W value incorrectly treated both alternative display
rails and sizing margins as simultaneous loads. The worst named simultaneous
case uses the 5 V display:

```text
vehicle 3.3 V bucket = 0.750 A x 3.3 V = 2.47500 W
AUX5 maximum         = 1.16001 A x 5.0 V = 5.80005 W
total maximum                                8.27505 W
```

The separate 3.3 V-display rail-capacity case is 1.050 A, or 1.313 A after a
25% sizing margin. Its total system output is 6.765 W. It is never added to the
5 V-display case.

For continuity with the earlier budget, the vehicle 3.3 V bucket below is the
total vehicle-source conversion bucket and includes the direct VEH_3V3 CAN
branch. The physical `MAIN_3V3` net is after TPS2116 and excludes CAN. A
USB-only current budget must therefore be rebuilt from the actual core loads,
not copied from the whole vehicle-source bucket.

### Load classifications

| Load | Continuous contract | Managed/short contract | Fault-only value |
|---|---:|---:|---:|
| ESP32-S3 base application | 100 mA at 3.3 V | Wi-Fi 355 mA **or** BLE 344 mA; do not sum | — |
| TCAN3404-Q1 | 8.2 mA | 55 mA dominant-bus case | 130 mA current-limited bus fault |
| NEO-M9N | 50 mA | 100 mA acquisition/unknown high-rate peak | — |
| Active GNSS antenna | 20 mA | — | 50 mA protected bias limit |
| microSD | 100 mA while logging | 200 mA transient; shed after bounded flush | — |
| Display | 500 mA at 5 V **or** 300 mA at 3.3 V | startup transient pending module selection | — |
| Ten-pixel shift light | 60.010 mA street, 120.010 mA track | 360.010 mA full-white | 1 A protected branch envelope |
| Sounder | counted as 300 mA continuously in TRACK thermal screening | program-managed 300 mA | — |
| Miscellaneous | 20 mA | — | — |

The 0.50 A shift branch remains a connector/protection qualification contract;
it is not the normal WS2812 load.

### Operating modes

| Mode | Vehicle 3.3 V bucket | AUX5 | Simultaneous AUX5 consumers | Output power | Contract |
|---|---:|---:|---|---:|---|
| PARKED | deep sleep / standby allocations | off | none | leakage budget below | vehicle attached, optional rails off |
| WAKE | 128.2 mA / 0.42306 W | off | none | 0.42306 W | event qualification |
| CORE | 198.2 mA / 0.65406 W | off | none | 0.65406 W | CAN/GNSS/core telemetry |
| STREET | 298.2 mA / 0.98406 W | 560.01 mA / 2.80005 W | 5 V display 500 mA + shift light 60.010 mA | 3.78411 W | continuous |
| TRACK | 298.2 mA / 0.98406 W | 920.01 mA / 4.60005 W | 5 V display 500 mA + shift light 120.010 mA + sounder 300 mA | 5.58411 W | continuous; conservatively includes sound |
| MAX | 750 mA / 2.475 W | 1.16001 A / 5.80005 W | 5 V display 500 mA + shift light 360.010 mA + sounder 300 mA | 8.27505 W | <=10 s and <=25% duty in any rolling 60 s |
| LOW-VOLTAGE SHED | 198.2 mA / 0.65406 W | off | none | 0.65406 W | no display, shift, sound, Wi-Fi or new SD writes |

The MAX time/duty limit is a `DESIGN_REQUIREMENT`, not implemented firmware.

### Input-current model

For screening:

```text
IIN = P3V3/(VIN x eta3V3) + P5V/(VIN x eta5V)
```

The assumed MAIN efficiencies at 6/8/10/12/14.4/18 V are
0.90/0.91/0.92/0.92/0.91/0.90. AUX efficiencies are
0.88/0.90/0.91/0.92/0.915/0.90. Exact LMQ66420MC5 curves are not published for
this implementation; all values in the following table are `ASSUMPTION +
CALCULATED`.

| Mode | 6 V | 8 V | 10 V | 12 V | 14.4 V | 18 V |
|---|---:|---:|---:|---:|---:|---:|
| WAKE | 0.0783 A | 0.0581 A | 0.0460 A | 0.0383 A | 0.0323 A | 0.0261 A |
| CORE / LOW-VOLTAGE SHED | 0.1211 A | 0.0898 A | 0.0711 A | 0.0592 A | 0.0499 A | 0.0404 A |
| STREET | 0.7125 A | 0.5241 A | 0.4147 A | 0.3428 A | 0.2876 A | 0.2336 A |
| TRACK | 1.0535 A | 0.7741 A | 0.6125 A | 0.5058 A | 0.4242 A | 0.3447 A |
| MAX, arithmetic stress only below 10 V | 1.5568 A | 1.1455 A | 0.9064 A | 0.7496 A | 0.6291 A | 0.5108 A |

Reducing both efficiencies by two percentage points and adding 3% for the
front end gives 1.6406/1.2066/0.9545/0.7892/0.6624/0.5381 A for MAX and
1.1102/0.8154/0.6450/0.5326/0.4467/0.3631 A for TRACK. The 6 V and 8 V MAX
columns are stress arithmetic, not permitted operating modes.

### Low-voltage policy

The later firmware implementation must enforce:

- prohibit MAX, full-white diagnostics and Wi-Fi below 10.0 V;
- below 9.5 V request a bounded SD flush and stop starting optional work;
- below 9.0 V for 100 ms enter LOW-VOLTAGE SHED: AUX5/display/shift/sound off,
  Wi-Fi off, and logging stopped after the bounded flush;
- exit shedding only above 10.0 V stable for 2 s;
- use controlled reset, CAN passive defaults and automatic recovery when
  MAIN_3V3 can no longer regulate.

Thresholds and timers remain `DESIGN_REQUIREMENT` values pending VBAT ADC
tolerance, front-end drop, crank waveforms and measured SD flush time/energy.

## Fuse selection and coordination

`0437002.WRA` is conditionally retained:

- AEC-Q200, 1206, fast acting, 2 A and 63 V;
- 62 mΩ nominal resistance, 0.20 V nominal drop and 0.40 W at rated current;
- 0.144 A²s nominal melting I²t;
- manufacturer continuous-use guidance of at most 80% of nameplate gives
  1.60 A at 25 °C; its 75 °C example gives 1.36 A after combined derating.

The corrected permitted currents fit that paper screen. MAX begins only at
10 V, where the conservative result is 0.955 A and is time limited. TRACK is
0.533 A at 12 V; even the disallowed 6 V TRACK stress is 1.110 A. Below 9 V
the system sheds to CORE. This is why a 2 A fuse is now defensible without
pretending that 12.458 W must operate at 6 V.

A 2.5 A or 3 A fuse reduces little normal loss but raises downstream fault
energy. The Bourns MF-LSMF200/33 PPTC is only 33 V and its hold current falls
to about 0.90 A at 85 °C. TPS4811-Q1-class current limiting adds hundreds of
microamps while enabled and cannot replace the upstream physical fuse.

The 2 A selection remains blocked from production release until tests establish
prospective OBD fault current, 50 A interrupt applicability, trace/connector
and harness protection, hot time-current behavior, converter and capacitor
inrush, failed-short TVS clearing, pulse repetition and aging.

## Reverse/overvoltage controller and thresholds

### LM74720 selection

`LM74720QDRRRQ1` is `VERIFIED_DATASHEET` as active, AEC-Q100 grade 1,
3–65 V operating, -60 to +65 V recommended at A, -65 to +70 V absolute at A,
85 V absolute C-to-A, and supplied in a 3 x 3 mm wettable-flank WSON. It
provides true reverse-current blocking and independent ideal-diode and
load-disconnect gate controls. Operating current is 27 µA typical/35 µA
maximum; shutdown is 1.5 µA typical/3.3 µA maximum.

The gate drive is 9.5–13 V. The reverse threshold is -12 to -1.3 mV and
reverse-current gate turn-off is at most 0.81 µs under the specified test
condition. Those values make USB no-backfeed technically different from the
old LM74502 implementation.

### OV divider

Freeze the prototype values:

```text
RTOP = 249 kΩ + 249 kΩ, each 0.1%
RBOTTOM = 28.0 kΩ, 0.1%
K = 1 + RTOP/RBOTTOM
Kmin = 18.750178
Kmax = 18.821321
```

Two top resistors distribute voltage/pulse stress and reduce contamination
risk at the sensitive node. With LM74720 OV rising threshold 1.13–1.33 V,
falling threshold 1.03–1.215 V, resistor tolerance, and a conservative
±1 µA combined OV-pin/PCB leakage envelope:

```text
VIN(rise,min) = 1.13  x Kmin - 1 µA x RTOP(min) = 20.690 V
VIN(rise,max) = 1.33  x Kmax + 1 µA x RTOP(max) = 25.531 V
VIN(fall,min) = 1.03  x Kmin - 1 µA x RTOP(min) = 18.815 V
VIN(fall,max) = 1.215 x Kmax + 1 µA x RTOP(max) = 23.366 V
```

The divider draws 22.814 µA at 12 V, 27.376 µA at 14.4 V and 34.221 µA at
18 V. Controller plus divider is 57.814 µA maximum at 12 V; allocate 60 µA.

This guarantees no static OV trip through 18 V, reconnect eligibility after
the source returns to 18 V, and an OV/PD-low command before 26 V within the
stated model. It does not bound completed disconnect/reconnect time or the
dynamic protected-node peak, and it does not guarantee the superseded 20 V
cutoff. The 1 µA allowance is deliberately much larger than the IC's published
OV input leakage maximum; layout cleanliness and measurement still matter.

The LM74720 support population is also part of the conditional capture
contract, not an unspecified schematic detail:

- `CGA5H2X7R2A224K115AE`, 220 nF ±10%, 100 V, X7R, soft termination,
  AEC-Q200 and in production, from A to POWER_GND. Its additive
  tolerance-plus-X7R temperature floor is 165 nF before DC bias; installed effective
  capacitance must remain at least 0.1 µF over 0–50 V, temperature and ageing.
  TDK's bias curve is reference-only, so model/lot verification remains a gate;
- tie VS to C, the Q1/Q2 common-drain node, through
  `CRCW06030000Z0EA`, 0 Ω, as in TI's two-FET application;
- `CGA6N3X7R2A225K230AE`, 2.2 µF ±10%, 100 V, X7R, soft termination and
  AEC-Q200, from the C/VS common-drain node to POWER_GND. The 100 V rating
  leaves 50 V static headroom when the defined +50 V source reaches C;
  installed effective capacitance must still be at least TI's 1 µF minimum
  over bias, tolerance and temperature;
- one `CGA6M3X7R1H225K200AE`, 2.2 µF ±10%, 50 V, X7R, soft termination
  and AEC-Q200, from CAP to C/VS. Its installed effective capacitance must
  likewise remain at least 1 µF over the specified differential, tolerance
  and temperature;
- `BZT52H-C18-Q`, 18 V nominal, AEC-Q101, SOD123F, across the disconnect
  FET gate/source in the polarity shown by TI; its 16.8–19.1 V working range
  at the stated test current is below the ±20 V FET gate limit but still
  requires transient validation;
- `LPS3015-104MRC`, 100 µH ±20%, AEC-Q200 grade 1, 3.4 Ω maximum DCR,
  0.24/0.25/0.26 A at 10/20/30% inductance drop, for the boost inductor;
  its 0.24 A 10%-drop current clears TI's greater-than 175 mA requirement;
- `CRCW0603330RFKEA`, 330 Ω ±1%, AEC-Q200, in series with PD. TI requires
  270–330 Ω when the input can exceed 48 V, so the upper value is selected
  to reduce controller pulldown stress;
- `CRCW0603100RFKEA`, 100 Ω ±1%, in series with
  `CGA2B3X7R1H103K050BE`, 10 nF ±10%, 50 V, X7R, soft termination and
  AEC-Q200, for the output-slew network as the prototype starting value.

The post-Q2 `VBAT_SWITCHED` filter shunts are separate components and a
separate electrical node: `CGA2B3X7R1H104M050BB` 100 nF plus
`CGA5L3X7R1H105K160AB` 1 µF to POWER_GND. Neither may be merged with C/VS.

The former `XPL2010-104MLC` example is not selected because Coilcraft marks
the XPL2010 series not recommended for new designs. The replacement
`LPS3015-104MRC` is the current recommended-for-new-designs, halogen-free
order code in the same 100 µH class. Its -40 to +125 °C ambient condition,
current and thermal data are reference ratings, not a guarantee of the
installed boost waveform.

TI data-sheet Sections 6.3, 8.3.1.2 and 9.2.3 control these provisions. Gate
charge, startup, inrush, reverse handoff, capacitor voltage and gate-clamp
stress must be validated before production release.

## Coordinated TVS network

Select two ST automotive unidirectional SMC TVSs in a common-anode,
anti-series connection:

- `SM15T47AY`: cathode to fused VBAT, anode to `TVS_MID`; 40.2 V
  standoff, 44.7–49.4 V breakdown, and 64.5 V clamp at 23.2 A;
- `SM15T33AY`: anode to `TVS_MID`, cathode to POWER_GND; 28.2 V
  standoff, 31.4–34.7 V breakdown, and 45.7 V clamp at 33 A.

Both are active, AEC-Q101, 1.5 kW at the stated 10/1000 µs waveform, and share
the same footprint. The + leg avalanches for a positive event while the - leg
conducts forward; the opposite occurs for a negative event. Place the fuse
upstream so a failed-short TVS is isolated, and keep the connector/TVS/ground
surge loop compact and separate from signal returns.

### Event screen

This ledger separates a defined source model from a component rating. `A`
and `C` are LM74720 pins; the second MOSFET is the PD-controlled disconnect
FET. Conditional voltage statements exclude harness/layout overshoot.

| Event / source model | TVS, controller and FET state | Downstream maximum / basis | A, C, PD, VDS and VGS stress / margin | Recovery / proof gate |
|---|---|---|---|---|
| +12/+14.4/+18 V DC operating points | Both TVSs are below standoff; GATE and PD paths are enabled and both FETs conduct. | At most 18 V minus path drop at the filter input, before normal ripple. | A,C <=18 V: 47 V to the 65 V recommended maximum. FET VDS is conduction drop. Specified `GATE-A=9.5–13 V`: at least 7 V to ±20 V. PD high-state/disconnect VGS still require capture because they lack that same min/max specification. | Measure normal ripple, hot leakage and both gate voltages. |
| +26 V for 60 s; rise/fall and source impedance undefined | Positive TVS stays below its 40.2 V standoff. OV commands PD low; the ideal-diode FET can remain forward biased while the disconnect FET opens. | Static isolation follows from the 25.531 V maximum rising threshold, but the dynamic peak is **not bounded** without the source edge and gate waveform. | Without fixture overshoot, A <=26 V, leaving 39 V to its 65 V recommended limit. C, PD, switching VDS and both VGS are open until nonlinear model/capture because stored filter energy can move protected nodes independently of A. The gate zener's 16.8–19.1 V data-sheet range leaves only 0.9 V from 19.1 V to ±20 V. | Return to 18 V is below the 18.815 V minimum modeled falling boundary, closing static recovery only. PD recharge, inrush, filter ring and recovery time remain open. |
| +38 V suppressed event; amplitude only defined | Positive TVS stays below 40.2 V standoff. It is **not valid** to say the controller is already open: a fast edge traverses OV decision delay and gate discharge. | **OPEN:** duration, edges and source impedance are absent, so energy and output peak cannot be calculated. The sensitivity below approaches the converter absolute limit and is not a bound. | Exactly 38 V without fixture overshoot leaves A 27 V to its recommended maximum. C, PD, switching VDS and both VGS remain open until nonlinear model/capture; nominal zener margin is only 0.9 V. | Define waveform/repetition/temperature, simulate nonlinearly, then capture A/C/PD/VDS/VGS/output. Recovery requires return below the falling boundary and its time is unbounded. |
| +50 V Thévenin, 2 Ω, 50 µs; edges/repetition undefined | TVS tolerance gives no avalanche at one endpoint through 1.507 A at the other: 46.99–50 V line, 0–3.54 mJ pair energy, 0–0.000114 A²s. OV commands opening, but completed-off time is not guaranteed. | **OPEN:** the sensitivity below exceeds inductor saturation, so 30.881 V is not an upper bound. The endpoint TVS arithmetic excludes simultaneous current into the load path. | Without fixture overshoot, A <=50 V, leaving 15 V to its 65 V recommended and 20 V to its 70 V absolute limit. C, PD, switching VDS and both VGS remain open until nonlinear model/capture. | When the source drops below the falling boundary static recovery is allowed. Model/measure TVS heat, total source/fuse current, gate turn-off, filter ring and reconnection. |
| -14 V for 60 s DC reversal | Negative TVS leg stays below 28.2 V standoff. The ideal-diode FET is the reverse barrier; PD FET is not credited as one. | Output bank is initially held then decays; no negative voltage is intentionally passed. | A=-14 V: 46 V to -60 V recommended. At held C=25.531 V, `C-A=39.531 V`: 45.469 V controller margin and 60.469 V FET margin. GATE must return to A so VGS approaches 0 V. | Verify leakage/heating, output decay, PD behavior and restart. TI's 2.9 µs GATE-on maximum uses a 10 nF test load, not this FET's unbounded maximum gate charge. |
| -100 V, 10 Ω, 2 ms rectangular screen; edges/repetition undefined | 33 V leg avalanches and 47 V leg conducts forward; ideal-diode FET opens. Endpoint model: 6.223–6.479 A, 35.208–37.774 V line clamp, 0.07744–0.08396 A²s, 0.456–0.470 J pair energy. | Output is initially held then decays. The separate conservative stress envelope uses A=-49.2 V and C=25.531 V, not the endpoint clamp. | `C-A=74.731 V`: 10.269 V to controller 85 V absolute and 25.269 V to FET 100 V. A has 10.8 V to -60 V recommended and 15.8 V to -65 V absolute. GATE-A should approach 0 V; PD/second FET require capture. | Verify TVS clamp/forward drop over temperature, overshoot, fuse coordination, holdup and restart. This 10.269 V is the smallest static event margin. |

#### Fast positive-event dynamic sensitivity

TI specifies OV-to-PD-low delay at 0.9 µs typical and **1.5 µs maximum**
under its stated conditions. The selected 330 Ω RPD is mandatory for the
defined source above 48 V and changes the gate-discharge waveform; TI's
Figure 6-12, the resistor pulse curve, the FET's typical-only 56 nC gate
charge, Miller plateau, zener current, 100 Ω/10 nF slew branch, parasitics and
moving source must all be included. The former shortcut `56 nC / 55 mA` is
therefore invalid and withdrawn. Completed-off time has no data-sheet-backed
maximum for the assembled network.

- For +38 V, treating the filter's first-cut maximum `Q=1.37` as an ordinary
  second-order stage gives `zeta=1/(2Q)=0.365`, `Mp=29.19%`, and
  `Vpeak=38+0.2919*(38-25.531)=41.639 V`: only 0.361 V below the LMQ66420
  42 V absolute rating. Hybrid topology, unknown source impedance, capacitor
  bias, nonlinear inductance and parasitics make this a `CALCULATED
  ASSUMPTION`, not an upper bound or pass.
- For +50 V/2 Ω only, an illustrative 4 µs conducting window used
  `Va(0)=Vout(0)=25.531 V`, `IL(0)=0`, `Ci=1.22 µF` maximum,
  `Co=9.4 µF` minimum and `L=1.76 µH` minimum, omitting load and damping.
  The equations `dVa/dt=((50-Va)/2-IL)/Ci`,
  `dIL/dt=(Va-Vout)/L`, `dVout/dt=IL/Co` give at 4 µs
  `Va=30.019 V`, `Vout=28.615 V`, `IL=14.310 A`; ideal lossless
  post-opening redistribution gives `Vout,peak=30.881 V`. Reproduction uses
  `Ceq=Ci*Co/(Ci+Co)=1.07985 µF`,
  `Vcm=(Ci*Va+Co*Vout)/(Ci+Co)=28.7763 V`,
  `Vdiff,pk=sqrt((Va-Vout)^2+L*IL^2/Ceq)=18.3226 V`, and
  `Vout,pk=Vcm+Ci/(Ci+Co)*Vdiff,pk=30.8811 V`. Since 14.310 A
  exceeds the inductor's 6.1 A saturation rating and 4 µs is not guaranteed,
  **30.881 V is not a maximum**.

The +26/+38/+50 release gate is exact generator edges/impedance followed by a
manufacturer-model transient of fuse, TVSs, both FETs, PD/slew/zener network
and bias-dependent filter, then hot/cold capture of source current, A, C, PD,
both VDS, both VGS, pre-inductor node, FILTERED_VEHICLE and converter pins.
Every capture must remain inside absolute maximum with the project's release
margin and recover without destructive ring.

Turn-on inrush also remains a conditional SOA gate. The maximum first-order
post-Q2 capacitance allocation is `1.22 + 20.68 + 144 + 2 x 25.3 =
216.5 µF`. TI's `IINRUSH = IPD_DRV x CLOAD / CdVdt` with `IPD_DRV=43–60 µA`
and the selected 10 nF ±10% gives 0.846–1.443 A; the corresponding ideal
0-to-25.531 V ramp is 6.531–3.830 ms. The 330 Ω RPD changes the 60 µA
turn-on source by at most about 20 mV and does not close this screen. Stored
energy is 70.6 mJ and the most aggressive initial linear-FET screen is
36.8 W. These are `CALCULATED` sensitivities, not SOA proof: compare the full
trajectory with hot STL125N10F8AG SOA and validate startup at capacitor
corners before schematic release.

The negative-pulse model assumes about 1 V forward drop in the other TVS.
Using a deliberately conservative 49.2 V network envelope
(45.7 V published negative-device clamp plus a 3.5 V engineering allowance)
and a protected node held at the new 25.531 V static threshold gives:

```text
LM74720 C-to-A and off-FET stress = 25.531 + 49.2 = 74.731 V
margin to LM74720 85 V absolute   = 10.269 V
margin to 100 V FET rating        = 25.269 V
margin from A=-49.2 V to -60 V recommended = 10.8 V
```

ST publishes a typical forward curve rather than a tabulated 3.5 V maximum;
the 49.2 V value is an engineering screen, not a guaranteed pair clamp.

The former `SM8SF24CA-Q` fails because 24 V standoff does not cover +26 V
for 60 s. A single `SM8SF33CA-Q` is not selected because +38 V lies inside
its 36.7–40.6 V breakdown range and its 53.3 V negative clamp leaves less
held-output margin. A Littelfuse `TPSMD40A + TPSMD28A` pair is a credible
same-power fallback with a specified forward limit, but is not the primary
prototype population.

## Power MOSFETs

Use two `STL125N10F8AG` back-to-back:

- active, AEC-Q101 and wettable-flank PowerFLAT 5 x 6;
- 100 V VDS, ±20 V VGS, 175 °C maximum junction;
- 4.6 mΩ maximum at 10 V gate and 56 nC typical gate charge.

A 1.7x hot-resistance screen gives:

```text
RPAIR = 2 x 4.6 mΩ x 1.7 = 15.64 mΩ
```

At the permitted conservative MAX current at 10 V, 0.9545 A, the screen is
14.9 mV and 14.3 mW. At 2 A it is 31.3 mV and 62.6 mW. The low conduction
loss does not prove hot SOA, gate timing or avalanche immunity.

For context only, applying ST's 16.2 °C/W junction-to-ambient comparison
metric to half of the pair loss gives about 0.12 °C rise per FET at the
permitted 10 V MAX screen and 0.51 °C per FET at 2 A. The metric assumes
ST's test board; it is not an installed thermal guarantee. At the same
permitted MAX current the fuse's 62 mΩ nominal cold resistance screens to
56.5 mW and the filter inductor's 22.1 mΩ maximum at 25 °C to 20.1 mW. At
the 12 V TRACK sensitivity current of 0.5326 A those values are 17.6 mW and
6.27 mW. They are comparison/lower-temperature screens, not hot bounds: the
fuse's own 0.40 W at 2 A rating corresponds to about 0.10 Ω, which gives
91.1 mW at 0.9545 A, while a 1.393 copper-temperature factor gives about
28.0 mW for the inductor. Installed hot rise and resistance remain
measurement gates.
The damping resistor carries no steady load current; its hot-plug pulse,
not its 1 W steady rating, controls qualification.

The 100 V class is necessary because the held-output negative screen is
74.731 V before layout overshoot. An 80 V part would have only 5.269 V static
margin. The previous 60 V DMT6007 pair is invalid.

## Conversion rails and thermal screen

### Vehicle 3.3 V converter / MAIN source

Freeze `LMQ66420MC3RXBRQ1`: active/production, AEC-Q100 grade 1, 2 A,
3.6–36 V startup, 3–36 V after startup, 42 V absolute transient rating,
2.2 MHz, fixed 3.3 V/adjustable, and wettable 14-pin 2.6 x 2.6 mm VQFN.

The named 1.05 A peak, not its 1.313 A sizing margin, is the thermal operating
case. At 85 °C ambient and the data-sheet 66.1 °C/W comparison metric, assumed
93/92/91% efficiency at 12/14.4/18 V gives about 102/105/108 °C junction.
The silicon is retained, subject to PCB measurement.

### AUX5

Conditionally freeze `LMQ66420MC5RXBRQ1` under these contracts:

| Contract | Current | Power | Simultaneous AUX5 consumers | Duration |
|---|---:|---:|---|---|
| STREET | 0.56001 A | 2.80005 W | 0.500 A display + 0.06001 A street LEDs | continuous |
| TRACK | 0.92001 A | 4.60005 W | 0.500 A display + 0.12001 A track LEDs + 0.300 A sound | continuous |
| MAX | 1.16001 A | 5.80005 W | 0.500 A display + 0.36001 A full-white LEDs + 0.300 A sound | <=10 s and <=25% rolling-60-s duty |
| sizing margin | 1.450 A | 7.250 W | 25% capacity margin, not a simultaneous-consumer mode | capacity calculation, not an operating load |
| LOW-VOLTAGE SHED | 0 A | 0 W | display, LEDs and sound hardware-off | below the defined policy threshold |

At 85 °C, 66.1 °C/W, and assumed 92/91/90% efficiency at 12/14.4/18 V,
TRACK screens to about 111/115/119 °C and MAX to 118/123/128 °C. Reducing
efficiency two points gives MAX about 128/132/137 °C. This supports a prototype
at 85 °C, not a hot production guarantee; a 105 °C enclosure can exhaust the
margin.

`LMQ66430MC5RXBRQ1` has the same published switch resistances and package,
so its higher current limit does not solve thermal loss and increases fault
energy. `LM63635DQDRRRQ1` and `LM53635LQRNLTQ1` are credible lower-loss
fallbacks with higher parked current, area or cost.

### Shared LMQ66420 capture population

Use the same external power-stage population for MC3 and MC5. This follows
TI Rev. E Table 8-5 for a fixed-output LMQ66420 at 2.2 MHz, while adding
prototype margin for capacitance tolerance, DC bias, temperature and aging:

| Function | Exact prototype population | Capture footprint / connection |
|---|---|---|
| `L` | `XGL5030-222MEC`, 2.2 µH ±20%, AEC-Q200 | Coilcraft XGL5030 manufacturer land pattern; 5.48 x 5.28 mm body envelope and 3.1 mm maximum height for `-222` |
| `CIN` | 2 x `CGA6P1X7R1N106K250AC`, 10 µF ±10%, 75 V, X7R, AEC-Q200 | EIA 1210; place immediately across VIN-to-PGND with the smallest possible hot loop |
| `COUT` | 4 x `CGA6P3X7R1E226M250AB`, 22 µF ±20%, 25 V, X7R, AEC-Q200; add one identical DNP tuning footprint | EIA 1210; return directly to the converter ground plane and sense VOUT at the bank |
| `CVCC` | 2 x `CGA3E1X7R1C105K080AC`, 1 µF ±10%, 16 V, X7R, AEC-Q200 | EIA 0603; place directly from VCC to GND |
| `CBOOT` | DNP | the Q-suffix device integrates the 0.1 µF bootstrap capacitor |

The exact-order inductor comparison is:

| Candidate | Qualification / status | DCR maximum at 25 °C | Published current evidence | Area / loss trade | Disposition |
|---|---|---:|---|---|---|
| `XGL4030-222MEC` | Coilcraft current family, AEC-Q200 | 15.0 mΩ | 3.1/5.0/7.0 A at 10/20/30% drop | smallest candidate, but its 10%-drop point is below the IC's 3.9 A maximum peak limit | rejected |
| `XGL5020-222MEC` | Coilcraft current family, AEC-Q200 | 18.8 mΩ | 3.3/5.4/7.6 A at 10/20/30% drop | lower profile, but higher loss and its 10%-drop point is below 3.9 A | rejected |
| `XGL5030-222MEC` | Coilcraft current family, AEC-Q200 | 10.6 mΩ | 4.2/6.8/9.4 A at 10/20/30% drop | largest of the XGL candidates, lowest DCR, and clears the ideal current-limit screen | **selected** |
| `XEL5030-222MEC` | Coilcraft current family, AEC-Q200 | 14.5 mΩ | 10.5 A at the specified 30% drop | similar area with higher DCR and no tabulated 10%-drop point | fallback only |

This accepts the selected inductor's roughly 5.5 mm body and direct one-piece
price snapshot of $2.62 to obtain the explicit current-limit and copper-loss
margin. Price and reel availability are `ASSUMPTION` snapshots to refresh at
BOM release; electrical ratings are `VERIFIED_DATASHEET`.

Use the exact manufacturer land patterns, not an unreviewed generic library
footprint. Tie MODE/SYNC to GND for 2.2 MHz AUTO operation and connect VOUT/FB
directly to the appropriate fixed output. Give AUX5 EN a hardware default-low
state; permit it to rise only after the main rail is valid and the voltage/load
policy authorizes AUX5.

The selected inductor has 9.2/10.6 mΩ typical/maximum DCR at 25 °C, 34 MHz
typical SRF, 4.2/6.8/9.4 A current for 10/20/30% inductance drop, and
9.6/12.9 A reference current for a 20/40 °C rise. Its 4.2 A 10%-drop point is
above the LMQ66420's 3.9 A maximum high-side peak-current limit. Coilcraft's
current and temperature figures are 25 °C/reference data, not application
absolute maxima; hot derating and installed temperature remain validation
items. The order-code ratings in this subsection are `VERIFIED_DATASHEET`.

TI treats the capacitor values in its application section as effective values.
The following deliberately explicit allocation is the capture acceptance
screen, not a claim that the independent factors are manufacturer-guaranteed
joint corners. The derating reserves are `ASSUMPTION`, the arithmetic is
`CALCULATED`, and meeting each displayed minimum is a `DESIGN_REQUIREMENT`:

```text
COUT = 4 x 22 µF x 0.80 tolerance x 0.85 temperature
                    x 0.75 DC-bias reserve x 0.90 aging reserve
     = 40.392 µF effective allocation >= 40 µF TI minimum

CIN  = 2 x 10 µF x 0.90 tolerance x 0.85 temperature
                    x 0.50 DC-bias reserve x 0.90 aging reserve
     = 6.885 µF effective allocation >= 4.7 µF TI minimum

CVCC = 2 x 1 µF x 0.90 tolerance x 0.85 temperature
                   x 0.90 DC-bias reserve x 0.90 aging reserve
     = 1.239 µF effective allocation >= 1 µF TI requirement
```

The COUT bank has 5.0x voltage-rating headroom at 5 V and 7.6x at 3.3 V.
The CIN bank's 75 V rating is 2.94x the 25.531 V protected-node ceiling and
exceeds TI's preferred twice-maximum-input screen of 51.062 V. CVCC has 4.64x
headroom over the 3.45 V data-sheet maximum VCC output. TDK labels its DC-bias
and ripple graphs as reference-only, so the prototype population is acceptable
only after a vendor model or lot measurement proves at least 40 µF COUT,
4.7 µF CIN and 1 µF CVCC at applied bias across the required temperature and
life condition. Populate the fifth COUT position if the four-part bank misses
40 µF; then repeat loop and startup validation.

### Inductor ripple, peak current and copper loss

For ideal CCM at fixed 2.2 MHz,
`delta_IL = VOUT x (VIN - VOUT) / (VIN x fSW x L)`. The following uses the
minimum-tolerance `L=1.76 µH`, so it is a maximum-ripple first-order screen;
it excludes switch drops, minimum on/off time, frequency foldback, core bias
and parasitics. The table is `CALCULATED`:

| VIN | 3.3 V delta_IL pp | MC3 1.050 A peak | 5 V delta_IL pp | MC5 STREET peak | MC5 TRACK peak | MC5 MAX peak |
|---:|---:|---:|---:|---:|---:|---:|
| 6 V | 0.384 A | 1.242 A | 0.215 A | 0.668 A | 1.028 A | 1.268 A |
| 10 V | 0.571 A | 1.336 A | 0.646 A | 0.883 A | 1.243 A | 1.483 A |
| 12 V | 0.618 A | 1.359 A | 0.753 A | 0.937 A | 1.297 A | 1.537 A |
| 14.4 V | 0.657 A | 1.378 A | 0.843 A | 0.981 A | 1.341 A | 1.581 A |
| 18 V | 0.696 A | 1.398 A | 0.933 A | 1.026 A | 1.386 A | 1.626 A |
| 25.531 V | 0.742 A | 1.421 A | 1.038 A | 1.079 A | 1.439 A | 1.679 A |

At nominal 2.2 µH and 12 V, ripple is 0.494 A for MC3 and 0.603 A for MC5,
24.7% and 30.1% of the 2 A device rating. Both exceed TI's approximately 10%
minimum-ripple guidance and lie in its normal 20-40% selection range. The
largest operational peak above, 1.679 A, retains 1.121 A to the data-sheet
2.8 A minimum peak-current limit and 2.521 A to the inductor's 4.2 A
10%-drop point. Even the non-operating 1.450 A AUX sizing case reaches only
1.969 A at 25.531 V in this screen. These margins do not replace load-step,
short-circuit or hot core-bias testing.

With triangular ripple,
`IL_RMS = sqrt(IOUT^2 + delta_IL^2 / 12)`. The table gives approximate
inductor copper loss in mW as `IL_RMS^2 x 10.6 mOhm`; each cell is
`25 °C / 125 °C`, where the second number applies an engineering copper
resistance factor of 1.393. AC/core loss is excluded.
The loss table is `CALCULATED`; the temperature factor is an `ASSUMPTION`.

| VIN | MC3 1.050 A | MC5 0.56001 A | MC5 0.92001 A | MC5 1.16001 A |
|---:|---:|---:|---:|---:|
| 6 V | 11.82 / 16.46 | 3.37 / 4.69 | 9.01 / 12.56 | 14.31 / 19.93 |
| 10 V | 11.98 / 16.68 | 3.69 / 5.14 | 9.34 / 13.01 | 14.63 / 20.38 |
| 12 V | 12.02 / 16.75 | 3.83 / 5.33 | 9.47 / 13.20 | 14.77 / 20.57 |
| 14.4 V | 12.07 / 16.81 | 3.95 / 5.51 | 9.60 / 13.37 | 14.89 / 20.74 |
| 18 V | 12.11 / 16.88 | 4.09 / 5.70 | 9.74 / 13.57 | 15.03 / 20.94 |
| 25.531 V | 12.17 / 16.96 | 4.28 / 5.96 | 9.93 / 13.83 | 15.22 / 21.20 |

The worst COUT triangular-ripple RMS is about 0.214 A for MC3 and 0.300 A
for MC5, or about 0.054/0.075 A per four equally sharing capacitors. The
first-order CIN switching-current bound is `IOUT/2`: 0.525 A for MC3 and
0.580 A for MC5 MAX, or at most 0.263/0.290 A per two equally sharing input
capacitors. Those currents are far below the multiampere span of TDK's
reference ripple-temperature plots, but equal sharing and installed MLCC
temperature must still be measured. At `COUT=40 µF`, the ideal capacitive
ripple terms are about 1.05 mV for 3.3 V and 1.47 mV for 5 V at the largest
listed ripple; ESR and switching spikes are additional.

### Startup and remaining converter gates

The COUT upper allocation for inrush is
`4 x 22 µF x 1.20 x 1.15 = 121.44 µF`. With TI's 2 ms minimum soft-start
time to 90%, a deliberately conservative linear full-voltage ramp gives about
0.200 A average COUT charge current for MC3 and 0.304 A for MC5. Adding the
named full load and half of the largest ripple gives first-order startup peaks
of about 1.621 A for MC3 and 1.983 A for MC5 MAX. AUX5 shall not start
concurrently with the main rail and shall remain off below the low-voltage
policy threshold. These are `CALCULATED` screens using an `ASSUMPTION`
of a linear ramp; the sequencing rule is a `DESIGN_REQUIREMENT`.

The upper local CIN allocation is 25.3 µF per converter. Charging it to
25.531 V adds at most about 646 µC and 8.24 mJ per converter, or 1.292 mC and
16.5 mJ for both local banks, before the already-calculated filter energy.
This charge is controlled by the LM74720 slew/inrush behavior, not by the buck
soft start.

Prototype release must therefore measure: effective CIN/COUT/CVCC at bias and
temperature; input/output ripple and MLCC rise; startup into every allowed
preload; LM74720 plus both-CIN inrush; line/load transients and Bode margin;
short/hiccup behavior; inductor temperature and bias; complete converter
efficiency and IC/inductor/PCB temperature at 6/10/12/14.4/18 V and the
25.531 V protection boundary. Use Coilcraft's loss model for the exact
waveform, then confirm it on hardware. No exact-MC20 efficiency curve is
inferred from TI's MC30 application plots.

Both selected converters require a four-layer thermal implementation based on
the TI layout: 2 oz outer/1 oz inner copper as the starting stack assumption,
solid inner ground, exposed-pad solder, thermal vias, short VIN-CIN-PGND loops,
minimal switch-node area, and substantial ground copper. Keep both switch
nodes and inductor fields away from GNSS/RF.

## Post-switch input filter

Use this prototype starting population after the disconnect FETs:

```text
switched input
  -> CGA2B3X7R1H104M050BB, 100 nF/50 V shunt
  -> CGA5L3X7R1H105K160AB, 1 µF/50 V shunt
  -> XEL4030V-222MEC, 2.2 µH
  -> four CGA6P3X7R1H475K250AB 4.7 µF/50 V direct capacitors
     plus one identical DNP tuning footprint
  -> converter input

in parallel with direct capacitors:
  WSL2512R3900FEA 0.39 Ω
  in series with EEH-ZC1H121P 120 µF/50 V
```

`XEL4030V-222MEC` is AEC-Q200 with a 120 V series rating, 2.2 µH ±20%,
22.1 mΩ maximum DCR, 6.1 A saturation and 5.8 A current rating. The direct
and input-side TDK capacitors are AEC-Q200 X7R. TDK specifies minimum
insulation resistance of 5 GΩ for the 100 nF part and 500 MΩ for the 1 µF
part.
The Panasonic hybrid is AEC-Q200, -55 to 125 °C, 120 µF ±20%, 28 mΩ maximum
ESR at 100 kHz and 60 µA maximum leakage. The Vishay WSL2512 is AEC-Q200,
0.39 Ω ±1%, and from a 1 W metal-strip family intended for pulse applications;
its exact value-dependent power derating and time/energy point must pass the
manufacturer pulse tool/model and physical test.

The capture acceptance range for the combined direct capacitance is
9.4–20.68 µF over 0–25.6 V DC bias, tolerance, temperature and aging. A vendor
graph/model is reference evidence, not a guaranteed minimum; obtain model or
measurement evidence before final population.

For nominal `L=2.2 µH`, nominal `Rd=0.39 Ω`, and the data-sheet maximum
28 mΩ hybrid ESR:

| Effective direct C | Nominal resonance | Nominal characteristic impedance | Nominal first-cut Q |
|---:|---:|---:|---:|
| 9.4 µF | 35.0 kHz | 0.484 Ω | 1.16 |
| 20.68 µF | 23.6 kHz | 0.326 Ω | 0.78 |

Those are nominal screens, not bounds. Including `L=1.76–2.64 µH`,
`Rd=0.3861–0.3939 Ω`, and ESR from zero to its 28 mΩ maximum gives the
corner screen `f0=21.54–39.13 kHz` and `Q=0.69–1.37`
(`CALCULATED`). Parasitics and frequency-dependent ESR are still absent,
so neither range proves stability.

The hybrid minimum is 96 µF, or 4.64 times the maximum allowed direct
capacitance. It satisfies the 4x first-cut guidance in TI SNVA538, but not the
more conservative 5x screen in SLVAFE0. This is an intentional prototype
starting point, not a final damping claim.

At 25.531 V and maximum 144 µF, ideal capacitor energy is about 46.9 mJ; an
ideal hard step would initially imply about 1.67 kW in 0.39 Ω. The LM74720
output slew network normally limits connection rate, but that ideal screen is
why the exact resistor point must pass the Vishay pulse tool/model and a
hard-hot-plug fault test rather than being inferred from its 1 W steady rating.

A plain LC/CLC network is rejected because of high-Q interaction with a
converter's negative incremental input impedance. A resistor in the full load
path is rejected on dissipation. Ferrite-only filtering may retain a DNP
footprint for high-frequency tuning, but it is not deterministic low-frequency
damping.

Production release requires impedance/VNA or equivalent transient evidence
over harness impedance, 6/12/14.4/18/25.6 V, load states, MLCC bias/
temperature/aging, inductor bias, converter negative input impedance, hot plug,
conducted emissions and radiated/GNSS coexistence.

## USB and vehicle source behavior

### Current limit and USB conversion

Conditionally freeze `TPS2553QDBVRQ1` ahead of the USB buck with
`CRCW060360K4FKEA`, 60.4 kΩ ±1%, AEC-Q200, from ILIM to ground. TI's
[TPS2553-Q1 data sheet](https://www.ti.com/lit/ds/symlink/tps2553-q1.pdf)
Section 8.5 gives `IOS,min = 25230 / RILIM^1.016` and
`IOS,max = 22980 / RILIM^0.94`, with current in mA and resistance in kΩ,
and states that external-resistor tolerance is not included. With
`RILIM,min = 59.796 kΩ` and `RILIM,max = 61.004 kΩ`, conservative
outward rounding gives a 387.2–491.3 mA population current-limit screen
(`CALCULATED`). This stays below 500 mA at the upper corner; every intended
steady load must stay below the 387.2 mA lower corner with margin.

The exact prototype starting population for `TPS62162QDSGRQ1` is:

- `CGA2B3X7R1H104M050BB`, 100 nF/50 V, at TPS2553 IN;
- `CGA6P1X7R1E106M250AC`, 10 µF ±20%/25 V, at TPS2553 OUT and
  TPS62162 VIN;
- `XFL3012-222MEC`, 2.2 µH ±20%, AEC-Q200, 97 mΩ maximum DCR;
- `CGA6M3X7R1C106K200AB`, 10 µF ±10%/16 V, X7R, AEC-Q200, at the
  TPS62162 output and TPS2116 VIN2;
- 100 kΩ from TPS62162 PG to `USB_3V3`; fixed-version FB to AGND and a
  Kelvin VOS connection at the output-capacitor bank.

TI [TPS62162-Q1 Table 2](https://www.ti.com/lit/ds/symlink/tps62162-q1.pdf)
explicitly marks 2.2 µH plus nominal 10 µF as a recommended LC combination;
22 µF is recommended, not the only stable value. TDK marks the named
[25 V input MLCC](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA6P1X7R1E106M250AC)
and
[16 V output MLCC](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA6M3X7R1C106K200AB)
as production AEC-Q200 parts, but its DC-bias curves are reference data.
Capture accepts the 10 µF nominal output population only after a vendor model
or lot measurement confirms the installed capacitance over 3.3 V bias,
temperature and aging and after loop/load-step testing. Place the buck and mux
so the named output capacitor also meets TPS2116's local 1 µF minimum;
otherwise add a local 1 µF AEC-Q200 capacitor and repeat the startup
calculation.

`ASSUMPTION + CALCULATED` USB power screening uses 4.75 V at the connector,
TPS2553 maximum `RON = 135 mΩ` and an 85% TPS62162 efficiency floor. The
vehicle 3.3 V bucket includes vehicle-only CAN, so the physical `MAIN_3V3`
loads are 120 mA WAKE, 190 mA CORE, 290 mA STREET/TRACK and 695 mA MAX.

| MAIN_3V3 load | VBUS input screen | Disposition |
|---:|---:|---|
| 100 mA | about 82.0 mA | post-ramp `USB_ENUM` steady ceiling before configuration; not an inrush ceiling |
| 120 mA | 98.36 mA | Existing WAKE assumption is prohibited before configuration; it has no defensible margin |
| 190 mA | 155.99 mA | configured USB-only CORE |
| 290 mA | 238.65 mA | configured USB-only normal operation |
| 400 mA | 330.03 mA | provisional steady output ceiling |
| 424 mA | 350.03 mA | arithmetic edge, not an operating contract |
| 695 mA | 577.53 mA | impossible on this USB path; prohibit MAX |

After the rails finish ramping and before configuration, `USB_ENUM` is a
`DESIGN_REQUIREMENT` of at most 100 mA total on `MAIN_3V3`, about 82.0 mA
steady at VBUS under the stated 4.75 V/85% screen. It is not an inrush ceiling.
GNSS, RF transmit, display, LEDs, sounder, SD writes and every other optional
branch must have a hardware default-off state. The existing 120 mA WAKE
assumption is not allowed in `USB_ENUM`. After the host contract is
established, provisionally cap operation at 350 mA VBUS steady and 400 mA
`MAIN_3V3`.

The USB-only prototype source contract is explicit: the source and cable must
advertise and sustain at least 500 mA at a 4.75 V connector voltage throughout
startup, because the direct 100 µF bulk can make the path draw within the
387.2–491.3 mA limiter band before enumeration. This does **not** establish
generic legacy USB 2.0 pre-enumeration compliance. The exact USB role, Type-C
advertisement/data circuitry and controlling current/inrush specification are
outside Task 5A.1; selecting them and proving host behavior remain
validation/product-release gates. The 100 mA `USB_ENUM` value is only the
post-ramp operating-load contract.

TPS2553-Q1 explicitly supports heavy capacitive loads and maintains
constant-current operation until the overload clears or thermal cycling
begins. Entering its current-limit band during startup is therefore not by
itself a contradiction. The following is an illustrative current-limit and
capacitance sensitivity, not a conservative time bound:

- TPS62162 COUT maximum is 11 µF; its typical 25 mV/µs slope requests
  275 mA;
- the selected TPS2116 VOUT bulk maximum is 120 µF; its typical 1.3 ms
  3.3 V soft start requests about 305 mA;
- adding the 100 mA `USB_ENUM` ceiling gives about 680 mA simultaneous
  output demand, or about 556 mA at VBUS under the screening model, so the
  387.2–491.3 mA limiter deliberately stretches the ramp;
- if the 387.2 mA lower current-limit corner were modeled with 4.75 V,
  135 mΩ and 85%, the equivalent 3.3 V capacity would be 468.5 mA. After the
  100 mA load, 368.5 mA would remain for capacitance, giving 1.173 ms for the
  upper 131 µF allocation by `C delta-V / I`;
- a deliberately pessimistic instantaneous limiter-loss screen is
  `4.75 V x 491.3 mA = 2.33 W`, or about 2.74 mJ if held for 1.173 ms.

Those time and energy values are `CALCULATED` illustrations, not guaranteed
bounds. The 135 mΩ on-resistance model is not valid while TPS2553 is in
current regulation; OUT may collapse toward TPS62162 UVLO, startup efficiency
is not specified, TPS2553 OUT also charges the 12 µF maximum buck-input
allocation, and the TPS62162 and TPS2116 ramps can interact. A nonlinear
current-limit/startup model and bench capture must replace the 1.173 ms
illustration. Validate monotonic
startup, FAULT behavior, host droop/inrush, repeated hot-plug temperature and
recovery at the 387.2 mA lower corner and with the 100 mA load applied. The
latest applicable USB specification and the exact Type-C/data implementation
remain controlling evidence; the 500 mA prototype-source requirement is not a
substitute for that product-level proof.

TI Section 9.2.4 also supports a two-level ILIM network. A screened
`210 kΩ || 84.5 kΩ` implementation would make the configured resistance about
60.255 kΩ, but the 210 kΩ state is specified at approximately 100–150 mA
before external-resistor tolerance. It does not guarantee that VBUS stays
below 100 mA, and an automotive
[`2N7002KQ-7`](https://www.diodes.com/datasheet/download/2N7002KQ.pdf)
implementation adds GPIO-default
and off-leakage dependencies. It is therefore not in the frozen starting
population. Reopen that option only if host/inrush testing rejects fixed
60.4 kΩ.

At the provisional 400 mA output ceiling and 85% efficiency floor, converter
loss screens to 232.9 mW. TI's 65.5 °C/W JEDEC comparison metric gives about
100.3 °C junction at 85 °C ambient and 120.3 °C at 105 °C ambient, only
4.7 °C below the 125 °C rating. The prohibited 695 mA USB MAX case screens to
about 131.5 °C at 105 °C ambient. These are conservative arithmetic screens,
not measured board temperatures. A dropout estimate gives about 3.58 V
required converter input at 400 mA, but the used switch-resistance maximum is
specified only for VIN at or above 6 V; hot 4.75 V dropout remains a test gate.

`TPS2116DRLR` provides reverse-current blocking and break-before-make source
selection at 3.3 V. It is a catalog, non-AEC part whose published electrical
limits used here extend through 105 °C; board-level environmental
qualification or an automotive-qualified replacement is a production gate.

Populate
[`EEEFK0J101AV`](https://na.industrial.panasonic.com/products/capacitors/aluminum-electrolytic-capacitors/lineup/aluminum-electrolytic-capacitors-surface-mount-type/series/89034/model/89622)
directly from TPS2116 VOUT to ground: 100 µF ±20%,
6.3 V, AEC-Q200, -55 to 105 °C, with 6.3 µA maximum leakage and 360 mΩ
maximum 100 kHz impedance. This implements TI's approximately 100 µF
alternative to an output clamp when reverse-current blocking can activate.
It is not hidden behind another load switch.

At its 59 mΩ maximum `RON` through 105 °C, the separate 1.050 A vehicle-buck
case leaves about 0.995 A through the mux after the 55 mA direct-CAN branch:
58.7 mV drop and 58.4 mW loss. The simultaneous 0.750 A bucket leaves about
0.695 A through the mux: 41.0 mV and 28.5 mW. With TI's 111.5 °C/W JEDEC
comparison metric those losses imply about 6.5 °C and 3.2 °C rise,
respectively (`CALCULATED`); this is not a board thermal guarantee.

TPS2116's 6.2 µs switchover is typical. At the selected bulk's 80 µF tolerance
minimum, ideal capacitive droop is about 31 mV at 400 mA, 54 mV at 695 mA and
77 mV at 995 mA. Capacitor impedance, board ceramics, source collapse rate and
the unbounded switchover-time distribution are additional. Validate the RCB
current spike, output overshoot, handoff droop and every capacitance/load
corner; fit an output clamp instead if the direct bulk does not pass.

The starting priority network is:

- `MODE = VEH_3V3`;
- `PR1` pulled up by 1.0 MΩ to `VEH_3V3` and down by 2.0 MΩ to ground;
- MAIN vehicle-buck PGOOD open drain pulls PR1 low during invalid startup;
- a valid vehicle rail selects VIN1; absent/invalid vehicle power selects
  VIN2/USB.

Validate the MODE undefined band, PGOOD sequencing, collapse, crossover,
reverse-current delay and all output-capacitance/load combinations.

This exact TPS2553/TPS62162/TPS2116 population is conditionally released for
prototype schematic capture. The condition is revoked before fabrication if
the 100 mA `USB_ENUM` load cannot be demonstrated from reset through
configuration, or if source/current-limit corner testing predicts UVLO
cycling, non-monotonic startup, unacceptable host inrush, RCB overshoot or
handoff droop. The silicon/topology remains a prototype freeze, not a USB or
automotive production qualification.

A dual
[`LM66100QDCKRQ1`](https://www.ti.com/lit/ds/symlink/lm66100-q1.pdf)
ideal-diode OR was screened but is not selected. TI specifies 140 mΩ maximum
at 3.6 V through 125 °C, 192 °C/W `RθJA` and up
to 8 µA output-to-input leakage; its simple highest-voltage OR does not
provide the required deterministic vehicle priority, and TI warns that the
basic dual circuit can turn both paths off when its inputs remain equal. It
does not close the capture with lower loss or fewer validation gates than
TPS2116.

### Four source states

| Vehicle | USB | Hardware result |
|---|---|---|
| absent | absent | all rails off/passive |
| present | absent | VEH_3V3 powers MAIN_3V3; CAN and vehicle-only loads available |
| absent | present | USB_3V3 powers MAIN_3V3 development/core domain; TCAN VCC and AUX5 remain physically off; no OBD backfeed |
| present | present | valid vehicle rail has priority; USB remains isolated and available for data/handoff; CAN authorization remains vehicle-qualified |

TCAN3404-Q1's unpowered digital pins and bus pins must remain high impedance.
Verify GPIO defaults and measure phantom current rather than relying only on
the transceiver table.

## Parked current

State: vehicle attached, MAIN vehicle converter enabled, MAIN_3V3 core asleep,
TCAN in standby, AUX5 and all optional loads off, USB absent. 3.3 V rail loads
are referred to 12 V using 60% low-load efficiency.

| Contribution | 12 V input allocation |
|---|---:|
| LM74720 operating maximum | 35.000 µA |
| OV divider | 22.814 µA |
| TVS pair | 2.000 µA |
| MAIN buck | 5.000 µA |
| AUX buck shutdown | 1.000 µA |
| vehicle sensing / gated ADC | 15.000 µA |
| TCAN standby | 7.800 µA |
| ESP module branch | 22.900 µA |
| TCA6408A | 4.600 µA |
| LP5814 shutdown | 0.140 µA |
| wake logic | 4.600 µA |
| disabled load switches | 1.500 µA |
| TPS2116 and priority network | 2.720 µA |
| `EEEFK0J101AV` MAIN bulk maximum leakage | 2.888 µA |
| four direct MLCC insulation allocation, held at 18 V maximum | 1.440 µA |
| LM74720/filter support-ceramic insulation allocation, held at 18 V maximum | 0.196 µA |
| damping hybrid maximum leakage | 60.000 µA |
| signal ESD | 5.000 µA |
| residual miscellaneous leakage reserve | 9.415 µA |
| USB buck/switch when USB is absent and isolated | 0 µA vehicle contribution |
| **mixed planning point** | **185.269 µA** |
| **conservative subtotal** | **204.013 µA** |
| **100% uncertainty/temperature design allowance** | **204.013 µA** |
| **paper design envelope** | **408.026 µA = 0.408 mA** |

A statistical `expected` value is not defensible before measurement because
the hybrid capacitor, ESD, module and contamination terms do not all have
applicable typical distributions. The 185.269 µA planning point substitutes
published typical values where available but retains allocations or maxima
where no useful typical exists. It is the closest responsible pre-measurement
estimate, not a guaranteed or probabilistic expectation. The 204.013 µA
subtotal uses the conservative values shown; the doubled result is the design
envelope.

The planning point is traceable as `27 + 22.814 + 0.4 + 1.5 + 0.25 + 15 +
4.58 + 22.9 + 4.6 + 0.046 + 4.6 + 1.5 + 1.14 + 2.888 + 1.440 + 0.196 + 60 +
5 + 9.415 = 185.269 µA`; terms without an applicable typical remain allocations
or maxima. The 2.888 µA term refers the capacitor's 6.3 µA maximum 3.3 V
leakage to 12 V at 60% efficiency. To keep the 6–18 V model conservative,
the direct-MLCC allocation is held at `4 x 18 V / 50 MΩ = 1.440 µA` rather
than its 12 V value. The 0.196 µA support-capacitor allocation rounds up
`18/2272 MΩ + 18/227 MΩ + 15.5/227 MΩ + 18/5 GΩ + 18/500 MΩ =
0.1951 µA`, the full-range minimum-insulation-resistance screen for the
separate A-to-ground, C/VS-to-ground, CAP-to-C/VS and post-Q2 100 nF/1 µF
parts. The 9.415 µA residual allocation explicitly includes a separate
1.354 µA planning screen for the four always-connected LMQ CIN capacitors,
enabled-MC3 COUT/CVCC banks and BZT52H-C18-Q leakage, leaving about 8.061 µA
for other small passive/PCB/contamination leakage. The 0.585 µA transferred
from the former round 10 µA reserve into the full-range direct/support rows
leaves the total unchanged. These are fixed full-range allocations in the
voltage model, not voltage-independent physical leakage. This is an allocation,
not a hot guarantee; complete-board temperature testing remains controlling.
The 1.354 µA screen is
`4 x 12 V / 50 MΩ = 0.960 µA` CIN,
`4 x 3.3 V / 22 MΩ x 3.3 V / (0.60 x 12 V) = 0.275 µA` COUT,
`2 x 3.45 V / 100 MΩ = 0.069 µA` CVCC, and 0.050 µA for the zener.
The zener value is specified only at 12.6 V and 25 °C, so its hot/13 V
behavior is part of the board-current gate.

For voltage screening, split the model into 135.551 µA constant allocations,
the 526 kΩ divider, and 99.5345 µA of 3.3 V rail current:

```text
IPARK(V) = 135.551 µA + V/526 kΩ
         + 99.5345 µA x 3.3 V / (0.60 x V)
```

| Vehicle input | Subtotal | Doubled design envelope |
|---:|---:|---:|
| 6 V | 238.20 µA | 0.476 mA |
| 8 V | 219.19 µA | 0.438 mA |
| 10 V | 209.31 µA | 0.419 mA |
| 12 V | 203.98 µA | 0.408 mA |
| 14.4 V | 200.94 µA | 0.402 mA |
| 18 V | 200.18 µA | 0.400 mA |

The small 12 V difference between the detailed-row sum and split model is
rounding. The paper result passes the <1.0 mA requirement and <0.50 mA stretch
target across the modeled range. It is not a hot guarantee: TVS, hybrid,
ESD, module/PSRAM, contamination, wake duty and converter behavior require
complete-board measurements. The 0.408 mA 30-day charge scale is about
0.294 Ah; it is not a storage-life claim.

## Manufacturability, availability and cost

All observations below are snapshots from 2026-08-17 and must be refreshed at
BOM release. Prices are indicative USD, normally quantity one unless stated.

| Part | Status / availability snapshot | Indicative price |
|---|---|---:|
| LM74720QDRRRQ1 | TI active; DigiKey/Mouser each showed thousands | $2.63 |
| STL125N10F8AG, x2 | ST active; DigiKey about 2,980 | $3.14 each |
| SM15T47AY + SM15T33AY | ST active; authorized stock observed | about $0.98 each |
| 0437002.WRA | Littelfuse active; DigiKey about 13,773 | $1.99 |
| LMQ66420MC3/MC5RXBRQ1 | TI active; authorized stock observed | $4.30 each |
| XGL5030-222MEC, x2 | Coilcraft current/AEC-Q200; direct stock observed | about $2.62 each |
| LMQ CIN/COUT/CVCC CGA banks | Exact TDK AEC-Q200 order codes are production parts | price refresh required |
| XEL4030V-222MEC | Coilcraft production/AEC-Q200; manufacturer page showed stock and direct unit price | $2.62 |
| TDK direct MLCC, x4 | TDK production; DigiKey stock observed | $1.02 each |
| EEH-ZC1H121P | Panasonic automotive hybrid; DigiKey about 2,291 | $1.67 |
| WSL2512R3900FEA | Vishay AEC-Q200; Mouser about 10,116 observed | about $0.95 |
| TPS2553QDBVRQ1 | TI active; supply availability volatile | about $1.60 assumption |
| TPS2116DRLR | TI active; high authorized stock | $1.01 |
| TPS62162QDSGRQ1 | TI active; DigiKey about 2,566 | $2.04 |
| XFL3012-222MEC | Coilcraft current/AEC-Q200; stock not guaranteed | price refresh required |
| USB CGA input/output capacitors | Exact TDK AEC-Q200 order codes are production parts | price refresh required |
| EEEFK0J101AV | Panasonic FK aluminum electrolytic, not a hybrid; exact availability requires refresh | price refresh required |
| LPS3015-104MRC | Coilcraft recommended for new designs/AEC-Q200 grade 1; release stock requires refresh | $0.95 manufacturer-page screen |
| LM74720 CGA support capacitors | Exact `CGA5H2...`, `CGA6N3...`, `CGA6M3...` and `CGA2B3...` codes are TDK production/AEC-Q200 parts; manufacturer pages showed authorized stock | included in the $2.00 B-support allowance; release price refresh required |
| LM74720 zener and CRCW support | Nexperia lists `BZT52H-C18-Q`; exact Vishay 0 Ω/100 Ω/330 Ω values use the AEC-Q200 D/CRCW family | included in the $2.00 B-support allowance; release stock/price and RPD pulse refresh required |
| LM74502QDDFRQ1, Architecture A delta | TI active; dated authorized stock was observed | $1.75 comparison allowance, not a release quote |
| LTC4368HMS-2#WPBF, Architecture C delta | ADI production/recommended for new designs; dated authorized stock about 559 | $7.57 comparison allowance, not a release quote |
| 12 mΩ shunt + alternate negative-TVS delta, Architecture C | Exact shunt and alternate TVS are not frozen because C is rejected | $0.50 + $0.14 architecture allowances |

The wettable-flank WSON/VQFN and PowerFLAT packages support optical solder-joint
inspection, but exposed-pad voiding and thermal vias still need assembler
rules and, where required, X-ray evidence. Confirm the exact SMC TVS and
PowerFLAT land patterns, stencil apertures, polarity marks, high-current
copper, connector rating and surge-return geometry. The selected filter uses
five large MLCC footprints, one SMC hybrid and a 2512 resistor; preserve
tuning/DNP flexibility until EMI and impedance measurements close.

Several TI exact parts showed volatile or zero stock on manufacturer pages
despite being active. Authorized-distributor availability, lifecycle and
second-source policy are BOM-release gates, not reasons to substitute an
electrically different look-alike.

## External reference survey and licensing

Manufacturer references control component ratings and calculations. TI
TIDA-01167 is useful surge-stopper/SOA methodology but its linear-pass design
is larger than this low-current disconnect architecture.

Secondary public hardware was used only for questions and test ideas:

- OpenXC's Ford reference vehicle interface publishes design material under
  CC BY 4.0, but its diode/TVS/cascaded-LDO power chain does not meet these
  requirements.
- Carloop hardware is GPLv3 and inherits OpenXC ideas; copying would require a
  license review.
- the local RejsaCAN hardware tree has no established root hardware license,
  so no permission to copy or derive is inferred;
- comma.ai panda is a useful HIL/vehicle-interface precedent, with license
  checked per file before any reuse.

No external schematic, layout or code was copied into AutoTelemetry.

## Validation and release gates

The architecture is approved only for a future prototype schematic capture.
Before production freeze or any compliance statement, close all of these:

1. define the exact +38 V source duration, impedance and repetition plus the
   applicable purchased-standard/OEM pulse set;
2. run manufacturer electrothermal/SPICE models and then bench the two-TVS
   network for dynamic clamp, forward drop, temperature, repetition and
   fixture/layout inductance;
3. measure fuse survival/clearing, time-current behavior, interrupt capability,
   inrush and failed-short TVS coordination hot and cold;
4. record LM74720 A, C, C-to-A, PD, both FET VDS/VGS and gate timing for
   normal, +26, +38, +50, -14, -100, source crossover and output-precharge
   cases;
5. confirm downstream survival through 25.531 V plus measured overshoot;
6. prove direct filter capacitance remains 9.4–20.68 µF and validate
   impedance, damping-resistor pulse, source/load transients and EMI;
7. validate TPS2116/TPS62162/TPS2553 four-state source injection, the
   post-ramp `USB_ENUM <=100 mA MAIN_3V3` contract, the explicit prototype
   startup source of at least 500 mA at 4.75 V, and the provisional
   350 mA VBUS/400 mA MAIN_3V3 configured ceilings; include the exact
   capacitor population, limiter tolerance, soft-start/current-limit
   interaction, host inrush, RCB bulk/clamp behavior, switchover, reverse
   leakage and 105 °C environmental limitation, without claiming generic
   legacy USB 2.0 pre-enumeration compliance;
8. measure MAIN/AUX efficiency, load-step, dropout and junction/board
   temperature at 12/14.4/18 V and 85 °C ambient for STREET/TRACK/MAX duty;
   separately exercise the 1.050 A `VEH_3V3`/MC3 capacity case, and decide
   whether 105 °C enclosure operation is required;
9. measure complete-board parked current over 6–18 V, temperature, wake duty,
   contamination and post-drive shutdown time;
10. verify low-voltage state transitions, bounded SD flush, CAN passive
    defaults, brownout and recovery before release firmware is approved;
11. perform the already-required full-load interference and
    EMI/GNSS/power-placement reviews before final routing/manufacture.

There is no claim of ISO 7637, ISO 16750, CISPR 25, UNECE R10, OEM, load-dump,
ESD, thermal or automotive system compliance.

## Primary sources

- TI, [LM74720-Q1 product page](https://www.ti.com/product/LM74720-Q1) and
  [data sheet](https://www.ti.com/lit/ds/symlink/lm74720-q1.pdf).
- TI, [LM74720-Q1 EVM guide](https://www.ti.com/lit/pdf/SNOU186).
- TI, [LM74502-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lm74502-q1.pdf),
  [LM7480-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lm7480-q1.pdf), and
  [LM74930-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lm74930-q1.pdf).
- TI, [TPS37-Q1 product page and exact-order data](https://www.ti.com/product/TPS37-Q1/part-details/TPS37A010122DYYRQ1)
  and [TPS48110-Q1 family product data](https://www.ti.com/product/TPS4811-Q1).
- Analog Devices, [LTC4368 product page, exact production models and Rev. C
  data sheet](https://www.analog.com/en/products/LTC4368.html).
- NXP, [FS27 product page](https://www.nxp.com/products/FS27),
  [FS23 product page](https://www.nxp.com/products/FS23), and
  [UJA1163A product page](https://www.nxp.com/products/UJA1163ATK).
- ST, [SM15T automotive TVS family data](https://www.st.com/resource/en/datasheet/sm15t36cay.pdf).
- ST, [STL125N10F8AG product page](https://www.st.com/en/power-transistors/stl125n10f8ag.html).
- ST, [STPM801 data sheet](https://www.st.com/resource/en/datasheet/stpm801.pdf).
- Infineon, [IAUA210N10S5N024 product page](https://www.infineon.com/part/IAUA210N10S5N024),
  [IAUCN08S7L018 product page](https://www.infineon.com/part/IAUCN08S7L018),
  [TLE9471 product page](https://www.infineon.com/part/TLE9471ES), and
  [TLE9261 product page](https://www.infineon.com/part/TLE9261BQX).
- Diodes Incorporated, [DMT10H020SDGQ automotive product page](https://www.diodes.com/part/view/DMT10H020SDGQ).
- Littelfuse, [437A fuse data sheet](https://www.littelfuse.com/assetdocs/littelfuse-fuse-437a-datasheet?assetguid=82c80a59-a4b9-4748-920b-3e2b65b813a9).
- Littelfuse, [TPSMD40A product data](https://www.littelfuse.com/products/overvoltage-protection/tvs-diodes/automotive-tvs-diodes/tpsmd/tpsmd40a)
  and [TPSMD series data sheet](https://www.littelfuse.com/assetdocs/tvs-diode-tpsmd-datasheet?assetguid=d280bb45-3774-486c-9d4c-0048b5a1d38c).
- Bourns, [MF-LSMF automotive PPTC data sheet](https://www.bourns.com/docs/product-datasheets/mf-lsmf.pdf)
  and [SM8SF-Q data sheet](https://www.bourns.com/docs/Product-Datasheets/SM8SF-Q.pdf).
- TI, [LMQ66420-Q1 Rev. E data sheet](https://www.ti.com/lit/ds/symlink/lmq66420-q1.pdf).
- Coilcraft, [XGL5030-222MEC product data](https://www.coilcraft.com/en-us/products/power/shielded-inductors/molded-inductor/xgl/xgl5030/xgl5030-222/)
  and [XGL5030 manufacturer data sheet](https://www.coilcraft.com/getmedia/e64ac115-95f2-45c7-b798-1b3769b91583/xgl5030.pdf).
- Coilcraft comparison data for
  [XGL4030-222MEC](https://www.coilcraft.com/en-us/products/power/shielded-inductors/molded-inductor/xgl/xgl4030/xgl4030-222/),
  [XGL5020-222MEC](https://www.coilcraft.com/en-us/products/power/shielded-inductors/molded-inductor/xgl/xgl5020/xgl5020-222/), and
  [XEL5030-222MEC](https://www.coilcraft.com/en-us/products/power/shielded-inductors/molded-inductor/xel/xel5030/xel5030-222/).
- TDK, [`CGA6P1X7R1N106K250AC` input capacitor](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA6P1X7R1N106K250AC),
  [`CGA6P3X7R1E226M250AB` output capacitor](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA6P3X7R1E226M250AB),
  [output-capacitor characterization sheet](https://product.tdk.com/system/files/dam/doc/product/capacitor/ceramic/mlcc/charasheet/cga6p3x7r1e226m250ab.pdf), and
  [`CGA3E1X7R1C105K080AC` VCC capacitor](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA3E1X7R1C105K080AC).
- TI, [LMQ66430MC5RXBRQ1 product data](https://www.ti.com/product/LMQ66430-Q1/part-details/LMQ66430MC5RXBRQ1),
  [LM63635DQDRRRQ1 product data](https://www.ti.com/product/LM63635-Q1/part-details/LM63635DQDRRRQ1),
  and [LM53635LQRNLTQ1 product data](https://www.ti.com/product/LM53635-Q1/part-details/LM53635LQRNLTQ1).
- TI, [TPS2116 product page](https://www.ti.com/product/TPS2116),
  [TPS62162-Q1 product page](https://www.ti.com/product/TPS62162-Q1), and
  [TPS2553-Q1 product page](https://www.ti.com/product/TPS2553-Q1).
- Nexperia, [PMEG6030EP-Q data sheet](https://assets.nexperia.com/documents/data-sheet/PMEG6030EP-Q.pdf).
- ST, [USBLC6-2SC6Y data sheet](https://www.st.com/resource/en/datasheet/usblc6-2sc6.pdf).
- Coilcraft, [XFL3012-222MEC](https://www.coilcraft.com/en-us/products/power/shielded-inductors/molded-inductor/xfl/xfl3012/xfl3012-222/).
- Coilcraft, [LPS3015-104MRC manufacturer data](https://www.coilcraft.com/en-us/products/power/shielded-inductors/ferrite-drum/lps/lps3015/lps3015-104/).
- TI, [input-filter damping note SNVA538](https://www.ti.com/lit/an/snva538/snva538.pdf),
  [damping guidance SLVAFE0](https://www.ti.com/lit/an/slvafe0/slvafe0.pdf),
  and [input-filter interaction note SNVA801](https://www.ti.com/lit/an/snva801/snva801.pdf).
- Coilcraft, [XEL4030V high-voltage inductor family](https://www.coilcraft.com/en-us/products/power/high-voltage-inductors/xel/xel4030v/).
- TDK, [CGA6P3X7R1H475K250AB characteristic sheet](https://product.tdk.com/system/files/dam/doc/product/capacitor/ceramic/mlcc/charasheet/cga6p3x7r1h475k250ab_200125.pdf).
- TDK, exact support-capacitor product data for
  [CGA5H2X7R2A224K115AE](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA5H2X7R2A224K115AE),
  [CGA6N3X7R2A225K230AE](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA6N3X7R2A225K230AE),
  [CGA6M3X7R1H225K200AE](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA6M3X7R1H225K200AE),
  [CGA2B3X7R1H103K050BE](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA2B3X7R1H103K050BE),
  [CGA2B3X7R1H104M050BB](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA2B3X7R1H104M050BB), and
  [CGA5L3X7R1H105K160AB](https://product.tdk.com/en/search/capacitor/ceramic/mlcc/info?part_no=CGA5L3X7R1H105K160AB).
- Nexperia, [BZT52H-Q series product data](https://www.nexperia.com/products/diodes/zener-diodes/series/BZT52H-Q-SERIES.html).
- Panasonic, [selected EEH-ZC1H121P product information](https://na.industrial.panasonic.com/products/capacitors/aluminum-electrolytic-capacitors/series/80/model/138),
  [EEH-ZC1H101P candidate data](https://industrial.panasonic.com/ww/products/pt/hybrid-aluminum/models/EEHZC1H101P),
  and [EEH-ZC1V680XV candidate data](https://industrial.panasonic.com/ww/products/pt/hybrid-aluminum/models/EEHZC1V680XV).
- Nichicon, [UCD series manufacturer data](https://www.nichicon.co.jp/products/pdfs/ucd.pdf).
- Vishay, [D/CRCW e3 automotive resistor data](https://www.vishay.com/docs/20035/dcrcwe3.pdf)
  for the exact 0 Ω, 100 Ω, 330 Ω and 60.4 kΩ support values, and
  [WSL Power Metal Strip resistor family](https://www.vishay.com/en/product/30100/).
