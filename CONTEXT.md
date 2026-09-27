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
