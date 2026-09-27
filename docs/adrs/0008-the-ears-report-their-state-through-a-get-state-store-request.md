# 0008. The ears report their state through a GET_STATE store request

## Status

Accepted

## Context

ADR 0003 has the phone read the ears' lighting and auto-animate on connect, through a new firmware state read that old firmware signals additively and that keeps the shipped watch working. This ADR decides that read.

- Every read of `ABF2` returns calibration, because the firmware refreshes calibration before each read ([research](../research/abf2-state-read.md)). The watch reads `ABF2` for calibration and needs byte 0 to be `0x03`. Its lighting and auto-animate reads of `ABF2` already fail.
- On iOS, universal_ble can resolve a pending `ABF2` read with an indication ([research](../research/universal-ble-ears-protocol.md)).
- Store requests (`0x06`) are written to `ABF1` and answered by typed, chunked indications on `ABF2`, matched by a client-chosen `corr`. Sub-opcodes `0x06` and `0x07` are reserved, `0x08` and up are free, and firmware answers any sub-opcode it doesn't know with `UNSUPPORTED_OPCODE` (`robo-cat-ears/docs/ble-protocol.md` §5, §7, §9).
- The protocol doc's growth rule: clients ignore trailing bytes, appended fields are fixed-width, and additive changes don't bump `protocol_version` (§8). The watch disconnects on any `protocol_version` other than 1.
- The watch ignores any indication that isn't a `0x06` frame whose `corr` matches its own pending request.
- The ears save lighting to flash and re-apply it at boot. With nothing saved, they show marquee, speed 50, red. A second, unused fallback elsewhere in the firmware is solid white.
- The ears save auto-animate to flash, but nothing restarts it at boot. ADR 0003's context says the ears re-apply it at power-on; that holds for lighting only.
- The ears hold only dimmed colours; brightness is applied on the controller (ADR 0003).
- The ears accept one controller at a time, so while the phone is connected only the phone changes their lighting or auto-animate.

Considered and rejected:

- Changing what an `ABF2` read returns: it breaks the watch's calibration read, and on iOS a read can be resolved by an indication.
- The unused `ABF3` or `ABF4` characteristics: the protocol doc tells clients not to depend on them.
- A feature-flags byte appended to CAPABILITY: it says only what `UNSUPPORTED_OPCODE` already says.
- A variable-length response ending in the colour list: nothing could ever be appended after it.
- Reporting whether the auto-animate task is running, leaving boot alone: auto-animate would silently turn off at every power cycle.
- Reporting "never set" lighting: the ears are showing the boot default, and the phone shows what they display.
- Re-reading state after connect: nothing but the phone can change it while the phone is connected.

## Decision

The ears answer a new store sub-opcode, `0x08` GET_STATE, with their current auto-animate and lighting.

- **Request.** `[0x06][corr][0x08][0x00][0x01]`, empty payload, written to `ABF1` like CAPABILITY.
- **Response.** A `0x06` indication on `ABF2` with status `OK` and a fixed 103-byte payload, big-endian:

  `[auto_mode_id:i16][frequency:i16][mode:u8][speed:u8][color_count:u8][r,g,b × 32]`

  The auto-animate fields mean what they do in §2.4, and the lighting fields mean what they do in §2.2. Colour slots past `color_count` are zero. Later fields are appended after the 32 colours.
- **Support signal.** The phone sends GET_STATE on every connect, right after CAPABILITY. An `UNSUPPORTED_OPCODE` answer means the ears need a firmware update. `protocol_version` stays 1 and CAPABILITY is unchanged.
- **Part of connecting.** The ears count as connected only once GET_STATE succeeds. A refusal shows the firmware-out-of-date message and a failure or timeout is a failed connect, all on the Connect gate within ADR 0005's 10 s. ADR 0003's connect prompt runs on the result before the controls appear.
- **Auto-animate is restored at boot.** The firmware restarts auto-animate from its saved value at power-on, so the saved value GET_STATE reports is what the ears are doing.
- **Default lighting.** With nothing saved, GET_STATE reports the boot default the LEDs show (marquee, speed 50, red). The unused white fallback is removed.
- **`ABF2` reads stay calibration.** The protocol doc is corrected where it says an `ABF2` read carries lighting.
- **The watch is unchanged.** It never sends GET_STATE and its calibration read still works, so no companion watch update is needed.

## Consequences

- One ears firmware change covers it: GET_STATE, the auto-animate restore at boot, the single default lighting, and the protocol doc updates. It carries the `ears-firmware` label when the map is sliced.
- Old firmware is caught in-band on the first connect, with no version bump the watch would reject.
- The watch could fix its broken lighting and auto-animate reads by adopting GET_STATE, but that's a watch change outside this effort.
- A fixed 103-byte response spends up to 93 bytes on empty colour slots, in exchange for a layout that can still grow.
- Auto-animate now survives a power cycle for every controller, the watch included.
