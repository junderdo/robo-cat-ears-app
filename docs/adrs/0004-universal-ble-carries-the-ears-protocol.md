# 0004. universal_ble carries the ears protocol

## Status

Accepted

## Context

- This app is GPL-3.0, like the ears firmware and the watch.
- `flutter_blue_plus` covers the whole ears protocol ([research](../research/flutter-blue-plus-ears-protocol.md)), but since 2.0.0 it ships under the FlutterBluePlus License. That licence forbids relicensing (§1.2) and requires a paid commercial licence for any for-profit use, development included (§1.3, §3). Those are further restrictions GPL-3.0 does not permit (§7, §10), so `flutter_blue_plus` 2.x can't be conveyed inside a GPL-3.0 app, whether or not the app is for-profit.
- The commercial licence is a one-time $999 for an individual and $2,999 for 1–9 employees ([price list](https://jamcorder.myshopify.com/products/flutterblueplus-commercial-license.json)).
- GPL-compatible alternatives, as of 2026-09-27 ([pub.dev](https://pub.dev)):
  - `universal_ble` 2.3.0 (BSD-3-Clause, Navideck): actively maintained. It covers scanning by service with RSSI, connect and disconnect, service discovery, writes with and without response, notifications, and a connection-state stream. On Android the app must call `requestMtu` itself; on iOS it only reads the MTU iOS negotiated, and can read 23 right after connect ([research](../research/universal-ble-ears-protocol.md)).
  - `flutter_reactive_ble` 5.6.0 (BSD-3-Clause, Philips Hue): active, but has 162 open issues and no `disconnect()`; you disconnect by cancelling the connection stream.
  - `bluetooth_low_energy` 6.2.1 (MIT): the least actively maintained, and it can't read the MTU on iOS.
  - `flutter_blue_plus` 1.36.8 (BSD-3-Clause): the last release before the licence change, frozen since September 2025.

Considered and rejected:

- Buying a `flutter_blue_plus` licence: it still leaves the GPL conflict, unless this app leaves the GPL family its firmware and watch share.
- Pinning `flutter_blue_plus` 1.36.8: it gets no fixes as Android and iOS move on.
- `flutter_reactive_ble`: a stream-lifetime disconnect fits ADR 0001's explicit disconnect less directly.
- `bluetooth_low_energy`: without an MTU read on iOS, the phone can't check the MTU against `max_chunk_bytes`.

## Decision

The phone's BLE stack is `universal_ble`.

## Consequences

- The app stays GPL-3.0, with no licence fee or build-time licence ping.
- The `flutter_blue_plus` research, and the decisions that cite it (ADR 0001's auto-connect risk), describe a plugin the app won't use. A research pass on `universal_ble` confirms or corrects chunking, iOS identity, and auto-connect behavior before the Connect gate and implementation lean on them.
