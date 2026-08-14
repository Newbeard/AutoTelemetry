# Telemetry v1 derivative hardware

Status: **documentation and review only; no schematic or PCB started**.

This directory is reserved for the derivative universal OBD-II telemetry hardware. The original RejsaCAN v3.x EasyEDA source, BOM, Gerbers, placements, and images remain unchanged under `Schematics/RejsaCAN v3.x (ESP32-S3 based board)/` as the reference design.

## Entry criteria for schematic capture

- Review and approve [`docs/requirements.md`](../../docs/requirements.md).
- Resolve the high-risk questions in [`docs/rejsacan-analysis.md`](../../docs/rejsacan-analysis.md).
- Approve [`docs/telemetry-v1-change-list.md`](../../docs/telemetry-v1-change-list.md) and the allocation in [`docs/interfaces.md`](../../docs/interfaces.md).
- Obtain the authoritative documents listed in the analysis.
- Define power/current, transient/EMC/environmental, antenna, display, shift-light, buzzer, connector, and enclosure requirements.

## Intended future structure

Create subdirectories only when the corresponding work is approved, for example:

```text
hardware/telemetry-v1/
  README.md
  schematic/       # future editable derivative source
  pcb/             # future layout source
  libraries/       # project-local reviewed symbols/footprints
```

Do not place generated Gerbers, pick-and-place, manufacturing BOMs, or order artifacts here until schematic and PCB reviews are complete.

