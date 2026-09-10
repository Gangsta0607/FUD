# FUD — technical documentation

How both apps work inside, and every non-obvious thing learned the hard way.
For the wire contract see `docs/PROTOCOL.md`; for users see `README.md`.

## Big picture

Two processes:

- **Server** (`FUD/`, macOS, SwiftUI menu-bar app): discovers client devices
  over mDNS, dials them, owns pairing, creates/owns a virtual display per
  device, captures it with ScreenCaptureKit, encodes with VideoToolbox, and
  pushes H.264/HEVC over UDP.
- **Client** (`FUDClient/`, iOS, one fat armv7+arm64 binary): announces
  itself, listens for the server's control connection, decodes and renders
  full-screen. The same process is also an AirPlay receiver (uxplay-based),
  so a Mac can mirror to the device the usual way. One protocol at a time:
  whichever starts streaming first stops the other's discoverability.

Roles are deliberately reversed (client listens, server dials) — the
AirPlay shape: the user sits at the Mac, everything is driven from there.

## Server workflow

**Startup** (`ServerController.shared`, created on first UI access, start
delayed 0.5 s): a `NetworkServer` (UDP listener on 7879 for video-plane
hello/stats; there is no TCP listener — v4 dials out only) and a
`ClientDiscovery` (NWBrowser on `_fud._tcp`).

**Discovery**: NWBrowser delivers service results; the TXT record is read
with a direct `DNSServiceQueryRecord` (`DNSTXTQuery.c`) because macOS hands
no browse-time TXT to third parties. The TXT carries `uuid`/`v`/`port`/`ip`/
`usb`. The browser's state is watched: a failed NWBrowser is rebuilt
(previously it died silently and discovery was dead until an app restart).

**Connect** (`connect(toAnnounced:)`): the card is pending from the first
tap (published `pendingAnnounced`), a second tap is a no-op. The dial plan
is the client's self-reported `ip=` list first (fresh — the client
re-publishes on interface change), topped up by `DNSAddrsForService`
(resolves every A record; early exit ~300 ms after the first answer). One
`NWConnection` per address — USB link-local and Wi-Fi are dialled together;
a per-path failure is normal and silent. Each link gets a 6 s "stuck in
.waiting" timeout; the whole attempt gets an 8 s verdict: nothing ready →
one silent redial; links ready but never answered the hello (a wedged
client) → dropped and redialled; only then an error is shown.

**Pairing**: `hello` carries the stored token (per client `uuid`). The
client answers `auth ok` (fast path), `auth pin` (PIN on the device's
screen, typed into the server card), or `auth fail`. A stored token answered
with `pin` is wiped — the client just said it is invalid.

**Streaming** (`ClientSession` + `StreamingPipeline`): on `auth ok` the
first link claims the device's pipeline through `DeviceCoordinator` — an
actor with an ordered event queue (claim/depart/cancelGrace/graceExpired),
because two sibling links both deciding "no pipeline yet" was a real race.
`beginPipeline` resolves the display provider (BetterDisplay or the private
CoreGraphics virtual display), ensures the virtual display, then starts
capture+encode. Sibling links get their own `config` and join silently;
if the owner dies, `depart` hands the pipeline to a survivor.

**Display lifecycle**: a virtual display is torn down 5 s after the last
link drops (grace — clients reconnect fast, and a rebuild gives the display
a new CGDirectDisplayID and scatters windows), immediately on a deliberate
disconnect, and synchronously at app termination (`NSApp.terminate` is what
runs `applicationWillTerminate`; killing the process any other way orphans
displays on the Mac). Known devices' displays are also swept when nothing
is connected — that mops up screens left by a crashed previous run.

**Settings**: bitrate is live-tunable; codec/fps/scale changes restart the
capture pipeline (a `restarting` message tells the client to suppress stall
detection for the gap). `updateSettingsAndRecreateDisplay` reshapes the
virtual display in place where possible, so macOS does not re-lay out
desktops.

**Auto-reconnect**: a device that paired during this server run and whose
announcement reappears (screen unlock, roam) is re-dialled automatically —
a drop without a reconnect is where "I have to press Connect again" came
from. A deliberate disconnect or device removal is never re-dialled.

**Stats → rate control**: the client's once-a-second stats packet splits
"the link is congested" (sharp −30 %) from "the device is behind" (lazy
−5 % steps, floored at a quarter of the ceiling). See `docs/PROTOCOL.md` §5
for the field semantics.

## Client workflow

**Startup** (`AppDelegate` → `AirPlayViewController.viewDidLoad`): the log
file is truncated, the AirPlay state machine starts, the FUD side starts
(control listener on 7878 + announcer), and lifecycle observers are
registered.

**Announcer** (`ClientAnnouncer`): publishes `_fud._tcp` with the TXT
identity + the device's own IPv4 list + `usb` flag; re-publishes whenever
the interface set changes (10 s watch) and whenever the app returns from
the background (NSNetService announcements do not survive suspension).

**Session** (`StreamingSession`): the server dials in →
`attachAcceptedTransport` (a redial from the same IP closes the replaced
transport). The session waits for the server's `hello`, sends `device`,
then runs auth in reverse (we show the PIN). Every link reports its
`config`; when all pending links have reported (or a 2 s timeout fires),
the UDP target is chosen by priority — link-local first — and `UDP_HELLO`
goes out. Heartbeats: the client pings every 5 s; a link with no `pong` for
6 s is dead and removed entirely (a dead stub used to block re-adding that
IP forever). A removed link also leaves the pending-CONFIG set, so it can
never wedge the collection.

**Decode/render**: `UDPDataTransport` → `FragmentAssembler` (one lost
fragment loses the frame → keyframe request, rate-limited to 200 ms) →
`VideoDecoder` (AVC/HEVC subclass of a shared base: Annex-B scan, one
sample buffer per frame, async VT decompression) → `FrameRenderer`
(`AVSampleBufferDisplayLayer` on iOS 8+, an OpenGL ES NV12 view below that).

**Stall detection**: no decoded frames for 2 s while the control link is
also silent = stalled (static screens are normal — their pongs stay fresh).
A frame stall with a stale pong on the current path fast-fails over to a
sibling. While frames are not flowing, `UDP_HELLO` is re-sent once a second
(heals a lost hello and keeps NAT warm).

**Lifecycle** (see "Suspension survival" below — the long version): on
`didEnterBackground` the whole network stack is torn down cleanly under a
background task; on `didBecomeActive` it is rebuilt: control listener, FUD
announcer, AirPlay server, decoder (fresh instance), display layer
(recreated unconditionally).

## The AirPlay receiver (uxplay)

`FUDClient/uxplay/` is the unmodified upstream tree, kept as a diff
baseline; everything compiled lives in `FUDClient/AirPlay/lib/`.

### Event channel (raop_event.c)

macOS 26 tears down the mirror audio stream seconds after SETUP when the
receiver advertises `eventPort=0`: current clients expect the reverse-HTTP
event channel a real Apple TV provides. `raop_event` listens on the
advertised port, speaks first with a server-initiated `updateInfo` plist on
accept, and re-sends it every 30 s. Ported from the FDH2/UxPlay issue #533
experiment.

### PIN mode

Access control is pair-setup-pin (SRP) against `raop->pin`, never the RTSP
digest: a client that passed pair-verify still hits the `passwd` callback at
SETUP, and answering with a password would 401 an already-paired Mac. The
onscreen PIN is fixed (`pin + 10000` marks it fixed so the stack does not
swap in a random one), lives in AuthManager (`fud_airPlayPIN`), and is
shared with FUD pairing — one switch («Требовать PIN», `fud_requirePIN`)
gates both.

### Suspension survival

iOS reclaims a suspended app's sockets, and screen lock *is* backgrounding.
Left alone, this produced on resume: a dead RTSP listener (one accept error
killed the httpd thread permanently), an mDNS responder spinning at 100 %
CPU on a dead fd, a FUD control listener refusing every dial, a
VideoToolbox session that malfunctioned (-12903) or hung `DecodeFrame`
forever, and a display layer sitting in a non-Failed status while showing
black.

The design now:

- `AirPlayViewController` observes `didEnterBackground`/`didBecomeActive`
  and tears down/rebuilds the whole network stack. The idle timer is
  disabled while streaming (a receiver is a display; manual lock still
  works).
- `AirPlayStateMachine.start` is "ensure running", not "restart": starting
  over a live server used to swap the `AirPlayServer` object out from under
  the C stack mid-teardown, and the old stack's queued conn callbacks then
  dereferenced the freed server — a main-thread `EXC_BAD_ACCESS`
  (symbolicated crash report exists). All RAOP conn callbacks resolve the
  server through the `weakActiveServer` slot instead of the raw `cls`
  pointer, because the C stack outlives the object.
- `httpd`: `EBADF`/`ENOTSOCK` from accept or select (a dead listen socket)
  exits the thread with a new `listener_died` callback after tearing down
  its connections; the state machine rebuilds the server. Transient accept
  errors (EMFILE & co) no longer kill anything. On the death path the dead
  fds are *not* closed — the OS already reclaimed them and the numbers may
  have been reused.
- `mdnsd`: a `select()` error exits the thread (a dead fd returns EBADF
  immediately and forever — that was the 100 % CPU spin).
- `TCPControlListener` (FUD :7878): the dispatch source's cancel handler
  owns the listen fd *by value*. Reading the ivar raced `_stop` clearing it
  and leaked the socket — after one background cycle every rebind failed
  EADDRINUSE forever.
- `VideoDecoder`: on a VT malfunction the session is rebuilt from the stored
  parameter sets (rate-limited), since AirPlay only sends SPS/PPS at stream
  start. `-12909` (missing reference) does *not* trigger a rebuild — that
  is a content gap which heals at the next keyframe, and rebuilding there
  loops forever (the rebuild resets the reference chain). A VT session whose
  create/decode spanned a freeze is cursed (DecodeFrame blocks on it
  forever), so on resume the AirPlay path gets a fresh decoder instance —
  `reset` on the same queue can never run, the queue is already wedged
  behind that call.
- `FrameRenderer`: the display layer is recreated unconditionally on resume
  (it can sit in a non-Failed status and render black forever), in addition
  to the existing throttled recreate-on-Failed.

### Empty codec packets on wired links

Current macOS emits h264-shaped codec packets with `payload_size 0`
(`header 01 00 16 01`) on wired links. They carry no information. Upstream
uxplay treats any empty codec packet as "client wants HEVC we did not
advertise" and tears the mirror thread down; we skip h264-shaped ones and
keep the unsupported-codec path only for genuinely h265-shaped ones
(`0x1e`/`0x5e`).

### Why AirPlay-over-USB was tried and removed

When the *sender's* link is Ethernet (and a USB tether looks like Ethernet
to macOS), current macOS sends HEVC instead of H.264 whenever the advertised
receiver height exceeds 1080 px — and if the receiver did not set the
`SupportsScreenMultiCodec` features bit 42, it withholds the VPS/SPS/PPS
config packet entirely, producing exactly the empty-packet + undecodable
stream described above (uxplay issues #233 and #231 map the behaviour:
"Both Ethernet → H265"). The codec is the sender's choice; there is no
receiver-side lever that forces H.264 at native 2048×1536 on a wired link,
and an A7 has no HEVC hardware (software HEVC at that resolution is a
slideshow). Wired low-latency display here belongs to the FUD protocol,
where the server encodes H.264 itself.

## Building the iOS client (the arclite trick)

Xcode 26 / clang 21 dropped `libarclite`, so linking an ARC app with
`-miphoneos-version-min` below 9.0 fails outright. The Makefile
(`FUDClient/Makefile`):

1. Links at 9.0 (`TARGET := iphone:clang:11.4:9.0`), the floor clang 21
   accepts.
2. Compiles each slice with its real per-arch minimum via CFLAGS and weak
   imports.
3. `tools/patch_min_version.py` then rewrites each slice's
   `LC_VERSION_MIN_IPHONEOS` back down (armv7 → 5.1, arm64 → 7.0).

One fat binary serves everything from an iPad 2 (iOS 5.1) to current
iPadOS: iOS 6's dyld skips the unknown arm64 slice and runs armv7, newer
devices run arm64. Each slice links its own libplist build
(`libplist.3.dylib` / `libplist-2.0.3.dylib`); both ship in `Frameworks/`
and resolve by install name. OpenSSL and fdk-aac are static.

Build caches go to `build/` (via `THEOS_BUILD_DIR` and
`_THEOS_RELATIVE_DATA_DIR`), final artifacts to `dist/`. `./fud clean`
removes them; nothing under either is source.

## deb vs ipa (why two packages for one binary)

The installd container sandbox on iOS 5–7 denies `iokit-open` on
`AppleVXD390UserClient`, so a containerized app gets
`VTDecompressionSessionCreate = -12913` and no hardware video decode. The
`.deb` installs into `/Applications` (jailbreak), outside the container —
the only way to hardware-decode on those systems. On iOS 8+ the sandbox no
longer blocks the decoder and either package works. The deb's postinst runs
`uicache` as mobile on fresh installs only.

## iOS 6 icons

SpringBoard on iOS 6 rounds corners and applies gloss only to App
Store-style container apps; `/Applications` apps are shown as-is. The
57/72/76 pt PNGs therefore have the rounded corners (with transparent, not
white, corners) baked in, and `UIPrerenderedIcon` is set.

## Version control

Fossil, not git. One repo per component (`fud-server.fossil`,
`fud-client.fossil`). A commit hook rejects messages containing `<`.
`.fossil-settings/ignore-glob` keeps build byproducts out of extras.

## Logging

- Server: `os_log`, subsystem `com.duckyouryan.FUD`. Info is not persisted;
  use a live `log stream --predicate 'subsystem == "com.duckyouryan.FUD"'
  --info`. Per-frame telemetry sits behind a `log.isEnabled` check so a 60
  fps stream costs nothing when nobody is listening.
- Client: `FLOG` writes `fud_debug.log` (`/var/mobile/` for the deb, the
  container's Documents for the ipa) and, once a FUD server has dialled in,
  POSTs each line to that Mac (`tools/log_collector.py`, port 30000).
  `./fud logs <ip>` fetches the fresher of the two on-device paths.

## Deploy tooling (`./fud`)

`./fud build|deploy|devices|logs|clean` — see the header comment. Deploy
notes: `deploy ipa` auto-picks the only connected device, otherwise asks,
and kills the running app first when the target is a jailbroken device with
known SSH (`build/last-client-ip`); `deploy deb` pipes the package over SSH
and kills the app before dpkg overwrites its files; `deploy server` pkills
the running menu-bar app before swapping `/Applications/FUD.app`.
