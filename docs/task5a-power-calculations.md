# Task 5A power calculations and schematic-capture gate

Status: **SUPERSEDED BLOCKER EVIDENCE**, 2026-08-17. Task 5A correctly stopped
before KiCad project creation because primary-source calculations invalidated
several frozen assumptions. Task 5A.1 resolves the active architecture and
requirements in
[task5a1-power-architecture.md](task5a1-power-architecture.md). The material
below remains the audit trail for the rejected Task 4.7 circuit and must not be
used as the current capture specification.

## Historical Task 5A result

The frozen vehicle-input chain cannot simultaneously satisfy its present 6–18 V operation, USB source-crossover, +26 V/60 s jump-start, ≤20 V overvoltage-cutoff, full-load and parked-current requirements. Per the Task 5A stop rule, affected schematic capture, ERC and PDF export were not performed.

The independent decision gates are:

1. select a threshold implementation that can guarantee operation through 18 V, cutoff by 20 V and the required USB/vehicle crossover;
2. reopen the input TVS because 24 V working standoff does not cover +26 V for 60 s;
3. define low-voltage/hot load shedding or reopen the 2 A fuse;
4. define whether AUX5 1.625 A is continuous or a managed peak, or reopen the regulator; and
5. select a lower-leakage damped filter or relax the <0.5 mA stretch target.

## LM74502-Q1 source values

Unless stated otherwise, these are VERIFIED_DATASHEET from TI SNOSDE0A, Tables 7.3 and 7.5.

| Parameter | Minimum | Typical | Maximum |
|---|---:|---:|---:|
| EN/UVLO falling threshold | 1.027 V | 1.14 V | 1.235 V |
| EN/UVLO rising threshold | 1.16 V | 1.24 V | 1.32 V |
| EN sink current at EN = 12 V | not specified | 3 µA | 5 µA |
| OV rising threshold | 1.165 V | 1.25 V | 1.333 V |
| OV falling threshold | 1.063 V | 1.143 V | 1.222 V |
| OV input leakage | 12 nA | 50 nA | 110 nA |
| Gate source current | 40 µA | 60 µA | 77 µA |
| Gate peak sink current | — | 2.37 A | — |
| Operating quiescent current | — | 45 µA | 65 µA |
| VS characterized operating range | 4 V | — | 60 V |
| Pin recommended range | −60 V | — | +60 V |
| Pin absolute maximum | −65 V | — | +65 V |

The prior 110 µA controller row was a conservative allocation, not the published maximum. Future budgets must label 110 µA as a DESIGN_REQUIREMENT if retained rather than VERIFIED_DATASHEET.

## UV / USB crossover incompatibility

For top resistor RT, bottom resistor RB, pin sink current IEN, and divider scale K = 1 + RT/RB:

~~~text
VIN_trip = VTH × K + IEN × RT
~~~

Before resistor tolerance or sink-current error, two necessary conditions are:

~~~text
guaranteed turn-on by 6.0 V:
K <= 6.0 / 1.32 = 4.545

guaranteed turn-off above even an idealized 5.0 V USB-derived SYS_IN:
K > 5.0 / 1.027 = 4.869
~~~

There is no overlap (CALCULATED). Using the 5.25 V USB upper bound increases the second constraint to K > 5.112. LM74502-Q1 does not reverse-block while enabled, so a prototype-only crossover test cannot replace a guaranteed off threshold.

The EN sink-current minimum is not specified and its 3–5 µA values are specified only at EN = 12 V. A high-value divider chosen to fit the 25 µA parked allocation therefore cannot receive a guaranteed worst-case threshold from the data sheet.

An illustrative, **not frozen**, 750 kΩ / 328 kΩ, 0.1% divider produces approximately 5.997 V only if 3 µA typical sink current is assumed. Treating 0–5 µA as the available interval gives an illustrative 3.371–7.818 V falling range (CALCULATED), which is not usable.

The prior typical reconnect arithmetic was also wrong. Ignoring the sink-current term, a 6.0 V typical falling threshold would give:

~~~text
6.0 V × 1.24 / 1.14 = 6.526 V
~~~

not 6.58 V. The additive sink-current term prevents even that simple scaling in an actual divider.

**PROPOSED CHANGE REQUIRED:** use a qualified precision window supervisor to control the LM74502 EN pin, use a controller that also provides reverse-current blocking and tighter thresholds, or change the voltage/source-crossover requirements. No option is selected here.

## OV incompatibility

Ignoring resistor and leakage error already proves the conflict:

~~~text
guaranteed operation through 18 V:
K >= 18 / 1.165 = 15.451

guaranteed cutoff by 20 V:
K <= 20 / 1.333 = 15.004
~~~

There is no overlap (CALCULATED). The first condition forces:

~~~text
Vtrip,max >= 1.333 × 15.451 = 20.596 V
~~~

The second condition permits:

~~~text
Vtrip,min <= 1.165 × 15.004 = 17.479 V
~~~

For completeness, an illustrative, **not frozen**, RT = 910 kΩ, RB = 68.1 kΩ, both 0.1%, gives:

~~~text
Vtrip = VREF × (1 + RT/RB) + IOV × RT

nominal rising  = 17.999 V
worst rising    = 16.712 V to 19.281 V
nominal falling = 16.462 V
worst falling   = 15.250 V to 17.684 V
divider current at 12 V = 12.27 µA
~~~

This pair meets the ≤20 V cutoff objective but can disconnect inside the 6–18 V normal envelope. Resistor tolerance or lower resistance cannot fix the comparator-reference spread.

**PROPOSED CHANGE REQUIRED:** change the threshold implementation, relax guaranteed 18 V operation, or relax the 20 V maximum cutoff. The downstream ≤24 V measured requirement remains separate.

## Input TVS

The frozen SM8SF24CA-Q values are VERIFIED_DATASHEET:

- bidirectional and AEC-Q101;
- VRWM = 24 V;
- VBR = 26.7–29.5 V at 5 mA;
- maximum clamp 38.9 V at 180 A;
- 7 kW at 10/1000 µs;
- 10 µA maximum leakage at 24 V and 25 °C.

The TVS is before the disconnect MOSFETs and remains connected during jump start. A +26 V/60 s source exceeds its 24 V working standoff and lies only 0.7 V below minimum breakdown. The data sheet does not guarantee leakage or non-avalanche behavior there for 60 s. Disconnecting the downstream load does not resolve this (VERIFIED_DATASHEET + CALCULATED).

Higher-standoff parts cannot be silently substituted because their published maximum clamps reduce the margin to the frozen 60 V MOSFET:

| Candidate | VRWM | Breakdown | Maximum clamp | Margin to 60 V |
|---|---:|---:|---:|---:|
| SM8SF24CA-Q | 24 V | 26.7–29.5 V | 38.9 V | 21.1 V |
| SM8SF28CA-Q | 28 V | 31.1–34.4 V | 45.4 V | 14.6 V |
| SM8SF30CA-Q | 30 V | 33.3–36.8 V | 48.4 V | 11.6 V |
| SM8SF33CA-Q | 33 V | 36.7–40.6 V | 53.3 V | 6.7 V |
| SM8SF36CA-Q | 36 V | 40.0–44.2 V | 58.1 V | 1.9 V |

**PROPOSED CHANGE REQUIRED:** reopen TVS/FET coordination after the exact +38 V pulse duration, source resistance, repetition and temperature are defined.

### Corrected source-current screening

The earlier calculation (50 − 38.9)/2 = 5.55 A used a clamp specified at 180 A. It is conservative for controller/FET voltage stress but is a lower, not upper, source-current estimate.

A two-point linear screen between the published breakdown and clamp endpoints gives, as an ASSUMPTION:

~~~text
+50 V / 2 Ω source: approximately 9.99–11.27 A
−100 V / 10 Ω source: approximately 7.01–7.28 A
~~~

For a rectangular 2 ms negative pulse:

~~~text
I²t = 0.098–0.106 A²s
~~~

This is 68–74% of the fuse's nominal 0.144 A²s melting value (CALCULATED). The real waveform is not rectangular, and nominal melting I²t is not a guaranteed withstand limit. Pulse repetition, temperature and measured TVS current remain mandatory.

## Input fuse

0437002.WRA / 0437002A is VERIFIED_DATASHEET as:

- Littelfuse 437A, AEC-Q200, fast-acting 1206;
- 2 A, 63 V;
- 62 mΩ nominal;
- 0.144 A²s nominal melting I²t at 1 ms;
- 0.20 V nominal drop and 0.40 W nominal dissipation at rated current;
- four hours minimum at 100% rating and 25 °C;
- five seconds maximum to open at 250%;
- one second maximum to open at 350%.

Littelfuse recommends continuous operation at no more than 80% of rating. Its 75 °C example applies 0.80 × 0.85 = 0.68 of rating:

~~~text
recommended continuous current at 25 °C = 1.60 A
recommended continuous current at 75 °C = 1.36 A
~~~

For the documented 12.458 W simultaneous rail envelope and 80% combined efficiency (ASSUMPTION):

| VIN | Input current |
|---:|---:|
| 6 V | 2.595 A |
| 9 V | 1.730 A |
| 12 V | 1.298 A |
| 18 V | 0.865 A |

The minimum voltage for that full load is:

~~~text
VIN >= 12.458 / (0.80 × 1.60) = 9.73 V at 25 °C
VIN >= 12.458 / (0.80 × 1.36) = 11.45 V at 75 °C
~~~

**PROPOSED CHANGE REQUIRED:** define low-voltage/hot load shedding while retaining the fuse, or reopen fuse rating and all downstream fault coordination. The prospective OBD fault current and exact applicable interrupt-rating table cell also remain to be verified.
## Conditional CdVdt / inrush screen

TI Equation 2 is:

~~~text
Cdvdt = IGATE × COUT / IINRUSH
IINRUSH,max = IGATE,max × COUT,max / Cdvdt,min
tramp = Cdvdt × VIN / IGATE
~~~

Using a conditional COUT,max = 150 µF and 22 nF ±10%:

~~~text
IINRUSH,max =
  77 µA × 150 µF / 19.8 nF
  = 0.583 A
~~~

| VIN | Typical ramp | Fast corner | Slow corner |
|---:|---:|---:|---:|
| 12 V | 4.40 ms | 3.09 ms | 7.26 ms |
| 18 V | 6.60 ms | 4.63 ms | 10.89 ms |
| 26 V | 9.53 ms | 6.69 ms | 15.73 ms |

The ideal capacitive-inrush-only fuse screen at 26 V is approximately 0.00227 A²s, 1.6% of nominal fuse melting I²t (CALCULATED). Converter startup current must be added.

22 nF plus the TI EVM's 100 Ω isolation-resistor precedent is a credible starting point, not a selected value. The exact effective filter capacitance, revised controller choice and load sequencing must be frozen first.

## MOSFET check

DMT6007LFGQ-7 is VERIFIED_DATASHEET as:

- 60 V VDS and ±20 V VGS;
- 6 mΩ maximum at 10 V gate and 8.5 mΩ maximum at 4.5 V gate;
- pins 1–3 source, pin 4 gate, pins 5–8 drain;
- 55 °C/W junction-to-ambient only on the manufacturer's 1 in², 2 oz test board;
- 20 mJ single-pulse avalanche rating;
- typical, 25 °C SOA curves.

Common sources and common gates are required, with one drain toward the raw input and the other toward the protected output.

Using the Task 4.7 hot factor of 1.7 (ASSUMPTION):

~~~text
RPAIR = 2 × 8.5 mΩ × 1.7 = 28.9 mΩ
~~~

| Current | Pair drop | Pair loss |
|---:|---:|---:|
| 1.298 A | 37.5 mV | 48.7 mW |
| 2.000 A | 57.8 mV | 115.6 mW |
| 2.595 A | 75.0 mV | 194.6 mW |

Normal conduction and gate stress pass: the controller's 13.9 V maximum gate drive leaves 6.1 V to the ±20 V gate rating.

The conditional approximately 0.6 A, below-20 V, 11 ms inrush point appears inside the typical 25 °C SOA; Diodes Incorporated's AP74701Q application section independently screens this MOSFET above 3.5 A at 43 V/1 ms. Hot SOA is still unproven, and one MOSFET must be assumed to carry the full linear stress.

Negative-pulse off-state stress is unresolved. With held output voltage:

~~~text
VDS ≈ VOUT + |VNEG_CLAMP| − VF
~~~

An approximately 30 V negative clamp gives 42.8 V at 13.5 V output, 47.3 V at 18 V output and 53.3 V at 24 V output. Using the 38.9 V published maximum clamp point can produce approximately 62.2 V at a held 24 V output, above the 60 V rating (CALCULATED). Exact precondition, clamp voltage and overshoot must be fixed before the MOSFET receives PASS status.

**Result: MARGINAL / REOPENED**, not a conduction-loss failure.

## Conditional post-switch input filter

The topology remains a reasonable starting point, but no exact filter is approved for capture.

Conditional candidate:

~~~text
VBAT_SWITCHED
  -> 100 nF + 1 µF / 50 V
  -> 2.2 µH XEL4030-222MEC
  -> direct effective capacitance >= 19.9 µF
  -> series damping branch: 0.22 Ω + 100 µF / 50 V EEHZC1H101P
  -> VEHICLE_PROTECTED / SYS_IN
~~~

The XEL4030-222MEC is AEC-Q200, 2.2 µH, 22.1 mΩ DCR maximum, 6.1 A saturation and 5.8 A RMS (VERIFIED_DATASHEET). The hybrid capacitor has up to 28 mΩ ESR and 50 µA specified maximum leakage; the exact pulse-rated 0.22 Ω resistor is not selected.

Using TI input-filter guidance:

~~~text
f0 = 1 / (2π√(L × Cdirect))
Z0 = √(L/Cdirect)
Cd >= 4 × Cdirect
|ZIN_converter| ≈ VIN² / PIN
~~~

Results:

- at 19.9 µF effective, f0 = 24.1 kHz and Z0 = 0.332 Ω;
- at 50.5 µF nominal, f0 = 15.1 kHz and Z0 = 0.209 Ω;
- minimum damping capacitance is 4 × 19.9 = 79.6 µF;
- the candidate calculated peak is approximately 0.255 Ω;
- worst screened converter negative input impedance at 6 V is approximately 2.51 Ω, giving about 9.8:1 separation.

These are CALCULATED screens, not stability proof. Exact MLCC DC-bias curves, SPICE/impedance analysis, source harness, load steps and EMI testing remain required.

The 50 µA capacitor leakage is incompatible with treating the <0.5 mA stretch budget as comfortably closed. A lower-leakage damping topology or an approved budget change is required before capture.

## Main-regulator screen

TI's LMQ66420-Q1 fixed-output starting network is VERIFIED_DATASHEET:

- 2.2 µH;
- at least 4.7 µF effective input capacitance;
- two 22 µF nominal output capacitors and at least 40 µF effective output capacitance;
- at least 1 µF effective VCC capacitance;
- MODE/SYNC grounded for AUTO;
- CBOOT/CFF DNP and fixed-output VOUT/FB sensed directly.

Exact MLCC order codes cannot be approved without manufacturer DC-bias and temperature curves.

At 2.2 µH and 2.2 MHz, nominal inductor ripple is:

| Rail | 12 V | 14.4 V | 18 V | 24 V |
|---|---:|---:|---:|---:|
| MAIN_3V3 | 0.494 A | 0.526 A | 0.557 A | 0.588 A |
| AUX5 | 0.603 A | 0.674 A | 0.746 A | 0.818 A |

At 24 V, the nominal peaks are approximately 1.607 A for the 1.313 A MAIN design current and 2.034 A for the 1.625 A AUX design current. Both are below the 2.8 A minimum high-side peak-current limit (CALCULATED + VERIFIED_DATASHEET).

## AUX5 thermal gate

At 85 °C ambient, with 88% efficiency (ASSUMPTION) and all conversion loss conservatively assigned to the IC:

~~~text
PLOSS = POUT × (1/η − 1)
TJ = TA + RθJA × PLOSS
~~~

| AUX5 load | Loss | TJ at 45 °C/W EVM | TJ at 50 °C/W target | TJ at 66.1 °C/W JEDEC |
|---|---:|---:|---:|---:|
| 1.30 A / 6.5 W named sum | 0.886 W | 125 °C | 129 °C | 144 °C |
| 1.625 A / 8.125 W design margin | 1.108 W | 135 °C | 140 °C | 158 °C |
| 2.00 A / 10 W rating | 1.364 W | 146 °C | 153 °C | 175 °C |

TI does not publish an exact MC5 efficiency curve for all required 12/14.4/18 V, 2 A conditions. The 1.30 A named simultaneous case is conditionally plausible only with measured efficiency and an achieved RθJA <= 50 °C/W. The 1.625 A margin cannot be guaranteed continuous at 85 °C from public data, and 2 A continuous is not defensible.

**PROPOSED CHANGE REQUIRED:** classify 1.625 A as a managed/short-duration envelope and 1.30 A as the continuous limit pending measurement, or select a lower-loss/higher-current regulator.

## Parked-current correction

Task 4.7's 0.424 mA estimate is not retained unchanged.

- The AUX5 LMQ66420-Q1 maximum shutdown VIN current of 1 µA was omitted.
- The controller's published maximum is 65 µA, while 110 µA may remain only as an explicitly conservative design allocation.
- The conditional hybrid damping capacitor specifies up to 50 µA leakage.

Keeping the existing conservative 110 µA controller allocation and adding AUX5 shutdown gives:

~~~text
base subtotal = 211.54 + 1.00 = 212.54 µA
doubled envelope = 425.08 µA = 0.425 mA
~~~

Adding the conditional capacitor's specified 50 µA maximum:

~~~text
subtotal = 262.54 µA
doubled envelope = 525.08 µA = 0.525 mA
~~~

The second result passes the <1 mA target by 0.475 mA but fails the <0.5 mA stretch target by 0.025 mA (CALCULATED). It is not a final budget because the filter part is not approved.

## Primary sources

- Texas Instruments, [LM74502-Q1 data sheet SNOSDE0A](https://www.ti.com/lit/ds/symlink/lm74502-q1.pdf), Tables 7.3/7.5, §§9.3.3–9.3.5 and §10.2.
- Texas Instruments, [LM74502Q1EVM guide SNOU191](https://www.ti.com/lit/pdf/SNOU191), schematic, inrush test and BOM.
- Diodes Incorporated, [DMT6007LFGQ data sheet DS40969](https://www.diodes.com/datasheet/download/DMT6007LFGQ.pdf), ratings, pinout and SOA.
- Diodes Incorporated, [AP74701Q data sheet](https://www.diodes.com/datasheet/download/AP74701Q.pdf), MOSFET SOA application example.
- Bourns, [SM8SF-Q data sheet](https://www.bourns.com/docs/Product-Datasheets/SM8SF-Q.pdf), electrical table and pulse graphs.
- Littelfuse, [437A data sheet](https://www.littelfuse.com/assetdocs/littelfuse-fuse-437a-datasheet?assetguid=82c80a59-a4b9-4748-920b-3e2b65b813a9), electrical/opening data and temperature re-rating.
- Texas Instruments, [LMQ66420-Q1 data sheet](https://www.ti.com/lit/ds/symlink/lmq66420-q1.pdf), electrical limits, Table 8-5, component selection and thermal information.
- Texas Instruments, [AN-2162 / SNVA489](https://www.ti.com/lit/an/snva489/snva489.pdf), [SNVA801](https://www.ti.com/lit/an/snva801/snva801.pdf), [SNVA538](https://www.ti.com/lit/an/snva538/snva538.pdf) and [SNVA810](https://www.ti.com/lit/an/snva810/snva810.pdf), damped input-filter guidance.
- Coilcraft, [XEL4030-222](https://www.coilcraft.com/en-us/products/power/high-voltage-inductors/xel/xel4030/xel4030-222/) manufacturer data.
- Panasonic, [EEHZC1H101P](https://industrial.panasonic.com/ww/products/pt/hybrid-aluminum/models/EEHZC1H101P) manufacturer data.

## Capture disposition

No KiCad project, schematic, custom symbol, ERC report or PDF review artifact was created. This avoids presenting unresolved proposed changes as production source of truth.
