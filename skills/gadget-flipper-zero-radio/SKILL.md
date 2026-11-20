---
name: gadget-flipper-zero-radio
description: >-
  Control a Flipper Zero running the Muse Radio peripheral app through a LAN bridge
  using Muse Home Link. Use its local HTTP API to transmit IR and sub-GHz commands
  and to learn remote codes; radio delivery is one-way and unconfirmed.
---

# Flipper Zero Radio Peripheral

Use this skill when the home has a Flipper Zero running the Muse Radio peripheral app, connected over BLE to a LAN bridge, and the user asks Muse to control an IR or sub-GHz device (TV, fan, shades, garage door, RF remote) through it.

## Architecture

```
Muse (cloud) → Home Link → bridge (Pi / Flipper One / phone) → BLE → Flipper Zero → IR / sub-GHz → device
```

- The Flipper Zero is the radio only. It runs the Muse Radio app, which exposes a BLE GATT service. It does not run the Muse SDK and does not speak to Muse directly.
- The bridge holds the BLE connection to the Zero and exposes a local HTTP API on the LAN. Muse reaches the bridge through the Home Link tunnel.
- The bridge also stores learned codes by name, so commands can reference a stored code (`tv_power`) instead of raw protocol parameters.

## Identify the Device

- Discover the bridge via mDNS (`_muse-flipper._tcp`) or its configured address. Confirm identity through `GET /status`: it must report the Flipper's name and serial. A BLE MAC prefix or an open port alone is not proof of the Muse Radio bridge.
- Confirm the Flipper is connected (`status.connected == true`) and the Muse Radio app is running before sending any command. A bridge with no Flipper attached cannot transmit.

## Prerequisites

Read and follow the Home Link networking and safety rules in `~/docs/devices/home_link.md`.

- A Flipper Zero owned by the user, running the Muse Radio peripheral app, within BLE range of the bridge and with line of sight to any IR target.
- A bridge on the same LAN as the Home Link, with the Flipper paired and the learned-code store available.
- User approval for the named device and action. Bridge access is not permission to transmit to every device in the house.

## BLE GATT Service (Flipper app contract)

Service UUID: `826499ba-4eba-4f50-a47d-a08d2ef81bdb`

| Characteristic | UUID | Access | Payload (JSON, UTF-8) |
|---|---|---|---|
| IR TX | `c41abca3-bec4-4ac2-80e8-9ab3abc5c58a` | Write | `{"protocol": "NEC", "address": "0x04", "command": "0x10", "repeats": 1}` or `{"code": "tv_power"}` |
| Sub-GHz TX | `9bc6130a-09df-40a3-8f65-fce7ddffe6a9` | Write | `{"frequency_hz": 433920000, "protocol": "Princeton", "payload": "0x...", "repeats": 1}` or `{"code": "garage_open"}` |
| Learn | `d21a4901-e861-471c-8a19-cd073ed0317d` | Write | `{"type": "ir", "name": "tv_power"}` — the Zero captures the next received signal and the bridge stores it under that name |
| Status | `7155c29e-a1b4-4c01-8930-2fb14059af93` | Read / Notify | `{"connected": true, "battery_pct": 78, "last_tx": {"type": "ir", "result": "ok", "at": "..."}}` |

## Workflow

1. Read `GET /status` on the bridge. Match the requested device to a stored code in `GET /codes` (name, type, protocol, frequency where applicable). Do not guess protocol parameters for a device with no stored code; use the learn flow first.
2. For IR, confirm line of sight: the Flipper's IR LED must face the target. IR is one-way and has no feedback from the device.
3. For a stored code, call `POST /ir/transmit` or `POST /subghz/transmit` with `{"code": "<name>"}`. For a raw command, pass the full protocol parameters from the bridge's documented format; do not invent frequencies or protocol names.
4. For a device with no stored code, use the learn flow: call `POST /learn` with the type and name, have the user press the original remote once while the Flipper listens, then read back the learned code and confirm it with the user before storing it as the device's code.
5. Observe the bridge response and the Flipper status. A successful HTTP response means the bridge delivered the command to the Flipper, nothing more.

## Verify the Result

- Inspect the bridge response and status; a successful response cannot prove the target device received the transmission. IR and sub-GHz are one-way radio.
- Ask for physical confirmation when actual state or successful radio delivery matters (did the TV turn on, did the door move). Before retrying, allow the device to settle; repeated radio commands can change the outcome (toggle power twice is off again).
- For stateful devices (TV power, garage door), distinguish the requested action from the achieved state. A toggle command has no known end state without observation.

## Limits

- No transmitting on frequencies or protocols the user does not own a device on. No emergency, aviation, or licensed bands. No replaying rolling codes except codes captured from the user's own remotes for the user's own devices.
- No new device pairing, no Flipper firmware flashing, no NFC payment or credential use, no BadUSB or HID actions. This skill covers IR and sub-GHz transmission and learning only.
- No assuming a code works because a similar device uses the same protocol. Codes are per-device; a learned code belongs to the device it was learned from.
- The Flipper must remain the user's own device, in their home, within BLE range of the bridge. Do not route transmissions through a Flipper that is not present.

## Sources

- [Flipper Zero firmware — IR worker API](https://github.com/flipperdevices/flipperzero-firmware/tree/dev/lib/infrared)
- [Flipper Zero firmware — sub-GHz transmitter API](https://github.com/flipperdevices/flipperzero-firmware/tree/dev/lib/subghz)
- [Flipper Zero firmware — BLE profile](https://github.com/flipperdevices/flipperzero-firmware/tree/dev/lib/ble_profile)
- [Muse Gadgets SDK — skills](https://github.com/facebookincubator/muse-gadget-sdk/tree/main/skills)
