# Robo Cat Ears mobile app

A phone app that controls a pair of Robo Cat Ears over Bluetooth, alongside the watch and web app.

## Language

**Ears**:
One pair of Robo Cat Ears, the device every controller talks to.
_Avoid_: device, peripheral, headset

**Controller**:
A client connected to the ears: the phone, the watch, or the web app. The ears accept one controller at a time.
_Avoid_: client, central, host

**Last ears**:
The ears this phone most recently connected to by the user's choice.
_Avoid_: saved device, paired ears

**Connect gate**:
The screen the app shows whenever no ears are connected; the controls are reachable only past it.
_Avoid_: scan screen, home screen

**Lighting**:
What the ears' LEDs show: colours, mode, speed, and brightness.
_Avoid_: glow settings, pattern

**Lighting profile**:
A named lighting saved on the phone.
_Avoid_: preset, pattern, theme

**Working lighting**:
The lighting the Glow tab is editing. While connected it is what the ears show; it is **unsaved** when no lighting profile holds it.
It is **edited** when it is unsaved but started from a lighting profile, which it keeps a link to.

**Apply**:
To write a lighting to the ears.

**Built-in animation**:
One of the 8 animations built into the ears' firmware, always available.
_Avoid_: preset, default animation

**Stored animation**:
An animation saved in one of the ears' slots, made in the web app. Its slot is its identity; names may repeat.
_Avoid_: custom animation, saved animation

**Auto-animate**:
The ears playing a built-in animation on their own at intervals.
_Avoid_: idle mode, animation mode

**Servo calibration**:
The ears' four per-axis offsets that correct each ear's resting position. Stored on the ears, shared by every controller.
_Avoid_: trim, servo settings

**Axis**:
One direction an ear moves: side-to-side or up-down, on the left or right ear. The ears have four.
_Avoid_: servo, azimuth, latitude
