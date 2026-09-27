# 0009. Profiles are managed from a bar on Glow, and edits stay linked to their profile

## Status

Accepted

## Context

ADR 0003 saves lighting as named profiles on the phone. A profile holds colours, mode, speed, and brightness. Glow edits are written to the ears live, picking a profile applies it, and the working lighting is unsaved when no profile holds it. On connect, the ears' lighting is shown as a profile if its frame equals that profile's dimmed frame. Otherwise the phone asks before replacing unsaved working lighting: **Save as profile**, **Apply to the ears**, or **Discard**. This ADR decides how profiles are created, named, and managed.

- Glow sits behind the Connect gate in the bottom navigation bar, Animate | Glow | Ears (ADR 0002), so profiles are only reachable while connected.
- Glow already holds the mode, speed, brightness, and up to 32 colours.
- A profile is about 100 bytes.
- Different brightnesses can dim to the same frame, so more than one profile can match the ears.
- Tuning a profile means editing it live and then keeping the result.

Considered and rejected:

- A chip row of profiles on Glow: it scrolls sideways past a few profiles and has no room to rename or delete.
- A separate Profiles page or a fourth tab: profiles are part of Glow, and ADR 0002 fixes three tabs.
- Edits saving into the selected profile: one stray drag silently rewrites it.
- Dropping the link to the profile on the first edit: overwriting a profile would take a name dialog.
- An **Update profile** option in the connect prompt: **Apply to the ears** keeps the link, and the bar's Save is one tap after.
- Confirm dialogs for picking, overwriting, and deleting: an Undo snackbar guards them without slowing the main action.
- A cap on the number of profiles, starter profiles, and blocking duplicate lighting: each adds a state nobody needs.

## Decision

A profile bar at the top of Glow shows the working lighting's state and opens a sheet that manages profiles. Edits keep a link to the profile they started from.

- **The bar.** It shows the current profile's name, "<name> · edited", or "Unsaved". Tapping it opens the profile sheet.
- **Edited.** Editing a profile makes the working lighting unsaved but keeps its profile: the bar reads "Sunset · edited" with inline **Revert**, which re-applies the profile, and **Save**, which overwrites it. When the lighting is plain Unsaved, the bar has one **Save**, which opens the Save as new dialog. When the working lighting equals a profile, the bar shows that profile.
- **The sheet.** **Save as new** sits at the top. Each row shows the name, the profile's colours at its brightness, a check on the current or edited profile, a drag handle, and an overflow menu with **Rename** and **Delete**. Tapping a row applies the profile and closes the sheet.
- **Names.** Save as new asks for a name, pre-filled "Profile N" and selected. Names are trimmed, 1 to 24 characters, and unique ignoring case. A clash shows an inline error.
- **Undo, not confirm.** Picking a profile over unsaved lighting, overwriting, and deleting happen at once, with an **Undo** snackbar. Undo restores the previous working lighting and its profile link. Deleting the current profile leaves the ears unchanged and makes the working lighting plain Unsaved.
- **The link persists.** The profile an edit started from is saved with the working lighting, so "Sunset · edited" survives a restart and a reconnect. The link is dropped when that profile is deleted.
- **Connect prompt.** ADR 0003's three options stay. When the working lighting is edited, the prompt names the profile ("Your edits to Sunset aren't saved"). **Apply to the ears** keeps the link.
- **Matching on connect.** Duplicate lighting is allowed. When more than one profile matches the ears, the last-selected one wins if it's among them, otherwise the first in list order.
- **Order and count.** Profiles stay in creation order, new ones at the bottom, and can be dragged to reorder. There is no cap.
- **First run.** The phone starts with no profiles.

## Consequences

- The working lighting stores a link to a profile alongside its lighting, and the phone remembers the last-selected profile.
- Every profile change can be undone from a snackbar, so the Glow tab needs an undo stack at least one step deep.
- The bar and its sheet are fixed ahead of the Glow tab's own layout, which fits below the bar.
- Profiles can't be exported, shared, or synced; ADR 0003 already keeps two phones apart.
