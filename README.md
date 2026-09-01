# ESPHome-NearbyGlass

An ESPHome port of [sh4d0wm45k/glass-detect](https://github.com/sh4d0wm45k/glass-detect) (the "GlassHole" detector
covered on [Hackaday](https://hackaday.com/2025/12/02/build-your-own-glasshole-detector/)): an ESP32 that scans BLE
advertisements for Meta Ray-Ban smart glasses and reacts when it finds one nearby.

Two hardware targets, two config files, sharing the same detection logic:

- **`glasshole.yaml`** — Seeed Studio XIAO ESP32C5. Wi-Fi (with AP/Improv provisioning) + BLE, connects to Home
  Assistant/ESPHome either directly on the LAN or over a WireGuard tunnel.
- **`glasshole-s3-eth.yaml`** — ESP32-S3 with W5500 Ethernet + PoE, no Wi-Fi at all. Networking is entirely wired,
  which — see [100% BLE scan time](#100-ble-scan-time-ethernet-variant) — is what buys back full BLE scan duty
  cycle, not just a faster chip.

Both log every detection.

## What it does

- **Detection**: layered signals, not raw OUI matching (see [Detection design](#detection-design) below for why).
  Broad ("any Meta device") and high-confidence ("Ray-Ban Meta glasses specifically") signals are tracked
  separately.
- **`binary_sensor.meta_glasses_detected`**: broad/over-inclusive — any Meta signal at all. Exposed to Home
  Assistant, occupancy class, auto-clears 10s after the last matching advertisement. **Will also fire for Quest
  headsets/controllers**, not just glasses.
- **`binary_sensor.glasses_high_confidence`** ("Ray-Ban Meta Glasses Detected (high confidence)"): the
  Luxottica-company-ID + Meta-service-UUID composite signal, or a glasses-specific name match. Shouldn't fire
  for Quest.
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

### XIAO ESP32C5 (`glasshole.yaml`)

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

### ESP32-S3 Ethernet/PoE (`glasshole-s3-eth.yaml`)

A generic ESP32-S3R8 Ethernet/PoE dev board — this one's sold as **UeeKKoo**, but it's the same reference design
sold under several other brand names (Waveshare's `ESP32-S3-ETH` among them): W5500 SPI Ethernet, optional PoE
module, an unused OV2640/OV5640 camera header, and an unused TF card slot. `esp32.board: esp32-s3-devkitc-1`
(a generic S3 board id — nothing here depends on this specific brand's board file), ESP-IDF framework.

- Native USB-Serial/JTAG over USB-C, same as the C5 — confirmed via `esptool`: ESP32-S3 (QFN56), 8MB embedded
  PSRAM, MAC read correctly.
- W5500 Ethernet pins (this reference design's fixed wiring, not configurable in hardware): `clk_pin: GPIO13`,
  `mosi_pin: GPIO11`, `miso_pin: GPIO12`, `cs_pin: GPIO14`, `interrupt_pin: GPIO10`, `reset_pin: GPIO9`. Sourced
  from two independent boards using this same reference design (Waveshare's published pinout and a generic
  "ESP32-S3-POE-ETH" pin table) that agree exactly — not verified against UeeKKoo's own documentation directly,
  since none could be found; if Ethernet doesn't link on your board, check its schematic against these pins
  first.
- External "GLASSHOLE" LED sign: wire your driver to **GPIO4** — a pin the W5500 SPI bus doesn't use. This
  board's camera/TF-card header pins aren't independently confirmed, but since this config never initializes a
  camera or SD card component, any pin not claimed by the `ethernet:` block above is free regardless of what
  else is silkscreened onto it.
- No Wi-Fi at all — see [100% BLE scan time](#100-ble-scan-time-ethernet-variant) for why that's deliberate, not
  just an oversight.

**Verification status**: compiled and flashed to the real board over `/dev/ttyACM0`. `esp32_ble_tracker` confirms
`Scan Interval: 320.0 ms / Scan Window: 320.0 ms / Continuous Scanning: YES` with no coexistence arbiter — it's
already detecting real devices nearby (a Quest 3) at a visibly higher rate than the XIAO ESP32C5's `60ms`-in-`100ms`
window. The W5500 driver initializes cleanly against the pin mapping above with no SPI/comm errors. **Ethernet
link is confirmed working**: with a cable connected and the board powered over PoE, `wireguard:` repeatedly logs
`Remote peer is online` with a fresh handshake timestamp every ~10s — that traffic can only flow over an
established, routed Ethernet link, so both the W5500 pin mapping and the wired network path are good on this
board. (The earlier `Connected: NO` / `Connecting failed; reconnecting` loop during initial bring-up was link-layer,
not a pin-mapping bug — no cable was plugged in yet at that point.)

## Setup

1. Copy `secrets.yaml.example` to `secrets.yaml` and fill in:
   - An OTA password and a 32-byte base64 API encryption key (generator command is a comment in the file) —
     needed by both variants.
   - Wi-Fi AP fallback password (`ap_fallback_password`) — XIAO ESP32C5 only; the S3/Ethernet variant has no
     Wi-Fi and ignores this key.
   - WireGuard tunnel config (device address, device private key, peer endpoint, peer public key) — needed by
     both variants. Generate a keypair with `wg genkey | tee device.key | wg pubkey > device.pub` and add the
     device as a peer on your WireGuard server.
2. Flash over USB (same command either way, just point it at the right file):

   ```
   python3 -m venv .venv && .venv/bin/pip install esphome   # or: uv venv .venv && uv pip install --python .venv/bin/python esphome
   .venv/bin/esphome run glasshole.yaml          # XIAO ESP32C5
   .venv/bin/esphome run glasshole-s3-eth.yaml   # ESP32-S3 Ethernet/PoE
   ```

   The device's serial port must be accessible to your user (on Linux, be in the `dialout` group, or
   `sudo chmod 666 /dev/ttyACM0` for a one-off fix — group membership persists across reboots, the chmod
   doesn't).

   On first compile, ESPHome downloads ESP-IDF and creates its own Python virtualenv for it via
   `python -m venv`. If your system Python is missing `ensurepip` (Debian/Ubuntu split it into a separate
   `python3.X-venv` package) and you can't `apt install` it, point `PYTHONEXEPATH` at a small wrapper script
   that runs `uv venv --seed` instead for that one call — `uv` doesn't need `ensurepip`.
3. Once online, the device exposes all sensors/lights to Home Assistant via the native API — no YAML integration
   needed. On the XIAO ESP32C5, connect to the "Glasshole Fallback Hotspot" AP first to provision real Wi-Fi
   credentials (see below); the S3/Ethernet variant just needs a network cable (PoE or DC power).

### Wi-Fi provisioning (XIAO ESP32C5 only)

`glasshole.yaml` has no static `ssid`/`password` on purpose — the device boots straight into the "Glasshole
Fallback Hotspot" AP every time until provisioned, either via that AP's captive portal or over USB via
`improv_serial`. Whichever you use, ESPHome saves the submitted credentials to flash and reconnects to that
network automatically on every later boot; the AP only reappears if that saved network stops being reachable.

## 100% BLE scan time (Ethernet variant)

`glasshole-s3-eth.yaml` has no `wifi:` component at all — Ethernet handles all networking. That's not just
"one less thing to configure," it's the actual mechanism for getting continuous BLE scanning:

- ESPHome's `esp32_ble_tracker` only compiles in the Wi-Fi/BLE radio-coexistence arbiter when a `wifi:`
  component is present (`software_coexistence`, set "iff wifi is configured and not disabled by the user" —
  see the comments above `_raise_defaulted_scan_window` in `esphome/components/esp32_ble_tracker/__init__.py`).
  With no `wifi:` block, that arbiter isn't compiled in at all — BLE has the 2.4GHz radio to itself, with no
  time-sharing negotiation with anything.
- That alone doesn't guarantee continuous scanning, though — `esp32_ble_tracker`'s own `scan_parameters` still
  has an `interval` (how often a scan window starts) and a `window` (how long it listens), and a window shorter
  than the interval leaves an idle gap between them regardless of radio contention. Setting `window: 320ms`
  equal to `interval: 320ms` closes that gap too. Combined, that's genuinely continuous (~100%-duty-cycle)
  active scanning — the XIAO ESP32C5 variant's `window: 60ms` inside a `100ms` interval is nowhere close.

If you add a `wifi:` component to the Ethernet variant for any reason (e.g. as a secondary network path),
you'll get the coexistence arbiter back and lose this — Wi-Fi and BLE would go back to sharing the radio.

## Detection design

Ray-Ban Meta glasses advertise over BLE using a **random address** (resolvable/static-random, per the Bluetooth
Core spec) that rotates — not a fixed factory MAC drawn from a stable IEEE OUI block. That makes raw MAC-prefix
matching fundamentally unreliable for this hardware, which is also why the original `glass-detect` sketch's 4
hardcoded prefixes (`7c:2a:9e`, `cc:66:0a`, `f4:03:43`, `5c:e9:1e`) don't hold up: looked up against the current
IEEE OUI registry, `cc:66:0a` and `5c:e9:1e` belong to **Apple, Inc.**, `f4:03:43` to **Hewlett Packard
Enterprise**, and `7c:2a:9e` isn't assigned to anyone. Their top bits also match the BLE spec's
random-address-type encoding, not real vendor-assigned OUI bits — strongly suggesting the original author
captured real glasses traffic but mislabeled random-address bytes as vendor MACs. Every other actively
maintained smart-glasses BLE detector project independently reached the same conclusion and abandoned
OUI-based matching:
[colonelpanichacks/oui-spy-unified-blue](https://github.com/colonelpanichacks/oui-spy-unified-blue),
[NullPxl/banrays](https://github.com/NullPxl/banrays),
[yjeanrenaud/yj_nearbyglasses](https://github.com/yjeanrenaud/yj_nearbyglasses).

So this config matches on BLE advertisement content instead, layered weakest to strongest (all tagged in
`detection_log`'s `match=` field so you can tell which fired):

1. `legacy_oui` — the 4 original prefixes, kept only as a low-confidence legacy fallback per the above.
2. `meta_company_id` — BLE manufacturer-data Company ID `0x01AB` (Meta Platforms) or `0x058E` (Meta Platforms
   Technologies). Real signal, but shared across **all** Meta hardware including Quest headsets/controllers.
3. `meta_service_uuid` — Meta's assigned 16-bit service UUID `0xFD5F`. Also shared across Meta products.
4. `glasses_luxottica` — Luxottica's Company ID `0x0D53` (the frame manufacturer) *together with* the Meta
   service UUID. High confidence, glasses-specific — this is the composite every independently-maintained
   project above converged on.
5. `glasses_name` — "Ray-Ban" / "Wayfarer" / "Oakley Meta" in the advertised name. Deliberately excludes a bare
   "Meta" match so a Quest advertising a "Meta Quest ..." name doesn't count as high-confidence glasses.

`binary_sensor.meta_glasses_detected` fires on *any* of the above (favoring over-alerting); only signals 4-5
also set `binary_sensor.glasses_high_confidence`. If you want the broad sensor to stop counting Quest/other
Meta hardware, drop the `meta_company_id`/`meta_service_uuid`/`legacy_oui` branches from the `if` in the
`on_ble_advertise` lambda in `glasshole.yaml` and key everything off `high_confidence_glasses`.

Meta ships new glasses hardware occasionally, which could mean new company IDs or service UUIDs — extend the
checks in the same lambda if one turns up.

**Live-verified against a real Meta Quest 3**: powering on a Quest 3 near the board produced a steady stream of
`match=meta_company_id` detections (advertised name `"Quest 3"`, RSSI -35 to -48 at close range), correctly
tripping `binary_sensor.meta_glasses_detected` while `binary_sensor.glasses_high_confidence` stayed off the
entire time — confirming the Luxottica/name discriminator works as intended. This also answers a question the
BLE spec left open going in: a Quest 3 broadcasts a general discoverable BLE presence just from being powered
on, not only during controller pairing — its advertised MAC's top bits (`0111...`) confirm it's a resolvable
private address, i.e. the same address-randomization behavior that makes OUI matching unreliable for Meta
hardware generally, not just the glasses.

## Notes on the port

- The original sketch's `BLEScan::setInterval(100)` / `setWindow(80)` are in 0.625ms BLE units (~62.5ms /
  ~50ms), not milliseconds — `scan_parameters` in `glasshole.yaml` uses ESPHome's millisecond units instead, so
  the values were approximated rather than converted 1:1. Detection logic (OUI prefixes + name substring match)
  is otherwise a direct port.
- `wireguard:` requires a `time:` source (used here for both tunnel handshakes and detection-log timestamps),
  which is why `time: platform: sntp` is present even though nothing else strictly needs it.
