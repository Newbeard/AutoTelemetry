# Power architecture baseline

Status: audited 2026-08-14. Detailed classified calculations are authoritative in [`power-budget.md`](power-budget.md); wake, protection, CAN, GNSS, display, shift-light and USB decisions are in [`power-wake-review.md`](power-wake-review.md).

## RejsaCAN v3.4 reference path

The v3.4 single-sheet schematic shows this functional path:

```text
POWER pin 1 ─ D6 DSS34 series diode ─ F1 MF-MSMF110/16-2 PTC ─ VCC
                                                           ├─ D4 SMF30A to GND
                                                           ├─ U3 threshold/hold logic
                                                           ├─ U4 LMR14006X buck ─ 3V3
                                                           └─ R18/R6 ADC divider ─ GPIO9

USB VBUS ─ D7 DSS34 ─ VCC/enable path
3V3 ─ U5 MT9700 controlled by GPIO21 ─ 3V3_SWITCHED
```

This is a descriptive reading of the source, not a validation. Exact diode direction/current paths and all net names must be rechecked in editable CAD when a derivative schematic is captured.

## Existing behavior

- Repository README operating claim: the reference board can be powered from 5–24 V. This is source attribution, not a Telemetry v1 requirement or transient rating; v1 targets 12 V passenger vehicles and does not require 24 V operation.
- U4 is LMR14006XDDCR: 4–40 V input and 600 mA maximum load (`VERIFIED_DATASHEET`, TI SNVSAG3). The repository schematic shows a 10 µH inductor, D5 catch diode, two 22 µF output capacitors and R9/R10 feedback for nominal 3.3 V (repository source values). None is automatically reused.
- D6 provides a series input element; F1 is a resettable fuse; D4 is a 30 V TVS-class part. These facts do not by themselves prove reverse-battery, jump-start, load-dump, pulse, thermal, or EMC compliance.
- U3 (BOM: ME2807A33M3G), Q1, R4/R5 and D1/D2/D3 form the threshold/hysteresis/hold network. R4 = 91 kΩ and R5 = 33 kΩ are repository source values. The README's approximately 13.7 V turn-on/13.0 V turn-off values are an upstream claim, not accepted Telemetry v1 thresholds; detector tolerance and the transfer equation were not established from a primary data sheet.
- GPIO17 FORCE_ON can hold the buck enabled after the input falls below the detector threshold. USB5V also feeds the enable network through a diode.
- GPIO8 SENSE_V_DIG observes the threshold state. GPIO9 SENSE_V_ANA measures VCC through repository values R18 = 120 kΩ and R6 = 33 kΩ. `CALCULATED`: ideal divider ratio is `33/(120+33)=0.2157`; continuous 12 V divider current is `12/153k=78.4 µA`. Telemetry v1 gates or increases the divider so vehicle sensing consumes ≤15 µA at the OBD input in parked mode (`DESIGN_REQUIREMENT`). ADC tolerance/nonlinearity, source impedance and clamping remain unresolved.
- U5 MT9700 creates 3V3_SWITCHED under GPIO21 control. The README's “max 500mA” label is an upstream claim, not a verified safe Telemetry v1 capacity. The derivative replaces this generic branch with calculated named domains.

## Wake/shutdown distinction

1. **Hardware-off:** when U4 SHDN is deasserted, 3.3 V disappears, so ESP32 and U2 also lose power. v3.4 can turn back on from the voltage threshold or USB, not from CAN.
2. **ESP32 sleep with main rail on:** GPIO17 holds U4 on. U2 can be placed in standby using GPIO38 high; TI documents receiver-active standby. This establishes feasibility only. Telemetry v1 replaces U2 with TCAN3404-Q1 WUP/standby and must bench-verify GPIO13 deep-sleep wake.
3. **Active CAN:** GPIO38 low selects high-speed mode; high impedance with R3 = 10 kΩ sets slope-control behavior. Firmware comments and TI terminology should be aligned in one driver.

## Approved pre-schematic derivative domains

| Domain | Candidate loads | Control intent | Open issue |
|---|---|---|---|
| Always-on/vehicle sense | Gated detector/divider, ESP wake inputs | Hardware/ESP | ≤15 µA OBD-input allocation (`DESIGN_REQUIREMENT`) |
| MAIN_3V3 | ESP32-S3, TCAN3404-Q1, switched branch inputs | Low-IQ automotive buck | ≥2.0 A (`DESIGN_REQUIREMENT`); LMQ66420-Q1 preferred candidate, not selected |
| GNSS_3V3 | NEO-M9N and antenna-bias controls | Independent load switch | ≥200 mA; off parked (`DESIGN_REQUIREMENT`) |
| SD_3V3 | microSD | Independent load switch | ≥250 mA; off parked (`DESIGN_REQUIREMENT`) |
| DISPLAY_3V3 | external module | Independent switch/BL control | ≥400 mA; off parked (`DESIGN_REQUIREMENT`) |
| AUX5 | display option, shift light, buzzer option | Independent 2 A automotive converter | ≥2.0 A aggregate; off parked (`DESIGN_REQUIREMENT`) |

`CALCULATED`: the 3.3 V simultaneous peak envelope is 1.050 A; after 25% margin it is 1.313 A, so the required regulator class is ≥2.0 A. `CALCULATED`: the 5 V external-load sum is 1.70 A and 15% margin produces 1.955 A, so AUX5 is ≥2.0 A. See the source-to-formula chain in `power-budget.md`.

## Parked result

The complete tree includes protection, buck IQ, vehicle sensing, TCAN standby, the ESP32 module branch, wake logic, disabled switches, protection leakage and miscellaneous leakage. `CALCULATED`: subtotal 80.3 µA and 100% allowance give ≤161 µA (0.161 mA) at 12 V. GNSS backup and LEDs are off. This supports the <1 mA requirement and <0.5 mA stretch target on paper; complete-board measurements over voltage/temperature remain mandatory.

## Required calculations and tests before layout

- Define supply cases: cranking, reverse polarity, jump start, load dump, ISO 7637 pulses, continuous overvoltage, and ground offset appropriate to target vehicles.
- Check every component's absolute maximum, energy, pulse duration, temperature derating, and failure mode; simulate/bench-test the complete protection chain.
- Calculate input/output capacitor ripple, inductor saturation, diode loss, buck thermal rise, startup/inrush, loop/stability requirements, and maximum simultaneous load.
- Measure active and parked-state current at temperature, including regulator, detector, CAN standby, divider, LEDs, GNSS backup, load switches, and leakage/back-power paths.
- Validate USB/vehicle dual-power behavior and ensure neither source is unintentionally back-fed.
- Define brownout/file-flush behavior and energy/time needed to close a log safely.

## Specialist-review flags

- U4 and U2 BOM entries are standard/catalog parts rather than explicitly automotive-qualified `-Q1` variants.
- A 30 V TVS plus a 40 V regulator is not sufficient evidence of load-dump survival.
- Reference CAN lines show termination but no dedicated CAN TVS/common-mode choke.
- The reference uses voltage-as-engine-running heuristic; smart charging and weak batteries can violate it.
- GPIO-held power and RTC wake behavior must be verified across every intended ESP32 sleep mode.
