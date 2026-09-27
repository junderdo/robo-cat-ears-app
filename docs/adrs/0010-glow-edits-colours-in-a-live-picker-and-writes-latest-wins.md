# 0010. Glow edits colours in a live picker and writes lighting latest-wins

## Status

Accepted

## Context

ADR 0002 puts the mode and speed inline on Glow, with the colour picker in a modal bottom sheet. ADR 0003 writes Glow edits to the ears live. ADR 0009 puts a profile bar at the top of Glow, and Glow's layout sits below it. This ADR decides the rest of the tab.

- **The frame.** A lighting frame is `[0x02][mode][speed][color_count][r,g,b × n]`, up to 32 colours (`robo-cat-ears/docs/ble-protocol.md` §2.2).
- **Modes.** The five modes are Solid, Breathing, Marquee, Chasing, and Rain. Every mode uses all the colours. Speed is 1–100, higher is faster, and it means a different frame delay in each mode. Solid ignores speed.
- **Zero colours.** The ears accept a frame with no colours. In Solid it turns the LEDs off. In the other modes it freezes them on their last frame. One colour works in every mode.
- **Writes are costly.** Every lighting write restarts the LED task (taking up to 1 s) and commits the lighting to flash. Writes share the ears' 10-deep command queue, and a command that finds the queue full is dropped silently.
- **The watch:**
  - shows swatches undimmed;
  - picks colours on a hue ring with a saturation/value triangle;
  - adds colours but can't edit one;
  - reorders by drag, writing on every swap mid-drag;
  - deletes by dragging onto a trash can, down to zero colours;
  - applies brightness 0–100 with gamma 2.2, where 0 sends black;
  - debounces speed and brightness by 300 ms.
- ADR 0002 disables the controls while reconnecting, and every reconnect runs GET_STATE and ADR 0003's connect prompt (ADR 0008).

Considered and rejected:

- A segmented button for the mode: five segments don't fit "Breathing" at portrait width.
- Drag-to-trash deletion: it's hard to find on a phone.
- Allowing zero colours: four modes would freeze the LEDs on a stale frame. Brightness 0 turns the lights off.
- A palette of preset colours, or separate hue, saturation, and value sliders: the ring matches the watch and reaches every colour.
- Writing the picked colour only on Done: it breaks ADR 0003's live edits.
- A fixed debounce, as the watch does: it shows nothing mid-drag.
- A fixed-rate throttle: it can still outrun the ears' queue.
- Writing on every reorder swap: each swap is a flash commit for a colour order nobody chose.
- Hiding the speed slider in Solid: the layout jumps when the mode changes.
- Showing speed as a number: 1–100 means a different timing in each mode.

## Decision

Glow is one scrolling page below the profile bar. Colours are edited in a picker sheet that writes live, and lighting writes go to the ears latest-wins.

- **Layout, top to bottom:**
  - **Colours.** A header, "Colours · 5 of 32", over a wrapping grid of swatches that ends in an **Add** tile.
  - **Mode.** A wrapping row of five choice chips.
  - **Speed.** A slider labelled "Slow … Fast", with no number. In Solid it's disabled, with the caption "Solid doesn't animate".
  - **Brightness.** A 0–100% slider. Its label reads "Off" at 0.
- **Swatches.** They show the undimmed colour. Brightness is applied with gamma 2.2 to the frame sent, as on the watch.
- **Colour list:**
  - Tap a swatch to open the picker sheet on that colour.
  - Long-press and drag to reorder. The new order is written on drop.
  - **Delete** is in the picker sheet and takes effect at once, with an **Undo** snackbar.
  - The list holds 1 to 32 colours. On the last colour, Delete is disabled with the hint "Lighting needs at least one colour". At 32, Add is disabled.
- **Picker sheet:**
  - A hue ring around a saturation/value square, plus a hex field.
  - The colour is written live as you drag.
  - **Done** keeps the colour. **Cancel**, or swiping the sheet away, restores the colour it opened on, or removes a colour being added.
  - **Add** appends a colour straight away, starting from the last colour in the list, and opens the sheet on it.
  - If the link drops, the sheet closes as if Done. The phone keeps the edit, and the reconnect's GET_STATE and connect prompt settle any difference with the ears.
- **Writes, latest-wins.** At most one lighting write is in flight at a time, written with response. When it completes, the newest working lighting goes next, if it changed. Intermediate states are skipped, never queued.
- **A failed write** shows a snackbar, "Couldn't update the ears' lighting". The phone keeps the edit. Every frame carries the whole lighting, so the next write repairs the ears.

## Consequences

- Writes pace themselves at the ears' round trip, so a slider drag never overflows the ears' queue. Each write is still a flash commit, bounded by that pace.
- The ears never see a zero-colour frame from the phone.
- Colours can be edited in place, which the watch can't do.
- A colour delete joins ADR 0009's profile changes on the undo stack.
- The app needs a colour picker widget: a hue ring with a saturation/value square and a hex field.
