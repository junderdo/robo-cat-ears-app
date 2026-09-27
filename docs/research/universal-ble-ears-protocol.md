# universal_ble and the ears protocol on Android and iOS

**Question.** ADR 0004 picks `universal_ble` over `flutter_blue_plus` (FBP). Does `universal_ble`
carry the whole ears protocol unmodified on Android and iOS? The FBP research
(`flutter-blue-plus-ears-protocol.md`, branch `docs/flutter-blue-plus-research`) settled `ABF2`
indications, MTU and chunking by `max_chunk_bytes`, device identity, and auto-connect. Which of
those findings still hold, and which change? Would any connect or auto-connect behaviour hold the
ears in the background, against ADR 0001?

**Short answer.** Yes. `universal_ble` carries the whole wire contract with no plugin changes, and
its default global command queue gives the app-wide serialization §12.1 asks for. It does less for
you than FBP in three places, and the app has to cover each one:

- It never asks for an MTU on Android.
- It never checks write length on either platform.
- On iOS a connect that times out in Dart stays pending natively.

Subscribe to `ABF2` with `indications`, never `notifications`. Identity and auto-connect behave as
the FBP research found. ADR 0001 holds, with one extra rule: always `disconnect()` after a failed
connect.

Version read: `universal_ble` 2.3.0 (pub.dev archive, 2026-09-07, the latest release). `main` carries
an unreleased 3.0.0 that changes the default queue; see §2.

## The wire contract, as far as it matters here

From `robo-cat-ears/docs/ble-protocol.md` and the firmware, unchanged from the FBP research except
where noted:

- Service `0xABF0`. `ABF1` declares WRITE, WRITE_NR and READ (`main/ble.c:165-166`). `ABF2` declares
  READ and INDICATE only (`main/ble.c:168-170`).
- **The CCCD handler checks the indicate bit only.** `enable_data_ntf = (cccd & CCCD_INDICATE_BIT)
  != 0` (`main/ble.c:549-562`). A CCCD write of `0x0001` (notify) leaves every `0x06` request
  dropped (§9.1).
- The ears set local MTU 512 but never start an MTU exchange (`main/ble.c:722`, `:678-681`).
  `max_chunk_bytes = spp_mtu_size - 3` is computed when `CAPABILITY` is answered (`main/ble.c:846-849`).
- A Write Long reaches `ESP_GATTS_EXEC_WRITE_EVT`, which prints and frees the buffer and never
  dispatches it (`main/ble.c:664-671`; protocol §1.4).
- One client at a time; advertising stops while connected (§1.3). Public address
  (`main/ble.c:99`).
- The store path writes with response (§5.1). §2 does not fix a write type for `0x01`–`0x05`.
  `ABF1` is `ESP_GATT_AUTO_RSP`, so both kinds of write are acknowledged or accepted
  (`main/ble.c:212-214`, `:656-663`).

## Findings

### 1. ABF2 indications: confirmed, but call `indications.subscribe()` explicitly

FBP picked the CCCD value from the characteristic's properties. `universal_ble` doesn't: the caller
chooses notifications or indications, and the Android plugin writes whatever it was told.

- **Android.** `setNotifiable` maps `INDICATION` to `ENABLE_INDICATION_VALUE` and `NOTIFICATION`
  to `ENABLE_NOTIFICATION_VALUE`. It writes the CCCD and then calls `setCharacteristicNotification`,
  with **no property check** (`UniversalBlePlugin.kt:495-600`, values at `:523-527`). The low-level
  `UniversalBle.subscribeNotifications(...)` on `ABF2` would therefore write `0x0001`. The ears would
  then drop every store request silently, and the client would see 5 s timeouts.
- **iOS/macOS.** `setNotifiable` rejects notifications on a characteristic without `.notify`, and
  indications without `.indicate`. It then calls `setNotifyValue(true)`
  (`UniversalBlePlugin.swift:349-374`). CoreBluetooth picks indications for an INDICATE-only
  characteristic. It enables "notifications only" only when both are declared (Apple,
  `setNotifyValue(_:for:)`).
- **The high-level API guards both platforms.** `characteristic.indications.subscribe()` checks
  that the characteristic declares INDICATE. `notifications.subscribe()` on `ABF2` throws
  "Operation not supported" before reaching native code (`ble_characteristic_extension.dart:142-181`).
- **Subscribing waits for the CCCD acknowledgement.** On Android the future completes in
  `onDescriptorWrite` (`UniversalBlePlugin.kt:1633-1708`). On iOS it completes in
  `didUpdateNotificationStateFor` (`UniversalBlePlugin.swift:776-789`). "Subscribed" means the
  ears have the CCCD set, as §9.1 requires. Confirmed.
- The OS sends the ATT confirmation for each indication. The app writes no code for it. Confirmed.
- **Value stream: corrected.** FBP put read results and indications on the same stream on both
  platforms. In `universal_ble`:
  - On Android, only `onCharacteristicChanged` (indications) feeds `onValueReceived`; `read()`
    results don't (`UniversalBlePlugin.kt:1615-1631` vs `:658-690`).
  - On iOS, any `didUpdateValueFor` on a characteristic that is notifying goes to the stream,
    including read results (`UniversalBlePlugin.swift:791-823`; "On iOS and MacOS this command will
    also trigger [onValueChange]", `universal_ble.dart:328`).
  - Filtering on the leading type byte (§1.2) is still required, for iOS.
- **iOS read/indication race (new).** CoreBluetooth reports read responses and indications through
  the same callback. The plugin resolves a pending `ABF2` read with whatever value arrives next,
  and that value may be an indication (`UniversalBlePlugin.swift:791-846`). The command queue
  doesn't help, because indications aren't queued operations. Don't read `ABF2` while a `0x06`
  request is outstanding.
- **Disconnects clear subscriptions and the service cache.** `updateConnection(false)` calls
  `CacheHandler.resetDeviceCache` (`universal_ble_platform_interface.dart:190-207`). Rediscover and
  resubscribe on every connect, as the protocol already requires. Confirmed.

**For us:** subscribe with `abf2.indications.subscribe()` and never call `subscribeNotifications`
for `ABF2`. Keep the one `ABF2` stream that splits `0x06` frames from state reads by their first
byte. Don't read `ABF2` while a store request is outstanding.

### 2. MTU and chunked writes: the app requests the MTU and enforces the length

**MTU request, Android: corrected.** FBP asked for MTU 512 inside `connect()`. `universal_ble`
doesn't. `connect()` only calls `connectGatt` (`UniversalBlePlugin.kt:277-321`), so the MTU stays 23
unless the app calls `requestMtu`.

- `requestMtu` calls `gatt.requestMtu` and completes from `onMtuChanged` with the negotiated value
  (`UniversalBlePlugin.kt:1051-1060`, `:1131-1152`).
- Android 14+ "requests the BLE ATT MTU to 517 bytes when the first GATT client requests an MTU, and
  disregards all subsequent MTU requests" (Android, `BluetoothGatt.requestMtu`). That still takes a
  request. The ears cap at 512, so either way expect MTU 512 and `max_chunk_bytes` 509.
- If the app skips the request, `CAPABILITY` reports `max_chunk_bytes` 20. That is valid but slow:
  15 payload bytes per frame, so about 54 frames for a full `LIST` and 55 for a worst-case `STORE`.
- **Known bug:** issue #307 (open) says `requestMtu` can time out on reconnect even when
  `onMtuChanged` reports success. The 2.3.0 code registers the waiter after the native call, on an
  unsynchronized list. It also ignores `requestMtu`'s boolean result, so a refused request waits for
  the 10 s queue timeout.
- If `requestMtu` fails or times out, go on to `CAPABILITY` anyway. The ears report whatever MTU
  was actually negotiated. This is exactly the case `max_chunk_bytes` exists for.

**MTU read, iOS: confirmed with a caveat.**

- `requestMtu` asks for nothing. It returns `maximumWriteValueLength(.withoutResponse) + 3` at the
  moment of the call (`UniversalBlePlugin.swift:494-504`).
- There is no MTU-change stream. FBP polled for about 2 s after connect and emitted `device.mtu`;
  `universal_ble` has nothing like it.
- Issue #131 (open) reports iOS returning 23 right after connect while the peripheral had
  negotiated 495. That is the race the FBP research warned about: the ears may answer
  `CAPABILITY` with 20 if iOS hasn't finished its exchange.
- The guard is the same as before, done by polling. Call `requestMtu` before `CAPABILITY`. If the
  result is 23, wait briefly and ask again. Or re-issue `CAPABILITY` later if `requestMtu() - 3`
  exceeds the `max_chunk_bytes` you hold.

**Write length: corrected.** FBP rejected writes longer than `MTU - 3` unless you passed
`allowLongWrite`. `universal_ble` has no such guard on either platform:

- **Android:** `writeValue` checks only the characteristic's WRITE and WRITE_NR properties
  (`UniversalBlePlugin.kt:692-788`, checks at `:716-743`).
- **iOS:** `writeValue` checks properties and connection state only
  (`UniversalBlePlugin.swift:394-431`).
- An over-length write with response goes to the stack as-is. The stack sends it as a Write Long,
  as FBP does with `allowLongWrite` on, and the ears silently discard Write Longs (§1.4).
- Android truncates an over-length write without response: "the data sent is truncated to the MTU
  size" (Android, `BluetoothGatt.requestMtu`).
- **The app must never pass `write()` more than `max_chunk_bytes`.** This covers `0x06` frames and
  `0x05` stream chunks.

**Write with response: confirmed.** Writes default to with-response (`withResponse = true`,
`ble_characteristic_extension.dart:61-76`). The future resolves on the ATT Write Response:
`onCharacteristicWrite` (`UniversalBlePlugin.kt:800-848`) or `didWriteValueFor`
(`UniversalBlePlugin.swift:709-725`). Both complete the oldest pending write first (CHANGELOG 2.2.0
and 2.3.0). That is the per-chunk ack §5.1 relies on.

- `ABF1` now declares WRITE, so the plugin's property check passes on both platforms.
- Older firmware without WRITE (§11.1) gets "Characteristic does not support write withResponse"
  from the plugin. It never reaches the ears.

**Write without response: works, but slow on iOS in 2.3.0.**

- **Android:** the future completes on `onCharacteristicWrite`, like a write with response.
- **iOS 2.3.0:** each write without response completes only when CoreBluetooth calls
  `peripheralIsReady(toSendWriteWithoutResponse:)` (`UniversalBlePlugin.swift:422-430`,
  `:699-707`). Apple documents that callback as coming "after a failed call to writeValue".
- Issue #271 (open against 2.1.0) measures per-write latency on iOS and macOS growing by 50–100 ms
  per write under the global queue, until writes time out. The fix (#300, `canSendWriteWithoutResponse`
  backpressure) is merged on `main` but unreleased (3.0.0).
- The protocol only needs write without response for `0x01`–`0x05`, and those work either way.
- **Writing everything to `ABF1` with response** avoids the iOS path. It gives an ack for free, at
  one connection interval (7.5–15 ms) per write.

**Serialization: confirmed.**

- The default is `QueueType.global`: one command at a time across the app, with a 10 s timeout per
  command (`ble_command_queue.dart:7`, `:12`; README, "Command Queue"). That is §12.1's "serialize
  app-wide" with no queue of our own.
- `connect()` and `readRssi()` bypass the queue (`universal_ble.dart:159-186`; CHANGELOG 2.3.0).
- A Dart-side timeout frees the queue but doesn't cancel the native operation, so the next command
  can collide with it on Android.
- **Unreleased 3.0.0 changes the default** to `QueueType.auto`, where "all other platforms [than
  Android] run commands in parallel" (`main` CHANGELOG). Pinning `^2.3.0` keeps us below 3.0.
  When upgrading, set `UniversalBle.queueType = QueueType.global` explicitly, or serialize in the
  app.

**For us:**

- **Android:** `requestMtu(512)` after discovery and before subscribing. Tolerate its failure.
- **iOS:** poll `requestMtu` until it leaves 23 before `CAPABILITY`, or re-issue `CAPABILITY` when
  it rises.
- **Both:** chunk by `min(max_chunk_bytes, mtuNow - 3)`, and assert every `ABF1` write fits. Write
  with response. Leave the queue on `global`.

### 3. Device identity: confirmed; MAC on Android, per-phone UUID on Apple

- **Android:** `deviceId` is `BluetoothDevice.getAddress()`, the MAC, upper case, from scan results
  and every GATT callback (`UniversalBlePlugin.kt:1550`, `:1569-1613`). The ears use a public
  address, so it is the same value the watch saves.
- **Android reconnect by saved ID needs no scan.** `connect` builds the device with
  `adapter.getRemoteDevice(deviceId)` (`UniversalBlePlugin.kt:309`).
- **iOS/macOS:** `deviceId` is `CBPeripheral.identifier.uuidString` (`UniversalBleHelper.swift:157-160`).
  Apple assigns that UUID "the first time a local manager encounters a peer" (Apple,
  `CBPeer.identifier`), so it belongs to one phone.
- **iOS reconnect by saved ID** looks in the plugin's scan cache first, then calls
  `retrievePeripherals(withIdentifiers:)`. An unknown UUID fails with "Unknown deviceId"
  (`UniversalBlePlugin.swift:211-212`, `:878-890`). That is the same lookup FBP does, with the same
  need for a scan as fallback.
- Device IDs are compared case-insensitively since 2.1.1 (CHANGELOG; `cache_handler.dart:12-16`).
- The `CAPABILITY` serial is still the only name shared across phones, the watch and the web app.
  It is still optional, and still not the slot-cache key (§6, §8.1). Nothing here depends on the
  plugin.

**For us:** unchanged from the FBP research. Tag the slot cache and save the last ears by
`deviceId`. Fall back to a scan on iOS. Key anything cross-client by the serial.

### 4. Reconnect and the background: auto-connect confirmed, plus a pending-connect trap on iOS

- **`autoConnect` defaults to false** (`universal_ble.dart:159-164`).
  - **Android:** `connectGatt(ctx, false, …, TRANSPORT_LE)` is a direct connect
    (`UniversalBlePlugin.kt:303-319`). On a drop without auto-connect, the plugin closes the GATT
    client and nothing reconnects (`:1569-1613`).
  - **iOS:** `manager.connect(peripheral, options: nil)` (`UniversalBlePlugin.swift:211-258`).
  - **Neither plugin reconnects on its own.** FBP re-called `connect` after a drop.
    `universal_ble`'s `autoConnectDevices` sets are bookkeeping only (`UniversalBlePlugin.swift:105`,
    `:650`; `UniversalBlePlugin.kt:1586`, `:1604-1612`).
- **`autoConnect: true` behaves like FBP's.**
  - **Android:** `connectGatt(autoConnect=true)`, and the GATT client stays open after a drop "for
    Android to reconnect" (`UniversalBlePlugin.kt:1611`). That is Android's "automatically connect
    as soon as the remote device becomes available" (Android, `BluetoothDevice.connectGatt`).
  - **iOS 17+/macOS 14+:** it sets `CBConnectPeripheralOptionEnableAutoReconnect`
    (`UniversalBlePlugin.swift:236-252`), which makes the system reconnect "when the link drops"
    (Apple).
  - Either would grab the ears whenever they advertise, which is what ADR 0001 rules out.
- **Pending-connect trap on iOS (new).**
  - `UniversalBle.connect` has a Dart-side timeout, 60 s by default. When it fires, the call throws
    `ConnectionException` and cancels nothing natively (`universal_ble.dart:159-186`). The native
    `connect` ignores the timeout entirely (`universal_ble_pigeon_channel.dart:66-79`).
  - CoreBluetooth "attempts to connect to a peripheral don't time out. To explicitly cancel a
    pending connection … call `cancelPeripheralConnection(_:)`" (Apple,
    `CBCentralManager.connect(_:options:)`).
  - So after ADR 0001's "one direct connect attempt" fails, iOS still holds a connection request.
    It completes the next time the ears advertise, even with `autoConnect: false`, possibly after the
    app is backgrounded, and locks the watch out.
  - `disconnect()` cancels it: it calls `cancelPeripheralConnection` whenever the state isn't
    `.disconnected` (`UniversalBlePlugin.swift:260-271`), and Dart calls the platform `disconnect`
    even when already disconnected, "to prevent auto-reconnect" (`universal_ble.dart:221-230`).
  - On Android the stack ends a direct connect by itself. Until it does, a second `connect()` throws
    `CONNECTION_IN_PROGRESS` (`UniversalBlePlugin.kt:287-301`). `disconnect()` releases it there
    too (`:323-334`, `:1444-1462`).
- **iOS background state restoration is opt-in by `Info.plist`.** The plugin creates its
  `CBCentralManager` with `CBCentralManagerOptionRestoreIdentifierKey` only when `UIBackgroundModes`
  contains `bluetooth-central` (`UniversalBlePlugin.swift:47-88`, `:112-122`; README, "iOS background
  state restoration"; CHANGELOG 2.0.4).
  - ADR 0001 declares no background mode, so restoration stays off.
  - If anything adds `bluetooth-central` later, restoration turns on silently, and the plugin re-adopts
    restored connections (`:561-579`).
- `AppleConnectionOptions` (relaunch on connection events) is off unless passed
  (`UniversalBlePlugin.swift:218-234`).
- **Android app kill.** Without `AndroidConnectionOptions(closeGattOnDetach: true)`, the link
  outlives a swiped-away app until the peripheral's supervision timeout (README, "Close GATT on app
  teardown"). The ears ask for 4 s (§1.3), so the watch waits at most a few seconds. The option is
  cheap to set anyway (`UniversalBlePlugin.kt:117-118`, `:282`).

**For us:** ADR 0001 holds as written: foreground only, `autoConnect: false`, and no
`bluetooth-central`. Add one rule to its implementation: **every connect attempt that fails or
times out, and every attempt still pending when the app backgrounds, ends in `disconnect()`**.
Pass a connect timeout shorter than 60 s. Set `closeGattOnDetach: true`.

### 5. Runtime permissions (not asked; for parity with the FBP research)

- **Android 12+:** `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT` are requested by `startScan()`, or by
  an explicit `requestPermissions()`. Location is requested only with `withAndroidFineLocation` /
  `requestLocationPermission` (`PermissionHandler.kt:226-270`; README, "Android").
  - `connect()` does not request permissions. Reconnecting to saved ears without a scan needs
    `requestPermissions()` first (README, "Manually Requesting Permissions").
  - A denial throws `UniversalBleException`, not an error string to match.
- **Manifest:** the README block, with `BLUETOOTH_SCAN` `neverForLocation`, matches the FBP
  "No Location" setup.
- **iOS:** `NSBluetoothAlwaysUsageDescription`. The README also lists
  `NSBluetoothPeripheralUsageDescription`, but that key is for peripheral mode, which we don't use.
- **macOS:** `com.apple.security.device.bluetooth`. Without it,
  `getBluetoothAvailabilityState()` returns `unsupported` (README).

### 6. Licence

BSD 3-Clause, Navideck Labs OÜ (`LICENSE`). No licence argument and no build-time ping, as ADR 0004
says.

## What this means for ADRs 0001 and 0004

- **ADR 0001** stands. Its context cites FBP for the auto-connect risk. `universal_ble`'s
  `autoConnect: true` carries the same risk (§4), so the reasoning transfers unchanged. What it
  doesn't cover is the pending-connect trap: "one direct connect attempt" needs an explicit
  `disconnect()` on failure or timeout. Otherwise iOS completes the attempt later, in the
  background.
- **ADR 0004** stands. Two of its claims need detail:
  - "`requestMtu` returns the negotiated MTU": on Android the app must call it (nothing requests an
    MTU for us), and on iOS it is a snapshot that can read 23 early.
  - "writes with and without response": both work, but 2.3.0's iOS write without response has an
    open latency bug (#271).

  "2.3.x" is the right pin: 3.0.0 changes the default queue.

## Open questions

- Does iOS finish its MTU exchange before `CAPABILITY`? Measure on the Mac: log `requestMtu()`
  against `max_chunk_bytes` across several connects. (Carried over from the FBP research; there is
  now no MTU stream to watch, only polling.)
- Does Android's `requestMtu` time out on reconnect in practice (issue #307)? If it does, the
  fallback of going on to `CAPABILITY` without an answer carries the load.
- Is an over-length write with response on iOS really sent as a Write Long, and dropped by the
  ears? The firmware analysis predicts it; nobody has observed it. The length assertion makes it
  moot, but a bench test would settle §1.4 for native clients.
- Should `0x01`–`0x05` go with response on every platform, or only on iOS until 3.0.0 ships the
  write-without-response fix? This affects live lighting-slider feel, if the phone streams `0x02`.
- Does a pending iOS connection request survive app suspension with no background mode?
  CoreBluetooth documents no timeout, but not what happens on suspension. Either way, the
  `disconnect()` rule covers it. Verifying it needs an iPhone.
- How long does an iOS `identifier` stay stable for a public-address peripheral? Carried over; Apple
  doesn't say.

## Sources

- `universal_ble` 2.3.0 package (https://pub.dev/api/archives/universal_ble-2.3.0.tar.gz):
  `README.md`, `CHANGELOG.md`, `LICENSE`, `lib/src/universal_ble.dart`,
  `lib/src/utils/ble_command_queue.dart`, `lib/src/queue.dart`,
  `lib/src/extensions/ble_characteristic_extension.dart`, `lib/src/extensions/ble_device_extension.dart`,
  `lib/src/interfaces/universal_ble_platform_interface.dart`, `lib/src/utils/cache_handler.dart`,
  `lib/src/universal_ble_pigeon/universal_ble_pigeon_channel.dart`,
  `android/src/main/kotlin/com/navideck/universal_ble/UniversalBlePlugin.kt`, `PermissionHandler.kt`,
  `darwin/universal_ble/Sources/universal_ble/UniversalBlePlugin.swift`, `UniversalBleHelper.swift`.
- `github.com/Navideck/universal_ble`: `main` `CHANGELOG.md` (unreleased 3.0.0, commit `245195cb`);
  issues and PRs #131, #271, #272, #281, #297, #300, #302, #304, #305, #307.
- Apple: [`CBCentralManager.connect(_:options:)`](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/connect(_:options:));
  [`CBPeripheral.setNotifyValue(_:for:)`](https://developer.apple.com/documentation/corebluetooth/cbperipheral/setnotifyvalue(_:for:));
  [`CBPeripheral.maximumWriteValueLength(for:)`](https://developer.apple.com/documentation/corebluetooth/cbperipheral/maximumwritevaluelength(for:));
  [`peripheralIsReady(toSendWriteWithoutResponse:)`](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate/peripheralisready(tosendwritewithoutresponse:));
  [`CBConnectPeripheralOptionEnableAutoReconnect`](https://developer.apple.com/documentation/corebluetooth/cbconnectperipheraloptionenableautoreconnect);
  [`CBPeer.identifier`](https://developer.apple.com/documentation/corebluetooth/cbpeer/identifier);
  [`retrievePeripherals(withIdentifiers:)`](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/retrieveperipherals(withidentifiers:)).
- Android: [`BluetoothGatt.requestMtu`](https://developer.android.com/reference/android/bluetooth/BluetoothGatt#requestMtu(int));
  [`BluetoothDevice.connectGatt`](https://developer.android.com/reference/android/bluetooth/BluetoothDevice#connectGatt(android.content.Context,%20boolean,%20android.bluetooth.BluetoothGattCallback,%20int)).
- `robo-cat-ears/docs/ble-protocol.md` §1, §2, §5, §6, §8.1, §9.1, §10, §11, §12;
  `robo-cat-ears/main/ble.c`.
- `flutter-blue-plus-ears-protocol.md` (branch `docs/flutter-blue-plus-research`); ADRs 0001 and 0004
  (branch `docs/wayfinding-operations`).
