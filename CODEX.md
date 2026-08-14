# CODEX.md — Universal OBD/CAN + GNSS Telemetry Project Rules

## Project context

This repository is a fork of:

`MagnusThome/RejsaCAN-ESP32`

The upstream project is used as the hardware reference/base for a new universal automotive telemetry device.

The derivative device is intended to combine:

* ESP32-S3
* CAN / OBD-II
* ISO-TP / UDS
* vehicle telemetry
* onboard GNSS
* external active GNSS antenna
* Bluetooth LE output to RaceChrono
* external display support
* external shift-light
* audible alarms
* microSD logging
* USB
* automotive sleep/wake
* support for multiple vehicles and multiple display types

## Core architectural principle

The project must remain modular.

Vehicle-specific logic, display-specific logic, RaceChrono transport, GNSS, CAN, OBD, logging and power management must not be unnecessarily coupled.

The hardware should be usable with multiple vehicle profiles and multiple external display types.

## Upstream protection

Do not destructively modify or delete original upstream files unless explicitly instructed.

Prefer creating derivative files under project-specific directories.

If an upstream schematic, PCB, BOM or source file must be modified, first preserve traceability to the original and clearly document the change.

Do not overwrite the original RejsaCAN design without explicit approval.

## Hardware development workflow

Do not jump directly to PCB routing.

The required order is:

1. inspect upstream design
2. document requirements
3. document architecture
4. analyze RejsaCAN v3.x
5. define KEEP / MODIFY / ADD / REMOVE
6. define GPIO and peripheral allocation
7. design schematic
8. run ERC/review
9. design PCB
10. run DRC/review
11. verify BOM
12. generate manufacturing files
13. perform final manufacturing review

Do not skip stages unless explicitly instructed.

## Current target hardware

The first derivative hardware revision is referred to as:

`Telemetry v1`

Telemetry v1 is expected to retain the useful RejsaCAN ESP32-S3 architecture while adding the missing telemetry-specific functions.

Expected additions include:

### GNSS

* u-blox NEO-M9N directly on the main PCB
* external active GNSS antenna
* U.FL RF connector
* GNSS UART connection to ESP32-S3
* target GNSS update rate of 20–25 Hz
* controlled GNSS power if practical
* RF-aware PCB placement and routing

Do not invent the GNSS RF implementation.

Use official u-blox hardware integration documentation before schematic/layout work.

### Display interface

The display is external and replaceable.

Initial target display:

* GC9A01
* 240×240
* SPI

Future displays may differ.

The main PCB should expose a sufficiently generic display interface rather than being tied permanently to one display model.

### Shift-light

The shift-light is external.

It must work independently of RaceChrono.

Do not assume an ESP32 GPIO can directly power the final LED load.

Evaluate the required driver, level shifting and power architecture.

### Buzzer / alarms

Provide support for audible alarms.

Use an appropriate transistor/MOSFET driver where required.

### Expansion and debugging

Preserve or provide useful:

* UART
* I2C
* GPIO
* test points
* USB
* microSD
* CAN test points
* GNSS UART test points
* power-rail test points

## OBD-II target interface

Primary vehicle connection is expected to use:

* pin 16: +12 V battery
* pin 4/5: ground
* pin 6: CAN-H
* pin 14: CAN-L

The device may remain permanently connected.

Therefore low-power operation and safe automotive power handling are mandatory design concerns.

## Firmware assumptions that affect hardware

The future firmware architecture is expected to contain modules for:

* CAN
* Generic OBD-II
* ISO-TP
* UDS
* vehicle profiles
* GNSS
* normalized telemetry/data core
* RaceChrono BLE
* display drivers
* shift-light
* alarms
* SD logger
* power management

Do not make hardware decisions that unnecessarily prevent this modular architecture.

## Product monorepo architecture

Treat this repository as the complete Telemetry product, not only a board fork. Preserve explicit ownership for automotive hardware, firmware, vehicle profiles, protocol specifications, RaceChrono integration, a future first-party app, displays, shift-light, alarms, logging, configuration, engineering tools, enclosure/mount CAD, tests and manufacturing evidence.

The normalized telemetry data core is the mandatory internal boundary. CAN, OBD-II, ISO-TP/UDS and GNSS publish canonical samples with units, timestamps, source, validity and quality. RaceChrono, device protocols, displays, shift-light, alarms, logger and optional lap processing consume this model and must not decode vehicle frames directly.

Support vehicle integration in explicit levels:

1. Level 1: Generic OBD-II.
2. Level 2: known declarative vehicle profile using passive CAN/DBC-derived definitions.
3. Level 3: explicitly enabled extended manufacturer diagnostics using ISO-TP/UDS.

Vehicle profiles must be data-driven and reviewable. Keep matching, CAN identifiers and frame format, bit rate, bit extraction, byte order, signedness, scale/offset/unit, destination channel, expected rate, validity, diagnostic addressing/request/response, source priority, provenance and metadata out of output adapters. Do not make DBC or OpenDBC a runtime requirement; importing permitted data into the canonical schema is acceptable when license and provenance are recorded.

LISTEN_ONLY and DIAGNOSTIC_POLLING are distinct operating states. LISTEN_ONLY must not transmit. All diagnostic transmission passes through one bounded scheduler with rate/bus budgets, timeouts, cancellation and backoff, including coexistence policy for other scan tools. Do not add arbitrary CAN transmit or safety-critical actuation to a production profile.

RaceChrono is one replaceable output integration, never the internal model or product-control protocol. First-party services must be transport-independent and versioned separately from BLE, Wi-Fi and USB bindings.

Display manager, renderer, controller drivers and layouts remain separate, and operation must be headless-capable. Shift-light works independently of RaceChrono/display. Alarm rules, alarm state and presentation are separate. Logger failure must not block acquisition. Lap processing is optional.

Use bounded queues and explicit backpressure/drop policies between runtime responsibilities. Define timeouts, health state, watchdog ownership and isolated restart/escalation so a failed display, storage device, network client or output cannot starve CAN/GNSS acquisition or cause uncontrolled transmission.

Product configuration must be centralized, schema-versioned, validated, atomically persisted, migratable and recoverable to safe defaults. Avoid incompatible per-module persistence formats.

## Repository ownership and dependency policy

Use these ownership boundaries unless an approved architecture change documents a better one:

* `hardware/telemetry-v1/`: derivative ECAD, BOM evidence and approved manufacturing releases
* `firmware/`: embedded composition and independently testable modules
* `profiles/`: canonical authored vehicle profiles and provenance
* `protocol/`: versioned first-party protocol specifications and compatibility fixtures
* `apps/`: first-party client applications
* `tools/`: host-side DBC, CAN, profile and log engineering tools
* `enclosure/`: independent core-device, display and vehicle-mount mechanical artifacts
* `tests/`: deterministic fixtures, replay, integration and hardware-in-loop assets

Before adding or copying a dependency, example, DBC, CAN capture or CAD asset, record its exact upstream, version/commit, license, notices, intended use and redistribution constraints. Absence of a license is not permission to copy or create a derivative. Prefer wrappers/adapters and pinned upstream dependencies over vendored source. Separate tooling-only dependencies from target firmware. Protect customer vehicle captures and location data; commit only synthetic or redistributable, redacted fixtures with provenance.

Mechanical work has three separately versioned layers: core-device enclosure, display enclosure and vehicle-specific mount. Native parametric CAD is source; meshes are derived. Electrical documents control connector, power, cable and RF limits. Do not infer production dimensions from photos or controller names, and do not make fit/safety claims without documented measurements and validation.

## Source discipline

For engineering conclusions, prefer authoritative sources in this order:

1. actual repository schematic / PCB / BOM / source files
2. component manufacturer datasheets
3. official Espressif documentation
4. official u-blox documentation
5. protocol standards / official technical documentation

Do not treat forum posts, blog posts or assumptions as authoritative when primary documentation is available.

Clearly mark anything uncertain.

Never fabricate:

* pin mappings
* component values
* GPIO availability
* current capability
* voltage limits
* RF requirements
* CAN termination
* protection behavior

## Automotive safety

Automotive power and CAN circuits require extra caution.

Always flag changes involving:

* reverse polarity protection
* load dump/transient protection
* ESD
* CAN transceiver protection
* CAN termination
* common-mode filtering
* DC/DC conversion
* sleep/wake circuitry
* brownout behavior

Do not claim automotive compliance without evidence.

## ESP32 constraints

Always verify:

* boot/strapping pins
* USB pins
* flash/PSRAM pins
* occupied GPIO
* wake-capable GPIO
* UART conflicts
* SPI conflicts
* ADC limitations
* peripheral power limits

Do not allocate GPIO based on memory or assumptions.

Check the actual ESP32-S3 variant and the RejsaCAN schematic.

## RF / GNSS constraints

GNSS layout must be treated as RF-sensitive.

Before routing:

* verify u-blox reference design
* verify antenna bias requirements
* verify required impedance
* verify grounding recommendations
* keep noisy switching circuits away from GNSS RF path
* minimize RF trace length
* document antenna connector choice

Do not route the GNSS RF section until official documentation has been reviewed.

## Manufacturing

Do not generate final manufacturing files until explicit approval.

Before Gerber/BOM/Pick-and-Place generation, verify:

* ERC
* DRC
* footprints
* component orientation
* pin 1 markings
* connector orientation
* BOM availability
* assembly constraints
* RF section
* automotive power section
* CAN section
* test points

## Git workflow

Do not force-push.

Do not rewrite upstream history.

Do not commit unrelated changes together.

Use clear commit messages.

Do not automatically merge upstream changes into derivative hardware without reviewing their effect.

## Required project documentation

Maintain at least:

* `docs/requirements.md`
* `docs/architecture.md`
* `docs/hardware-spec.md`
* `docs/interfaces.md`
* `docs/power.md`
* `docs/rejsacan-analysis.md`
* `docs/telemetry-v1-change-list.md`

Create additional documentation only when it improves traceability.

## Mandatory report for ChatGPT review

After every meaningful engineering task, produce a concise review report for the user to paste back into ChatGPT.

Create or update:

`docs/chatgpt-review.md`

This file must contain the latest review packet.

Use this exact structure:

## Task completed

Describe what was done.

## Files changed

List every file created or modified.

## Engineering findings

List the important technical conclusions.

For hardware findings, include source evidence such as:

* schematic file
* sheet
* component reference
* BOM line/item
* source-code file
* datasheet section

## Decisions made

List any choices made during the task.

## Assumptions

List every assumption.

If there are none, write:

`None.`

## Uncertainties / unresolved questions

List everything that still needs verification.

Do not hide uncertainty.

## Risks

List technical risks discovered.

## GPIO / peripheral changes

Show any GPIO or peripheral allocation changes in a table.

If none changed, state that explicitly.

## Power / CAN / RF impact

Explain whether the task changed or affected:

* automotive power
* CAN
* GNSS/RF
* USB
* ESP32 boot/strapping

## Datasheets / primary sources consulted

List the primary technical sources used.

## Recommended next step

Recommend exactly one next engineering step.

## STOP condition

At the end of every task, stop after completing the requested scope.

Do not automatically continue into schematic editing, PCB layout, manufacturing generation or firmware implementation unless explicitly asked.

# Engineering Research and Calculation Rules

These rules apply to ALL hardware engineering tasks in this repository.

## Source priority

For electrical calculations, component selection, schematic design, PCB design and engineering review, use primary sources in this order:

1. Component manufacturer datasheets.
2. Manufacturer application notes.
3. Manufacturer reference designs.
4. Official Espressif documentation.
5. Official u-blox documentation.
6. Relevant automotive/protocol standards.
7. Other authoritative technical documentation.

Do not use forum posts, blogs, distributor descriptions or search-result snippets as authoritative sources when primary documentation is available.

Every important component selection and numerical design decision must be traceable to a primary source.

## Numerical engineering decisions

Every numerical design decision must show:

* source value;
* source document and section/table/page when available;
* formula;
* calculation;
* engineering margin;
* resulting design requirement.

Every numerical value must be classified where practical as:

* `VERIFIED_DATASHEET`
* `CALCULATED`
* `DESIGN_REQUIREMENT`
* `ASSUMPTION`

Never silently turn an assumption into a design fact.

## Missing information

If an authoritative datasheet or required specification cannot be found:

1. mark the item unresolved;
2. state what information is missing;
3. explain why it matters;
4. do not invent a value;
5. do not finalize the affected component selection.

## Power calculations

For every regulator or power architecture evaluate at minimum:

* VIN minimum;
* VIN nominal;
* VIN maximum;
* expected automotive transient environment;
* VOUT;
* typical load;
* maximum continuous load;
* transient/peak load;
* conversion efficiency;
* regulator power dissipation;
* thermal margin;
* quiescent current;
* shutdown current;
* enable behavior;
* required input/output capacitance;
* switching frequency;
* relevant EMI implications;
* component/package thermal limits.

Do not select a regulator only from its headline maximum-current rating.

## Parked-current budget

For any design intended to remain connected to OBD-II, calculate the COMPLETE parked-current budget.

Include, where applicable:

* input protection leakage;
* regulator quiescent current;
* always-on regulator current;
* wake-controller current;
* CAN-transceiver sleep current;
* ESP32 sleep current;
* voltage-divider current;
* GNSS backup-domain current;
* load-switch leakage;
* indicator LED current;
* ESD/protection leakage;
* other always-powered circuitry.

Do not quote ESP32 deep-sleep current as the device parked current.

## Automotive power protection

For each protection element identify the condition it is intended to address, such as:

* reverse battery;
* over-voltage;
* load dump;
* fast transients;
* cranking/brownout;
* ESD;
* conducted noise.

Verify voltage, current, power and energy ratings from authoritative documentation.

Never claim ISO, CISPR, UNECE or other automotive compliance without appropriate testing and evidence.

## CAN engineering

For CAN hardware evaluate:

* transceiver exact part number;
* qualification grade;
* supply voltage;
* common-mode range;
* standby/sleep current;
* wake capability;
* unpowered-node behavior;
* bus loading;
* ESD capability;
* CAN-line protection;
* optional common-mode filtering;
* termination configuration.

For an OBD-connected node, do not assume that an additional 120 Ω termination resistor is required.

Explicitly justify termination configuration.

Distinguish clearly between:

* passive/listen-only operation;
* active CAN transmission;
* OBD-II diagnostic polling;
* ISO-TP;
* UDS.

## GNSS engineering

For GNSS use official u-blox documentation as the primary source.

Evaluate:

* module supply voltage;
* typical current;
* peak current;
* backup-domain requirements;
* startup behavior;
* cold/warm/hot start implications;
* active-antenna supply;
* antenna bias current;
* antenna protection;
* RF impedance requirements;
* UART electrical levels;
* UART bandwidth;
* intended navigation update rate;
* enabled constellations;
* enabled messages.

For high-rate GNSS, calculate required UART throughput rather than assuming a baud rate is sufficient.

Do not design or route the RF section without consulting the official hardware integration documentation.

## Display interface

For each supported display evaluate separately:

* controller current;
* backlight current;
* supply voltage;
* logic voltage;
* SPI frequency;
* required SPI bandwidth;
* refresh requirements;
* MISO behavior;
* cable length;
* signal-integrity implications.

Do not assume the display controller current represents total display-module current.

## Shift-light

Calculate worst-case shift-light power.

Evaluate:

* LED technology;
* number of LEDs;
* voltage;
* maximum current;
* data signaling;
* logic-level compatibility;
* cable length;
* connector rating;
* output protection;
* EMI implications.

Do not power significant LED loads directly from ESP32 GPIO.

## USB

Always evaluate:

* USB-only power;
* vehicle-only power;
* simultaneous USB + vehicle power;
* back-feed paths;
* ESD;
* CC configuration;
* native ESP32-S3 USB requirements.

USB must not unintentionally energize the vehicle-side power system.

## Component comparison

When selecting an important component, compare reasonable candidates in a table containing, where applicable:

* exact manufacturer part number;
* manufacturer;
* qualification grade;
* operating voltage;
* current capability;
* quiescent current;
* shutdown current;
* temperature range;
* relevant protections/features;
* package;
* lifecycle/status;
* availability;
* advantages;
* disadvantages;
* primary-source reference.

Do not choose a component solely because it is familiar or already used by RejsaCAN.

## Reproducibility

Preserve engineering calculations in repository documentation.

Another competent engineer should be able to reproduce the reasoning from:

source data → formula → calculation → margin → design decision.

## Design reviews

Before progressing between major hardware stages, perform a review.

Required gates:

Requirements
→ architecture
→ component selection
→ power/wake review
→ schematic
→ ERC
→ schematic review
→ PCB placement
→ PCB routing
→ DRC
→ PCB review
→ manufacturing package
→ manufacturing review.

Do not silently skip a gate.

## Uncertainty rule

When uncertain, do not guess.

Document:

* what is known;
* what is unknown;
* why it matters;
* how it can be verified.

A clearly documented unresolved item is preferable to an unsupported engineering decision.
