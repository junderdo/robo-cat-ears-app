# 0002. The controls sit behind a Connect gate

## Status

Accepted

## Context

The phone must carry the watch's four capabilities: Connect, Animate, Glow, and Servo calibration. The watch lays them out as four screens in a circular swipe loop: Scan, Animate, Glow, and Settings. Settings holds a single button that opens calibration as a modal. Glow opens its mode and speed and its colour picker as modals because the screen is small. A connect jumps to Animate (`robo-cat-ears-watch/components/brookesia_app_robo_cat_ears/esp_brookesia_app_robo_cat_ears.cpp`).

- With no ears connected there is nothing to control. The controls start from state read off the ears, and ADR 0001 already sends a failed auto-connect to the scan list.
- Animate and Glow are used repeatedly. Disconnect and calibration are occasional and apply to the ears as a whole.
- Glow holds up to 32 colours, so any screen that also holds Glow gets long.
- Disconnecting turns off auto-connect (ADR 0001), so it must stay a deliberate action.

Considered and rejected:

- The watch's swipe loop with Connect as a peer screen: it leaves controls reachable with nothing to control.
- Top tabs for Animate and Glow, with calibration and Disconnect in an overflow menu: this hides Disconnect, the phone's way to hand the ears to the watch.
- One scrolling control screen: Glow's colour list makes it long, and calibration buried in it is easy to nudge.
- A home hub with a card per capability: an extra tap for the capabilities used most.
- Landscape and tablet layouts: the watch has nothing to match, and the only test hardware is one Android phone.

## Decision

Connect is a gate. The controls sit behind it in a bottom navigation bar: **Animate | Glow | Ears**.

- **The gate.** The app shows Connect whenever no ears are connected. During a connect, the ears' row reads "Connecting to <ears>…" until the protocol version check passes. The controls then open, and the stored-animation list shows its own loading state until the store fetch settles.
- **Opening tab.** The controls open on the tab last used, saved per phone. The first connect opens on Animate.
- **Ears tab.** It shows the ears' name, firmware version, and serial, plus Disconnect and the entry to Servo calibration. Calibration is a full page pushed from there.
- **Glow.** The mode and speed sit inline on the Glow screen. The colour picker is a modal bottom sheet.
- **Connection status.** Every tab's app bar shows the ears' name and link state. During the 30 s retry window a "Reconnecting…" banner shows and the controls are disabled.
- **Connection lost for good.** When the retry window runs out, or the protocol version doesn't match, the app returns to the gate with the last ears pinned at the top. Their row reads "Disconnected: tap to reconnect" or shows the firmware-out-of-date message.
- **Android back.** From Glow or Ears, back goes to Animate. From Animate it leaves the app, which starts ADR 0001's grace period. Back never disconnects.
- **Orientation.** Portrait only, with phone layouts only.

## Consequences

- The controls only ever show two link states, connected and reconnecting. Every not-connected state lives on the gate.
- A version mismatch is caught before the controls open, so the app never flashes into them and back out.
- Disconnect is one tab away and never happens as a side effect of navigation.
- The app saves the last-used tab alongside the last ears.
- Each tab's detailed behaviour can be specified on its own.
- Wide screens get the phone layout stretched. A navigation rail can be added later without changing this structure.
