# 0003. Lighting lives in per-phone profiles, read back from the ears on connect

## Status

Accepted

## Context

- The ears keep the last lighting and auto-animate they were sent, and re-apply them at power-on. No controller can read them back: every read of `ABF2` returns servo calibration ([research](../research/abf2-state-read.md)), and the ears send no notification of their lighting at connect.
- One lighting frame carries the mode, the speed, and every colour (`robo-cat-ears/docs/ble-protocol.md` §2.2), so any lighting write replaces all of it.
- Brightness is applied on the controller with gamma. The ears only ever receive the dimmed colours.
- The watch saves one set of colours and brightness per watch. It doesn't save the mode or speed, and it writes lighting only when the user edits it.
- The CAPABILITY serial is the only ears identity the phone and the watch share.
- The watch requires `protocol_version` to equal 1 exactly and hangs up otherwise.
- The project is pre-1.0. The phone may require current ears firmware, and the watch may be updated alongside it.

Considered and rejected:

- Lighting saved per ears, keyed by the serial: it raises the question of what new ears start with, for a user who has one pair.
- Saving only the colours and brightness, as the watch does: the first colour edit after a restart would reset the ears' mode and speed.
- Pushing the phone's lighting on every connect: it silently undoes whatever the watch set.
- Showing the phone's own lighting without reading the ears: the phone would show lighting the ears aren't displaying.
- Falling back to that for old ears firmware: the project is pre-1.0, so requiring current firmware is simpler.

## Decision

The phone saves lighting as named profiles on the phone, and on connect it reads the ears' lighting back through a new firmware state read.

- **Per phone.** Profiles, the working lighting, and the last auto-animate value sent are stored on the phone, not keyed by ears.
- **Profiles.** A profile holds colours, mode, speed, and brightness. Auto-animate isn't part of a profile.
- **Live edits.** Edits on the Glow tab are written to the ears live, debounced, so while connected the working lighting is what the ears show. Picking a profile applies it to the ears.
- **Unsaved.** The working lighting is unsaved when no profile holds it.
- **On connect.** The phone reads the ears' lighting and auto-animate. The ears' lighting is shown as a saved profile if its frame equals that profile's dimmed frame. Otherwise it's shown as unsaved, with the reported colours at 100% brightness.
- **Connect prompt.** If the working lighting is unsaved and differs from the ears, the phone asks before replacing it:
  - **Save as profile**, then load the ears' lighting.
  - **Apply to the ears**, where it stays unsaved.
  - **Discard**, and load the ears' lighting.
- **Old firmware.** Ears without the state read get the firmware-out-of-date message. They're detected by an additive signal, never by a `protocol_version` bump the watch would reject.

## Consequences

- An ears firmware change is on the critical path: a typed state read for lighting and auto-animate that keeps the watch working, with a companion watch update if one is needed.
- The phone shows what the ears are actually displaying, including changes made by the watch.
- For lighting the phone didn't set, the original brightness can't be recovered, so the colours show dimmed at 100%.
- Two phones never share profiles.
- The watch keeps its own single saved lighting. Profiles are phone-only.
