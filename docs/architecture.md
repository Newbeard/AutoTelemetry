# Telemetry v1 architecture

## System boundary

Telemetry v1 is a permanently connectable OBD-II node. RejsaCAN v3.4 is the evidence-bearing reference; the derivative is a new design, not an edit of upstream production files.

The approved pre-schematic direction is the hybrid rail-on block diagram in [`power-wake-review.md`](power-wake-review.md): source-isolated OBD/USB input, always-powered low-IQ MAIN_3V3 for ESP32/CAN, switchable GNSS/SD/display branches, and a normally-off AUX5 external-load rail. Rail capacities and margins are derived in [`power-budget.md`](power-budget.md).

## Hardware partitioning

1. **Vehicle boundary:** OBD power and CAN enter through protection appropriate to the final electrical test profile.
2. **Always-powered wake path:** low-IQ MAIN_3V3 remains on; ESP32-S3 deep sleep, TCAN3404-Q1 standby, gated vehicle sense, timer and MODE provide wake. `CALCULATED`: ≤0.161 mA parked input envelope at 12 V.
3. **Main 3.3 V domain:** ≥2.0 A (`DESIGN_REQUIREMENT`) low-IQ automotive buck supplies ESP32/CAN and the individually switched 3.3 V branches.
4. **Switchable/noisy peripherals:** GNSS, display and SD each have named switches. A ≥2.0 A normally-off AUX5 rail supplies protected 5 V external loads (`DESIGN_REQUIREMENT`).
5. **RF region:** NEO-M9N, antenna bias, U.FL, RF trace, and keep-out form a controlled GNSS sub-system physically separated from switching and high-edge-rate nodes.

## Firmware partitioning

```text
drivers (TWAI, UART GNSS, SPI SD/display, GPIO, USB/BLE)
                 │
protocols (OBD-II, ISO-TP, UDS, GNSS parser)
                 │
normalized telemetry/data core + timestamp service
        ┌────────┼─────────┬──────────┬──────────┐
 vehicle profiles   BLE/RaceChrono  display   logger/alarms
```

- Drivers own hardware and queues; protocol modules do not render UI or know vehicle wiring.
- Vehicle profiles convert raw signals to normalized names, units, validity, and timestamps.
- Display drivers consume a display-neutral view model. A vehicle/display pairing is configuration, not a fork.
- Shift-light and alarm rules consume normalized telemetry locally and do not depend on a BLE connection.
- Logging uses a bounded queue and explicit overflow policy so slow SD writes cannot block CAN reception.
- Power management coordinates peripheral quiesce, file flush, GNSS state, CAN standby, sleep source setup, and the hardware hold signal.

## Timing and bus ownership

- CAN reception must have the highest data-ingest priority and a non-blocking path.
- GNSS UART rate must include headroom for every enabled UBX/NMEA message at 20–25 Hz; configure a suitable baud rate rather than assuming the reference UART default.
- microSD and display may share GPIO39/40/41 SPI signals, but require separate CS lines, drivers that release MISO when deselected, serialized transactions, and verified signal integrity over the external display cable.
- Use a monotonic local clock for all samples, then correlate GNSS time separately.

## Power states

| State | Main buck | ESP32 | CAN receiver | Peripherals | Wake/exit |
|---|---|---|---|---|---|
| Active | On | Running | Active | As needed | Policy |
| Parked/deep sleep | On | Deep sleep | TCAN3404-Q1 standby/WUP | Off | CAN_RX, timer, voltage hint, MODE, USB |
| Hardware off | Off | Off | Off | Off | Loss of both OBD and USB; true cold-CAN wake is not the v1 strategy |
| USB debug | On from reverse-blocked USB path | Running | Powered but passive unless OBD is valid | SD and optionally GNSS; AUX5 off by default | USB removal or valid vehicle policy |

CAN activity cannot cold-start the v3.4 hardware once its common 3.3 V rail is absent. Telemetry v1 avoids that failure mode by keeping MAIN_3V3 alive at low current. A TCAN1043A-Q1-class hardware-off alternative is documented but not recommended for v1.

## Design gates

1. Approve requirements, evidence analysis, pin plan, and current budgets.
2. Obtain current component/module/antenna/display data sheets and define automotive test pulses.
3. Capture a new derivative schematic with review checkpoints for power, CAN, straps/USB, and GNSS RF.
4. Run ERC and independent calculations/review; only then begin PCB placement/routing.
5. Do not generate manufacturing outputs until schematic and layout reviews close.
