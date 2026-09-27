# 0006. The Animate tab plays without a playing state

## Status

Accepted

## Context

ADR 0002 puts Animate first in the bottom navigation bar and gives the stored-animation list its own loading state. ADR 0003 keeps the last auto-animate value sent on the phone. This ADR decides what the tab shows and does.

- Built-in animations are fire-and-forget: `[0x01][id]`, ids 1–8, with no reply and no stop command (`robo-cat-ears/docs/ble-protocol.md` §2.1).
- The ears play commands one after another from a 10-deep queue and drop overflow. A playing animation blocks the queue, and a stored animation can run for up to about 65 s.
- A stored PLAY's OK means accepted, not finished. Nothing reports when playback ends, and the protocol says no client should show a "Playing…" state (§7.5). A timeout means the outcome is unknown, so the client re-reads LIST (§9.2). `SLOT_EMPTY` or `SLOT_OUT_OF_RANGE` means the list is stale.
- LIST gives each stored animation a slot and a name, with no duration. It is read once per connection and can't change while this phone is connected, because the ears take one controller at a time (§1.3).
- Auto-animate is `[0x04][mode_id][frequency]`, a write without response. The watch sends only on (mode 1, 100 an hour) or off. The ears then play built-in 1 or 2 at random.
- The watch's labels for built-ins 1/2 ("Right"/"Left") and 6/7 ("Radar"/"Curious") don't match what the firmware plays.

Considered and rejected:

- A "Playing…" state or a stop button: the ears report neither, and there's no stop command.
- Dropping a stored tap while another request is in flight, as the watch does: the tap is silently lost.
- Pull-to-refresh on the stored list: the list can only change between sessions.
- A frequency control for auto-animate: beyond parity with the watch.
- Copying the watch's built-in labels: four of them name the wrong animation.

## Decision

The tab plays on tap and never shows a playing state.

- **Layout.** One scrolling page:
  - the auto-animate switch row;
  - the 8 built-ins as a 4-across grid of icon-and-label tiles;
  - "On your ears": stored animations as a list of name rows in slot order.
- **Built-in labels**, by id, from what the firmware plays, with left and right on the wearer's side: "Left ear", "Right ear", "Happy", "Sad", "Bounce", "Tilt", "Radar", "Twitch".
- **Tapping.** A tap gives a ripple and a haptic tick and nothing else.
  - Built-in taps are sent as they come and queue on the ears.
  - Stored taps queue on the phone, one store request in flight at a time. Each gets a 5 s timeout counted from when it's sent.
- **Stored list states**, shown in the section:
  - Loading: "Loading the ears' animations…" with a spinner.
  - Empty: "No animations stored on the ears", with a hint that the web app adds them.
  - Failed: "Couldn't read the ears' animations", with **Try again**.
- **Play outcomes.**
  - Stale (`SLOT_EMPTY` or `SLOT_OUT_OF_RANGE`): a snackbar, "That animation is no longer on the ears", and a quiet re-read of the list.
  - Timeout: nothing shown, and a quiet re-read of the list.
  - Any other error status: a snackbar, "Couldn't play that animation".
- **Auto-animate.** An on/off switch: on sends mode 1 at 100 an hour, off sends mode 0.
  - The switch flips on tap, and the value is saved as the last sent (ADR 0003).
  - A failed write flips it back, with a snackbar: "Couldn't change auto-animate".
- **Unchanged from other ADRs.** While reconnecting, ADR 0002's banner shows and the controls are disabled. Firmware out of date is caught at the gate (ADR 0005).

## Consequences

- The tab has no playback state to model: taps are sends, and only failures show.
- Several quick taps can queue up seconds of animation that can't be cancelled.
- The phone's store requests need a queue, not a single in-flight slot.
- A stored PLAY queued on the ears behind a long animation times out and triggers a re-read, which may also wait behind that animation.
- The phone and watch label four built-ins differently until the watch is fixed.
