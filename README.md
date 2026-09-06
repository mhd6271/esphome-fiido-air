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
  - source: github://YOURNAME/esphome-fiido-air
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

| Entity | Status |
| --- | --- |
| Battery SOC | works |
| Battery Voltage | works |
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
