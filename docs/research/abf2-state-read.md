# What an ABF2 read returns

**Question:** Does a GATT read of characteristic `ABF2` return the ears' current lighting, animation
mode, and calibration, or does the ears' READ handler's calibration refresh overwrite the others? The
answer decides how the phone learns the ears' state on connect.

**Answer:** Every read of `ABF2` returns servo calibration (`[0x03][8 bytes]`), whatever was last
written and whenever the read happens. Lighting and animation mode can't be read from the ears with
the current firmware. The watch's lighting and auto-animate reads therefore always fail their
type-byte check and are silently dropped.

Sources: ears firmware at `~/personal/projects/robo-cat-ears/robo-cat-ears` (commit `e7561e3`), watch
at `~/personal/projects/robo-cat-ears-watch/robo-cat-ears-watch` (commit `8c027d5`). Found by reading
the code; no hardware test was run.

## Ears: one value attribute, rewritten before every read

- `ABF2`'s value attribute is `ESP_GATT_RSP_BY_APP` (`main/ble.c:221-222`), so the stack doesn't answer
  reads itself. Every read reaches the app's `ESP_GATTS_READ_EVT` handler.
- The handler first calls `controller_update_servo_calibration_characteristic()` (`main/ble.c:622-624`,
  commented "Always update calibration data before responding"). Then it reads the attribute back
  and sends it as the response (`main/ble.c:627-644`).
- `controller_update_servo_calibration_characteristic` loads calibration from NVS, falling back to
  zeros (`servo_calibration_init`). It packs `[0x03][left_azi][left_lat][right_azi][right_lat]` (i16
  big-endian, 9 bytes) and writes it into both `spp_data_notify_val` and the GATT attribute with
  `esp_ble_gatts_set_attr_value` (`main/controller.c:255-317`). The frame layout is
  `[type][payload]` (`main/types/ble_packet_types.h:75-95`), with `0x03` = `DATA_TYPE_SERVO_CALIBRATION`
  (`main/types/ble_packet_types.h:48`).
- Three other writers put different frames into the same attribute. The refresh overwrites all of
  them before any client can read them:

  | When | Writer | Frame it leaves | What the next read returns |
  |---|---|---|---|
  | Connect | `controller_update_lighting_characteristic()` (`main/ble.c:729-730`) | `0x02` lighting | calibration |
  | Lighting command applied | `process_lighting_command` → same (`main/controller.c:447-451`) | `0x02` lighting | calibration |
  | Animation-mode command applied | `process_animation_mode_command` → `controller_update_animation_mode_characteristic()` (`main/controller.c:322-364`, `:393`) | `0x04` animation mode | calibration |
  | Calibration command applied | `process_servo_calibration_command` → calibration refresh (`main/controller.c:484`) | `0x03` calibration | calibration |

- `controller_handle_read` (`main/controller.c:488-530`) returns the buffer without refreshing, but
  nothing calls it. The only caller of any read path is the `ble.c` handler above.
- The ears' own protocol doc contradicts the code. `docs/ble-protocol.md:54-57` and `:133-134` say the
  readable `ABF2` value carries lighting state and is "the only readable state the service exposes".
  In the code, that readable state is calibration.

## Watch: the lighting and auto-animate reads always fail

- `BluetoothService::readDataPacket` ignores its `data_type` argument except in a log line. It issues
  one plain `esp_ble_gattc_read_char` on `ABF2` (`components/services/bluetooth_service/bluetooth_service.cpp:822-876`)
  and hands the raw bytes to the caller (`:372-401`).
- **Lighting** (Glow screen, `glow_screen.cpp:700`): `LightingService::readLightingData` rejects any
  frame whose first byte isn't `0x02` (`components/services/lighting_service/lighting_service.cpp:268-274`).
  It gets `0x03`, logs "Invalid data type", and never invokes the screen's callback. The screen keeps
  what it loaded from its own NVS.
- **Auto-animate** (Animate screen, `animate_screen.cpp:297`):
  `AnimationModeService::readAnimationModeData` checks for `DataType::ANIMATION` (`0x01`), not
  `ANIMATION_MODE` (`0x04`) (`components/services/animation_mode_service/animation_mode_service.cpp:188`, `:224`).
  The ears never place `0x01` on `ABF2`. This read would fail even if the refresh were removed, so it
  is a separate watch bug. Either way the switch never reflects the ears.
- **Calibration** (Calibration screen, `calibration_screen.cpp:260`): checks for `0x03`
  (`components/services/calibration_service/calibration_service.cpp:241-247`), which it always gets.
  This is the only read that works.

## Is the suspected bug real?

Yes, and it is in the ears firmware: `main/ble.c:624` makes `ABF2` a calibration-only readable
value. It doesn't lose data. Lighting and mode are still in NVS, and the ears still apply them. What
it breaks is any client learning them by reading. Deleting line 624 alone would not fix this. The read
would then return whichever frame was written last, and calibration would become unreadable after
connect, because connect writes lighting. A real fix needs a read that says which state it wants,
such as a request on `ABF1` answered by an `ABF2` indication, like the store's `CAPABILITY`/`LIST`
sub-opcodes (`docs/ble-protocol.md:554`). The watch's `0x01` check is a second, independent bug, in
the watch repo.

## What the phone should do on connect

- **Calibration:** read `ABF2` and accept it only if byte 0 is `0x03` and it has 9 bytes. This is
  reliable today.
- **Lighting and auto-animate:** don't expect them from the ears. Keep them in the phone's own
  storage, as the watch already does for colors (`glow_screen.cpp`: it ignores the ears' colors when
  it has its own). Show the stored values, and push them when the user changes them. If the phone
  needs the ears' actual values, for example after the watch changed them, that needs a firmware
  change first.
- Parse by type byte and never trust a read to be the type you asked for. That keeps the phone
  correct if the firmware later changes what `ABF2` holds.

## Confirming on hardware (optional)

With nRF Connect or similar: connect, write a lighting frame (`0x02 …`) to `ABF1`, then read `ABF2`.
The code predicts byte 0 is `0x03` and the length is 9. The watch's logs give the same signal: the Glow
screen logs "Invalid data type in response: got 0x03".
