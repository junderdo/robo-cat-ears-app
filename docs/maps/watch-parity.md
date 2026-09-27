# Phone app parity with the watch

Trello card: [Phone app parity with the watch](https://trello.com/c/QfT0bxox)

## Destination

A parity spec: every capability of the watch app (junderdo/robo-cat-ears-watch) has decided phone-native behavior and is sliced into vertical `mobile-app` implementation cards, with nothing left to decide before building.

## Notes

- Parity means the same capabilities with phone-native UX, not a copy of the watch's screens. Watch behavior that exists only because of its hardware needs no phone equivalent.
- The capabilities in scope:
  - **Connect**: scan with signal strength, connect and disconnect, auto-reconnect to the last ears, the firmware-out-of-date message.
  - **Animate**: the auto-animate toggle, the 8 built-in animations, and playing animations stored on the ears, with their loading, empty, stale, and error states.
  - **Glow**: up to 32 colours to add, reorder, and delete, with a colour picker; 5 modes plus speed; brightness applied on the phone with gamma; colours and brightness saved on the phone.
  - **Servo calibration**: four axes, sent live.
- Android and iOS from one BLE stack; `flutter_blue_plus` fits the protocol, pending its licence. Test hardware: an Android phone and a Mac; no iPhone, so iOS BLE behavior is specified but verified later.
- The wire contract is `robo-cat-ears/docs/ble-protocol.md`. The ears accept one controller at a time, so the phone and the watch cannot both be connected.
- Watch reference: `robo-cat-ears-watch/components/brookesia_app_robo_cat_ears/` (screens) and `components/services/` (BLE, animation store, lighting). Related ADRs live in `milk-lab-creations/docs/adr/`.
- Grilling tickets use the `grilling` and `domain-modeling` skills. Every resolved decision gets an ADR in `docs/adrs/`; research lands in `docs/research/`.

## Decisions so far

- [Does reading ABF2 return the ears' current lighting, mode, and calibration state?](https://trello.com/c/HK9ZrdEY): no. Every read returns calibration, so the phone reads calibration from the ears and keeps lighting and auto-animate itself. [Research](../research/abf2-state-read.md)
- [How does flutter_blue_plus handle the ears protocol on Android and iOS?](https://trello.com/c/YfneC3gc): it covers the whole protocol unmodified. Chunk by CAPABILITY's `max_chunk_bytes`, never long writes; identity is the address on Android but a per-phone UUID on iOS, so only the CAPABILITY serial is shared across controllers; a background auto-connect would lock the watch out. [Research](../research/flutter-blue-plus-ears-protocol.md)
- [How do the phone and watch share the ears' single connection?](https://trello.com/c/LIaROpUn): the phone holds the ears only in the foreground (15 s grace on Android, none on iOS), auto-connects once on open, retries 30 s after a drop, shows busy ears as "not found", and an explicit disconnect turns off auto-connect. [ADR 0001](../adrs/0001-phone-holds-the-ears-only-in-the-foreground.md)

## Not yet specified

- The detailed behavior of each capability screen: states, errors, debouncing, and how calibration's ±1000 wire range is presented. Waits on the screen layout decision.
- Whether a screen prototype is worth building before the per-screen specs.
- How the Dart protocol layer is tested, possibly with test vectors drawn from the protocol doc, and whether iOS settles its MTU before CAPABILITY is sent.
- Slicing the decided behavior into implementation cards.

## Out of scope

- Watch-hardware behavior: idle dim and screen-off power steps, screen brightness, watch battery and PMU readings, wrist flick, the power button.
- The watch's "Ears Sys Info" panel, a placeholder until the ears send power data.
- Web-app-only features: streaming a custom animation (0x05) and a timeline editor.
- Backlog ideas the watch doesn't have: renaming the ears, saving and loading lighting patterns, gyro motion control, voice or music control, syncing with nearby ears.
- Ears firmware changes, such as accepting more than one controller, or advertising a busy state so a phone could show it.
- The watch reconnecting after its own Disconnect button: its disconnection callback starts the 5 s reconnect timer on every disconnect. A watch bug, not phone work.
