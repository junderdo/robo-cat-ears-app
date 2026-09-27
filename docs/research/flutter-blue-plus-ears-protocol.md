# flutter_blue_plus and the ears protocol on Android and iOS

**Question.** What does `flutter_blue_plus` (FBP) give us on Android and iOS for the ears' BLE protocol:
`ABF2` indications, MTU and chunked writes, device identity, reconnect behaviour (auto-connect and
background), and runtime permissions? What does each mean for this app?

**Short answer.** FBP covers the whole wire contract without plugin changes, and its per-app
operation queue already serializes GATT calls the way the protocol requires. Two things need care.
iOS picks its own MTU, so the app must trust `max_chunk_bytes` from `CAPABILITY` and never use long
writes. iOS also never shows a MAC address, so the phone's device key cannot be the watch's key.

Versions read: `flutter_blue_plus` 2.3.12 (the tcg-proxy-card-app pin), `flutter_blue_plus_android`
9.0.3, `flutter_blue_plus_darwin` 9.0.4. 2.3.13 (2026-09-22) is the latest; its changes are Android
connection-state fixes and nothing that touches the findings below.

## The wire contract, as far as it matters here

From `robo-cat-ears/docs/ble-protocol.md` and the firmware:

- Service `0xABF0`, advertised by UUID and by the name `ROBO_CAT_EARS` (§1.1). `ABF1` is where the
  client writes: WRITE, WRITE_NR and READ. `ABF2` is where the ears answer: READ and **INDICATE
  only**. This is now unconditional in `main/ble.c:165-170`.
- One client at a time. The ears stop advertising while connected (§1.3).
- The ears set local MTU 512 but never start an MTU exchange; they record whatever the client
  negotiates (`main/ble.c:678-681`, `:722`). `max_chunk_bytes = MTU - 3` comes back in
  `CAPABILITY`. A write longer than `MTU - 3` turns into a Write Long, which the firmware **silently
  drops** (§1.4).
- Connect sequence: subscribe to `ABF2`, then `CAPABILITY`, then `LIST` (§6). Any `0x06` request sent
  before the CCCD is set is dropped (§9.1).
- `CAPABILITY` = `[version][slot_count][max_chunk_bytes:u16][serial:6]`. The serial is optional:
  it is present only when the payload is 10 bytes or longer and not all zero (§8.1).
- The ears use a **public** address (`main/ble.c:99`). The watch saves that address to NVS and
  auto-reconnects to it (`robo-cat-ears-watch/.../bluetooth_service.cpp:227`, `:623`).

## Findings

### 1. ABF2 indications: `setNotifyValue(true)` is enough on both platforms

- **Android.** FBP picks the CCCD value from the characteristic's properties. With INDICATE and
  no NOTIFY it writes `ENABLE_INDICATION_VALUE`. If a characteristic declares both, NOTIFY wins
  unless you pass `forceIndications: true`, which is Android-only.
  (`flutter_blue_plus_android` `FlutterBluePlusPlugin.java:1165-1190`)
- **iOS/macOS.** `setNotifyValue` maps to `CBPeripheral.setNotifyValue`, and CoreBluetooth
  chooses indications for an INDICATE-only characteristic. `forceIndications` is asserted off there.
  (`flutter_blue_plus` `lib/src/bluetooth_characteristic.dart:252-266`; README, "Subscribe to a
  characteristic")
- `setNotifyValue` waits for the CCCD write to be acknowledged before it returns
  (`bluetooth_characteristic.dart:283-300`). "Subscribed" therefore means the ears have the CCCD
  set, which is what §9.1 requires before the first `0x06` request.
- The OS stack sends the ATT confirmation for each indication. The app writes no code for it.
- `onValueReceived` fires for `read()` results **and** for indications (README, "onValueReceived is
  never called"). A read of `ABF2` and a store response arrive on the same stream. Filter
  on the leading type byte, as the protocol intends (§1.2). With current ears firmware a read
  always returns calibration (`0x03`), not lighting; see [abf2-state-read.md](abf2-state-read.md).
- Subscriptions are cleared on disconnect and services must be rediscovered on every reconnect
  (`flutter_blue_plus.dart:502-517`; README, "Connect to a device"). That matches the protocol's
  rule to re-run the connect sequence every time.

**For us:** no platform branches for indications. Wrap `ABF2` in one stream that splits
`0x06` frames from state reads by their first byte.

### 2. MTU and chunked writes: trust `max_chunk_bytes`, never `allowLongWrite`

- **Android.** `connect()` asks for MTU 512 by default and awaits `requestMtu` before it returns
  (`bluetooth_device.dart:106-194`). The ears cap at 512, so expect MTU 512 and
  `max_chunk_bytes` 509, the same as Chrome.
- **iOS/macOS.** There is no MTU request API: `requestMtu` throws "iOS does not allow mtu requests
  to the peripheral". iOS negotiates the MTU itself, "typically 135 to 255", with no callback. FBP
  polls `maximumWriteValueLength(.withoutResponse) + 3` for about 2 s after connect and emits
  `device.mtu` changes as it moves (darwin `FlutterBluePlusPlugin.m:744`, `:1179-1260`, `:2415-2444`;
  README, "MTU").
- **Both platforms reject writes over `MTU - 3`** with an error unless `allowLongWrite: true`
  (Android `getMaxPayload`, `FlutterBluePlusPlugin.java:946-953`, `:1808-1830`; darwin `:517`,
  `:2415-2438`). That default protects us. With `allowLongWrite` on, iOS and Android split the value
  into a Write Long, and the firmware throws Write Longs away. **Never set `allowLongWrite`.**
- Frame counts at a typical iOS MTU of 185 (`max_chunk_bytes` 182, 177 payload bytes per frame): a
  full `LIST` (801 B) takes 5 frames and a worst-case `STORE` (819 B) takes 5. At 7.5–15 ms per
  indication round trip that is still well under the 5 s timeout. The protocol already handles this;
  only the frame count changes.
- **Race to watch for (iOS).** The ears compute `max_chunk_bytes` from their MTU at the moment
  `CAPABILITY` arrives. If iOS has not finished its MTU exchange by then, the answer is 20. That is
  valid but slow: about 55 frames for a big `STORE`. Discovery and subscribing usually give the
  exchange time to finish, but nothing guarantees it. A cheap guard is to wait for `device.mtu` to
  settle, or to use `min(max_chunk_bytes, mtuNow - 3)` and re-issue `CAPABILITY` if `mtuNow` rises
  later.
- FBP's default `OperationQueueMode.global` lets one BLE operation run at a time across the whole
  app (`flutter_blue_plus.dart:49`, `:103-130`; `_bleOperationMutexKey` `:153`). That is exactly
  §12.1's "serialize app-wide", with no queue of our own. Writes with response resolve on the ATT
  write response, which gives the per-chunk ack §5.1 relies on.

**For us:** chunk by the `max_chunk_bytes` the ears return, clamped by `mtuNow - 3`; leave
`allowLongWrite` off; do not hardcode 509. The Mac is a fair proxy for testing iOS's automatic MTU.

### 3. Device identity: `remoteId` is a MAC on Android and a per-phone UUID on Apple

- **Android:** `remoteId` is the Bluetooth address, `05:A4:22:31:F7:ED` style. It "never changes"
  and, because the ears use a public address, it is the same value the watch saves.
  (README, "The remoteId is different on Android versus iOS & macOS")
- **iOS/macOS:** `remoteId` is a CoreBluetooth `NSUUID`. "The first time a local manager encounters
  a peer, the system assigns the peer a UUID" (Apple, `CBPeer.identifier`). It is local to that
  phone, never equal to the MAC, and FBP warns "this uuid will periodically change". Apple's guide
  expects you to store it and pass it to `retrievePeripheralsWithIdentifiers:`, and says to fall
  back to a scan when that returns nothing (Core Bluetooth Programming Guide, "Reconnecting to
  Peripherals"). FBP's `connect` on darwin does exactly that lookup and fails with "Peripheral not
  found" when the UUID is unknown (`FlutterBluePlusPlugin.m:358-372`).
- **The one identity every client can see is the `CAPABILITY` serial**:
  `SHA-256("milklab-ears-serial-v1" ‖ factory eFuse MAC)[0..6]`. Opaque, stable across firmware
  updates, frozen (`milk-lab-creations/docs/adr/0002-how-a-pair-of-ears-is-identified.md`). But it is
  optional (absent on pre-serial firmware and on eFuse failure), and the ADR and protocol §6 both say
  it must **not** be the cache key for the slot list. Use the platform handle for that.

**For us:**

- Tag the per-connection slot cache with `remoteId`, as §6 requires.
- Save `remoteId` to reconnect to the last ears (Android: the MAC; iOS: the UUID, with a scan as
  fallback).
- Key anything meant to follow a pair of ears across phones, the watch, or the web app by the
  **serial**, and handle "no serial" explicitly.
- The phone cannot learn the watch's saved address on iOS, and the two cannot compare MAC-based
  keys. The serial is the only shared name.

### 4. Reconnect: auto-connect exists on both, but its trade-offs differ

- `connect(autoConnect: true)` returns immediately, does not time out, and reconnects "whenever your
  device is found". **It is incompatible with the `mtu` argument**, which is asserted, so you must
  call `requestMtu` yourself on Android after each reconnect (README, "Auto Connect";
  `bluetooth_device.dart:112-114`). If you forget, the MTU stays 23 and `CAPABILITY` reports
  `max_chunk_bytes` 20.
- **Android** passes it through as `connectGatt(ctx, autoConnect=true, …, TRANSPORT_LE)`
  (`FlutterBluePlusPlugin.java:726-760`), the OS-level "connect as soon as it becomes available"
  (Android, "Connect to a GATT server"). FBP also re-arms it when the adapter turns back on
  (`flutter_blue_plus.dart:489-498`).
- **iOS 17+** sets `CBConnectPeripheralOptionEnableAutoReconnect`, which makes the system
  reconnect after a link drop (darwin `:383-388`; Apple docs for that constant). FBP also calls
  `connect` again itself when a link drops (`flutter_blue_plus.dart:519-530`). CoreBluetooth
  "connection requests do not time out" (Apple, background processing guide).
- **Background.** iOS needs `UIBackgroundModes: bluetooth-central` to be woken for BLE events, and
  `FlutterBluePlus.setOptions(restoreState: true)` to be relaunched after the system kills the app
  (it sets `CBCentralManagerOptionRestoreIdentifierKey`, darwin `:123-145`). Each wake gives about
  10 s. FBP calls background use "an advanced use case … You may have to fork it". Android has no
  FBP support for this and points to a foreground service (`flutter_foreground_task`). (README,
  "Using Ble in App Background"; Apple, "Core Bluetooth Background Processing")
- **The ears accept one client and stop advertising while connected** (protocol §1.3). A phone with
  a pending auto-connect will grab the ears whenever they advertise. That races the watch's own
  auto-reconnect and locks the watch out for as long as the phone holds the link. Auto-connect in
  the background is the worst case, because the user is not even looking at the phone.

**For us:** the phone and the watch cannot share a connection. Whoever connects first owns the
ears. If the phone reconnects in the foreground only, with no auto-connect and a disconnect when
the app is backgrounded, the watch stays usable. Background auto-connect would need an explicit
"phone takes over" decision. None of the iOS background behaviour can be verified without an iPhone.

### 5. Runtime permissions: FBP prompts on Android; iOS prompts on first use

- **Android 12+:** `BLUETOOTH_SCAN` (with `neverForLocation`) and `BLUETOOTH_CONNECT` are runtime
  permissions. Android 11 and lower need `ACCESS_FINE_LOCATION` at runtime to scan
  (Android, "Bluetooth permissions"). FBP asks for whatever is missing itself, just before `scan` or
  `connect` (`ensurePermissions`, `FlutterBluePlusPlugin.java:1547-1590`, called from `:388-526`,
  `:695-704`). A denial comes back only as error text: "Permission … required". The
  tcg-proxy-card-app maps that string to `BlePermissionDeniedException`
  (`lib/bluetooth/flutter_blue_plus_service.dart:144-149`).
- **Manifest:** the FBP "No Location" block, as used verbatim in tcg-proxy-card-app's
  `AndroidManifest.xml`. We do not need location, since the ears are found by name or service UUID.
- **iOS:** `NSBluetoothAlwaysUsageDescription` in `Info.plist`. The system prompt appears on first
  CoreBluetooth use, and `adapterState` reports `unauthorized` on denial.
- **macOS:** turn on App Sandbox → Hardware → Bluetooth
  (`com.apple.security.device.bluetooth`). tcg-proxy-card-app has no macOS target, so there is no
  prior art for this. (README, "Add permissions for iOS / macOS")

**For us:** copy the tcg-proxy-card-app manifest and plist entries, add the macOS entitlement for
bench testing, and surface `unauthorized` / permission-denied as a settings deep link, not a
generic failure.

### 6. Licence (not asked, but it gates using FBP)

FBP is under the FlutterBluePlus License. It is free for personal, nonprofit and educational use.
"Any use … by or for a for-profit company or corporation — including commercial use by individuals
— requires the purchase of a commercial license", and that includes development. `connect()`
requires a `license:` argument, and builds may send a licence ping with the package and app name
(`LICENSE.md` §1.3–1.4, §3; CHANGELOG 2.3.5). tcg-proxy-card-app passes `License.nonprofit`.

## Open questions this raised

- Does iOS reliably finish its MTU exchange before `CAPABILITY`? Check on the Mac by logging
  `device.mtu` against `max_chunk_bytes`. If it does not, choose between a settle-wait and a
  re-issued `CAPABILITY`.
- Is Milk Lab Creations for-profit? If so, a commercial FBP licence is needed before development
  counts as licensed.
- How long does an iOS `remoteId` stay stable for a public-address peripheral? FBP says it
  "periodically changes"; Apple's docs don't say. The phone needs the scan fallback either way.
- Should the phone ever hold the ears in the background, given that it locks the watch out?

## Sources

- `flutter_blue_plus` 2.3.12 package (pub.dev): `README.md`, `CHANGELOG.md`, `LICENSE.md`,
  `lib/src/bluetooth_characteristic.dart`, `lib/src/bluetooth_device.dart`,
  `lib/src/flutter_blue_plus.dart`. 2.3.13 changelog from
  https://pub.dev/packages/flutter_blue_plus/changelog.
- `flutter_blue_plus_android` 9.0.3: `android/src/main/java/com/jmx/flutter_blue_plus/FlutterBluePlusPlugin.java`.
- `flutter_blue_plus_darwin` 9.0.4: `darwin/flutter_blue_plus_darwin/Sources/flutter_blue_plus_darwin/FlutterBluePlusPlugin.m`.
- Apple: [`CBPeer.identifier`](https://developer.apple.com/documentation/corebluetooth/cbpeer/identifier);
  [`CBConnectPeripheralOptionEnableAutoReconnect`](https://developer.apple.com/documentation/corebluetooth/cbconnectperipheraloptionenableautoreconnect);
  [Core Bluetooth Programming Guide, Reconnecting to Peripherals](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/BestPracticesForInteractingWithARemotePeripheralDevice/BestPracticesForInteractingWithARemotePeripheralDevice.html);
  [Core Bluetooth Background Processing for iOS Apps](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html).
- Android: [Bluetooth permissions](https://developer.android.com/develop/connectivity/bluetooth/bt-permissions);
  [Connect to a GATT server](https://developer.android.com/develop/connectivity/bluetooth/ble/connect-gatt-server).
- `robo-cat-ears/docs/ble-protocol.md` §1, §5–§8, §9.1, §10, §12; `robo-cat-ears/main/ble.c`.
- `milk-lab-creations/docs/adr/0002-how-a-pair-of-ears-is-identified.md`.
- `robo-cat-ears-watch/components/services/bluetooth_service/bluetooth_service.cpp`.
- Prior art: `tcg-proxy-card-app/lib/bluetooth/flutter_blue_plus_service.dart`,
  `android/app/src/main/AndroidManifest.xml`, `ios/Runner/Info.plist`.
