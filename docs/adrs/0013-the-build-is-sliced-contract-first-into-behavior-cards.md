# 0013. The build is sliced contract first into behavior cards

## Status

Accepted

## Context

ADRs 0001–0012 decide the phone's behavior for watch parity plus lighting profiles, and an ears firmware change (ADR 0008). The app is still the Flutter starter. Each card should deliver behavior a reviewer can exercise end to end, not one layer across many features.

- ADR 0012's test infrastructure has no behavior of its own. The fake ears, the fixture copies, and `package:clock` are needed by the first card that talks to the ears.
- The phone copies the control-protocol fixture byte for byte, and the fixture is generated from `robo-cat-ears/docs/ble-protocol.md`. ADR 0008 put the doc's GET_STATE section in the firmware change, which would put firmware ahead of all phone work.
- Real ears on today's firmware refuse GET_STATE, so until the firmware lands every phone card can only be exercised against the fake.
- ADR 0003's connect prompt offers **Save as profile**, so it can't ship before profiles.
- The repo has no CI, and no ADR decides it.

Considered and rejected:

- Firmware, then fixture, then phone, in strict order: the phone waits on firmware for no gain.
- Phone tests with hand-written bytes until the fixture exists: they build the drift ADR 0012 guards against.
- Standalone cards for the fake ears or the test harness: horizontal slices with nothing to exercise.
- One card for all of Connect, or for Glow with profiles: too big to review.
- A reduced connect prompt (Apply or Discard) before profiles: two versions of one prompt.

## Decision

The contract goes first, then a walking-skeleton connect card, then one card per behavior. The cards live in Trello's **Backlog** with the `mobile-app` label, the ears cards also `ears-firmware`, each with a `Blocked by:` first line.

- **The contract moves out of the firmware card.** [Put GET_STATE in the BLE contract and generate the control-protocol fixture](https://trello.com/c/QnZgsy2s) writes GET_STATE and the `ABF2` correction into `ble-protocol.md`, with the new fixture and its generator. The firmware and the phone both build against it in parallel.
- **Test infrastructure lands with its first user.** [Connect to ears from the gate and disconnect from the Ears tab](https://trello.com/c/OM6TUn6A) brings in the fake ears platform, `FAKE_EARS`, the fixture copies and drift checks, `package:clock` with `fake_async`, the on-hardware checklist, and a GitHub Actions workflow (analyze, test, drift checks). Later cards add the fake's knobs and their checklist lines. The macOS target is its own card.
- **Glow is split in two** at the write path: mode, speed, and brightness first, colour editing second.
- **The connect prompt ships with profiles.** Until then, a connect replaces the working lighting with the ears' lighting unless the ears already show it.

The build order, for one developer:

| Order | Card | Blocked by |
| --- | --- | --- |
| 1 | [Put GET_STATE in the BLE contract and generate the control-protocol fixture](https://trello.com/c/QnZgsy2s) | none |
| 2 | [Connect to ears from the gate and disconnect from the Ears tab](https://trello.com/c/OM6TUn6A) | 1 |
| 3 | [Run the app on macOS for BLE checks](https://trello.com/c/UvUDoLtn) | 2 |
| 4 | [Ears answer GET_STATE and restore auto-animate at boot](https://trello.com/c/l9o52aaj) | 1 |
| 5 | [Explain Bluetooth permission and handle Bluetooth off on the gate](https://trello.com/c/0L1g0aI6) | 2 |
| 6 | [Auto-connect to the last ears and ride out drops](https://trello.com/c/Sud7038d) | 2 |
| 7 | [Play built-in animations and toggle auto-animate](https://trello.com/c/cFtgVEs2) | 2 |
| 8 | [List and play the animations stored on the ears](https://trello.com/c/IK8YTc1L) | 2 |
| 9 | [Calibrate the servos live from the Ears tab](https://trello.com/c/iZROgabW) | 2 |
| 10 | [Change Glow mode, speed, and brightness live on the ears](https://trello.com/c/i2od4jTc) | 2 |
| 11 | [Edit Glow colours in a live picker](https://trello.com/c/JfemMCKN) | 10 |
| 12 | [Save, pick, and manage lighting profiles, and ask before replacing unsaved lighting](https://trello.com/c/7ZmBbyKy) | 11 |

The macOS target comes early because the Mac is the second BLE test machine. The firmware comes before anything past the skeleton, so later cards can be checked on real ears.

## Consequences

- The first merge is a doc and a fixture in `robo-cat-ears`, not code in this repo.
- The first phone card is the largest: it carries the whole connect sequence and the test harness.
- Until the firmware card ships, real ears show "Ears firmware is out of date", and phone cards are exercised on the fake.
- Between the Glow cards and profiles, lighting edited on the phone while disconnected is lost on the next connect.
- Cards 5–10 depend only on the skeleton and can be built in any order or in parallel.
- ADR 0008's single firmware change is now two cards: the contract and the firmware.
