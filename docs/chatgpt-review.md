## Task completed

Completed the second pre-schematic Telemetry v1 engineering phase. Audited prior documentation under the repository traceability rules; created reproducible power/parked-current budgets; compared power and wake architectures; and resolved preliminary automotive-input, CAN, GNSS, display, shift-light, buzzer and USB electrical architecture. No schematic, PCB, footprint, firmware or manufacturing file was changed.

## Files changed

- `docs/power-budget.md` — created classified load, rail, margin, parked-current and regulator-comparison calculations.
- `docs/power-wake-review.md` — created architecture, state machine, protection, CAN, GNSS, display, shift-light, buzzer, USB and decision review.
- `docs/requirements.md` — added parked-current/domain requirements and audit disposition; removed 24 V as a v1 requirement.
- `docs/architecture.md` — adopted the hybrid rail-on domain and state model.
- `docs/hardware-spec.md` — recorded classified electrical envelopes and proposed parts/interfaces.
- `docs/interfaces.md` — expanded the universal display and shift-light contracts; classified repository GPIO evidence.
- `docs/power.md` — audited reference claims and summarized approved domain capacities and parked result.
- `docs/rejsacan-analysis.md` — classified repository evidence, superseded stale preliminary power/wake conclusions and refreshed unresolved questions.
- `docs/telemetry-v1-change-list.md` — updated KEEP/MODIFY/ADD actions and review gates.
- `docs/chatgpt-review.md` — replaced with this latest review packet.

## Engineering findings

- `VERIFIED_DATASHEET`: ESP32-S3-WROOM-1 RF peak is 355 mA for the highest documented Wi-Fi case; the module supply must be able to deliver at least 0.5 A. BLE's documented maximum peak is 344 mA and is not added simultaneously to Wi-Fi.
- `VERIFIED_DATASHEET`: NEO-M9N four-GNSS continuous tracking is 36 mA and peak acquisition is 100 mA at documented conditions; active antennas are commonly 5–20 mA in the u-blox integration manual. High-rate current remains unguaranteed.
- `DESIGN_REQUIREMENT`: interchangeable microSD, display, shift-light and buzzer loads use conservative envelopes because exact products are not selected: SD 200 mA peak at 3.3 V; display 300 mA at 3.3 V or 500 mA at 5 V; shift light 1 A at 5 V; buzzer 200 mA.
- `CALCULATED`: MAIN_3V3 transient sum is 1.050 A; 25% margin gives 1.313 A. The required architecture rating is ≥2.0 A (`DESIGN_REQUIREMENT`).
- `CALCULATED`: AUX5 external-load sum is 1.70 A; 15% margin gives 1.955 A. The required switched rating is ≥2.0 A (`DESIGN_REQUIREMENT`).
- `CALCULATED`: the complete 12 V parked tree subtotal is 80.3 µA. A 100% uncertainty/temperature allowance gives ≤161 µA (0.161 mA). This includes protection/source leakage, buck IQ, vehicle sensing, TCAN standby, ESP module allocation, wake logic, disabled load switches, interface ESD and miscellaneous leakage.
- `DESIGN_REQUIREMENT`: release parked current is <1.0 mA across the specified parked range; <0.50 mA at 12 V/25 °C is the stretch target. Both are feasible on paper, but require populated-board tests including wake duty cycle.
- The reference's 5–24 V README wording is not a v1 requirement or transient-survival proof. Telemetry v1 is a 12 V passenger-vehicle design; 24 V support is not required.
- The reference DSS34/PTC/SMF30A/LMR14006X chain is not accepted as a coordinated automotive protection solution. Fuse, reverse MOSFET, TVS/surge clamp, filter and regulator must be selected together after a 12 V test profile is agreed. No standards-compliance claim is made.
- `CALCULATED`: an additional 120 Ω terminator on the already terminated 60 Ω-equivalent vehicle bus yields 40 Ω. Optional split 120 Ω termination is therefore DNP/OFF by default and populated only for isolated bench endpoint use.
- TCAN3404-Q1 is the recommended transceiver: AEC-Q100, single 3.3 V supply, maximum 17 µA standby, WUP wake reporting, ±58 V bus standoff, ±30 V common-mode range and high-impedance unpowered bus pins (`VERIFIED_DATASHEET`). Add a dedicated low-capacitance AEC-Q101 CAN TVS and a bypassed/DNP common-mode-choke option.
- GNSS uses a switched ≥200 mA 3.3 V branch with protected/filtered active-antenna bias. V_BCKP follows switched VCC in v1, giving zero parked GNSS current; investigate u-blox host UBX save/restore instead of a cell/supercapacitor.
- With VCC/V_BCKP removed, GNSS normally cold-starts; host-restored assistance is preferred to a permanent backup load. The u-blox product summary's 24 s cold and 2 s aided/hot values are typical test-condition comparisons, not product acceptance limits (`VERIFIED_DATASHEET`).
- `ASSUMPTION`: 200 serial bytes per 25 Hz navigation epoch. `CALCULATED`: 50 kbit/s at 8N1; ≤50% occupancy requires ≥100 kbit/s. Use 230,400 bit/s (`DESIGN_REQUIREMENT`), then validate the exact UBX message set/configuration.
- The proposed display interface supplies dual ground, switched 3.3 V, optional switched 5 V, SPI including optional MISO, CS/DC/reset/backlight control, optional I2C and INT/TE. Initial cable is ≤200 mm and SPI ≤20 MHz (`DESIGN_REQUIREMENT`). `CALCULATED`: 240×240 RGB565 is 921,600 bits/frame and approximately 16.3 frame/s at 20 MHz with an assumed 75% bus efficiency.
- The simplest robust v1 shift light is a short-cable, protected/current-limited 5 V/1 A addressable-LED branch with buffered 5 V data, ESD, damping and default-off behavior. Longer/noisier cables require an intelligent/differential external module.
- USB power is source-ORed with reverse blocking before the main buck. Vehicle-only and USB-only operation are supported; simultaneous connection back-feeds neither USB VBUS nor OBD pin 16. USB-only mode powers development logic and selected internal peripherals but not AUX5 external loads by default.

## Decisions made

- Recommend Architecture C: hybrid rail-on MAIN_3V3 for ESP32/TCAN plus individually switched GNSS, SD, display and AUX5 domains.
- Prefer LMQ66420-Q1 as the main-buck candidate, not a final selection; full transient, worst-case IQ, passives, stability, thermal and EMI work remains.
- Use rail-on ESP32 deep sleep plus TCAN3404-Q1 standby/WUP rather than a TCAN1043A-Q1 hardware-off wake domain for v1.
- Replace SN65HVD230 with TCAN3404-Q1 and default CAN termination to DNP/OFF.
- Do not power GNSS backup continuously in v1.
- Use 230,400 bit/s for the preliminary GNSS UART contract.
- Provide both constrained switched 3.3 V and optional switched 5 V display supplies through a keyed/module-specific contract.
- Use a 5 V addressable shift-light interface with a data buffer and 1 A current limit.
- Use a MOSFET buzzer driver; exact buzzer and clamp remain unselected.
- Use reverse-blocked OBD/USB source OR and keep AUX5 off by default in USB-only mode.

## Assumptions

- 100 mA is the representative active ESP application current; actual firmware/RF duty cycle is unknown.
- 50 mA is the provisional continuous NEO-M9N allocation at 25 Hz because the official 36 mA tracking measurement is at 1 Hz.
- The GNSS antenna draws no more than 20 mA normally.
- The serial-output envelope is 200 bytes per navigation epoch.
- Representative SD/display active values used for an indicative typical case are 50 mA and 150 mA; exact modules are not selected.
- A legacy/interchangeable addressable RGB LED may draw 60 mA full-white, despite the reviewed WS2812B-2020 specifying 12 mA per channel.
- Main-converter efficiency is 85% for indicative active calculations and 60% for low-load rail-current referral; both require bench replacement.
- Display-bus payload efficiency is 75% for refresh calculation.
- Shift-light cable is local/short; beyond 0.5 m triggers an intelligent/differential-module requirement pending EMC tests.

## Uncertainties / unresolved questions

- Exact 12 V transient, cranking, jump-start, reverse-battery, ESD, temperature and acceptance test profile, including any OEM-specific severity.
- Exact protection, main regulator, AUX5 regulator, load-switch, source-OR, CAN TVS and optional choke orderable parts and their full worst-case calculations.
- ESP32-S3-WROOM-1-N16R8 complete branch current with PSRAM/flash, pulls and intended deep-sleep wake sources over temperature.
- GPIO13 deep-sleep CAN-wake operation and all peripheral GPIO high-impedance/back-power behavior.
- Exact active antenna, microSD, display/adapter, shift-light and buzzer; their maxima, inrush, fault behavior, temperature grade and availability.
- Exact 25 Hz UBX message list/throughput, high-rate current and host save/restore behavior; the module's up-to-25 Hz support for all supported constellation combinations is verified.
- Display connector family, pin numbering, cable impedance/ground arrangement and module power keying.
- Product ground/chassis strategy, enclosure, cable environment and regulatory markets.

## Risks

- A 42 V-class buck can be overstressed if a selected TVS clamps above its safe input during the actual load-dump current; clamp coordination is the highest power risk.
- N16R8 PSRAM/module leakage or GPIO phantom powering could exceed the parked allocation even though the ESP32 chip deep-sleep figure is small.
- RF/SD/display current overlap and external-load inrush can cause brownout unless capacitance, switch current limit, sequencing and regulator transient response are verified.
- Fast buck/SPI/LED edges and external cables can desensitize GNSS or fail conducted/radiated EMC.
- A wrong display adapter or dual-rail configuration could apply 5 V to a 3.3 V-only module.
- CAN wake noise/retries can raise average parked current; diagnostic transmission after wake could disturb a vehicle if passive and active policies are not separated.
- USB/vehicle source-selection leakage or reverse conduction could energize a host or vehicle net unexpectedly.
- SD power loss during shutdown can corrupt logs unless brownout detection and flush energy/time are characterized.

## GPIO / peripheral changes

No GPIO allocation changed in this task. Electrical peripheral decisions changed as follows:

| Peripheral | Existing preliminary GPIO | Electrical decision |
|---|---:|---|
| CAN RX/TX/STB | 13 / 14 / 38 | TCAN3404-Q1; RXD WUP; reset defaults to standby |
| GNSS RX/TX | 15 / 16 | 230,400 bit/s contract; switched ≥200 mA GNSS rail |
| SPI SCLK/MOSI/MISO | 39 / 40 / 41 | Shared SD/display with separate CS, arbitration and off-state isolation |
| Display CS/DC/reset/BL | 47 / 48 / 12 / 7 | Universal connector and driven backlight control |
| Shift-light | 6 | Buffered data plus protected 5 V/1 A branch |
| Buzzer | 42 | MOSFET driver, ≤200 mA branch envelope |
| MODE | 10 | Non-strap wake input; leakage/debounce still to be designed |

## Power / CAN / RF impact

- **Automotive power:** changes from an unverified reference 600 mA/single-switch path to a calculated ≥2 A MAIN_3V3, independently switched peripherals and ≥2 A AUX5. The reference protection is not reused without new coordination.
- **CAN:** replaces standard SN65HVD230 with automotive TCAN3404-Q1, dedicated CAN TVS, optional DNP choke and DNP split termination; enables reliable low-current rail-on wake.
- **GNSS/RF:** adds a switched ≥200 mA domain, zero-current parked backup policy, protected active-antenna bias and a 230,400-bit/s UART contract. RF trace/layout remains deliberately unstarted.
- **USB:** adds reverse-blocked source OR and explicit behavior for all four source cases; native GPIO19/20 USB remains unchanged.
- **ESP32 boot/strapping:** no allocation change. GPIO45 microSD CS remains a documented strap risk; power-off isolation and wake GPIO behavior require schematic/bench verification.

## Datasheets / primary sources consulted

- Repository `Schematics/RejsaCAN v3.x (ESP32-S3 based board)/RejsaCAN v3.4 - Schematic.json` and matching single-sheet PNG, including U2/U3/U4/U5, D4/D6, F1, input dividers, USB source path and CAN termination.
- Espressif, *ESP32-S3-WROOM-1/1U Data Sheet* v1.8, Tables 1-1 and 6-2 through 6-7.
- TI, *SN65HVD230 3.3-V CAN Transceiver* SLLS500K.
- TI, *TCAN3404-Q1/TCAN3403-Q1* SLLSFQ6A, Tables 7-1/7-2 and §§8–9.
- TI, *TCAN1043A-Q1* SLLSF27, electrical characteristics and mode descriptions.
- TI product/data-sheet material for LMQ66420-Q1, LM53602-Q1 and TPS7B82-Q1.
- Infineon product/data-sheet material for TLS4120D0EP V33.
- Analog Devices product/data-sheet material for MAX20004/MAX20006/MAX20008.
- u-blox, *NEO-M9N-00B Data Sheet* UBX-19014285 R08 and *NEO-M9N Integration Manual* UBX-19014286 R10.
- Swissbit, *PS-66 SD-LxPT Product Data Sheet* Rev. 1.01.
- GalaxyCore, *GC9A01A Data Sheet* v1.0 preliminary, §7.2 Table 44.
- Worldsemi, *WS2812B-2020 Data Sheet* v1.3.
- STMicroelectronics, *LDP01-xxAY automotive load-dump TVS data sheet*, *TN1517 TVS FAQ*, and ESDCAN04-2BWY product data.
- ISO 7637-2:2011 and ISO 16750-2:2023 official scope pages.

## Recommended next step

Perform the exact-component selection review for the 12 V input-protection chain, MAIN_3V3/AUX5 regulators, source OR, load switches and CAN protection against an agreed electrical test profile, producing worst-case loss, thermal, pulse-energy, stability and leakage calculations before schematic capture.

## STOP condition

The requested pre-schematic power/wake architecture review is complete. Stop here and wait for review; do not begin schematic capture, PCB layout, footprint assignment, firmware, procurement or manufacturing outputs.
