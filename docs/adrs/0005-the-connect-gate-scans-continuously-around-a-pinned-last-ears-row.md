# 0005. The Connect gate scans continuously around a pinned last-ears row

## Status

Accepted

## Context

ADR 0002 makes Connect a gate that shows whenever no ears are connected. ADR 0001 sets its connection rules: one automatic connect per app open, "Ears not found" with a hint, "Disconnected: tap to reconnect", and the firmware-out-of-date message on the last ears' row. This ADR decides what the gate shows and does in each state.

- Every pair of ears advertises the same name, `ROBO_CAT_EARS`, with the `0xABF0` service UUID and no scan response (`robo-cat-ears/main/ble.c`). CAPABILITY holds no name or firmware version, and its serial is only readable after connecting (`robo-cat-ears/docs/ble-protocol.md`).
- The scan ID is the MAC on Android and a per-phone UUID on iOS ([research](../research/universal-ble-ears-protocol.md) §3).
- `universal_ble` scans until stopped. On Android it silently postpones a scan start beyond 5 in 30 s, and the app isn't told. Both platforms report RSSI repeatedly for the same ears.
- `universal_ble` reports adapter state as a stream. `enableBluetooth()` opens the system dialog on Android and always fails on iOS. `requestPermissions()` throws on denial, and it can't tell "don't ask again" apart from a plain denial. `connect()` doesn't request permissions.
- Every failed or cancelled connect must end with `disconnect()`, or iOS completes it later and grabs the ears ([research](../research/universal-ble-ears-protocol.md) §4).
- The watch scans for 50 s behind a "Scan for Ears" button. It shows name, MAC, raw dBm and 3 signal bars (above −60, above −80, otherwise), sorted strongest first. It has no connect timeout of its own.

Considered and rejected:

- A timed scan with a "Scan again" button, as on the watch: repeated taps hit Android's silent throttle, and the button looks dead.
- Scanning during the automatic connect on open: the connect is competing for the radio, and the list would flicker and then vanish.
- The same "Ears not found" wording for every failed connect: once the link has come up, the ears aren't busy, and the hint would mislead.
- Auto-connecting when not-found last ears reappear: ADR 0001 rejects retrying for as long as the phone is in the foreground.
- Labelling ears by a serial remembered after the first connect: it adds storage just for a label, and renaming ears is out of scope.
- Showing dBm: meaningless to most users.

## Decision

The gate scans continuously, with the last ears pinned in one row whose status line changes.

- **Last-ears row.** Once any ears have connected, the last ears sit in one row pinned on top, whatever their signal. The row shows the ears' label, a status line, and signal bars once the scan sees them. Those ears never appear a second time in the scanned list.
- **Labels.** Ears are "Robo Cat Ears" plus the last 4 hex digits of the scan ID. The same label is used in "Connecting to <ears>…" and in the app bar.
- **Signal.** 3 bars at the watch's thresholds, smoothed over recent readings. Scanned rows sort strongest first and reorder at most once a second.
- **Scanning.** It runs while the gate is on screen and the app is in the foreground, and pauses during a connect. A row drops out after about 10 s without a sighting. There's no refresh button.
- **Automatic connect on open.** There's no scan during it. The last-ears row reads "Connecting to <ears>…" above an empty, dimmed list. Scanning starts if the attempt fails.
- **Connecting.** A tapped row shows "Connecting…" and the other rows are disabled. Tapping the connecting row again cancels. An attempt that hasn't passed the version check within 10 s fails. Every cancel or failure ends with `disconnect()`.
- **Failure notes**, on whichever row was tapped:
  - The link never came up: "Ears not found. Disconnect the watch or other phone and try again."
  - The link came up, then a later step failed or timed out: "Couldn't connect. Try again."
- **Recovery.** A note clears when the scan next sees those ears, and the row reads "Tap to connect". Nothing connects automatically on a sighting.
- **After an explicit Disconnect.** The last-ears row shows a quiet "Disconnected", with no error styling, then "Tap to connect" once scanned.
- **Firmware out of date.** The row keeps a warning until a manual connect passes the check:
  - Ears with an older `protocol_version`, or without ADR 0003's lighting state read: "Ears firmware is out of date. Update the ears to use them with this app."
  - Ears with a newer `protocol_version`: "This app is out of date. Update it to use these ears."
  - Ears that fail the check never become the last ears.
  - There's no update link.
- **Bluetooth off.** A full-gate state, "Bluetooth is off", replaces the list, with the last-ears row greyed above it. Android offers **Turn on**, and iOS says to use Settings or Control Centre. When the adapter returns, the automatic connect runs if it hasn't yet on this open; otherwise scanning starts. If Bluetooth goes off while connected, the app skips ADR 0002's 30 s retry and returns here, with the last-ears row reading "Disconnected: tap to reconnect".
- **Permissions.**
  - The first request only comes after a one-screen explanation with **Continue**. After that, the app requests silently before each scan or connect.
  - Denial shows a full-gate state, "Bluetooth permission needed", with both **Allow** and **Open settings**. On Android below 12 it says "Nearby devices and location".
  - A revoked permission is handled like Bluetooth off.
- **Empty states.**
  - With no last ears: "Looking for ears…" and a hint to switch the ears on and disconnect the watch.
  - With only the last ears in range: a small "Looking for other ears…" line.

The 10 s row expiry and the 10 s connect timeout are tunable constants.

## Consequences

- The list is always current, with no scan button to learn.
- Continuous scanning costs battery while the gate is showing. The gate only shows while nothing is connected and the app is in the foreground.
- Two pairs of ears are told apart by a suffix that differs between phones and doesn't match the watch's MAC display on iOS.
- The connect sequence must report whether the link came up, to choose between "Ears not found" and "Couldn't connect".
- On Android below 12 with location services off, the list stays empty with no explanation, because `universal_ble` can't check location services.
- Firmware updates for the ears have no path from the app. The out-of-date message only names the stale side.
