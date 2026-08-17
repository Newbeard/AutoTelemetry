# Telemetry v1 requirements

Status: Task 4.6 research freeze, audited 2026-08-17. Remaining blockers are listed in [`schematic-architecture.md`](schematic-architecture.md). Numerical evidence uses `VERIFIED_DATASHEET`, `CALCULATED`, `DESIGN_REQUIREMENT`, and `ASSUMPTION` as defined in [`power-budget.md`](power-budget.md).

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
| ALM-01 | Audible alarm | Proposed TPA2005D1-Q1 class-D driver and onboard 8 Ω, ≥1 W speaker within a 300 mA AUX5 branch; exact speaker/opening/back volume require acoustic qualification |
| LOG-01 | microSD logging | Concurrent CAN/GNSS logging without electrical bus contention with the display |
| BLE-01 | RaceChrono BLE link | BLE profile/protocol to be confirmed against current RaceChrono documentation |
| EXP-01 | Expansion and debug | Expose I2C, UART/debug access, useful spare GPIO, and named test points for CAN-H/L, vehicle input, 3.3 V, GNSS UART, and ground |
| PWR-01 | Permanent OBD installation | Verify active, transient and complete parked current against [`power-budget.md`](power-budget.md); release limit <1.0 mA, stretch <0.50 mA at 12 V/25 °C (`DESIGN_REQUIREMENT`) |
| PWR-02 | Wake sources | Rail-on ESP32/CAN standby wake from CAN, timer, MODE, vehicle-voltage hint and USB (`DESIGN_REQUIREMENT`) |
| PWR-03 | Power domains | LMQ66420MC3RXBRQ1 for ≥2.0 A MAIN_3V3 and ≥2.0 A switched AUX5; TPS22919-Q1 separately switches GNSS, SD, DISPLAY_3V3, and DISPLAY_5V |
| PWR-04 | USB source isolation | Support vehicle-only, USB-only, simultaneous and unpowered cases with no back-feed to OBD pin 16 or USB VBUS (`DESIGN_REQUIREMENT`) |
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
| UI-02 | Status indication | A proposed LP5814-controlled common-anode RGB indicator reports state without consuming three MCU GPIOs and remains off parked |
| FUT-01 | Future tire telemetry boundary | A separate future module may publish normalized pressure, temperature, tread-temperature and health data; no tire electronics or transport is added to v1 |
| TST-02 | Layered verification | Host tests, CAN/GNSS replay, virtual/physical ECU simulation and later HIL precede controlled vehicle validation |
| RES-01 | Fault isolation | Acquisition cannot be blocked by display, storage, network, or client work; queues are bounded and timeouts, health state, restart policy, and watchdog ownership are explicit |
| TST-01 | Deterministic verification | Core, profiles, scheduling, protocols, outputs, migrations, overload, and fault behavior are testable with deterministic clocks and provenance-controlled fixtures |
| MEC-01 | Independent mechanical layers | Core enclosure, display enclosure, and vehicle mount are separately versioned and connected through controlled mechanical/electrical contracts |

## Safety and validation requirements

- Preserve ESP32-S3 GPIO19/GPIO20 for native USB and do not load GPIO0/GPIO3/GPIO45/GPIO46 without a strap analysis.
- Preserve GPIO42 for JTAG MTMS. microSD CS is GPIO11, not GPIO45; low-speed rail enables and card detect are assigned to TCA6408AQPWRQ1.
- Validate OBD input against a written 12 V passenger-vehicle transient profile. 24 V operation is explicitly not a Telemetry v1 requirement (`DESIGN_REQUIREMENT`). An input-voltage range is not a transient-survival specification.
- Review CAN protection, common-mode range, ESD, termination, grounding, and non-automotive-qualified reference components.
- Perform regulator worst-case input/transient, load, thermal, stability, startup, shutdown, reverse-polarity, and back-power analyses.
- Establish RF keep-outs, controlled-impedance rules, antenna bias filtering/protection, and conducted/radiated noise targets before GNSS layout.
- Define connector pin numbering, load limits, short-circuit behavior, cable length, ESD protection, and hot-plug behavior for every external interface.
- Add test points without creating high-stub or antenna structures on CAN, USB, SPI, or GNSS RF nets.

## Deferred decisions

- Exact automotive pulse severity, temperature grade, enclosure, cable environment, compliance markets, and production volume.
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
- Earlier termination, switched-load capacity, start threshold, display current and peripheral-current wording is superseded by the classified values or explicit unresolved items in `power-budget.md` and `power-wake-review.md`.
- GPIO numbers are evidence from the v3.4 schematic/pinout and are classified in `interfaces.md`; they are not new electrical calculations.
