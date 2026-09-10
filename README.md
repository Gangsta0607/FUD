# FUD

Turn an iPhone or iPad into an extra display for your Mac. Two ways in one
app:

- **FUD mode** — a small menu-bar app on your Mac creates a virtual display for the
  device and streams it over the local network. Plug the device in
  over USB and the video automatically prefers the cable; unplug it and the
  stream hops to Wi-Fi in about a second, no restart.
- **AirPlay mode** — the same device shows up in your Mac's Screen
  Mirroring menu like an Apple TV. Nothing to configure.

Works on absurdly old hardware: the client runs from iOS 5.1 to current iPadOS, with hardware video decoding all the way down.
Tested on iPad 2 (iOS 6.1) and iPad mini 2 (iPadOS 12.5.8).

## What it looks like in daily use

- Open FUD on the device. It sits there showing its name.
- On the Mac, the menu-bar app lists every device it can see. Press
  «Подключить» / Connect, type the PIN shown on the device's screen (once
  per device — after that it just connects), and the display appears.
- Lock the device, walk away, come back — the stream picks itself back up.
- Per-device settings (resolution, codec, fps, bitrate) live in the
  device's settings window on the Mac; the Mac remembers them.
- For AirPlay, just pick the device in Control Center → Screen Mirroring.

## Installing

Grab the latest release:

- **The Mac server**: `FUD.app` — unzip anywhere, run. It lives in the menu
  bar. On first launch macOS asks for Local Network permission (needed to
  find devices). For virtual displays with arbitrary aspect ratios it can
  use [BetterDisplay](https://github.com/waydabber/BetterDisplay) if you
  have it; otherwise it falls back to its own virtual display.
- **The iOS client**, one binary, two packages:
  - `FUD.ipa` — for iOS 8 and later. Install with your favourite
    sideloading tool.
  - `com.duckyouryan.fud.deb` — for jailbroken devices, iOS 5 and later.
    This one is required on iOS 5–7: only a jailbreak install gets hardware
    video decode there (Apple's app container blocks it).

## Notes and fine print

- Everything happens on your local network. No accounts, no cloud, nothing
  leaves the LAN.
- Pairing is protected by a one-time PIN (no-auth pairing is also possible), after
  which the Mac holds a token.
- The AirPlay receiver speaks to current macOS (tested against macOS 26),
  including its newer expectations like the event channel.


## Building from source

Requirements: Xcode (26 known-good) for the server; for the client,
[Theos](https://theos.dev) with `iPhoneOS11.4.sdk`, `ldid`, and
`libimobiledevice` + `ideviceinstaller` for device installs.

Then:

```
./fud build  ipa|deb|server|client|all [--release]
./fud deploy ipa|deb|server|client|all
./fud devices          # list connected iOS devices
./fud logs <ip>        # fetch the debug log from a jailbroken client
./fud clean [--all]    # build/, plus dist/ with --all
```

`deploy` builds first if the artifact is missing. `deploy ipa` auto-picks
the only connected device; `deploy deb` pipes the package over SSH and
remembers the address for next time; `deploy server` installs to
`/Applications`.

## Documentation

- `docs/PROTOCOL.md` — the wire protocol (v4). Normative; both sides
  conform to it.
- `docs/TECHNICAL.md` — how both apps work inside: workflows, the uxplay
  delta, the suspension-survival design, the build tricks, and the rest of
  the hard-won knowledge.

Parts of https://github.com/fdh2/uxplay were used for that project.
