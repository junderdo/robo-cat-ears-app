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
- Android and iOS from one BLE stack, `flutter_blue_plus`. Test hardware: an Android phone and a Mac; no iPhone, so iOS BLE behavior is specified but verified later.
- The wire contract is `robo-cat-ears/docs/ble-protocol.md`. The ears accept one controller at a time, so the phone and the watch cannot both be connected.
- Watch reference: `robo-cat-ears-watch/components/brookesia_app_robo_cat_ears/` (screens) and `components/services/` (BLE, animation store, lighting). Related ADRs live in `milk-lab-creations/docs/adr/`.
- Grilling tickets use the `grilling` and `domain-modeling` skills. Every resolved decision gets an ADR in `docs/adrs/`; research lands in `docs/research/`.

## Decisions so far

## Not yet specified

- The detailed behavior of each capability screen: states, errors, debouncing, and how calibration's ±1000 wire range is presented. Waits on the screen layout and the shared-connection decisions.
- Whether a screen prototype is worth building before the per-screen specs.
- How the Dart protocol layer is tested, possibly with test vectors drawn from the protocol doc.
- Slicing the decided behavior into implementation cards.

## Out of scope

- Watch-hardware behavior: idle dim and screen-off power steps, screen brightness, watch battery and PMU readings, wrist flick, the power button.
- The watch's "Ears Sys Info" panel, a placeholder until the ears send power data.
- Web-app-only features: streaming a custom animation (0x05) and a timeline editor.
- Backlog ideas the watch doesn't have: renaming the ears, saving and loading lighting patterns, gyro motion control, voice or music control, syncing with nearby ears.
- Ears firmware changes, such as accepting more than one controller.
