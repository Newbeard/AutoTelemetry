# Enclosure and mounting architecture

Status: architecture only. No CAD geometry, print files, or manufacturing drawings are approved by this document.

## Separation of concerns

The mechanical system has three independent, replaceable layers:

```text
vehicle-specific mount  ->  display enclosure  ->  display electrical contract
                                  |
                                  +----------->  core-device enclosure
```

- The **core-device enclosure** protects the Telemetry v1 PCB, OBD harness entry, USB, microSD, GNSS antenna connection, and service points.
- A **display enclosure** fits a particular display module and implements the generic display cable contract.
- A **vehicle mount** adapts a display enclosure to a particular cabin without changing electronics or application logic.

No vehicle name, display controller, or dashboard geometry belongs in the core-device enclosure source.

## Removable user-I/O accessories

The shift-light is a fourth accessory assembly, not a change to the three enclosure/mount layers. It contains exactly ten small addressable LEDs on a narrow flex/rigid-flex or narrow PCB, local bypass capacitors, black optical wells, a diffusing face, cable strain relief and a removable Dual Lock-class attachment. The bend axis, component keep-outs, minimum bend radius, cable exit and service replacement are controlled parameters. It must not become a structural part of the display or core enclosure.

The core enclosure must reserve an acoustically intentional path for the proposed onboard 8 Ω speaker: protected opening or membrane, defined back volume, no water/debris trap and no direct stress on the speaker. It must also provide a visible RGB status light pipe/window and distinct access for MODE versus recessed RESET/BOOT. Exact geometry awaits selected components and acoustic/ingress tests.

## Parametric source-of-truth rules

- Native editable CAD is the source of truth; STEP is the neutral review/interchange format.
- Repeated dimensions such as board outline, connector keep-outs, wall thickness, fastener sizes, display aperture, cable bend radius, and mount-interface spacing must be named parameters rather than unrelated literals.
- PCB outline, holes, connector envelopes, keep-outs, and component maximum heights must originate in an approved ECAD export or controlled interface drawing.
- Display and vehicle-mount models consume documented interfaces; they must not infer dimensions from photographs.
- Printed prototypes are fit checks, not release evidence. Material, temperature, UV, flammability, vibration, creep, and mounting loads remain qualification inputs.

## Customer adaptation workflow

Inputs may include vehicle make/model/year/trim, desired location, selected display, manufacturer drawings, annotated photographs with reference scales, caliper measurements, 3D scans, and photogrammetry where its accuracy is characterized. Every dimensional input records its origin, method, uncertainty, and applicability; imagery alone is not dimensional proof.

1. Select a released core-device enclosure revision.
2. Select a display module/profile with an approved electrical and mechanical interface.
3. Capture the target vehicle location, restricted zones, sight lines, airbag deployment paths, control clearances, and attachment method.
4. Instantiate the vehicle-mount template using measured and traceable parameters.
5. Check cable routing, strain relief, connector access, GNSS lead routing, removal/service access, and display visibility.
6. Produce a prototype and perform static fit, vibration, temperature, glare, and driver-interference checks.
7. Record the vehicle, trim, model-year range, measurement method, materials, print/manufacturing settings, and validation evidence.
8. Release the mount independently of device electronics and display firmware.

## Controlled outputs

Each released enclosure or mount should provide:

- native parametric CAD and its tool/version;
- STEP for review and integration;
- 3MF or STL only as a derived fabrication artifact;
- a dimensioned interface drawing and revision identifier;
- material, process, inserts/fasteners, orientation, and assembly notes;
- compatibility and validation records.

Derived meshes must never be the sole editable source.

## Mechanical/electrical contracts

| Boundary | Architecture constraint |
|---|---|
| Core PCB | Use approved outline, holes, component heights, RF keep-outs, antenna-end clearance, and test/service access. |
| OBD harness | Provide retention and strain relief; loads must not transfer into solder joints. |
| Connector selection | Electrical voltage/current, signal integrity, ESD, hot-plug, keying and retention requirements take precedence over mechanical convenience. |
| Cable exits | Direction and bend envelope are controlled parameters; exits must not violate connector sweep, strain relief, RF, service, or vehicle-clearance constraints. |
| USB and microSD | Preserve insertion/removal clearance and prevent accidental vehicle-side back-feed assumptions. |
| GNSS | Do not pinch or sharply bend the coax; preserve U.FL mating/service space and RF keep-outs. |
| Display cable | Initial assumption remains <=200 mm and SPI <=20 MHz until signal-integrity validation; provide bend and connector retention space. |
| Shift-light cable | ≤0.5 m, strain relieved, power/ground ≥26 AWG (24 AWG preferred), data ≥28 AWG with adjacent/twisted ground; longer runs require an intelligent/differential module and EMC review. |
| Display | Aperture, active area, bezel, viewing angle, backlight heat, buttons/touch, and connector sweep are profile parameters. |
| Mount interface | Display enclosure owns a stable attachment datum; vehicle mount owns vehicle geometry and breakaway/retention behavior. |
| Mounting points | Location, datum, fastener/insert specification, pull-out/load case, installation access and tolerance are versioned interface data. |
| Venting/sealing | Must follow measured dissipation and ingress requirements; neither is currently specified. |

Electrical limits in [hardware-spec.md](hardware-spec.md) and [interfaces.md](interfaces.md) remain authoritative. Mechanical models may constrain routing, but may not silently change pinout, voltage, current, cable length, or RF requirements.

## Safety constraints

- Do not obstruct the driver's sight line, controls, telltales, vents, or emergency egress.
- Do not locate hardware or cables in an airbag deployment path.
- Avoid sharp edges, loose projectiles, unqualified adhesives, and attachments that damage safety-critical trim.
- Evaluate surface temperature, solar loading, vibration, fatigue, creep, glare, and night-time reflections.
- Vehicle compatibility claims require a documented physical fit check; a shared chassis code is not sufficient evidence.

## Repository ownership

```text
enclosure/
  core_device/                 # product enclosure and PCB interface
  displays/
    gc9a01/                    # one replaceable display enclosure family
  vehicle_mounts/
    bmw_e81/                   # one vehicle-specific adapter family
```

Each directory begins with a scope README. CAD or derived outputs are added only after the relevant interface inputs and toolchain are selected.

Released source geometry should accumulate as a reusable, indexed library of measured vehicle references, mounts, display housings and adapters rather than isolated customer meshes.

## Unresolved inputs

- Telemetry v1 PCB outline, hole pattern, component envelopes, connector locations, and thermal map.
- Enclosure ingress, temperature, UV, flammability, impact, vibration, and material requirements.
- Core/display attachment standard and cable connector family.
- Mount load cases and acceptable vehicle attachment methods.
- Exact GC9A01 module mechanical drawing and target BMW E81 trim/location measurements.
- Exact shift-light pixel/flex construction, diffuser, adhesive system, bend limits and connector.
- Exact speaker, acoustic opening/back volume, ingress treatment, status light pipe and service-button access geometry.
