# 0012. The protocol layer is tested against a fake ears platform and shared fixtures

## Status

Accepted

## Context

The phone's protocol layer carries rules that break quietly if they're wrong: ADR 0011's MTU wait, ADR 0008's GET_STATE on connect, ADR 0010's latest-wins lighting writes and gamma, and the universal_ble rules from the [research](../research/universal-ble-ears-protocol.md) (request the Android MTU, subscribe explicitly, write ABF1 with response, `disconnect()` after every failed or timed-out connect). There's one Android test phone, a Mac, and no iPhone.

- universal_ble 2.3.0 lets an app replace its platform: `UniversalBle.setInstance(UniversalBlePlatform)`, where `UniversalBlePlatform` is abstract.
- The ears ecosystem already shares golden bytes. `milk-lab-creations/docs/spec/wire-format-fixture.json` holds the stored-animation keyframe format. `robo-cat-ears/test/` carries a byte-identical copy, checked by `check-fixture-drift.sh`, and a C conformance test reads it. `gen_serial_vectors.py` is written from the spec, not the firmware, so agreement means the derivation is right.
- The watch dims each channel by `(brightness / 100)^2.2`, rounded with `lroundf`, and sends colours unchanged at 100 and black at 0 (`robo-cat-ears-watch/.../screens/glow_screen.cpp`).
- Every protocol rule runs on timers: the 250 ms / 2 s MTU wait, the 10 s connect timeout, the 30 s retry, the 15 s Android grace, one lighting write in flight.
- The iOS Simulator has no Bluetooth.

Considered and rejected:

- An app-owned transport interface with a thin universal_ble adapter: the plugin calls the ADRs require would live in the adapter, which only hardware would test.
- Hand-written bytes in each Dart test: they agree with the phone's reading of the spec, not with an independent one.
- Keeping the canonical control-protocol fixture in the phone repo: the protocol doc and the firmware are its other readers.
- A scripted fake that each test programs frame by frame: every tab's tests would re-script the same conversations.
- Automated tests on a device: one phone and real ears can't run unattended.

## Decision

The protocol layer is tested off-device against a fake ears that replaces universal_ble's platform, with bytes from shared fixtures. Hardware checks what a fake can't.

- **The seam.** `FakeEarsPlatform` extends `UniversalBlePlatform` and is installed with `UniversalBle.setInstance`. App code calls `UniversalBle` directly. Methods the app doesn't use throw.
- **A stateful fake.** It holds calibration, lighting, auto-animate, and stored-animation slots, and answers the way the firmware does. Each test sets knobs for failures: how `max_chunk_bytes` grows across CAPABILITYs, `UNSUPPORTED_OPCODE` for GET_STATE, stale slots, a drop mid-transfer, a request never answered, busy or absent ears in a scan, and which platform it's running on. It logs every frame and plugin call, so tests can assert on the wire.
- **Fixtures.** A new control-protocol fixture (JSON) holds the golden bytes for CAPABILITY, store requests and chunked responses at several `max_chunk_bytes`, the 103-byte GET_STATE payload, lighting frames of 1 to 32 colours, calibration, play, auto-animate, and error codes. It's written from `ble-protocol.md` by an independent generator, and its canonical copy sits beside `robo-cat-ears/docs/ble-protocol.md`. The phone carries byte-identical copies of it and of `wire-format-fixture.json`, each with a drift check. A firmware conformance test against the new fixture is optional and not part of this effort.
- **Gamma vectors** come from the watch's formula at brightness 0, 1, 50, 99, and 100.
- **Time.** Every timer runs through `package:clock`, and tests drive it with `fake_async`, the fake ears' reply delays included. No test sleeps in real time.
- **Three layers**, all on the dev machine: codec unit tests against the fixtures; session tests of the protocol layer against the fake (connect sequence, store transfers, errors, drops); widget tests of the gate and each tab with the whole stack over the fake.
- **On hardware.** A manual checklist in `docs/agents/on-hardware-checks.md`, run on the Android phone with real ears before a release: the MTU reaching 100 and how long it takes (to tune ADR 0011's guesses), a 32-colour frame, a stored-animation list transfer, GET_STATE on current and old firmware, calibration moving the servos, the foreground grace, the watch connecting after a failed phone connect, signal strength and the scan-ID label. An iOS section stays marked unverified.
- **The Mac.** The app gets a macOS target, used for BLE checks as an approximation of iOS CoreBluetooth (MTU reporting, identity UUIDs, disconnect), not a substitute for an iPhone.
- **Fake ears in the app.** Debug builds run against the fake with `--dart-define=FAKE_EARS=true`, for UI work and screenshots. Release builds never include it.

## Consequences

- The ADRs' plugin rules are asserted in tests, not only remembered.
- A universal_ble upgrade that changes its platform interface breaks the fake at compile time, which flags the upgrade for a look.
- `UniversalBlePlatform` is wide, so the fake carries throwing stubs for most of it.
- The fixture is new work in `robo-cat-ears`, with a generator of its own. The GET_STATE cases can't be confirmed against real ears until the ADR 0008 firmware lands.
- The macOS target is a second platform to keep building, and it only approximates iOS.
- The fake is a second implementation of the ears' behavior that can drift from the firmware. The fixtures pin the bytes, and the hardware checklist catches drift in behavior.
