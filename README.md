# ESPHome-NearbyGlass

An ESPHome port of [sh4d0wm45k/glass-detect](https://github.com/sh4d0wm45k/glass-detect) (the "GlassHole" detector
covered on [Hackaday](https://hackaday.com/2025/12/02/build-your-own-glasshole-detector/)): an ESP32 that scans BLE
advertisements for Meta Ray-Ban smart glasses and reacts when it finds one nearby.

Targets the **Seeed Studio XIAO ESP32C5**, connects to Home Assistant/ESPHome either directly on the LAN or over a
WireGuard tunnel, and logs every detection.

## What it does

- **Detection**: matches BLE advertisement MAC addresses against known Meta/Facebook OUI prefixes
  (`7c:2a:9e`, `cc:66:0a`, `f4:03:43`, `5c:e9:1e`), and falls back to matching "Ray-Ban" or "Meta" in the
  advertised device name. Same logic as the original Arduino sketch.
- **`binary_sensor.meta_glasses_detected`**: exposed to Home Assistant, occupancy class, auto-clears 10s after
  the last matching advertisement.
- **`text_sensor.detection_log`**: publishes a new timestamped record (`epoch mac=... name="..." rssi=... match=...`)
  for every detection. Home Assistant's logbook/history for this entity *is* the persistent detection log — no
  external server required.
- **`text_sensor.last_detected_glasses_mac`**: MAC of the most recent match, for quick automations.
- **LED sign + board LED**: the external sign output flashes 4 times on detection (matching the original sketch),
  plus a 2-flash boot self-test. The XIAO's onboard LED mirrors the same pattern so you get feedback with zero
  extra wiring.
- **Remote connectivity**: a `wireguard` tunnel runs alongside normal Wi-Fi so the ESPHome API/Home Assistant stay
  reachable even when the device is out on a network that isn't the home LAN. `binary_sensor.wireguard_connected`
  reports tunnel state; the device never blocks boot waiting on the tunnel.
- **Joining a foreign Wi-Fi network**: two ways in, both already built into ESPHome:
  - **Hotspot/AP mode**: if it can't join the configured SSID, the device opens a "Glasshole Fallback Hotspot" AP
    with a captive portal for entering new credentials.
  - **USB serial provisioning**: `improv_serial` lets you push new Wi-Fi credentials over the same USB connection
    used to flash it (e.g. via the ESPHome Web/Improv flow, or `improv-cli`), which is the natural path since the
    device is USB-attached to a host.

## Hardware

**Seeed Studio XIAO ESP32C5** (`seeed_xiao_esp32c5` board, ESP-IDF framework — ESP32-C5 doesn't support Arduino
in ESPHome). Verified against real hardware:
- Native USB-Serial/JTAG, enumerates as `/dev/ttyACM0` (Espressif USB JTAG/serial debug unit, `303a:1001`) — no
  separate USB-UART chip or driver needed.
- 8MB flash.
- External "GLASSHOLE" LED sign: wire your driver to **GPIO1** (the `D0` header pin).
- Onboard user LED: **GPIO27**.

Any other ESP32 with BLE should also work if you change `esp32.board` back to something like `esp32dev` — but then
drop the `wireguard`/ESP-IDF-only assumptions and re-check pin numbers for your board.

**Verified on real hardware**: this config was compiled and flashed to a XIAO ESP32C5 over its native
USB-Serial/JTAG port. It boots cleanly, brings up Wi-Fi, and falls back to the `Glasshole Fallback Hotspot` AP
(serving its captive portal at `192.168.4.1`) when the configured SSID isn't reachable. `esptool` prints a
"Crystal frequency mismatch! ... configured for 0MHz" warning on flash — that's a false positive for this chip:
the ESP32-C5's `XTAL_FREQ` Kconfig only offers `XTAL_FREQ_AUTO` (value `0`), which is the correct default (the
chip detects its 40/48MHz crystal itself at boot via efuse bits). There's no `CONFIG_XTAL_FREQ_48` option for
this target, unlike some other ESP32 variants, so don't add one.

## Setup

1. Copy `secrets.yaml.example` to `secrets.yaml` and fill in:
   - Wi-Fi credentials, an OTA password, and a 32-byte base64 API encryption key (generator command is a comment
     in the file).
   - WireGuard tunnel config (device address, device private key, peer endpoint, peer public key). Generate a
     keypair with `wg genkey | tee device.key | wg pubkey > device.pub` and add the device as a peer on your
     WireGuard server.
2. Flash over USB:

   ```
   python3 -m venv .venv && .venv/bin/pip install esphome   # or: uv venv .venv && uv pip install --python .venv/bin/python esphome
   .venv/bin/esphome run glasshole.yaml
   ```

   The device's serial port must be accessible to your user (on Linux, be in the `dialout` group, or
   `sudo chmod 666 /dev/ttyACM0` for a one-off fix).

   On first compile, ESPHome downloads ESP-IDF and creates its own Python virtualenv for it via
   `python -m venv`. If your system Python is missing `ensurepip` (Debian/Ubuntu split it into a separate
   `python3.X-venv` package) and you can't `apt install` it, point `PYTHONEXEPATH` at a small wrapper script
   that runs `uv venv --seed` instead for that one call — `uv` doesn't need `ensurepip`.
3. Once online, the device exposes all sensors/lights to Home Assistant via the native API — no YAML integration
   needed.

## Extending the OUI list

Meta ships new glasses hardware occasionally, which can mean new MAC prefixes. Add them to the `meta_prefixes`
array in the `esp32_ble_tracker.on_ble_advertise` lambda in `glasshole.yaml`.

## Notes on the port

- The original sketch's `BLEScan::setInterval(100)` / `setWindow(80)` are in 0.625ms BLE units (~62.5ms /
  ~50ms), not milliseconds — `scan_parameters` in `glasshole.yaml` uses ESPHome's millisecond units instead, so
  the values were approximated rather than converted 1:1. Detection logic (OUI prefixes + name substring match)
  is otherwise a direct port.
- `wireguard:` requires a `time:` source (used here for both tunnel handshakes and detection-log timestamps),
  which is why `time: platform: sntp` is present even though nothing else strictly needs it.
