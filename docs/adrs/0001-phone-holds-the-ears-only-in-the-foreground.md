# 0001. The phone holds the ears only in the foreground

## Status

Accepted

## Context

The ears accept one controller at a time and stop advertising while one is connected (`robo-cat-ears/docs/ble-protocol.md` §1.3). The phone and the watch therefore can't both be connected, and whoever connects first owns the ears until it lets go.

- The watch makes one direct connect attempt to its saved address when its app opens. After any disconnect it retries every 5 s while its panel is on, and it disconnects when the user leaves its app. It skips retries after a protocol version mismatch (`robo-cat-ears-watch/components/brookesia_app_robo_cat_ears/esp_brookesia_app_robo_cat_ears.cpp`).
- A pending `flutter_blue_plus` auto-connect grabs the ears the moment they advertise. In the background that locks the watch out while nobody is looking at the phone ([research](../research/flutter-blue-plus-ears-protocol.md) §4).
- iOS suspends a backgrounded app within seconds unless it has a background mode, so a Dart timer can't be trusted to disconnect it later. We have no iPhone to verify iOS background behavior.
- Busy ears don't advertise, so the phone can't tell ears held by another controller from ears that are off or out of range. Telling them apart would need a firmware change, which is out of scope.

Considered and rejected:

- Holding the ears in the background with `bluetooth-central` and a foreground service: it locks out the watch and can't be verified on iOS.
- Retrying for as long as the phone is in the foreground: the phone would grab the ears the instant the watch releases them.
- A background task on iOS to run a grace period: more platform code for a small gain.

## Decision

The phone is a controller only while it's in the foreground.

- **Backgrounding.** Android disconnects after a 15 s grace period, so a quick app switch keeps the link. iOS disconnects as soon as the app is backgrounded.
- **Auto-connect.** When the app opens or comes back to the foreground, it makes one direct connect attempt to the last ears. On iOS that uses the saved `remoteId`, with a scan as fallback. It never uses `autoConnect: true`. If the attempt fails, the phone shows the scan list with the last ears pinned at the top and a "not found" note.
- **Unexpected drop.** In the foreground the phone retries for 30 s, then shows "Disconnected: tap to reconnect".
- **Busy ears.** The phone has no busy state and no take-over action. When the ears aren't found, it shows "Ears not found" with a hint to disconnect the watch or other phone.
- **Explicit disconnect.** It turns off auto-connect, across launches, until the user connects by hand. The last ears stay remembered and pinned in the scan list. Any manual connect makes those ears the last ears and turns auto-connect back on.
- **Protocol version mismatch.** It ends the retry window and turns off auto-connect to those ears until the user connects by hand. The firmware-out-of-date message stays on screen.

The 15 s grace period and the 30 s retry window are tunable constants.

## Consequences

- The watch can take the ears whenever the phone is in the background, or within 15 s of it going there on Android. Handing the ears to the watch is an explicit disconnect on the phone.
- No background BLE: no `bluetooth-central` mode, no foreground service, and no `restoreState`.
- The lifecycle rule differs by platform (a grace period on Android, none on iOS). On iOS, coming back from a quick app switch costs a reconnect and a re-run of the connect sequence.
- "Ears not found" covers busy, off, and out of range. Users learn about the one-controller limit from the hint, not from a distinct state.
- The phone saves an auto-connect flag with the last ears.
