# User I/O, configuration and future Tire Module architecture

Status: Task 5A.1 user-I/O/power-domain amendment, 2026-08-17. The vehicle-input, USB-source and AUX5 choices are conditional prototype/capture selections with unresolved bench gates. This document does not authorize firmware, KiCad schematic/PCB work, flex layout, CAD, Tire Module electronics or manufacturing, and makes no compliance claim. See [`task5a1-power-architecture.md`](task5a1-power-architecture.md).

## Scope and disposition

Telemetry v1 keeps the ESP32-S3, one Classical CAN channel, GPIO map, 1 A protected shift-branch fault envelope and three-wire shift-light contract. Task 5A.1 conditionally retains LMQ66420MC5RXBRQ1 for vehicle-only AUX5 under the STREET/TRACK/MAX contracts below; the continuous 2 A interpretation is superseded. The future AutoTelemetry Tire Module remains a separate product.

Task 4.7 resolves two previously proposed Task 4 hardware changes:

- **APPROVED DIRECTION — sounder:** replace the MOSFET-only buzzer path with a 5 V mono class-D amplifier and small speaker. GPIO17 and the AUX5 domain remain unchanged.
- **APPROVED — status:** replace the one-color expander LED sink with one RGB LED driven by an I2C RGB driver. No direct ESP32 GPIO is added; expander P6 becomes the driver's hardware enable/default-off control.

Those Task 4.7 user-I/O decisions remain in force, but their rail behavior is now governed by the Task 5A.1 source partition.

## Task 5A.1 source and visible-output contract

The core rail is now selected after conversion: vehicle `LMQ66420MC3RXBRQ1` produces `VEH_3V3` on TPS2116 VIN1; USB `TPS2553QDBVRQ1` plus fixed-3.3 V `TPS62162QDSGRQ1` produces `USB_3V3` on VIN2; TPS2116 VOUT is `MAIN_3V3`. TCAN3404 VCC and the normally-off AUX5 converter remain physically vehicle-only. A user setting or service UI must never override that isolation.

| Vehicle | USB | Core / user-visible behavior |
|---|---|---|
| absent | absent | Device and all indicators/outputs are off |
| present | absent | Valid vehicle PGOOD gives VIN1 priority; core and CAN are available, while optional display/shift/sound rails follow state and load policy |
| absent | present | `USB_3V3` powers only the `MAIN_3V3` development/core domain; CAN, AUX5, 5 V display, shift light and sounder remain physically off |
| present | present | Valid vehicle power retains priority; USB remains isolated and available for data/handoff. Loss of vehicle PGOOD may hand the core to USB, while CAN/AUX5 turn off with the vehicle rail |

TPS2116 reverse-current blocking plus LM74720 vehicle-side reverse blocking provide the intended no-backfeed architecture, but four-state ramps, brownout, handover and abnormal connections remain bench gates. USB presence never authorizes CAN transmission. USB-only uses the exact conditional TPS2553/TPS62162/TPS2116 population and begins with optional loads off. The TPS2553 60.4 kΩ ±1% RILIM screens a 387.2–491.3 mA fault/current-limit population band, not a load contract. The prototype startup source/cable must advertise and sustain ≥500 mA at 4.75 V. After ramp, `USB_ENUM` is ≤100 mA `MAIN_3V3` (about 82 mA VBUS), not an inrush ceiling; the provisional configured ceiling is 350 mA VBUS steady and 400 mA `MAIN_3V3`, with MAX prohibited. This does not establish generic legacy USB 2.0 pre-enumeration compliance; later firmware must honor the actual host/Type-C contract.

AUX5 user modes are constrained as follows: STREET 0.56001 A continuous, TRACK 0.92001 A continuous, and MAX 1.16001 A for no more than 10 s and 25% of any rolling 60 s. A 1.450 A sizing value is not a user-operating mode. Below 10.0 V the power manager prohibits MAX/full-white diagnostics and Wi-Fi; below 9.5 V it starts a bounded SD flush and no new optional work; below 9.0 V for 100 ms it enters `LOW-VOLTAGE SHED`, turning AUX5/display/shift/sound/Wi-Fi off and stopping new SD writes; recovery requires >10.0 V stable for 2 s. UI status must explain shedding without offering an unsafe override. All thresholds and timers remain prototype requirements pending ADC/front-end, crank and SD-flush measurement.

## External flex shift-light

### Frozen product contract

The remote assembly is exactly 10 individually addressable RGB light points on a thin flexible PCB. Only the LEDs, one manufacturer-required local decoupling capacitor per pixel, interconnect copper, connector/cable termination, and purely mechanical light-control/strain-relief features may be remote. There is no remote MCU, regulator, button, analog brightness wire, or separate control board.

The interface remains:

| Signal | Contract |
|---|---|
| `SHIFT_5V` | switched and protected 5 V; 0.50 A qualified-load design envelope; existing 1.0 A current-limited switch/fault envelope |
| `GND` | dedicated power/data return; not a shield substitute |
| `SHIFT_DATA` | regenerated 5 V single-ended addressable-pixel data |
| Cable | attached or locking/keyed three-conductor harness, <=0.5 m |
| Connector | >=1.5 A per power contact after environmental derating; keyed, latching, touch-safe and strain relieved |

The strip is off during reset, `PARKED/SLEEP`, USB-only service, and a shift-branch fault. Maximum allowed software brightness must be electrically safe even though normal patterns use less power.

### Addressable LED comparison

Values are manufacturer data unless marked otherwise. Availability is a purchasing snapshot, not a lifetime guarantee.

| Family / representative | Package and optics | Electrical/data | Temperature and manufacturing | Decision |
|---|---|---|---|---|
| Worldsemi `WS2812B-2020` / preferred V6 family member | 2.0 x 2.0 x about 0.84 mm; earlier v1.3 gives 150/500/100 mcd minima and 2020 format | V6 family page: 3.3-5.5 V, 12 mA per color, 8-bit/color, 800 kbit/s one wire; every pixel reshapes data; V6 VIH is 0.55 VDD in the reviewed data sheet | V6 reviewed data: -40 to +85 C; compact four-pad part is suitable for flex assembly, but handling/MSL and exact sourcing require contract-manufacturer review | **Preferred family frozen.** Exact `WS2812B-2020-V6` order/revision remains `PROVISIONAL` until English controlled data sheet, lot identity, optical sample and production availability are verified. |
| OPSCO `SK6812MINI-E` | 3.2 x 2.8 x 1.78 mm, 120 degree class package | 3.7-5.5 V; 12 mA per color; 8-bit/color; 800 kbit/s; >=1.2 kHz PWM; regenerated one-wire output; VIH >=0.7 VDD | -40 to +85 C, MSL 5a; larger/taller than 2020 | Viable fallback with better published temperature range than old WS2812 revisions, but less mechanically attractive. |
| Brightek `ISA2020VGBC1CKA1` | 2.0 x 2.0 x 0.75 mm; 140 degree; 75/230/55 mcd typical at 5 mA/color | 5 mA/color version, 8-bit class smart-pixel family, single-wire variants and sleep support | -40 to +85 C shown for the family; manufacturer data and CAD are available | Attractive low-current alternate. Do not substitute without protocol, timing, local-passive, brightness and sourcing qualification. |
| Brightek `ISA2020VGBC1CJK3` | 2.0 x 2.0 x 0.75 mm; 140 degree | 20 mA/color, 8-bit PWM plus 5-bit brightness, separate data and clock | -40 to +85 C, MSL 5 | Rejected for v1: four-wire electrical contract and 600 mA white load provide no demonstrated product benefit. |

Old WS2812B-2020 revisions are not interchangeable by name: reviewed v1.0 data use 16 mA/color and a 0.7 VDD input threshold, while v1.3 uses 12 mA/color and a fixed 2.7 V VIH. The BOM and incoming inspection must therefore control the exact revision; unqualified marketplace substitutions are prohibited.

Each pixel gets the data-sheet-required 100 nF local capacitor unless the final controlled revision explicitly requires a different network. Place local bulk capacitance at the cable entry only after inrush and pattern-edge measurements; it does not replace per-pixel decoupling.

### Reproducible 10-pixel power envelope

For the preferred 12 mA/color family:

```text
Ichannel       = 12 mA                         [VERIFIED_DATASHEET]
Ipixel_white   = 3 x 12 mA = 36 mA            [CALCULATED]
I10_white      = 10 x 36 mA = 360 mA          [CALCULATED]
Iq10_V6        <= 10 x 1 uA = 0.010 mA         [VERIFIED_DATASHEET + CALCULATED]
Iqualified_max = 360.010 mA, rounded 0.361 A   [DESIGN_REQUIREMENT]

Normal race pattern definition:
10 pixels, one color channel, 50% protocol/PWM brightness
Inormal        = 10 x 12 mA x 0.50 + 0.010 mA
               = 60.010 mA                    [CALCULATED]

Ibranch_design = 0.361 A x 1.25
               = 0.451 A, rounded to 0.50 A   [CALCULATED + DESIGN_REQUIREMENT]
```

A full-brightness one-color TRACK flash is about 120.01 mA. Full-white at 100% remains an allowed software command and is the qualified maximum, not a normal race pattern.

The existing TPS1H100B-Q1 branch remains configured as a 1.0 A protected/fault envelope. The 1.0 A switch rating is deliberately distinct from the 0.50 A qualified load contract: it supplies inrush/interchange/fault headroom but does not authorize a 1 A remote strip. The connector remains >=1.5 A after derating.

For a representative 26 AWG stranded copper cable, Belden publishes 44.4 ohm/1000 ft = 0.146 ohm/m. A 0.5 m one-way cable has a 1.0 m round trip:

```text
Vdrop_25C = 0.50 A x 0.146 ohm = 0.073 V       [CALCULATED]
Vdrop_hot = 0.073 V x 1.25 = 0.091 V           [ASSUMPTION: 25% hot/aging allowance]
```

Allowing 50 mV for the mated connector gives a 0.141 V design drop. Use 26 AWG minimum for `SHIFT_5V` and `GND`; 24 AWG is preferred where bend/connector geometry permits. A 28 AWG data conductor is acceptable. The final cable assembly requires its own temperature, flex, current and contact-resistance qualification.

### Data path and EMC

```text
ESP32-S3 GPIO6
  -> CAHCT1G126-Q1, VCC=5 V, OE held disabled until SHIFT5 is valid
  -> 33 ohm source-series footprint
  -> connector-side low-capacitance ESD device
  -> <=0.5 m cable with data adjacent to/twisted with GND
  -> pixel 1 DI -> regenerated chain through pixel 10
```

The retained AHCT buffer is required even though the preferred V6 VIH is lower. It provides a controlled 5 V launch level, isolation while the strip is unpowered, and margin against revision substitution, ground shift and cable loss. Direct 3.3 V drive is not the product contract. The starting series value is 33 ohm, with 22-47 ohm as the measurement range. Select the exact ESD part after connector geometry is known; its capacitance and clamp behavior must preserve the 800 kbit/s waveform.

Do not share load current through a thin signal return. Keep loop area small, avoid a cable shield carrying DC return current, suppress branch power before data, and verify idle/reset transitions, hot plug, ESD, radiated emissions and pixel-chain failure. A failed early pixel can interrupt downstream data; this is accepted for the simple v1 strip and must be detected by a lamp-test/user diagnostic rather than extra remote electronics.

### Mechanical architecture

The future assembly is:

```text
flex PCB + 10 discrete smart RGB pixels + local capacitors
+ black flexible TPU/silicone optical separator/surround
+ attached strain-relieved three-conductor cable
```

Ten dark wells keep the points visually distinct. The flex has a declared bend axis, minimum static bend radius, component keep-outs and stiffened cable transition; components and solder joints do not sit in the repeated-bend zone. Attachment is removable automotive-grade hook-and-loop, Dual Lock, or an equivalent replaceable pad. Adhesive, solar load, glare/reflection, residue, temperature, vibration and visor compatibility require validation. No CAD or flex layout is frozen here.

### Brightness and pattern configuration

The Configuration Manager owns enable, source channel, RPM start/progression/flash thresholds, ten point colors, progression direction/pattern, flash cadence, and profile brightness. User brightness is 1-100%; 0% is equivalent to disabled. Initial policy defaults are NIGHT 10%, DAY 40%, and TRACK 70% (`DESIGN_REQUIREMENT` defaults, not optical pass limits). TRACK may be configured up to 100%. Gamma/color correction and current limiting occur in software over the addressable protocol; there is no analog brightness wire. Missing/stale RPM makes the strip dark and reports a fault rather than holding the last pattern.

## Onboard audible warning

### Architecture comparison

| Architecture | Control and sound | Power/space | Environment/limitations |
|---|---|---|---|
| Active buzzer | Internal oscillator; easy on/off and coarse PWM gating | Low parts count, often tens of mA | Fixed dominant frequency and limited clean volume control; cannot provide a useful family of tones. |
| Passive piezo | Efficient near resonance; PWM tone and duty control | Low average current, may require boosted/bridge voltage for high SPL | Narrow response and strong enclosure/resonance dependence. Reviewed TDK PS example is only 60 dBA at 10 cm and -10 to +70 C. |
| Electromagnetic/magnetic transducer | Direct tone control; examples reach 80-85 dBA at 10 cm and -40 to +85 C | Up to 60-100 mA; low-side driver and inductive clamp | TDK automotive examples are EOL/not recommended for new design; narrow resonant behavior and flyback management. |
| Small speaker plus amplifier | Wide tone/pattern range and true amplitude control; best path to STREET/TRACK profiles | More PCB area, speaker back-volume/opening and about 0.30 A 5 V branch | Acoustic output depends on selected speaker, enclosure and installation; requires EMI and hearing-safety validation. |

**Recommendation:** small 8 ohm speaker plus a mono analog-input class-D amplifier.

**APPROVED DIRECTION:** use exact order code `TPA2005D1TDGNRQ1` in its MSOP-PowerPAD package. TI specifies 2.5-5.5 V operation and -40 to +105 C for the T-suffix order code, 1.4 W typical into 8 ohm at 5 V/10% THD, 84% efficiency at 400 mW, 2.8 mA quiescent current, 0.5 uA shutdown, and short/thermal protection. The exact speaker remains `PROVISIONAL`; it must be at least 1 W rated, 8 ohm, suitable for the final temperature and mounting environment, and qualified in the real enclosure. PUI `AS01808AO-SC18-WP-R` demonstrates that a 1 W, 8 ohm, 18 x 13 x 2.5 mm, IP68-face device with 96 +/-3 dBA at 10 cm in a 1 cc test volume is feasible; it is a reference, not the frozen transducer.

The existing GPIO17 generates a high-rate PWM/audio waveform into a reconstruction/coupling network; it never sources speaker current. The amplifier drives the speaker differentially from AUX5. Hardware gain and input clamp/filter values must ensure that every GPIO/PWM state is safe. Software amplitude sets volume; tone frequency, cadence and envelope provide multiple alarm signatures. `STREET` and `TRACK` are bounded volume presets, with user mute and percentage control. Critical alarm state remains active when the audible presentation is acknowledged or muted.

Power envelope for a 1 W configured speaker limit:

```text
Pout_limit       = 1.00 W                         [DESIGN_REQUIREMENT]
eta_design       = 0.75                           [ASSUMPTION, below cited 84% point]
Iamp_output      = 1.00 W / (5 V x 0.75) = 267 mA [CALCULATED]
Iamp_plus_Iq     = 267 + 2.8 = 270 mA             [CALCULATED]
Isounder_design  = 300 mA                         [DESIGN_REQUIREMENT]
```

AUX5 off provides complete PARKED/SLEEP shutdown. Mute while active is a zero-amplitude waveform; the schematic may use amplifier shutdown if a safe control becomes available without reopening GPIO, but it is not required for parked current. A class-D output is BTL: neither speaker terminal may be grounded, and no flyback diode is used across the speaker. Output filtering/EMI follows the amplifier data sheet and actual speaker lead geometry.

No maximum-SPL claim is made before enclosure testing. Acceptance requires measured A-weighted level and spectrum at the driver's ear in STREET and representative track-cabin noise, alarm recognition with helmet/hearing protection use cases, distortion/thermal testing, and a configurable safe upper limit. Enclosure opening/back-volume geometry remains a mechanical blocker.

## Alarm presentation

Alarm rules, alarm state and presentation remain independent:

```text
normalized channel -> alarm rule -> alarm state/latch/acknowledge
                                      |-> sounder
                                      |-> display
                                      |-> shift light
                                      +-> future app
```

Thresholds, hysteresis, delays, missing-data behavior, priority and latching are configuration/profile data, never global constants in an output driver. Acknowledgement may silence the current audible presentation but cannot convert a critical alarm to a healthy state.

## MODE, RESET and BOOT

GPIO10 remains the MODE input: normally-open switch to ground, 47 kohm pull-up to MAIN_3V3, firmware debounce, and no strap conflict.

Frozen conceptual behavior:

- short press after debounce and before 1 s: acknowledge/silence the highest-priority active alarm presentation; when no alarm is active, advance the local display page or emit a headless status indication;
- long press >=3 s: request Configuration Mode only when policy considers the device stationary/safe; reject visibly during update, shutdown, or unsafe vehicle state;
- holding MODE during reset may request a non-destructive recovery/configuration service mode, but must not erase configuration or enable CAN transmission by itself;
- factory reset requires an authenticated UI/service confirmation, not an accidental long press.

RESET/EN and BOOT/GPIO0 remain distinct PCB/service controls. The enclosure must expose them through labelled recessed holes or removable service access for development and recovery, while preventing routine accidental operation. BOOT remains a strap and has no normal user function.

## Single RGB status indicator

One user-visible RGB indicator communicates state; no always-on power LED is added. Exact color/flash policy remains firmware policy, with patterns available for boot, vehicle/CAN detection, GNSS search/fix, BLE connection, configuration, update and fault. Priority must prevent a benign connection indication from hiding an update/fault state. It is off in PARKED/SLEEP except a bounded, explicitly requested service diagnostic.

**APPROVED:** fit TI `LP5814DRLR`, a catalog I2C four-channel constant-current RGBW driver, on MAIN_3V3. TI lists 2.5-5.5 V, 0.1-51 mA/channel, 8-bit dot-current and PWM control, autonomous patterns, 0.1 uA typical/0.3 uA maximum shutdown, -40 to +125 C, and an 8-pin SOT-5X3 package. Use three sinks with one common-anode RGB LED; exact LED and optical current remain provisional. Shared I2C adds no direct GPIO. Reassign `EXP_P6` from `STATUS_LED_N` to `STATUS_DRV_EN` with a hardware default-off pull. This is an expander-function change, not an ESP32 allocation change.

## Central Configuration Manager

One Configuration Manager owns the canonical settings tree. Modules receive immutable validated snapshots and report capabilities/status; they do not maintain independent persistent user settings.

A stored configuration contains a schema version, monotonic revision, hardware/firmware compatibility, active profile reference, values, provenance, integrity metadata and migration history. Writes follow:

```text
read current -> edit candidate -> schema/range/cross-field validation
-> safety-policy validation -> stage -> atomic commit
-> publish new snapshot -> retain last-known-good
```

Power loss at any step yields either the old or fully validated new revision. Failed migration or corrupt storage falls back to last-known-good, then factory-safe defaults with Generic OBD and no diagnostic transmission. Export/import redacts secrets. Factory reset is explicit, authenticated where a writable interface exists, and preserves the recoverable factory image.

Consumers include vehicle profile/CAN scheduler, GNSS, RaceChrono, display, shift light, alarms/sounder, logger, power manager, web UI and future app. The future app and web UI use the same first-party Device Protocol operations; neither owns a parallel model.

## Web Configuration Mode

Normal mode keeps Wi-Fi disabled; BLE/RaceChrono and local acquisition/outputs are unaffected. A safe MODE long press starts a time-limited SoftAP configuration session.

Frozen architecture:

1. Require physical-presence entry and safe vehicle/update state.
2. Start a uniquely named SoftAP with per-device or per-session high-entropy credentials; WPA2/WPA3 and protected management frames are enabled where supported.
3. Use an explicit local address and optional mDNS. Do not use a captive portal in v1: interception behavior, OS heuristics and attack surface outweigh convenience.
4. Require application authentication/session nonce and CSRF protection in addition to Wi-Fi association. Do not expose unauthenticated writes.
5. Validate a complete candidate through the Configuration Manager; risky CAN/diagnostic changes require explicit confirmation and remain inactive until safe activation.
6. Expire after 10 minutes of inactivity, on explicit exit, vehicle motion/unsafe state, shutdown or update completion; then erase ephemeral session secrets and disable Wi-Fi.

The exact HTTP/TLS certificate and ownership-provisioning model remains a threat-model blocker. A private SoftAP alone is not treated as authorization. During vehicle-powered Configuration Mode, available CAN/GNSS acquisition and critical local alarms continue with bounded resources; large transfers and display/network work cannot starve them. In USB-only mode, CAN and AUX5-dependent presentation remain physically unavailable, and optional 3.3 V loads stay within the configured USB budget. Wi-Fi/BLE coexistence, GNSS RF desense, AUX rail load and enclosure temperature require measurement. Normal BLE/RaceChrono operation must recover automatically when configuration mode exits.

The future UI scope is vehicle/profile and CAN mode, safe polling policy, display type/layout/brightness, complete shift-light settings, alarm thresholds/volume/mute, user-safe GNSS settings/diagnostics, RaceChrono status, logging/storage, device versions/diagnostics/reboot/factory reset, and updates. Raw CAN transmission, arbitrary register writes and unbounded engineering values are excluded outside a separately gated service build/mode.

## User-I/O validation and expert diagnostics

Shift-light, sounder, display, BLE and configuration-mode Wi-Fi participate in controlled A/B interference tests and the mandatory `FULL_LOAD_INTERFERENCE_TEST` defined in [system-validation-plan.md](system-validation-plan.md). Test cases include maximum permitted shift brightness and rapid transitions, low/STREET/TRACK sound profiles and tones, active display updates, BLE streaming and Wi-Fi coexistence where meaningful. Results compare power-rail, CAN, GNSS, radio, storage and runtime-health evidence against a stable baseline.

Useful health may be available through serial/debug, an expert/service Web UI page, the first-party Device Protocol, a future app and/or SD diagnostic logs. Normal user dashboards show actionable state, not unexplained CAN counters, memory minima or engineering-only RF statistics.

## OTA firmware

Use ESP-IDF `esp_https_ota` with server certificate verification and the bootloader OTA data mechanism. Reserve two application slots plus a factory/recovery path sized from measured firmware. A candidate has signed manifest/image, hardware revision and minimum bootloader/schema compatibility, semantic version/build identity, security version, size and digest.

Update is user-initiated, prohibited in unsafe power/vehicle state, written only to the inactive slot, verified before boot selection, and interruption safe. First boot is `PENDING_VERIFY`; only a bounded self-test covering config mount/migration, watchdog, CAN-safe default and core health marks it valid. Reset/crash before confirmation rolls back. Progress/error is shown by the status LED and first-party API. Secure Boot v2, flash encryption and eFuse anti-rollback are production security decisions; do not burn irreversible anti-rollback state until the signing, recovery and manufacturing-key process is validated.

## Independently updateable vehicle profiles

Independent profile update is allowed only when a profile can remain declarative and its schema/runtime compatibility is provable. A package is staged separately from active data and carries ID, profile/schema version, supported hardware/firmware range, provenance/license, integrity digest, signature/key ID and fixture/test identity. Activation is transactional with semantic validation, resource/bus-budget checks, optional dry run, last-known-good rollback and an immutable built-in Generic OBD fallback. A profile cannot ship executable native code or bypass the diagnostic scheduler. The final package/serialization format is intentionally not frozen.

## Future AutoTelemetry Tire Module

The optional Tire Module is a separate future product. It may acquire four-wheel pressure, TPMS internal temperature, multi-point surface/tread temperature and sensor health, then send measurements to the main unit. The main unit timestamps, validates and normalizes them before any RaceChrono, logger, display or app adapter.

Future validation covers missing module/node, stale pressure, stale temperature, invalid sensor, RF/link loss, module reboot, main-unit reboot, data recovery and RaceChrono forwarding. Tire failure degrades only tire channels and cannot stop the main unit's CAN/GNSS acquisition, RaceChrono telemetry, logger, display or alarms.

Reserve canonical concepts such as `TIRE_PRESSURE_FL/FR/RL/RR` in Pa, `TIRE_TEMPERATURE_FL/FR/RL/RR` in degC, and indexed tread temperatures with explicit left-to-right orientation and sensor position metadata. Exact registry IDs remain subject to the channel-registry freeze.

RaceChrono's established custom-CAN behavior supports channels decoded in a Vehicle Profile. RaceChrono maintainer guidance specifies `Tyre temperature <position>` for center/overall temperature and indexed postfixes 1 through 8 from left edge to right edge, with positions FL/FR/RL/RR/Front/Rear; exact naming matters for the tire overlay. Pressure can be recorded as custom OBD/CAN data, but a public authoritative canonical `Tyre pressure` naming/postfix contract was not found. That UI/channel mapping must be reverified against the target RaceChrono release before claiming overlay compatibility.

RaceChrono need not talk to tire sensors. AutoTelemetry can expose normalized tire values through its existing RaceChrono DIY CAN/BLE output, with a RaceChrono vehicle profile mapping frames/equations to the established tire channels. RaceChrono remains an output adapter and never owns Tire Module pairing/calibration/configuration.

### Future link evaluation

| Link | Benefit | Cost/risk | Current reservation |
|---|---|---|---|
| BLE / ESP-NOW class wireless | No new connector; existing ESP32-S3 radio can prototype it | 2.4 GHz coexistence, pairing/security, latency/loss and sensor-node power qualification | Software/protocol capability only |
| UART / RS-485 | Deterministic point-to-point cable and simple framing | Would need connector/transceiver/protection and ground/fault analysis on the main PCB | No v1 hardware |
| CAN | Robust multi-drop and automotive tooling | A truly independent bus needs another controller/transceiver/connector; sharing vehicle CAN is not acceptable by default | No v1 hardware |
| Existing USB/I2C expansion | Useful on a bench | Not a frozen automotive field link; cable/ESD/power limits unsuitable without redesign | No product claim |

**Decision:** no current Telemetry v1 hardware reservation is required solely for the Tire Module. Keep the future transport behind the first-party Device Protocol and evaluate wireless first; if reliability or installation requires a wired independent link, use a separate gateway/module or a later hardware revision.

## Second-CAN decision

ESP-IDF v6.0.2 documents one TWAI controller/driver instance in ESP32-S3 and Classical CAN only. Espressif's ESP32-C6 v1.5 data sheet lists two TWAI controllers, Wi-Fi 6/BLE 5.3/802.15.4, a 160 MHz single high-performance RISC-V core and 512 KB HP SRAM; it does not preserve the frozen S3 N16R8 dual-core/8 MB PSRAM resource envelope.

The local RejsaCAN v6.x C6 self-test is useful evidence that both C6 TWAI instances can be exercised. ESP32-CAN-X2 demonstrates another mature topology: ESP32-S3 native TWAI plus MCP2515 over 10 MHz SPI and IRQ, with a second transceiver. That approach consumes SPI/GPIO/interrupt resources, adds controller latency/buffering and PCB/protection complexity. Espressif also identifies an SPI MCP2518FD as the route when S3 CAN FD/external control is required.

No current requirement needs two independent buses: Telemetry v1 connects to one vehicle OBD CAN bus, and the Tire Module has no frozen CAN transport. Therefore the MCU and one-CAN freeze remain unchanged.

If a later requirement proves two independent CAN buses are necessary, first prefer a separate Tire/sensor gateway to preserve fault and product boundaries. For a future main-unit redesign, compare C6 dual TWAI against S3 plus an automotive-qualified external controller using measured CPU/RAM/PSRAM, radio, SPI, latency, connector, protection, power and software requirements. Do not select solely by controller count.

## Primary sources

- Worldsemi, [WS2812 family](https://world-semi.com/ws2812-family/) and [WS2812B-2020 v1.3 data sheet](https://www.mouser.com/datasheet/2/737/4684_WS2812B_2020_V1_3_EN-1900866.pdf).
- OPSCO, [SK6812MINI-E data sheet](https://www.digikey.com/en/htmldatasheets/production/8367381/0/0/1/sk6812mini-e.html).
- Brightek, [TOP ICLED 2020](https://www.brightek.com/products/visible/id-699.html).
- TI, [CAHCT1G126-Q1](https://www.ti.com/product/SN74AHCT1G126-Q1), [TPS1H100-Q1](https://www.ti.com/product/TPS1H100-Q1), [TPA2005D1-Q1](https://www.ti.com/lit/ds/symlink/tpa2005d1-q1.pdf), and [LP5814](https://www.ti.com/product/LP5814).
- TDK, [piezoelectric buzzer catalogue](https://product.tdk.com/en/system/files?file=dam%2Fdoc%2Fproduct%2Fsw_piezo%2Fsw_piezo%2Fpiezo-buzzer%2Fcatalog%2Fpiezoelectronic_buzzer_ps_en.pdf) and [electromagnetic buzzer catalogue](https://product.tdk.com/system/files/dam/doc/product/sw_piezo/sw_piezo/em-buzzer/catalog/electromagnetic_buzzer_sd_en.pdf).
- PUI Audio, [AS01808AO-SC18-WP-R data sheet](https://puiaudio.com/file/specs-AS01808AO-SC18-WP-R.pdf).
- Belden, [1213A 26 AWG cable data](https://www.belden.com/products/cable/electronic-wire-cable/multi-conductor-cable/1213a).
- Espressif, [ESP32-S3 TWAI v6.0.2](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/twai.html), [ESP32-C6 data sheet v1.5](https://documentation.espressif.com/esp32-c6_dataSheet_en.pdf), [ESP HTTPS OTA](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/esp_https_ota.html), [OTA rollback/anti-rollback](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/ota.html), [Secure Boot v2](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/security/secure-boot-v2.html), and [Wi-Fi security](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/wifi-security.html).
- RaceChrono, [DIY devices tutorial](https://racechrono.com/article/2572), [CAN logging](https://racechrono.com/article/faq/how-do-i-log-can-bus-messages), and [maintainer tire-temperature channel guidance](https://racechrono.com/forum/d/2293-2293).
- Autosport Labs, [ESP32-CAN-X2 documentation](https://wiki.autosportlabs.com/ESP32-CAN-X2).

## Unresolved verification items

- Exercise every user-visible state through vehicle-only, USB-only, both-source, neither-source, brownout and low-voltage-shed transitions; verify truthful status, no unsafe override, bounded USB load and no accidental CAN/AUX5 power.
- Obtain and archive the controlled English WS2812B-2020-V6 data sheet; verify exact revision, lot marking, authorized supply, brightness bins, MSL, local capacitor and real maximum/inrush current.
- Select and acoustically qualify the speaker, back volume/opening, sound spectrum and maximum safe level in the enclosure and representative vehicles.
- Freeze the physical shift connector and connector-side ESD after mechanical/EMC review.
- Threat-model and prototype configuration ownership/authentication/TLS before any writable network interface.
- Reverify RaceChrono pressure and tire-overlay channel naming in the release targeted for integration.
