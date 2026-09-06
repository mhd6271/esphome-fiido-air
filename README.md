# ESPHome Fiido Air

ESPHome external component that exposes a **Fiido Air** to Home Assistant over BLE.

This is a fork of [dzikus/esphome-fiido-bms](https://github.com/dzikus/esphome-fiido-bms),
which targets the Fiido C11 Pro and M1 Pro. The Air uses the same wire protocol but a
different BLE service, so upstream refuses to connect to it. This fork changes the three
UUID constants and nothing else.

> **Status: early.** The BLE link, the frame format and battery telemetry are confirmed
> working on a Fiido Air. Most of the remaining entities are inherited from upstream and
> have **not** been verified on this bike. See [Entity status](#entity-status).

---

## What is different from upstream

Upstream looks for the service `00010203-0405-0607-0809-0A0B0C0DFFE0` with `FFE1` (notify)
and `FFE2` (write). The Air exposes a different service, and — importantly — the roles of
the two characteristics are **reversed** relative to the C11 and M1.

| | C11 Pro / M1 Pro | Fiido Air |
| --- | --- | --- |
| Service | `00010203-…-0A0B0C0DFFE0` | `C3E6FEA0-E966-1000-8000-BE99C223DF6A` |
| Notify (Rx) | `…FFE1`, handle `0x12` | `C3E6FEA2-…`, handle `0x12` |
| Write (Tx) | `…FFE2`, handle `0x10` | `C3E6FEA1-…`, handle `0x10` |
| Negotiated MTU | 247 | 247 |
| Service handle range | — | `0x0E`–`0x13` |

Everything else — frame layout, CRC, poll table, register map — appears to be identical.
A hand-crafted STATS poll (`46 64 55 35 05 47`) is answered with a frame carrying the same
`46 64` Fiido signature.

The entire change lives in `components/fiido_bms/fiido_link.cpp`:

```cpp
static const auto SERVICE_UUID     = ESPBTUUID::from_raw("c3e6fea0-e966-1000-8000-be99c223df6a");
static const auto NOTIFY_CHAR_UUID = ESPBTUUID::from_raw("c3e6fea2-e966-1000-8000-be99c223df6a");
static const auto WRITE_CHAR_UUID  = ESPBTUUID::from_raw("c3e6fea1-e966-1000-8000-be99c223df6a");
```

If you point upstream at an Air without this change, you get:

```
[E][fiido_bms]: FFE1/FFE2 not found, not a Fiido BMS?
```

Getting the notify/write assignment backwards produces a subtler failure: `resolve()`
succeeds, polls are sent, but nothing ever comes back, and the log shows
`esp_ble_gattc_get_descr_by_char_handle error, status=10` — there is no CCCD on the
characteristic being subscribed to.

---

## Hardware

Developed against a Fiido Air (250 W MIVICE drive, single speed, no display, fingerprint
unlock plus bundled smartwatch).

The bike's BLE address is not published anywhere; scan for it with nRF Connect or with an
`on_ble_advertise` lambda and substitute your own into `ble_client.mac_address`.

ESP32 side: any board that can run `esp32_ble_tracker` + `ble_client`. Reference setup is a
plain ESP32 dev board on Wi-Fi with the `esp-idf` framework.

Nothing exotic is needed — a generic ESP32 dev board (ESP32-WROOM-32, the kind sold in
multipacks) is enough, and the ones I use are
[here](https://link.amazon/B049fpjEo).

> *Affiliate disclosure: that is an Amazon affiliate link. If you buy through it I earn a
> commission at no extra cost to you. Any equivalent ESP32 board works just as well — this
> project does not depend on that particular one.*

---

## Installing from Home Assistant

If you already run ESPHome, skip to [Minimum config](#minimum-config). This section is the
full path from a stock Home Assistant to a flashed ESP32.

### 1. Install the ESPHome Device Builder add-on

Settings → Add-ons → Add-on Store, search for **ESPHome Device Builder**, install it. The
first install pulls a container image and can take ten minutes or more.

Before starting it, enable *Start on boot*, *Watchdog* and *Show in sidebar*, then hit
**Start**. ESPHome now appears in the Home Assistant sidebar.

If the add-on is not in the store, its repository is not registered: ⋮ menu →
Repositories → add `https://github.com/esphome/home-assistant-addon`.

### 2. Create the device

Open ESPHome Builder from the sidebar → **+ Create device** → give it a name (`ebike`) →
pick **ESP32** as the platform. Let the wizard write your Wi-Fi credentials when it asks;
it stores them in `secrets.yaml` and the generated config references them.

The wizard produces a working skeleton. Click **Edit** on the new device and replace the
contents with the configuration from the next section — but keep the `api:` encryption key
and the `ota:` password that the wizard generated for you.

### 3. Add the external component

Nothing to download or copy. The `external_components:` block in the config points at this
repository, and ESPHome fetches the source itself at compile time:

```yaml
external_components:
  - source: github://mhd6271/esphome-fiido-air
    components: [fiido_bms]
    refresh: always
```

`refresh: always` matters while you are iterating — without it ESPHome caches the fetched
repository for 24 hours and will happily rebuild from a stale copy after you push a change.
Drop it once things are stable.

If you want to edit the component locally instead — quicker for experiments, since there is
no push between attempts — copy `components/fiido_bms/` to `/config/esphome/components/fiido_bms/`
using the File Editor or Studio Code Server add-on, and point at it with:

```yaml
external_components:
  - source:
      type: local
      path: components
    components: [fiido_bms]
```

The path is relative to the YAML file.

### 4. Flash the ESP32

**The first flash has to happen over USB.** OTA only works once ESPHome is already running
on the board.

Connect the ESP32 to the machine your browser is running on, then in ESPHome Builder click
the ⋮ menu next to the device → **Install** → **Plug into this computer**. ESPHome compiles
(several minutes on the first build), then offers **Open USB flasher**, which hands over to
ESPHome Web in a new tab. Pick the serial port and let it write.

Use a Chromium-based browser — Chrome or Edge. Firefox and Safari do not support WebSerial
and cannot flash. Also make sure you have a USB *data* cable; charge-only cables are a
classic time sink. Some boards need a button held to enter download mode.

If the ESP32 is physically plugged into the Home Assistant server rather than your laptop,
choose **Plug into your Home Assistant server** instead and skip the browser step.

Every later change installs wirelessly: same menu, **Install** → **Wirelessly**.

### 5. Adopt in Home Assistant

Once the board boots and joins Wi-Fi, Home Assistant discovers it under Settings → Devices
& Services. Confirm, and paste the API encryption key from your YAML when prompted.

At this point the device exists but has no data yet — you still need the bike's BLE address
in `ble_client.mac_address`. See [Finding your bike's MAC](#finding-your-bikes-mac).

---

## Finding your bike's MAC

The address is not printed anywhere on the bike. Two ways to get it:

**nRF Connect** (free, Android/iOS, no extra hardware): scan with the bike switched on and
look for a device that appears when you wake the bike and vanishes when it sleeps. On iOS
you get an opaque UUID rather than a MAC, so use Android if you can.

**The ESP itself**, which avoids the chicken-and-egg problem of the phone occupying the
bike's only BLE connection slot:

```yaml
esp32_ble_tracker:
  scan_parameters:
    active: true
  on_ble_advertise:
    then:
      - lambda: |-
          ESP_LOGD("ble_scan", "%s | %s | %d",
                   x.address_str().c_str(),
                   x.get_name().c_str(), x.get_rssi());
```

Flash that, watch the logs, wake the bike, and look for the address that shows up. Remove
the block afterwards — it is noisy.

Check that the address is stable across a power cycle of the bike. A first byte whose top
two bits are `01` (for example `41:42:…`) indicates a resolvable private address, which
rotates and cannot be pinned in `ble_client`.

---

## Minimum config

Requires **ESPHome 2025.7.0 or newer** (the component uses the sub-device API).

```yaml
esphome:
  name: ebike
  min_version: 2025.7.0

esp32:
  board: esp32dev
  framework:
    type: esp-idf

external_components:
  - source: github://mhd6271/esphome-fiido-air
    components: [fiido_bms]

esp32_ble_tracker:
  scan_parameters:
    active: true

ble_client:
  - id: ble_air
    mac_address: XX:XX:XX:XX:XX:XX

fiido_bms:
  - id: hub_air
    ble_client_id: ble_air

sensor:
  - platform: fiido_bms
    fiido_bms_id: hub_air

binary_sensor:
  - platform: fiido_bms
    fiido_bms_id: hub_air

select:
  - platform: fiido_bms
    fiido_bms_id: hub_air

switch:
  - platform: fiido_bms
    fiido_bms_id: hub_air

number:
  - platform: fiido_bms
    fiido_bms_id: hub_air

button:
  - platform: fiido_bms
    fiido_bms_id: hub_air
```

### Running alongside a Bluetooth proxy

`bluetooth_proxy` claims all three default connection slots and leaves none for
`ble_client`. ESPHome warns about this at compile time. Either drop the proxy from this
device, or raise the limit:

```yaml
esp32_ble:
  max_connections: 4
```

Four concurrent BLE links on one ESP32 is not a restful arrangement. A dedicated ESP for
the bike is the calmer option.

---

## Entity status

Inherited from upstream, so the full C11/M1 entity set is created. What has actually been
observed on the Air:

<img width="347" height="400" alt="image" src="https://github.com/user-attachments/assets/e2271217-b8e7-471d-8de4-17d3374b67aa" />


| Entity | Status |
| --- | --- |
| Battery SOC | works |
| Battery Voltage | works — note the Air is a 36 V pack, not 48 V like the C11 and M1 |
| Speed | works |
| Total Distance | works |
| Trip Distance | works |
| BLE Connected | works |
| Motor Temperature | reads constant 0 °C — sensor may not exist on this drive |
| Speed Limit (select) | `unknown`, never populated |
| PAS Limit (binary sensor) | constant off |
| everything else | **untested** |

The Air is a single-speed bike with a torque sensor and no display, so several upstream
controls have no physical counterpart here — gear selection, gear count, throttle, speed
unit. Whether the BMS still accepts writes to those registers is unknown; nothing has been
verified, so treat them as unsupported until someone checks.

Unwanted entities are easiest to hide in Home Assistant itself (device page → entity →
disable) rather than by patching the component.

---

## Known constraints

**One BLE central at a time.** While ESPHome holds the link, the Fiido app cannot pair —
and, conversely, the bundled smartwatch or a phone with the app open will keep ESPHome from
connecting. Turn off the `bluetooth` switch on the HA device to release the link.

**The bike must be awake.** With the controller asleep, the BMS does not answer polls. You
may see a BLE connection and no data at all.

**The bike will not sleep while the link is held.** Leave `auto_shutdown` on unless you
have a reason not to; otherwise the bike keeps drawing standby current.

---

## Debugging

Useful logger configuration when something does not come up:

```yaml
logger:
  level: VERBOSE
  logs:
    esp32_ble_tracker: INFO
    wifi: INFO
    api: INFO
    esp32_ble_client: VERBOSE
    ble_client: VERBOSE
    fiido_bms: VERBOSE
```

`level: DEBUG` is not enough — it is the compile-time ceiling, and per-component VERBOSE
lines are never emitted below it.

A healthy start looks roughly like:

```
[I][fiido_bms]: [XX:XX:…] Connection opened
[D][fiido_bms]: [XX:XX:…] FFE2 (write)=0x12  FFE1 (notify)=0x10
[I][fiido_bms]: [XX:XX:…] READY (FFE1 notify enabled)
[V][fiido_bms]: [XX:XX:…] POLL STATS -> 46.64.55.35.05.47 (6)
```

(The `FFE1`/`FFE2` labels in those log lines are upstream strings and were left alone; on
the Air they refer to `FEA1`/`FEA2`.)

---

## Credit and upstream

All of the actual work — the protocol reverse engineering, the register map, the component
architecture — is [dzikus](https://github.com/dzikus)'s. This fork only swaps three
constants.

The right long-term home for Air support is upstream, with the UUIDs made configurable
rather than forked. If you have an Air, adding your findings to the upstream issue tracker
is more useful than starting another fork.

## License

GPL-3.0, same as upstream.
