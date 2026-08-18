# Telemetry v1 requirements

Status: Task 5A.1 conditional prototype/capture re-freeze, 2026-08-17. The authoritative vehicle-input and source-selection decision is [task5a1-power-architecture.md](task5a1-power-architecture.md). Capture remains conditional on its stated calculations, tolerances, prototype gates, and review conditions; this status is not automotive qualification or compliance evidence.

## Scope and invariants

- Use RejsaCAN v3.4 as a reference, while keeping every upstream schematic, PCB, Gerber, BOM, placement, and manufacturing file unchanged.
- Create derivative hardware only under `hardware/telemetry-v1/` after this document set is reviewed.
- Connect to OBD-II pin 16 (battery), pins 4/5 (ground), pin 6 (CAN-H), and pin 14 (CAN-L).
- Support permanent installation with average parked input current <1.0 mA over the specified parked voltage/temperature range (`DESIGN_REQUIREMENT`); <0.50 mA at 12 V and 25 °C is the stretch target (`DESIGN_REQUIREMENT`).
- Retain USB firmware download, recovery, configuration, and debugging, plus microSD logging.
- Do not claim automotive robustness until the design has defined and passed relevant electrical, EMC, thermal, and environmental tests.
- Telemetry v1 includes only 12 V passenger-vehicle OBD power/protection/wake, ESP32-S3, one Classical CAN channel, OBD/ISO-TP/UDS readiness, onboard NEO-M9N with external active antenna, microSD, USB-C, BLE, interchangeable display, shift light, onboard audible alarm, MODE, status, and debug.
- TPMS, tire-temperature sensing, IMUs, analog sensor hubs, external sensor networks, a second CAN channel, and unrelated expansion are explicitly excluded from Telemetry v1.

## Functional requirements

| ID | Requirement | Initial acceptance criterion |
|---|---|---|
| CAN-01 | One Classical CAN channel using the ESP32-S3 TWAI controller | Receive and transmit 11/29-bit CAN at required vehicle bit rates; listen-only mode supported |
| CAN-02 | Generic OBD-II, ISO-TP, UDS, and raw/vehicle-profile decoding | Hardware must not couple protocol choice to vehicle or display choice |
| CAN-03 | Configurable CAN termination | Optional split 120 Ω (`DESIGN_REQUIREMENT`), DNP/OFF by default for OBD use; bench-only population documented |
| GNSS-01 | Onboard NEO-M9N-00B with external active antenna | UART at 230,400 bit/s; target navigation rate 20–25 Hz subject to configured constellations/messages and link budget; professional-grade qualification limitation recorded |
| GNSS-02 | U.FL antenna interface and active-antenna bias | Implement only from the current u-blox integration manual and selected antenna data sheet |
| GNSS-03 | Power control | Software-controlled GNSS power or backup strategy, with cold/warm-start trade-off documented |
| DSP-01 | Interchangeable external display | Initial GC9A01 240×240 SPI display; later ST7789/AMOLED drivers without core redesign |
| DSP-02 | Display connector | Frozen 14-position logical contract, switched 3.3 V/400 mA and optional 5 V/600 mA, cable ≤200 mm, initial SPI ≤20 MHz; physical connector remains a mechanical blocker |
| SHF-01 | External shift-light output | Exactly ten WS2812-compatible addressable pixels, qualified load ≤0.50 A, separately protected by TPS1H100B-Q1 at a retained 1 A fault envelope, AHCT buffer, cable ≤0.5 m |
| ALM-01 | Audible alarm | Approved-direction TPA2005D1TDGNRQ1 class-D driver and onboard 8 Ω, ≥1 W speaker within a 300 mA AUX5 branch; exact speaker/opening/back volume require acoustic qualification |
| LOG-01 | microSD logging | Concurrent CAN/GNSS logging without electrical bus contention with the display |
| BLE-01 | RaceChrono BLE link | BLE profile/protocol to be confirmed against current RaceChrono documentation |
| EXP-01 | Expansion and debug | Expose I2C, UART/debug access, useful spare GPIO, and named test points for CAN-H/L, vehicle input, 3.3 V, GNSS UART, and ground |
| PWR-01 | Permanent OBD installation | Verify active, transient and complete parked current against [`power-budget.md`](power-budget.md); release limit <1.0 mA, stretch <0.50 mA at 12 V/25 °C (`DESIGN_REQUIREMENT`) |
| PWR-02 | Wake sources | Rail-on ESP32/CAN standby wake from CAN, timer, MODE, vehicle-voltage hint and USB (`DESIGN_REQUIREMENT`) |
| PWR-03 | Power domains | `LMQ66420MC3RXBRQ1` is frozen as the vehicle converter producing `VEH_3V3`; the mux output is `MAIN_3V3`. `LMQ66420MC5RXBRQ1` is conditional for vehicle-only AUX5 pending thermal/load-step proof. Both use the exact shared `XGL5030-222MEC`, CIN/COUT/CVCC and CBOOT-DNP population frozen in `task5a1-power-architecture.md`. TPS22919-Q1 separately switches GNSS, SD, DISPLAY_3V3, and DISPLAY_5V. CAN and AUX5 are vehicle-only domains. |
| PWR-04 | USB source isolation | Use `VEH_3V3` and USB-derived `USB_3V3`, selected by TPS2116 for `MAIN_3V3`, with no back-feed or USB CAN/AUX5 power. The prototype startup source/cable must advertise and sustain ≥500 mA at 4.75 V. Post-ramp `USB_ENUM` is ≤100 mA `MAIN_3V3` (about 82 mA VBUS), not an inrush ceiling; provisional configured ceilings are 350 mA VBUS steady and 400 mA MAIN_3V3, with MAX prohibited. This contract does not establish generic legacy USB 2.0 pre-enumeration compliance (`DESIGN_REQUIREMENT`). |
| PWR-05 | Electrical envelope | Full-system operation is not permitted across the entire 6–18 V range. At 6 V, require controlled core/CAN/wake survival only. The vehicle path must have no static OV trip through 18 V, issue the OV/PD-low command on a rising input no later than 25.531 V and therefore before 26 V, and permit reconnect by 18 V. Completed isolation/recovery time and the dynamic downstream peak require model-and-capture acceptance. Verify +26 V/60 s, the exact +38 V suppressed-load-dump source, and −14 V/60 s without damage (`DESIGN_REQUIREMENT`) |
| PWR-06 | Crank/brownout and load shedding | Permit MAX for ≤10 s and ≤25% duty in any rolling 60 s only at ≥10.0 V; prohibit MAX, full-white diagnostics and Wi-Fi below 10.0 V; request logger flush below 9.5 V; after input remains below 9.0 V for 100 ms, shed nonessential loads and retain controlled core/CAN/wake survival; restore shed loads only after input remains above 10.0 V for 2 s. Avoid reboot loops, SD corruption and unintended CAN transmission. Exact timer/tolerance implementation remains a design requirement pending prototype measurement |
| PWR-07 | Severe load dump | Severe unsuppressed load dump is outside the v1 guaranteed envelope unless a later purchased-standard/OEM test profile is designed and passed |
| PWR-08 | Vehicle input implementation | Conditionally select `0437002.WRA` 2 A fuse, common-anode ST `SM15T47AY` + `SM15T33AY`, `LM74720QDRRRQ1`, and two `STL125N10F8AG` 100 V back-to-back N-MOSFETs. Production release requires all analytical and prototype gates in `task5a1-power-architecture.md` |
| FW-01 | Modular firmware | Independent CAN, OBD-II, ISO-TP, UDS, vehicle profile, GNSS, normalized data core, BLE, display, shift-light, alarms, logger, and power modules |

## Product and data architecture requirements

| ID | Requirement | Initial acceptance criterion |
|---|---|---|
| ARC-01 | Product monorepo | Hardware, firmware, vehicle profiles, protocols, clients, tools, enclosures, tests, and release evidence have explicit owners and versioned interfaces |
| DAT-01 | Normalized telemetry core | Every consumer uses canonical channel IDs, units, timestamps, source, validity, age, quality, and arbitration; consumers do not decode vehicle frames |
| DAT-02 | Source arbitration | Overlapping sources use deterministic eligibility, priority, quality, freshness, hysteresis, and fallback rules defined in `telemetry-data-model.md` |
| VEH-01 | Level 1 vehicle support | Generic OBD-II operation is available without a manufacturer profile, subject to confirmed vehicle/protocol support |
| VEH-02 | Level 2 vehicle support | Known profiles declaratively decode passive 11/29-bit Classical CAN frames and default to LISTEN_ONLY |
| VEH-03 | Level 3 vehicle support | Extended manufacturer telemetry uses explicitly enabled ISO-TP/UDS definitions and the bounded diagnostic scheduler |
| VEH-04 | Declarative profiles | Profiles express matching, bit rate, frame/decode fields, validity, diagnostic addressing/request/response, source priority, metadata, and support level without output-specific logic |
| CAN-04 | Explicit bus modes | LISTEN_ONLY cannot transmit; DIAGNOSTIC_POLLING is a deliberate state with configured rate/bus budgets, timeouts, backoff, and health reporting |
| CAN-05 | Tester coexistence | Diagnostic polling backs off or disables on congestion, arbitration loss/error escalation, or evidence of another diagnostic tester; user override remains available |
| OUT-01 | Replaceable consumers | RaceChrono, first-party protocol, display, shift-light, alarms, logger, and optional lap engine consume normalized data independently |
| OUT-02 | Headless operation | CAN/GNSS acquisition, logging, protocol, shift-light, and alarms can operate with no display installed or initialized |
| PRT-01 | Transport-independent API | First-party services and schemas are versioned independently from BLE, Wi-Fi, and USB bindings and support capability discovery |
| CFG-01 | Central configuration | Product configuration is schema-versioned, validated, atomically persisted, migratable, exportable, and recoverable to a safe default |
| CFG-02 | Local web configuration | Physical MODE authorization starts a time-bounded authenticated SoftAP session; normal telemetry remains bounded and Wi-Fi turns off on exit |
| UPD-01 | Recoverable firmware update | Signed HTTPS OTA writes only an inactive slot, checks compatibility, self-tests pending firmware and rolls back on failure |
| VEH-05 | Independently updateable profiles | Declarative, non-executable profile packages are authenticated, schema/runtime checked, staged and recoverable to built-in Generic OBD |
| UI-01 | MODE behavior | Debounced short press acknowledges an active alarm or changes local indication; ≥3 s long press requests safe configuration mode; erase/reset needs separate confirmation |
| UI-02 | Status indication | An approved LP5814-controlled common-anode RGB indicator reports state without consuming three MCU GPIOs and remains off parked |
| FUT-01 | Future tire telemetry boundary | A separate future module may publish normalized pressure, temperature, tread-temperature and health data; no tire electronics or transport is added to v1 |
| TST-02 | Layered verification | Host tests, CAN/GNSS replay, virtual/physical ECU simulation and later HIL precede controlled vehicle validation |
| TST-03 | Staged product validation | Major prototype evidence progresses through host/software, bench, stationary vehicle, road and track/slalom stages; later stages do not replace earlier evidence |
| TST-04 | Full-load interference gate | `FULL_LOAD_INTERFERENCE_TEST` simultaneously stresses permitted shift-light, sounder, SD, display, BLE, target-rate GNSS and representative CAN/diagnostic load while rail, RF, bus, storage and runtime health are recorded |
| DIA-01 | Runtime diagnostics | CAN, GNSS, BLE, Wi-Fi, SD/logger and system services expose meaningful health/error counters or state where practical, with defined semantics and expert/service access |
| RES-01 | Fault isolation | Acquisition cannot be blocked by display, storage, network, or client work; queues are bounded and timeouts, health state, restart policy, and watchdog ownership are explicit |
| RES-02 | Peripheral failure isolation | GNSS, BLE, Wi-Fi/configuration, display, SD, Tire Module, shift-light and sound failures cannot destabilize unrelated core acquisition/outputs |
| LAY-01 | EMI/GNSS/power placement gate | A dedicated placement/return-path/cable-coupling review passes before final PCB routing or manufacturing release |
| TST-05 | Professional-validation boundary | Internal functional, bench, vehicle and track tests are never represented as regulatory compliance or automotive qualification |
| TST-01 | Deterministic verification | Core, profiles, scheduling, protocols, outputs, migrations, overload, and fault behavior are testable with deterministic clocks and provenance-controlled fixtures |
| MEC-01 | Independent mechanical layers | Core enclosure, display enclosure, and vehicle mount are separately versioned and connected through controlled mechanical/electrical contracts |

## Safety and validation requirements

- Preserve ESP32-S3 GPIO19/GPIO20 for native USB and do not load GPIO0/GPIO3/GPIO45/GPIO46 without a strap analysis.
- Preserve GPIO42 for JTAG MTMS. microSD CS is GPIO11, not GPIO45; low-speed rail enables and card detect are assigned to TCA6408AQPWRQ1.
- Validate OBD input against [`task5a1-power-architecture.md`](task5a1-power-architecture.md), [`automotive-electrical-profile.md`](automotive-electrical-profile.md), and [`input-protection-architecture.md`](input-protection-architecture.md). Telemetry v1 is a 12 V passenger-vehicle product. Six volts is a controlled core/CAN/wake survival point, not a full-system operating guarantee; reference-design pulse conditions remain test inputs, not compliance claims.
- Review CAN protection, common-mode range, ESD, termination, grounding, and non-automotive-qualified reference components.
- Perform regulator worst-case input/transient, load, thermal, stability, startup, shutdown, reverse-polarity, and back-power analyses.
- Establish RF keep-outs, controlled-impedance rules, antenna bias filtering/protection, and conducted/radiated noise targets before GNSS layout.
- Define connector pin numbering, load limits, short-circuit behavior, cable length, ESD protection, and hot-plug behavior for every external interface.
- Add test points without creating high-stub or antenna structures on CAN, USB, SPI, or GNSS RF nets.
- Control staged procedures, instrumentation, evidence and future acceptance limits through [system-validation-plan.md](system-validation-plan.md).

## Deferred decisions

- Purchased-standard/OEM pulse details, repetition/acceptance classes, exact temperature grade, enclosure/cable environment, severe-unsuppressed-load-dump need, compliance markets, and production volume.
- Exact implementation tolerances for the 100 ms low-voltage shed timer and 2 s recovery timer remain design requirements pending prototype measurement; their nominal thresholds and durations are not permission to run MAX loads below 10 V.
- Whether a later hardware-off CAN wake variant is worth its added 5 V/AON sequencing; it is not required for Telemetry v1 unless rail-on measurements fail.
- Exact physical display connector family and module adapters; the electrical rail/current/cable contract is frozen.
- Exact onboard 8 Ω speaker, acoustic opening/back volume, sealed acoustic path, and measured in-cabin sound-pressure acceptance limits.
- GNSS V_BCKP follows switched GNSS power in v1; host save/restore performance remains to be tested.
- Firmware platform/dependency versions, profile/config serialization formats, canonical wire encoding, full security/threat model, signing-key process, and runtime resource budgets.
- Exact profile matching policy, supported diagnostic services per vehicle, and the criteria for detecting/coexisting with another scan tool.
- First-party app platforms and BLE/Wi-Fi/USB transport bindings.
- Core/display mount interface, enclosure material/process, environmental limits, and mount load cases.

## Audit disposition

- Earlier 5–24 V wording was a repository/reference claim, not a Telemetry v1 requirement; it is no longer used as a design fact.
- Earlier full-operation-at-6–18 V wording is superseded: 6 V now means controlled core/CAN/wake survival after load shedding, not unrestricted operation.
- Earlier OV targets of turn-off at or below 20 V and `VEHICLE_PROTECTED` at or below 24 V are superseded by the Task 5A.1 tolerance-bounded static-command requirements: no trip through 18 V, OV/PD-low command by 25.531 V before 26 V, and reconnect eligibility by 18 V. Completed switching and dynamic peak remain validation gates.
- The corrected Task 5A.1 maximum-load total is 8.27505 W; the historical 12.458 W aggregate is superseded and must not be used for fuse, regulator, thermal, or low-voltage conclusions.
- Earlier termination, switched-load capacity, start threshold, display current and peripheral-current wording is superseded by the classified values or explicit unresolved items in `power-budget.md` and `power-wake-review.md`.
- GPIO numbers are evidence from the v3.4 schematic/pinout and are classified in `interfaces.md`; they are not new electrical calculations.
