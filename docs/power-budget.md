# Telemetry v1 power budget

Status: Task 4 component-freeze budget, 2026-08-14. It is not a released schematic or an automotive-compliance claim.

## Evidence convention

- `VERIFIED_DATASHEET`: stated in the cited manufacturer document under the stated conditions.
- `CALCULATED`: derived here; the formula and inputs are shown.
- `DESIGN_REQUIREMENT`: an envelope the eventual circuit must meet. It is not a measured load.
- `ASSUMPTION`: provisional input that must be replaced by a selected-part value or measurement.

Datasheet “typical” values are not guaranteed maxima. Unknown marketplace display, card, antenna, light and buzzer variants therefore receive conservative design envelopes rather than invented consumption.

## Source loads

| Load | Rail | Typical / operating basis | Maximum or peak basis | Budget used | Evidence |
|---|---:|---:|---:|---:|---|
| ESP32-S3-WROOM-1 N16R8 | 3.3 V | 81.3 mA, modem-sleep CPU at 240 MHz with peripheral clocks | 355 mA Wi-Fi TX peak; BLE TX peak 344 mA | 100 mA typical, 355 mA peak | `VERIFIED_DATASHEET`: Espressif module datasheet v1.8, Tables 6-4 through 6-7. Supply is 3.0–3.6 V and the external supply must deliver at least 0.5 A. `ASSUMPTION`: 100 mA application typical until firmware is measured. Wi-Fi and BLE peaks are alternative RF cases, not summed. |
| ESP32 deep sleep allocation | 3.3 V | Chip/module table: 7–8 µA depending on RTC retention | N16R8 module/PSRAM board result not guaranteed by the chip figure | 50 µA rail-current allocation | `VERIFIED_DATASHEET`: module datasheet Table 6-7; its note says module current can exceed the chip table with PSRAM. `DESIGN_REQUIREMENT`: complete populated ESP32 branch ≤50 µA at 25 °C after strap/pull-network optimization; verify over temperature. |
| NEO-M9N | switched 3.3 V | 36 mA continuous tracking, four GNSS | 50 mA acquisition continuous; 100 mA peak | 50 mA continuous, 100 mA peak | `VERIFIED_DATASHEET`: u-blox NEO-M9N-00B Data sheet R08, Table 14, measured at 3.0 V, 25 °C and 1 Hz. High-rate consumption is not specified, so 50 mA is the provisional continuous requirement and must be measured at 25 Hz. |
| Active GNSS antenna | GNSS 3.3 V bias | 5–20 mA typical range | Antenna not selected | 20 mA continuous, 50 mA fault-limited output | `VERIFIED_DATASHEET`: u-blox integration manual R10 §3.2.2.1 gives typical active-antenna current of 5–20 mA; NEO-M9N VCC_RF is rated to source 50 mA in data-sheet Table 14. `DESIGN_REQUIREMENT`: selected antenna ≤20 mA normal; bias path survives/limits a short. |
| microSD | switched 3.3 V | 80 mA write typical for the Swissbit longevity/high-endurance example | 100 mA write maximum for that example; final card unspecified | 100 mA continuous, 200 mA peak | `VERIFIED_DATASHEET`: Swissbit PS-66 SD-LxPT data sheet Rev. 1.01 operating-write table. `DESIGN_REQUIREMENT`: socket rail supports 200 mA transient because interchangeable cards/inrush are not bounded by that example. |
| GC9A01A controller | switched 3.3 V | Exact operating current unresolved | Exact maximum unresolved | included in display-module envelope | `VERIFIED_DATASHEET`: GC9A01A preliminary data sheet v1.0 Table 44 gives VCI 2.5–3.3 V and IOVCC 1.65–3.3 V, but the reviewed table does not establish complete module/backlight current. No unsupported controller-current number is used. |
| Representative 1.28-inch display module | switched 3.3 V or 5 V | Module/backlight not selected | Marketplace modules differ | 300 mA at 3.3 V **or** 500 mA at 5 V | `DESIGN_REQUIREMENT`: connector envelopes, including controller, backlight and cable loss. Only one supplied rail may be enabled/configured for a module. Replace after exact module selection. |
| TCAN3404-Q1 | always-on 3.3 V | 7 mA recessive typical | 55 mA dominant maximum at 60 Ω; 130 mA current-limited bus-fault case; 17 µA standby maximum at 150 °C | 8.2 mA active max, 55 mA dominant peak, 17 µA parked | `VERIFIED_DATASHEET`: TI SLLSFQ6A Tables 7-1/7-2. The 130 mA fault current is a protection/thermal case, not an ordinary rail capacity load. |
| External shift light | switched 5 V | application dependent | Ten legacy-class RGB LEDs could be 10 × 60 mA = 600 mA | 1.0 A connector/rail envelope | `VERIFIED_DATASHEET`: WS2812B-2020 v1.3 specifies 3.7–5.3 V, VIH ≥2.7 V and 12 mA per color working current, implying 360 mA for ten all-white LEDs. `ASSUMPTION`: legacy/interchangeable pixels may require 60 mA each. `CALCULATED`: 10 × 60 mA = 600 mA. `DESIGN_REQUIREMENT`: 1.0 A protected output gives 67% margin over 600 mA. |
| Buzzer | switched 5 V or 3.3 V | Part not selected | Magnetic/piezo, active/passive unknown | 200 mA output envelope | `DESIGN_REQUIREMENT`: MOSFET-driven, current-limited/fused branch; select clamp only after load type is known. |
| TPS22919-Q1 load switches ×4 | associated rail | 90 mΩ typical | 1.5 A device rating; 2 nA typical shutdown | ≤1 µA aggregate parked allocation | `VERIFIED_DATASHEET` device characteristics; `DESIGN_REQUIREMENT` allocation includes temperature/leakage margin. Reverse-current behavior and discharge population must be verified per branch. |
| Miscellaneous PCB loads | 3.3 V | sensors, pull-ups and status circuits not selected | — | 20 mA active; 10 µA parked | `DESIGN_REQUIREMENT`; indicator LEDs must be off in sleep. |

## Active rail calculations

### Direct 3.3 V rail

Worst simultaneous transient envelope, with a 3.3 V display:

```text
I3V3_peak = ESP_RF + CAN_dominant + GNSS_peak + antenna
           + SD_peak + display_envelope + miscellaneous
           = 355 + 55 + 100 + 20 + 200 + 300 + 20
           = 1,050 mA                         [CALCULATED]

Required capacity = 1.050 A × 1.25 = 1.313 A [CALCULATED]
Selected architecture rating ≥2.0 A          [DESIGN_REQUIREMENT]
```

The 25% calculation margin covers overlap and first-pass tolerance, while the 2 A requirement leaves 687 mA (52% above the 1.313 A result) for startup and later refinements. It does not replace regulator thermal, inductor-saturation, capacitor, loop-stability or RF-burst testing.

An indicative active case, not a guaranteed product value, is:

```text
I3V3_typ = 100 + 7 + 50 + 10 + 50 + 150 + 20 = 387 mA [ASSUMPTION + CALCULATED]
P3V3_typ = 3.3 V × 0.387 A = 1.28 W                    [CALCULATED]
```

Here 50 mA SD and 150 mA display are assumptions pending actual parts. At 12 V and an assumed 85% converter efficiency, input current is `1.28 W / (12 V × 0.85) = 125 mA` (`CALCULATED`). At the 1.05 A envelope, 3.465 W output and assumed 85% efficiency imply 0.340 A at 12 V. These are architecture comparisons, not final converter loss calculations.

### Switched 5 V auxiliary rail

```text
IAUX5_peak = display_5V + shift_light + buzzer
           = 0.50 + 1.00 + 0.20 = 1.70 A              [CALCULATED]
Required capacity = 1.70 A × 1.15 = 1.955 A            [CALCULATED]
AUX5 rating ≥2.0 A, current limited, normally off      [DESIGN_REQUIREMENT]
```

The 5 V display allocation is not added when a module uses the 3.3 V display allocation. Firmware must not enable incompatible display supplies; hardware keying or population options must prevent simultaneous connection.

### Individual switched branches

| Branch | Peak sum | Margin calculation | Resulting requirement |
|---|---:|---:|---:|
| GNSS_3V3 including antenna | 100 + 20 = 120 mA | 120 × 1.25 = 150 mA | ≥200 mA, short-protected antenna bias |
| SD_3V3 | 200 mA | 200 × 1.25 = 250 mA | ≥250 mA; bulk capacitance set after card inrush measurement |
| DISPLAY_3V3 | 300 mA | 300 × 1.25 = 375 mA | ≥400 mA |
| DISPLAY_5V | 500 mA | 500 × 1.20 = 600 mA | ≥600 mA within the 2 A AUX5 total |
| SHIFT_5V | 1.0 A envelope | already includes 67% over 600 mA | ≥1.0 A, protected |
| BUZZER | 200 mA envelope | part still unknown | ≥200 mA, driver/clamp matched to load |

All rows are `CALCULATED` plus `DESIGN_REQUIREMENT` unless otherwise stated. Load-switch voltage drop and heating must be checked at the stated branch current.

## Complete parked-current tree

Recommended state: protected vehicle input present; main 3.3 V buck remains enabled; ESP32-S3 is in deep sleep; TCAN3404-Q1 is in standby; GNSS, SD, display, AUX5, shift-light, buzzer and all indicators are off. The table expresses current at the 12 V OBD input.

| Always-powered item | Rail allocation | 12 V input contribution | Basis |
|---|---:|---:|---|
| LM74502H-Q1 reverse-controller supply current plus input protection leakage | — | 110 µA | `VERIFIED_DATASHEET`: controller maximum operating supply-current bound used conservatively; TVS/MOSFET/fuse leakage must fit inside this allocation over temperature |
| Main buck own IQ | input | 5 µA | `DESIGN_REQUIREMENT`; LMQ66420-Q1 advertises 1.5 µA typical, but a guaranteed implementation maximum is not yet established |
| Vehicle detector and gated divider | input | 15 µA | `DESIGN_REQUIREMENT`; continuous 120 kΩ + 33 kΩ reference divider would draw `12/153k = 78.4 µA` and is therefore not retained continuously |
| TCAN3404-Q1 standby | 17 µA at 3.3 V | 7.8 µA | `VERIFIED_DATASHEET` 17 µA max; `ASSUMPTION` 60% low-load conversion; `CALCULATED`: `17µA×3.3/(12×0.60)` |
| ESP32 module branch | 50 µA at 3.3 V | 22.9 µA | `DESIGN_REQUIREMENT` 50 µA; same 60% conversion calculation |
| Wake/button/timer logic | 10 µA at 3.3 V | 4.6 µA | `DESIGN_REQUIREMENT`; button itself consumes zero except while pressed |
| Disabled load switches and rail discharge paths | — | 5 µA | `DESIGN_REQUIREMENT`, total referred to input |
| CAN-line and USB ESD / source-isolation leakage | — | 5 µA | `DESIGN_REQUIREMENT`; selected parts must prove it over temperature |
| GNSS V_BCKP | off | 0 µA | Recommended no-always-on-backup choice |
| LEDs, display, SD, AUX5, buzzer | off | 0 µA | `DESIGN_REQUIREMENT`; no always-on indicator |
| Miscellaneous leakage allocation | — | 10 µA | `DESIGN_REQUIREMENT` |
| **Subtotal** | | **185.3 µA** | `CALCULATED`; TCA6408A and residual logic leakage are included in the miscellaneous allocation pending a complete netlist |
| **100% uncertainty/temperature allowance** | | **185.3 µA** | `DESIGN_REQUIREMENT` margin |
| **Expected design envelope at 12 V** | | **≤370.6 µA (0.371 mA)** | `CALCULATED` |

The 60% low-load efficiency is an `ASSUMPTION`, deliberately below the selected regulator's headline light-load efficiency; bench characterization must replace it. At 0.371 mA, the calculation has 0.629 mA margin to the 1 mA requirement and 0.129 mA to the 0.5 mA stretch target. Accordingly:

- `<1 mA average parked` is realistically achievable (`DESIGN_REQUIREMENT`) with substantial tolerance.
- `<0.5 mA average parked` is also realistically achievable at 12 V/room temperature (`DESIGN_REQUIREMENT`), but is not accepted until complete-board measurement over voltage and temperature.
- Release limits: <0.50 mA typical at 12 V and 25 °C; <1.00 mA over the specified parked voltage/temperature range (`DESIGN_REQUIREMENT`). Wake retries, periodic timer work and post-drive shutdown time must be included in the eventual time-average measurement.

For scale only, a continuous 1 mA draws `24 mAh/day` and `0.72 Ah/30 days`; 0.371 mA draws `0.267 Ah/30 days` (`CALCULATED`, ignoring battery self-discharge and temperature). This is not a claim of safe storage duration for any vehicle battery.

## Architecture comparison

| Criterion | A: battery → 3.3 V | B: battery → 5 V → 3.3 V | C: always-on 3.3 V logic/CAN + switched peripheral rails |
|---|---|---|---|
| Conversion | One conversion for logic | Two conversions for 3.3 V loads | One conversion for logic; 5 V only when demanded |
| Heat/efficiency | Best for dominant 3.3 V load | Cascade loss; an LDO would dissipate `(5−3.3)I`, 0.66 W at 0.387 A | Similar to A in normal logic path; external-load heat isolated |
| Parked IQ | Low if buck remains enabled | Sum of primary and secondary IQ | Low with all peripheral switches off |
| Parts/area | Lowest | Higher | Highest, but explicit fault and sleep boundaries |
| RF/transients | Requires automotive front end and low-noise layout | 5 V intermediate can isolate loads but adds switchers | GNSS can have filtered switched branch; noisy 5 V rail remains off unless needed |
| Display/shift light | Needs boost or modules restricted to 3.3 V | Native 5 V available continuously unless gated | Provides both constrained 3.3 V display and switched 5 V external-load options |
| USB | Needs source OR before the buck or separate USB regulator | USB naturally matches 5 V but still needs reverse blocking | Source OR feeds logic buck; AUX5 is not back-powered by USB by default |
| Sleep/startup | ESP/CAN rail-on possible | More sequencing and back-power paths | Explicit ESP/CAN wake domain, staged peripheral startup and current limiting |
| Manufacturability | Simple | More converters/capacitors | More nets/test points, but clearer validation and configuration |

**Recommendation:** Architecture C, implemented as a hybrid rail-on system: protected battery/USB source OR → low-IQ 2 A automotive 3.3 V buck for ESP32/CAN, individually switched GNSS/SD/display branches, and a separately switched 2 A 5 V auxiliary converter for external display/shift-light/buzzer needs. This preserves reliable CAN/timer/button wake without a second wake MCU and still meets the parked-current targets on paper.

## Regulator comparison and freeze

| Candidate | Qualification / input / output | IQ and shutdown | Package / status | Advantages | Disadvantages / unresolved |
|---|---|---|---|---|---|
| TI LMQ66420-Q1 | AEC-Q100; 3–36 V operating, 42 V transient; 2 A; fixed/adjustable 3.3/5 V | 1.5 µA typical IQ; 250 nA typical shutdown | MC3 QFN, active; availability checked in Task 4 | **Frozen silicon for MAIN_3V3 and AUX5**; low parked IQ, 2 A, low-EMI features | Exact feedback/passive values, effective capacitance, thermal design and clamped transient margin remain schematic calculations |
| TI LM53602-Q1 | AEC-Q100; 3.5–36 V, 42 V transient; 2 A; 3.3/5 V | 24 µA no-load typical; 1.7 µA shutdown typical | VQFN; active; distributor availability not assessed | Mature automotive family, 2.1 MHz | Higher rail-on IQ; minimum VIN less favorable in crank |
| Infineon TLS4120D0EP V33 | Automotive; 3.7–40 V; 3.3 V, 2 A | <31 µA stated IQ | exposed-pad package; active; distributor availability not assessed | 40 V input and 2 A | Higher IQ; input minimum and transient margin must be reconciled |
| ADI MAX20006 | AEC-Q100; 3.5–36 V; 4/6/8 A family | 25 µA no-load typical | TQFN; active; distributor availability not assessed | High load margin, 40 V load-dump-tolerant family | Oversized, more area/cost and higher IQ for this budget |
| TI TPS7B82-Q1 (AON-only comparator) | AEC-Q100; 3–40 V, 45 V transient; 300 mA LDO | 2.7 µA typical / 5 µA max light-load; 300 nA shutdown | HVSSOP/WSON; active; distributor availability not assessed | Simple very-low-IQ small AON rail | Not suitable as active main rail: at 12 V to 3.3 V and 0.3 A it would dissipate 2.61 W (`CALCULATED`) |

Primary sources are listed below. LMQ66420MC3RXBRQ1 silicon is frozen for both converters. TI Table 8-5 provides a 2.2 µH, 4.7 µF input, 1 µF VCC, and two 22 µF nominal output-capacitor starting point; effective capacitance ≥40 µF, feedback, ripple, loss, stability, thermal performance, pulse headroom, and exact passive order codes remain schematic-release calculations.

## Primary sources

- Espressif, [ESP32-S3-WROOM-1/1U Data Sheet v1.8](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf), §§3.1, 6.5–6.8 and Tables 1-1, 6-2, 6-4 through 6-7.
- u-blox, [NEO-M9N-00B Data Sheet UBX-19014285 R08](https://content.u-blox.com/sites/default/files/NEO-M9N-00B_DataSheet_UBX-19014285.pdf), §§2.5, 3.1, 4.2 and Tables 7, 11, 14.
- u-blox, [NEO-M9N Integration Manual UBX-19014286 R10](https://content.u-blox.com/sites/default/files/NEO-M9N_Integrationmanual_UBX-19014286.pdf), §§2.4, 2.6, 3.2.
- TI, [TCAN3404-Q1/TCAN3403-Q1 Data Sheet SLLSFQ6A](https://www.ti.com/lit/ds/symlink/tcan3404-q1.pdf), §§6–9.
- Swissbit, *PS-66 SD-LxPT Product Data Sheet*, Rev. 1.01, electrical characteristics operating-current table.
- GalaxyCore, [GC9A01A Data Sheet v1.0 preliminary](https://buydisplay.com/download/ic/GC9A01A.pdf), §7.2 Table 44.
- Worldsemi, [WS2812B-2020 Data Sheet v1.3](https://cdn-shop.adafruit.com/product-files/4684/4684_WS2812B-2020_V1.3_EN.pdf), pp. 2–3.
- TI product data: [LMQ66420-Q1](https://www.ti.com/product/LMQ66420-Q1), [LM53602-Q1](https://www.ti.com/product/LM53602-Q1), [TPS7B82-Q1](https://www.ti.com/product/TPS7B82-Q1).
- Infineon, [TLS4120D0EP V33](https://www.infineon.com/part/TLS4120D0EP-V33).
- Analog Devices, [MAX20004/MAX20006/MAX20008](https://www.analog.com/en/products/max20006.html).

## Open verification items

- Measure ESP32-S3-WROOM-1-N16R8 branch current with flash/PSRAM and all pulls in the intended deep-sleep configuration.
- Select exact microSD, display module/backlight, active antenna, shift-light and buzzer; replace envelopes with guaranteed maxima and inrush waveforms.
- Establish load overlap policy and actual firmware duty cycles.
- Complete chosen regulator loss, junction-temperature, inductor, capacitor, stability, startup and conducted/radiated EMI calculations after the automotive transient profile and schematic are defined.
- Verify every disabled peripheral signal is high-impedance so it cannot be phantom-powered through GPIO protection structures.
