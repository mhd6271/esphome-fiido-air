# Fiido Air BLE protocol

Reconstructed upstream from a live C11 Pro and M1 Pro 2025, then re-verified on a Fiido Air.
Kept in step with `components/fiido_bms/fiido_protocol.h`.

The Air speaks the same wire protocol as the C11 and M1. Only the BLE transport differs.
Everything below the transport layer — frame layout, CRC, poll table, register map — is
inherited from upstream and has been confirmed to work on the Air only where explicitly
marked.

## Transport

This is the one place the Air genuinely differs.

Service `C3E6FEA0-E966-1000-8000-BE99C223DF6A`, handle range `0x0E`–`0x13`:

| characteristic | handle | direction |
| --- | --- | --- |
| `C3E6FEA2-…` | 0x12 | notify, bike to host |
| `C3E6FEA1-…` | 0x10 | write, host to bike |

Note the reversal against upstream. On the C11 and M1 the *first* characteristic of the
pair notifies and the second is written to; on the Air it is the other way round. The
handles themselves land in the same places, which makes the swap easy to miss — the CCCD
sits at `0x13`, after the characteristic at `0x12`, and that is what gives the assignment
away.

Writes go out as write-without-response. Negotiated MTU is 247, same as upstream, and no
frame in this protocol comes near it.

### Symptoms of getting the transport wrong

Wrong service UUID: `resolve()` fails outright and the component logs
`FFE1/FFE2 not found, not a Fiido BMS?`.

Right service, swapped characteristics: `resolve()` succeeds, the link reports READY, polls
go out — and nothing ever comes back. The tell is
`esp_ble_gattc_get_descr_by_char_handle error, status=10` in the ESPHome log, meaning there
is no CCCD on the characteristic being subscribed to.

## Frame layout

Identical to upstream. Confirmed on the Air: a hand-written STATS poll is answered with a
frame carrying the same `46 64` signature.

```
poll:    46 64 55 <len> <addr> <crc>
write:   46 64 <type> <payload_len> <addr> <payload...> <crc>
notify:  46 64 AA <payload_len> <addr> <payload...> <crc>
```

- `46 64` is `"Fd"`.
- `<type>` is `AA` for an L0 write and `FF` for a J0 write. A notify carries `AA` in the
  same position, meaning something else.
- `<crc>` is the XOR of every preceding byte.
- A poll's `<len>` is how many bytes the answer will carry, and it has to match what the
  parser expects. `poll_len()` asserts that at compile time.

Writes are fire and forget. The BMS sends no notify for a write; confirmation comes from a
forced STATS poll.

### Which write type

ADDR 0x39 latches only under J0. An L0 write to it is acknowledged and dropped, which reads
as success. `compute_masked_write()` picks the type from the address.

**Untested on the Air.** No write path has been exercised on this bike yet.

## Polls

| name | addr | payload | carries | Air |
| --- | --- | --- | --- | --- |
| STATS | 0x05 | 53 | speed, distance, gear, SOC, status flags (0x05..0x39) | answers |
| METER | 0x60 | 13 | display HW/SW, mode | untested |
| SPEEDLIM | 0x3C | 1 | speed limit value in km/h | no usable answer |
| BOOST | 0x52 | 1 | PAS boost level | untested |
| DISPLAY | 0x57 | 2 | brightness and guard time | untested |
| BATTERY | 0x7B | 13 | capacity, voltage, current, manufacturer | answers |
| MOTOR | 0x96 | 12 | version, wheel, temperature, rated power | answers, temperature always 0 |
| CTRL | 0xAF | 12 | controller HW/SW, voltages, current, temperature | untested |
| ENERGY | 0xC8 | 12 | crank torque and rpm, energy, uptime | untested |

A handshake poll of ADDR 0x0D goes out once per connection.

The Air has no display, so METER and DISPLAY are unlikely to mean anything here even if
they answer.

## STATS payload

Inherited from upstream and unchanged. Offsets are into the payload, which starts at ADDR
0x05, so ADDR `XX` normally sits at `XX - 0x05`. Three fields predate that rule and are kept
as measured: total distance at 23 (4B), trip at 27 (2B), speed at 29 (2B), all in tenths.

| offset | addr | meaning |
| --- | --- | --- |
| 31 | 0x24 | SOC in percent |
| 32 | 0x25 | gear count as a nibble pair, not a plain number |
| 33 | 0x26 | current gear |
| 34 | 0x27 | bit 7 controller, 6 cruise, 5 speed limit, 3 light, 1 start mode, 0 insensitivity |
| 35 | 0x28 | bit 7 speed unit (1 = mph), 6 show total km |
| 37 | 0x2A | bit 5 brake |
| 38 | 0x2B | bit 6 bike guard, 5 double speed, 1 throttle (inverted) |
| 39 | 0x2C | bit 7 PAS limit, 6 slow mode on boot, 4 key sound (inverted) |
| 51 | 0x38 | bits 3:2 speaker (00 = audible) |
| 52 | 0x39 | bit 3 auto screen off, 1 ring; bits 7:5 are written as zero |

On the Air, SOC, speed, trip distance and total distance decode correctly, so the payload
layout holds at least that far. The status bits have not been individually verified; PAS
limit at 0x2C bit 7 reads constant zero.

The checksum is a plain XOR, so a corrupted frame can still validate. Distance, speed and
SOC are therefore range-checked before they reach an entity, and a sample outside its bound
is dropped rather than published.

## Speed limit

Upstream procedure, two writes:

1. ADDR 0x2C bit 7, the PAS cap, if it differs from the target.
2. after 50 ms, ADDR 0x3C with the value and ADDR 0x27 bit 5 with the flag.

Clearing the PAS bit makes the BMS write 0x3C = 25 by itself, so phase two has to land after
that. Setting the cap needs no delay.

Readable combinations are value 100 with the flag off, or 6 and 25 with the flag on.
Anything else is the resting state the BMS re-arms after a ride and is ignored rather than
published.

**On the Air this does not appear to apply.** The SPEEDLIM poll never yields a value the
parser accepts, and the select entity stays `unknown`. The PAS limit bit reads constant
zero. The working assumption is that the Air's 25 km/h cut-off lives in the MIVICE
controller firmware with no register exposed over BLE — but that is an assumption, not a
measurement. Nobody has tried writing to 0x3C on this bike.

## Behaviour the BMS imposes

Upstream observations, carried over. None of these have been re-tested on the Air.

- The controller has to be on before a gear or gear-count write is accepted.
- Bit 3 of ADDR 0x27, the light, survives an off/on cycle, so the bike would come back on
  with the lamp lit. The component clears it on the falling edge of the controller.
- A read-back that latches does not prove the function works. Several bits persist on
  hardware with no support for them. ADDR 0x2D..0x34 carry the capability bits that say
  which functions the bike declares support for.

That last point deserves emphasis on the Air, which is a single-speed bike with a torque
sensor, no display and no throttle. Gear, gear count, throttle, speed unit and horn all have
entities, and several of them may well accept writes without anything happening. Reading the
capability bits at 0x2D..0x34 would be the principled way to settle which functions this
bike actually declares — that has not been done yet.

## Open questions on the Air

- Motor temperature at the MOTOR poll reads a constant 0 °C. Either the MIVICE drive has no
  temperature sensor, or the value sits at a different offset on this bike. A raw MOTOR frame
  captured after a ride would settle it: a real sensor moves.
- The pack is 36 V, not 48 V as on the C11 and M1, and the voltage decodes correctly anyway.
  That suggests the BATTERY block carries no model-dependent scaling.
- No write has been attempted at all. Everything in the write path is inherited on faith.
- The capability bits at 0x2D..0x34 have not been read out.

Raw frames from another Air would be welcome; so would a correction to any of the above.
