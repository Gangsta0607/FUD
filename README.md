# FUD

Turn an iPad (iOS 5+) or iPhone (I mean you can if you want to) into an extra display for your Mac (or iPhone). It supports AirPlay (audio and video, not separately) and custom macOS 13+ server (for multiple devices simultaneously and also USB connection for iOS 9 and higher (maybe even 10 or higher, needs testing, for sure works on iPadOS 12)).
Was tested on iPad 2 (iOS 6.1 and 8.4.1 a little), iPad mini 2 (iPadOS 12.5.8), MacBook Air M2 (macOS 26), MacBook Air M2 (macOS 15).

## Installation
For iOS 5-7 install deb package (because VideoToolbox is private on these iOSes and app sandbox doesn't allow loading private libraries), for 8+ you can choose between deb and ipa (not signed, use AppSync or Sideloadly or whatever). If you want multiple devices at once or USB connection (for lower latency, for example) you need to install macOS app (just download ZIP and put FUD.app wherever you like).

## Using
Just open an app on your iDevice and connect to it via AirPlay or FUD server. FUD server has different settings (quality, scaling, fps and others), AirPlay is controlled by macOS (or iOS, you can connect any iDevice with AirPlay support). 

## Possible issues
- App icon on iOS has white corners, too lazy to deal with it
- Server doesn't check or wait until user actually allows local network scanning or screen recording, too lazy to deal with it and it is needed one time
- Might not appear in server app or AirPlay list or stop streaming after first frame or something like that - just reconnect or restart client app (or server). It was mostly fixed but still might happen (though it didn't happen during testing)
- Anything really (let me know)

## Building from source

Requirements: Xcode (26 known-good) for the server; for the client,
[Theos](https://theos.dev) with `iPhoneOS11.4.sdk` and `iPhoneOS6.1.sdk`, `ldid`, and
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

- `docs/PROTOCOL.md` — description of protocol used in a custom server.
- `docs/TECHNICAL.md` — how both apps work inside: workflows, the uxplay
  delta, the suspension-survival design, the build tricks, and the rest of
  the hard-won knowledge.

Parts of https://github.com/fdh2/uxplay were used for that project.
