# 0007. Servo calibration is live with Revert, and the Ears tab drops the firmware version

## Status

Accepted

## Context

ADR 0002 puts an Ears tab at the right of the bottom navigation bar. It shows the ears' name, firmware version, and serial, plus Disconnect and the entry to a full-page Servo calibration. This ADR decides what the tab and the page show and do.

- The ears send no firmware version. CAPABILITY holds only `protocol_version`, `slot_count`, `max_chunk_bytes`, and the serial. The protocol sends only what a client can't derive (`robo-cat-ears/docs/ble-protocol.md` §8). The serial is all-zero when the ears can't read their eFuse (§8.1).
- Calibration is `[0x03][left_azi][left_lat][right_azi][right_lat]`, four i16 offsets in −1000..+1000 (§2.3). The firmware adds each offset to that servo's angle in degrees and clamps the result to 0–180° (`robo-cat-ears/main/servo.c`).
- Every calibration write saves to the ears' flash and snaps all four servos to their 90° centre (`robo-cat-ears/main/controller.c`).
- Reading `ABF2` returns the ears' calibration ([research](../research/abf2-state-read.md)).
- The watch uses four sliders from −15 to +15, labelled "Left Azi", "Left Lat", "Right Azi", and "Right Lat". It reads calibration when the page opens and sends after 300 ms without a slider change. Its OK sends nothing, and its Cancel resets the sliders without sending, so the ears keep the change. The web app doesn't calibrate.
- Auto-animate can play a built-in at any time and move the ears away from centre.
- Disconnect turns off auto-connect (ADR 0001), and reconnecting is one tap on the gate (ADR 0005).

Considered and rejected:

- Adding a firmware version to the ears: it's only for display, and the protocol sends nothing a client doesn't need.
- Showing `protocol_version`: it means nothing to a user.
- The wire's full ±1000 range: the angle clamps long before it's reached.
- Sending on a 300 ms pause during a drag, as the watch does: the ears snap to centre and write flash mid-drag.
- The watch's OK and Cancel: changes are already on the ears, so OK means nothing, and back would need its own meaning.
- Pausing auto-animate while calibrating: if the app dies or the link drops, the ears are left with auto-animate off.
- Confirming Disconnect: reconnecting is one tap.

## Decision

Calibration changes are live and kept, with Revert as the undo. The Ears tab shows identity without a firmware version.

- **Ears tab.** From top to bottom:
  - the ears' label (ADR 0005) and serial, or "Serial unavailable" when it's all-zero;
  - a "Servo calibration ›" row that pushes the calibration page;
  - **Disconnect**, styled as destructive, with no confirm;
  - the phone app's version, in small text at the foot.
- **Axes.** One row per axis: "Left ear side-to-side", "Left ear up-down", "Right ear side-to-side", "Right ear up-down". Each row has a slider with −/+ buttons.
- **Range.** ±15° in 1° steps, labelled in degrees, such as "+3°". A value read outside ±15 shows its number with the slider pinned at the end until it's moved.
- **Opening the page.** The phone reads calibration from the ears, with the controls disabled and a spinner. Once the read succeeds, the phone re-sends the same values so the ears sit at centre. A hint reads "Adjust each axis until the ear sits straight."
- **Read failed.** "Couldn't read calibration" with **Retry**. The controls stay disabled, so the phone never writes a guess.
- **Sending.** A slider sends when released. −/+ taps send after 300 ms without another tap. Every send carries all four axes.
- **Write failed.** A snackbar reads "Couldn't update calibration", and the phone re-reads the ears' values.
- **Undo.**
  - There's no OK or Cancel. Back keeps the changes.
  - **Revert** sends the values read when the page opened. It's enabled once anything has changed.
  - **Reset to zero** sets all four axes to 0 after a confirm.
- **Auto-animate.** Left as it is. While it's on, the page shows a note: "Auto-animate is on and may move the ears."
- **Reconnecting.** ADR 0002's banner shows and the controls are disabled. When the link returns, the phone re-reads and re-centres as on opening. Revert keeps the values from when the page opened.

The 300 ms tap debounce is a tunable constant.

## Consequences

- The ears move to centre once for each deliberate adjustment, never mid-drag.
- Each opening of the page costs one flash write on the ears, for the re-centre.
- The page always shows what the ears hold, even after the watch changed it.
- The Ears tab departs from ADR 0002 by showing no firmware version. The firmware-out-of-date message on the gate (ADR 0005) is the only firmware signal.
- Values beyond ±15° can be kept but not set from the phone.
- The watch's Cancel still leaves its change on the ears until the watch is fixed.
