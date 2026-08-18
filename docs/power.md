# Power architecture baseline

Status: Task 5A.1 **conditionally re-frozen for future prototype schematic
capture**, 2026-08-17. The authoritative circuit, calculations, requirements
and remaining release gates are in
[`task5a1-power-architecture.md`](task5a1-power-architecture.md). No schematic,
PCB, firmware, measurement or compliance result is implied.

## Current Task 5A.1 baseline

```text
OBD16 -> 0437002.WRA 2 A fuse
      -> VBAT_FUSED_CLAMPED
         +-> common-anode SM15T47AY + SM15T33AY shunt -> POWER_GND
      -> LM74720QDRRRQ1 + 2 x back-to-back STL125N10F8AG
      -> damped post-switch filter
      -> FILTERED_VEHICLE
         -> LMQ66420MC3RXBRQ1 -> VEH_3V3
         -> normally-off LMQ66420MC5RXBRQ1 -> AUX5

USB -> TPS2553-Q1 -> TPS62162-Q1 USB_3V3
VEH_3V3 + USB_3V3 -> TPS2116 -> MAIN_3V3
VEH_3V3 directly powers TCAN3404-Q1; AUX5 is vehicle-only.
```

The LM74720 prototype OV divider is `249 kΩ + 249 kΩ` over
`28.0 kΩ`, all 0.1%. With resistor tolerance and a conservative ±1 µA
node-leakage envelope, OV rising is 20.690–25.531 V and falling is
18.815–23.366 V. This guarantees no static trip through 18 V, a reconnect
command is eligible at 18 V, and the OV/PD-low command occurs before 26 V.
Completed switching, downstream peak and recovery time remain dynamic gates.
It deliberately supersedes the incompatible
18–20 V cutoff and 24 V protected-node requirements.

The actual worst named simultaneous output is 8.27505 W: a 2.475 W complete
vehicle 3.3 V/MC3 bucket plus 5.80005 W AUX. The bucket includes direct
vehicle-only CAN current; physical muxed `MAIN_3V3` does not. MAX is allowed
only at or above 10 V for at most 10 s and 25%
duty in a rolling 60 s. A low-voltage policy begins an SD flush below 9.5 V,
sheds AUX/display/shift/sound/Wi-Fi below 9.0 V for 100 ms, and recovers only
above 10.0 V for 2 s. Those values are design requirements for later firmware,
not an implementation.

`LMQ66420MC3RXBRQ1` remains frozen for the vehicle 3.3 V source. The
`LMQ66420MC5RXBRQ1` AUX selection is conditionally frozen for
0.56001 A STREET, 0.92001 A TRACK, and 1.16001 A managed MAX. The corrected
parked-current envelope is 0.408 mA at 12 V and 0.476 mA at the modeled 6 V
worst point, passing both paper targets but still requiring hot board
measurement.

Both LMQ variants use the exact shared conditional population:
`XGL5030-222MEC`; two `CGA6P1X7R1N106K250AC` input capacitors; four
`CGA6P3X7R1E226M250AB` output capacitors plus one DNP; two
`CGA3E1X7R1C105K080AC` VCC capacitors; and CBOOT DNP. The USB chain uses
60.4 kΩ ±1% TPS2553 RILIM, the exact TPS62162 population and
`EEEFK0J101AV` 100 µF at TPS2116 VOUT frozen in the authoritative record.
Its prototype startup source/cable must advertise and sustain at least
500 mA at 4.75 V; post-ramp `USB_ENUM` is at most 100 mA `MAIN_3V3`
(about 82 mA VBUS), not an inrush ceiling or a generic legacy USB 2.0
pre-enumeration compliance claim.

## Superseded reference and Task 4.7 context

Everything below this heading is retained for provenance. Its old component
chain, thresholds, load sums and parked-current result are not active design
instructions when they conflict with Task 5A.1.

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
2. **ESP32 sleep with main rail on:** GPIO17 holds U4 on. U2 can be placed in standby using GPIO38 high; TI documents receiver-active standby. This establishes reference feasibility only. Telemetry v1 uses TCAN3404DRQ1 WUP/standby with CAN_RX on GPIO13 and a separate vehicle-activity wake input on GPIO8; both wake paths require prototype verification.
3. **Active CAN:** GPIO38 low selects high-speed mode; high impedance with R3 = 10 kΩ sets slope-control behavior. Firmware comments and TI terminology should be aligned in one driver.

## Frozen pre-schematic derivative domains

| Domain | Candidate loads | Control intent | Open issue |
|---|---|---|---|
| Always-on/vehicle sense | Gated detector/divider, ESP wake inputs | Hardware/ESP | ≤15 µA OBD-input allocation (`DESIGN_REQUIREMENT`) |
| MAIN_3V3 | ESP32-S3, TCAN3404DRQ1, switched branch inputs | LMQ66420MC3RXBRQ1 | ≥2.0 A; rail remains on while parked |
| GNSS_3V3 | NEO-M9N and antenna-bias controls | TPS22919QDCKRQ1 | ≥200 mA; off parked |
| SD_3V3 | microSD | TPS22919QDCKRQ1 | ≥250 mA; off parked |
| DISPLAY_3V3 | external module | TPS22919QDCKRQ1 and 2N7002KQ BL control | ≥400 mA; off parked |
| AUX5 | display option, shift-light and sounder source | LMQ66420MC5RXBRQ1 | 2.0 A silicon class; continuous hot-load capability is not established; off parked; enabled by GPIO21 |
| DISPLAY_5V | optional display rail | TPS22919QDCKRQ1 from AUX5 | ≥600 mA; mutually exclusive with incompatible display power |
| SHIFT_5V | external ten-pixel light | TPS1H100BQPWPRQ1 from AUX5 | 0.50 A qualified load, 1.0 A protected fault envelope; cable ≤0.5 m |
| SOUNDER5 | onboard sounder | approved-direction TPA2005D1TDGNRQ1 from AUX5 | 300 mA design envelope; exact speaker provisional |

`CALCULATED`: the 3.3 V simultaneous peak envelope is 1.050 A; after 25% margin it is 1.313 A, so the required regulator class is ≥2.0 A. The 5 V named-load sum is 1.30 A and 25% margin produces 1.625 A. Task 5A found 1.30 A conditionally plausible only pending measured converter efficiency and demonstrated `RθJA ≤50 °C/W`; 1.625 A must be managed/short-duration pending proof, and 2 A continuous at 85 °C is not defensible. See the source-to-formula chain in [`power-budget.md`](power-budget.md) and the thermal screen in [`task5a-power-calculations.md`](task5a-power-calculations.md).

## Parked result

The Task 4.7 complete tree separately includes the LM74502-Q1 controller, SM8SF24CA-Q, both buck-converter parked currents, UV/OV/sensing, TCAN, ESP32, TCA6408A, LP5814 shutdown, disabled switches, USB isolation, signal ESD and miscellaneous leakage. Adding the AUX5 converter's 1 µA maximum shutdown current gives a `CALCULATED` subtotal of 212.54 µA; a 100% allowance gives ≤425.08 µA (0.425 mA) at 12 V. This passes <1.0 mA and the <0.50 mA room target on paper by 0.575 mA and 0.075 mA. A conditional damping-capacitor candidate with 50 µA maximum leakage would instead give `2×(212.54+50)=525.08 µA`, failing the <0.50 mA stretch target; a lower-leakage solution or requirement change is a Task 5A decision gate. Complete-board measurements remain mandatory.

The Task 4.7 historical candidate chain was `0437002A` fuse → bidirectional `SM8SF24CA-Q` → `LM74502QDDFRQ1` with back-to-back `DMT6007LFGQ-7` MOSFETs → damped post-switch C-L-C topology → converters. Task 5A reopens its threshold implementation, TVS/FET/fuse coordination and exact filter population; it is not approved for capture. The 6–18 V operation, +26 V/60 s jump, +38 V suppressed-load-dump, −14 V/60 s reverse and ≤24 V downstream values remain `DESIGN_REQUIREMENT` inputs pending an approved compatible implementation. At the 12.458 W calculated output envelope and assumed 80% efficiency, input current is 2.595 A at 6 V, 1.730 A at 9 V and 1.298 A at 12 V; the 2 A fuse therefore needs an explicit low-voltage/hot load policy or must be reopened.

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
