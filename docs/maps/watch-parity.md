# Phone app parity with the watch

Trello card: [Phone app parity with the watch](https://trello.com/c/QfT0bxox)

## Destination

A parity spec, plus lighting profiles: every capability of the watch app (junderdo/robo-cat-ears-watch), and saved lighting profiles, has decided phone-native behavior and is sliced into vertical implementation cards, with nothing left to decide before building.

## Notes

- Parity means the same capabilities with phone-native UX, not a copy of the watch's screens. Watch behavior that exists only because of its hardware needs no phone equivalent.
- The capabilities in scope:
  - **Connect**: scan with signal strength, connect and disconnect, auto-reconnect to the last ears, the firmware-out-of-date message.
  - **Animate**: the auto-animate toggle, the 8 built-in animations, and playing animations stored on the ears, with their loading, empty, stale, and error states.
  - **Glow**: up to 32 colours to add, reorder, and delete, with a colour picker; 5 modes plus speed; brightness applied on the phone with gamma; colours and brightness saved on the phone.
  - **Servo calibration**: four axes, sent live.
  - **Lighting profiles**: beyond the watch, added by [ADR 0003](../adrs/0003-lighting-lives-in-per-phone-profiles.md).
- Android and iOS from one BLE stack: `universal_ble` ([ADR 0004](../adrs/0004-universal-ble-carries-the-ears-protocol.md)). Test hardware: an Android phone and a Mac; no iPhone, so iOS BLE behavior is specified but verified later.
- The wire contract is `robo-cat-ears/docs/ble-protocol.md`. The ears accept one controller at a time, so the phone and the watch cannot both be connected.
- Watch reference: `robo-cat-ears-watch/components/brookesia_app_robo_cat_ears/` (screens) and `components/services/` (BLE, animation store, lighting). Related ADRs live in `milk-lab-creations/docs/adr/`.
- Pre-1.0: the phone may require current ears firmware, and firmware changes are in scope where a decision needs them. Ears firmware tickets carry the `ears-firmware` label too. The shipped watch must keep working, through a companion watch update if needed.
- Grilling tickets use the `grilling` and `domain-modeling` skills. Every resolved decision gets an ADR in `docs/adrs/`; research lands in `docs/research/`.

## Decisions so far

- [Does reading ABF2 return the ears' current lighting, mode, and calibration state?](https://trello.com/c/HK9ZrdEY): no. Every read returns calibration, so the phone reads calibration from the ears and keeps lighting and auto-animate itself. [Research](../research/abf2-state-read.md)
- [How does flutter_blue_plus handle the ears protocol on Android and iOS?](https://trello.com/c/YfneC3gc): it covers the whole protocol unmodified. Chunk by CAPABILITY's `max_chunk_bytes`, never long writes; identity is the address on Android but a per-phone UUID on iOS, so only the CAPABILITY serial is shared across controllers; a background auto-connect would lock the watch out. [Research](../research/flutter-blue-plus-ears-protocol.md)
- [How do the phone and watch share the ears' single connection?](https://trello.com/c/LIaROpUn): the phone holds the ears only in the foreground (15 s grace on Android, none on iOS), auto-connects once on open, retries 30 s after a drop, shows busy ears as "not found", and an explicit disconnect turns off auto-connect. [ADR 0001](../adrs/0001-phone-holds-the-ears-only-in-the-foreground.md)
- [How are the four capabilities laid out as phone screens?](https://trello.com/c/xlsoi2vr): Connect is a gate, and the controls sit behind it in a bottom navigation bar, Animate | Glow | Ears. Ears holds Disconnect and a full-page servo calibration; every not-connected state lives on the gate; portrait-only phone layouts. [ADR 0002](../adrs/0002-controls-sit-behind-a-connect-gate.md)
- [Are saved glow colours and brightness per phone or per ears?](https://trello.com/c/gGiccVK5): per phone, as named lighting profiles. Glow edits write live; on connect the phone reads the ears' lighting and auto-animate through a new firmware read, and asks before replacing unsaved lighting; ears without the read are out of date. [ADR 0003](../adrs/0003-lighting-lives-in-per-phone-profiles.md)
- [Amend the BLE protocol contract to admit a native phone client](https://trello.com/c/FuZ6FsUa): done. The phone is a supported, play-only client with no wire change; §13 now rules out only the web app on iOS. `robo-cat-ears/docs/ble-protocol.md` and `milk-lab-creations/docs/adr/0003-a-native-phone-app-is-a-supported-client.md`, on each repo's `docs/phone-client-contract` branch
- [Which BLE plugin carries the ears protocol, given flutter_blue_plus's licence?](https://trello.com/c/3gG1mEVn): `universal_ble` (BSD-3). flutter_blue_plus 2.x's licence adds restrictions GPL-3.0 forbids, so it can't ship in this app, for-profit or not. [ADR 0004](../adrs/0004-universal-ble-carries-the-ears-protocol.md)
- [How does universal_ble handle the ears protocol on Android and iOS?](https://trello.com/c/i0TL3bnJ): it carries the whole protocol unmodified with its default global queue, but the app must request the Android MTU, keep every write within `max_chunk_bytes`, subscribe to indications explicitly, write ABF1 with response, and end every failed or timed-out connect with `disconnect()` or iOS grabs the ears later. [Research](../research/universal-ble-ears-protocol.md)

## Not yet specified

- Whether a screen prototype is worth building before the per-screen specs, now that the layout is decided.
- How the Dart protocol layer is tested, possibly with test vectors drawn from the protocol doc, and how the phone waits out iOS's early MTU of 23 before CAPABILITY (poll or re-issue).
- Slicing the decided behavior into implementation cards.

## Out of scope

- Landscape and tablet layouts: the watch has nothing to match and the test hardware is one phone ([ADR 0002](../adrs/0002-controls-sit-behind-a-connect-gate.md)).
- Watch-hardware behavior: idle dim and screen-off power steps, screen brightness, watch battery and PMU readings, wrist flick, the power button.
- The watch's "Ears Sys Info" panel, a placeholder until the ears send power data.
- Web-app-only features: streaming a custom animation (0x05) and a timeline editor.
- Backlog ideas the watch doesn't have: renaming the ears, gyro motion control, voice or music control, syncing with nearby ears.
- Ears firmware changes no decision needs, such as accepting more than one controller, or advertising a busy state so a phone could show it.
- The watch reconnecting after its own Disconnect button: its disconnection callback starts the 5 s reconnect timer on every disconnect. A watch bug, not phone work.
