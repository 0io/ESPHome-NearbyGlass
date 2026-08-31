# ESPHome-NearbyGlass

An ESPHome port of [sh4d0wm45k/glass-detect](https://github.com/sh4d0wm45k/glass-detect) (the "GlassHole" detector
covered on [Hackaday](https://hackaday.com/2025/12/02/build-your-own-glasshole-detector/)): an ESP32 that scans BLE
advertisements for Meta Ray-Ban smart glasses and reacts when it finds one nearby.

Same detection logic as the original Arduino sketch, but as an ESPHome config so it drops straight into Home
Assistant with no custom firmware to maintain:

- **Detection**: matches BLE advertisement MAC addresses against known Meta/Facebook OUI prefixes
  (`7c:2a:9e`, `cc:66:0a`, `f4:03:43`, `5c:e9:1e`), and falls back to matching "Ray-Ban" or "Meta" in the
  advertised device name.
- **`binary_sensor.meta_glasses_detected`**: exposed to Home Assistant, occupancy class, auto-clears 10s after
  the last matching advertisement.
- **`text_sensor.last_detected_glasses_mac`**: the MAC of the most recent match, for logging/automations.
- **LED sign**: drives the GPIO2 output (matches the original PCB's LED pin) through a binary `light`, flashing
  4 times whenever a match is detected, plus a 2-flash pattern on boot as a self-test.

## Hardware

Any ESP32 dev board works out of the box (`esp32dev`). If you're building the original `gd-pcb` sign from the
upstream repo, wire its LED driver to GPIO2, or change the `pin:` under `output:` in `glasshole.yaml` to match
your board.

## Setup

1. Copy `secrets.yaml.example` to `secrets.yaml` and fill in your Wi-Fi credentials, an OTA password, and a
   32-byte base64 API encryption key (a command to generate one is included as a comment in the file).
2. Flash and adopt into ESPHome/Home Assistant as usual:

   ```
   esphome run glasshole.yaml
   ```

3. Once online, the device exposes the binary sensor, text sensor, and the LED sign light entity to Home
   Assistant via the native API — no YAML integration needed.

## Extending the OUI list

Meta ships new glasses hardware occasionally, which can mean new MAC prefixes. Add them to the `meta_prefixes`
array in the `esp32_ble_tracker.on_ble_advertise` lambda in `glasshole.yaml`.

## Notes on the port

The original sketch's `BLEScan::setInterval(100)` / `setWindow(80)` are in 0.625ms BLE units (~62.5ms /
~50ms), not milliseconds — `scan_parameters` in `glasshole.yaml` uses ESPHome's millisecond units instead, so
the values were approximated rather than converted 1:1. Detection logic (OUI prefixes + name substring match)
is otherwise a direct port.
