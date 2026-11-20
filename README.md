# Muse Flipper Zero

Flipper Zero as a Muse radio peripheral: IR and sub-GHz control for dumb
devices, over BLE, bridged to the LAN.

- `skills/gadget-flipper-zero-radio/SKILL.md` — the Muse Gadgets skill: GATT
  service contract (UUIDs), bridge HTTP API, learn flow, and safety rules.
  Drops into the `muse-gadget-sdk` `skills/` directory.
- Architecture: Muse → Home Link → bridge (Pi / Flipper One / phone) → BLE →
  Flipper Zero → IR / sub-GHz → device.
- The Flipper runs the Muse Radio peripheral app (not yet built). It does not
  run the Muse SDK — the SDK's Noise-tunnel transport is a port, not a flash.

Related: the Home Link project lives in `muse-home-link`.
