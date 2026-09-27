# 0011. The phone waits on the ears' CAPABILITY for a usable MTU

## Status

Accepted

## Context

The phone can't write more than `MTU - 3` bytes in one ATT write. The MTU starts at 23 and grows only after an MTU exchange (`docs/research/universal-ble-ears-protocol.md` §2):

- **Android.** universal_ble never requests an MTU. The app has to call `requestMtu`, which can time out even when the exchange succeeds (issue #307). Android 14+ settles on 517 after the first request, and the ears cap it at 512.
- **iOS.** iOS negotiates on its own, but `requestMtu` only reports the current value, and it can report 23 just after connect while the exchange is still running (issue #131).

What the ears do:

- **They know the real MTU.** The firmware updates it on the exchange and recomputes `max_chunk_bytes` on every CAPABILITY answer (`robo-cat-ears/main/ble.c:678-680`, `main/store.c:43-46`). A second CAPABILITY reports the link as it is now.
- **Only lighting needs a large MTU.** Store requests are chunked by `max_chunk_bytes`, and store responses and GET_STATE are chunked by the ears, so they work at any MTU, just slower. Calibration, auto-animate, and built-in play frames are under 20 bytes. A lighting frame (`0x02`) must be one write: the firmware doesn't reassemble it and drops Write Longs. A full 32-colour frame is 100 bytes, and at MTU 23 only 5 colours fit.
- Protocol §1.4 says to "design nothing that reads or assumes the MTU"; `max_chunk_bytes` is the contract.
- The watch requests the MTU and holds off being ready until the exchange completes.

Considered and rejected:

- Polling universal_ble's `requestMtu` until it leaves 23: that relies on the plugin value issues #131 and #307 get wrong.
- Using the first CAPABILITY answer without waiting: on iOS it can report 20.
- Requiring a near-maximum MTU, such as 185, so stored-animation transfers are also fast: it would turn phones away for speed alone.
- Connecting anyway with lighting capped to what fits, or read-only: it clashes with profiles that hold more colours and adds a state to every Glow screen.
- A firmware change to send lighting in chunks: it's ears work for a case nobody has seen yet.
- A separate "link too small" message on the gate: the case should be vanishingly rare, and "Couldn't connect" with retry already fits it.

## Decision

A connect waits until the ears report a `max_chunk_bytes` of at least 100, which fits a full lighting frame. The ears' CAPABILITY is the only MTU the phone relies on.

- **On Android**, call `requestMtu(512)` after discovery, before subscribing. Continue if it fails or times out.
- **On both platforms**, send CAPABILITY. If `max_chunk_bytes` is under 100, send CAPABILITY again every 250 ms for up to 2 s. The phone doesn't read universal_ble's MTU on iOS.
- The wait is part of connecting: it happens before GET_STATE (ADR 0008), within ADR 0005's 10 s connect timeout, and the gate shows "Connecting…".
- **If it never reaches 100**, the connect fails. The phone calls `disconnect()`, logs the MTU it reached, and the gate shows ADR 0005's "Couldn't connect" with its usual retry.
- Every write stays within the `max_chunk_bytes` from the last CAPABILITY.

## Consequences

- One MTU source, the same on both platforms, which neither plugin bug can affect.
- Once connected, every lighting frame fits in one write, so Glow and profiles never have to handle a small MTU.
- A connect can take up to 2 s longer when the MTU is slow to grow. The 250 ms interval and the 2 s budget are guesses to tune on the Android test phone, and on an iPhone later.
- A phone whose MTU stays small can't connect at all. If on-hardware testing ever finds one, chunked lighting in the firmware becomes a new ticket, and the logged MTU helps diagnose it.
