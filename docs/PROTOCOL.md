# The FUD Protocol

Protocol version: **4**.

This document is the contract between the server (macOS, `FUD/`) and the
client (iOS, `FUDClient/`). All implementations must conform; when code and
this document disagree, one of them gets fixed — never silently one side.

## Overview

The roles are reversed from the naive layout: **the client announces itself
and listens; the server discovers it and dials in** — the AirPlay shape,
where the receiver shows up in a list and the sender picks it. The user sits
at the Mac; the device may be in another room, so all setup (choosing the
device, pairing, start/stop) happens on the server.

Unchanged across versions: video flows server → client over UDP, stats flow
client → server, messages on the control channel are one JSON object per
line with `t` naming the type. Unknown keys and unknown types are ignored —
that is what lets one side grow a field without updating the other in step.

### Channels

| Channel | Transport | Port | Who listens |
|---|---|---|---|
| Discovery | mDNS (`_fud._tcp`) | — | both sides |
| Control | TCP | 7878 | **client** |
| Video | UDP | 7879 | server |
| Stats | UDP (same socket) | 7879 | server |

## 1. Discovery (mDNS)

The client publishes itself via Bonjour: type `_fud._tcp`, name = device
name, port = the control port. The TXT record:

```
uuid=8C7F...
v=4
port=7878
ip=169.254.241.81,192.168.43.56
usb=1
```

- `uuid` — permanent device identity; the server stores pairing tokens
  under it.
- `ip` — the client's own IPv4 addresses, comma-separated, link-local
  (169.254/16, the USB cable) first. The client re-publishes whenever its
  interface set changes, so this is fresher than mDNS A records (the iOS
  link-local A record flaps). The server dials these addresses directly;
  resolving the service endpoint only tops the list up. Old clients omit
  the field — then resolution is the only source of addresses.
- `usb=1` — the client has a link-local interface (a cable); a UI badge.
- If both sides come up empty-handed, the connection goes to the Bonjour
  **service endpoint** and the OS resolves addresses, link-local included.

Why the TXT query dance: macOS does not hand browse-time TXT records to
third-party apps (NWBrowser results carry `metadata=<none>`), so the server
fetches the TXT with a direct `DNSServiceQueryRecord`. The full device
description does not fit this channel anyway — it travels in a `device`
message over TCP (see §2).

## 2. Control (TCP 7878, server → client)

The user picks a device in the server UI (or the server auto-redials a known
one) → the server opens TCP to the client's control port and says:

```json
{"t":"hello","v":4,"name":"MacBook Pro","token":"a1b2..."}
```

- `token` — the token THIS client issued on a previous pairing; the server
  keeps tokens per client `uuid`. Absent on first connect.

The client answers with `device` first, then `auth`:

```json
{"t":"device","device":{"uuid":"8C7F...","name":"iPad 2","platform":"ios",
 "os":"12.5.8","model":"iPad4,5","screen":{...},"decode":{"avc":true,"hevc":false}}}
```

The `device` message delivers the §3 description (it replaced the device
object in the mDNS TXT — see §1). Its `uuid` repeats the TXT identity so a
manual connect-by-IP (typed by hand, no announcement) can still learn who it
is talking to: the server keys such a link under a `manual:ip` placeholder
and re-keys it to the real `uuid` from this message. Old clients omit
`uuid`; the server must survive that (such a link stays manual and always
pairs).

```json
{"t":"auth","status":"pin"}
{"t":"auth","status":"ok","token":"a1b2..."}
{"t":"auth","status":"fail","reason":"Invalid PIN"}
```

- `pin` — pairing required: the client shows a PIN on its screen.
- `ok` — the token was accepted (or the client does not require pairing);
  the reply's `token` is the client's current token, which the server stores
  under the `uuid` and sends in the next `hello`.
- `fail` — wrong PIN; the server wipes its stored token. A server that gets
  `pin` in answer to a stored token must wipe it: the client just said it is
  invalid.

**PIN**: the client generates and shows the PIN; the user types it into the
server's UI; the server sends:

```json
{"t":"pin","code":"4821"}
```

The client answers `auth ok` or `auth fail` and keeps the link up either
way — there can be several attempts.

After `auth ok` the server sends `config` — the parameters of the stream it
is about to push:

```json
{"t":"config","fps":60,"bitrate":10000000,"cap":15000000,
 "width":2048,"height":1536,"udpPort":7879,"codec":"hevc"}
```

| Field | Meaning |
|---|---|
| `fps` | target frame rate |
| `bitrate` | target bitrate, bits/s |
| `cap` | current network ceiling (DataRateLimits), bits/s |
| `width`, `height` | encoded frame size in pixels |
| `udpPort` | where to send `UDP_HELLO` and stats |
| `codec` | `"avc"` or `"hevc"` |

`config` arrives right after authentication and again on every change —
including when the rate controller moves the bitrate or the ceiling. The
client must survive a `codec` change on a live connection: a decoder is tied
to its codec, so a change means building a new one.

The client then opens its UDP socket and sends `UDP_HELLO:<uuid>` to the
server's `udpPort` — video and stats follow §4 and §5.

Remaining messages, in the direction of whoever initiates the action:
`keyframe` client → server (a frame was lost, an IDR is needed; not more
than once per 200 ms), `restarting` server → client (the capture pipeline is
about to be torn down and recreated — the client suppresses its stall
detection for the expected gap), `bye` both ways (orderly disconnect).
`ping`/`pong`: the CLIENT pings every 5 s, the server answers `pong`; a link
with no `pong` for 6 s is dead, and the client is the one who notices.

### Multi-link

One control link per client address, so USB and Wi-Fi are up at the same
time. Every link authenticates and gets its own `config`. The client picks
the UDP target by priority (link-local cable first) among alive links that
reported a config; when the streaming link dies, the best survivor takes
over; when the cable comes back, video moves to it. A redialled link from
the same address replaces the old one. On the server side only the first
link claims the capture pipeline; siblings join silently and inherit it if
the owner dies.

## 3. The device description

The `device` object from §2. The client reports everything it can measure
itself; the server guesses nothing and keeps no model table.

```json
{
  "uuid": "8C7F...",
  "name": "iPhone",
  "platform": "ios",
  "os": "18.5",
  "model": "iPhone17,5",
  "manufacturer": "Apple",
  "screen": {
    "pixelWidth": 2556, "pixelHeight": 1179,
    "pointWidth": 852, "pointHeight": 393,
    "uiPixelWidth": 2556, "uiPixelHeight": 1179,
    "scale": 3.0,
    "maxFps": 60,
    "safeArea": {"top": 0, "left": 59, "bottom": 21, "right": 59}
  },
  "decode": {"avc": true, "hevc": true}
}
```

| Field | Required | Meaning |
|---|---|---|
| `uuid` | no | permanent identity; only needed for manual connect-by-IP (an mDNS connect already knows it from the TXT). Old clients omit it |
| `name` | yes | the OS-reported name. On iOS 16+ without a special entitlement this is the model ("iPhone"), not the user-set name — so the server treats it as a hint and shows its own configured name (default: the Bonjour announcement name, then `model`) |
| `platform` | yes | `"ios"` |
| `os` | yes | OS version string |
| `model` | yes | iOS: `uname().machine` (`iPad4,4`) |
| `manufacturer` | no | absent on iOS |
| `screen` | yes | see below |
| `decode` | yes | which codecs the client can decode |

### `screen`

**All sizes are given in LANDSCAPE, long edge first** — the orientation the
client displays in.

| Field | Required | Meaning |
|---|---|---|
| `pixelWidth`, `pixelHeight` | yes | the physical panel in pixels |
| `pointWidth`, `pointHeight` | yes | the same in points (pixels / `scale`) |
| `uiPixelWidth`, `uiPixelHeight` | no | what the UI layer believes (`bounds * scale` on iOS). Differs from the physical panel in iOS compatibility mode, when an app is built without a launch storyboard. Diagnostic only: a mismatch with `pixelWidth` means the app is not rendering natively |
| `scale` | yes | density multiplier (2.0, 3.0, 2.75…) |
| `maxFps` | yes | panel maximum. When the OS cannot say (iOS < 10.3), report `60` |
| `safeArea` | no | insets in **points** to the area the app can really draw in: notch, rounded corners, home indicator, system bars. An absent field means "the client cannot know" (iOS < 11), which is not the same as all zeros |

### `decode`

```json
{"avc": true, "hevc": false}
```

"the OS can decode this at all" — hardware or software, deliberately not
distinguished: a slow software decode is the user's call to make, not a
reason to hide the option. On iOS: `avc` is always `true`; `hevc` is `true`
from iOS 11, where VideoToolbox's public HEVC decoder appeared.

## 4. Video (UDP 7879)

Binary; JSON does not belong here — this is tens of thousands of packets a
second.

The client announces itself on this port first:

```
UDP_HELLO:<uuid>
```

This packet tells the server where to send video, and it is also what moves
the stream when links change. It is single and unacknowledged, so the client
repeats it twice, 400 ms apart, on start, and then once a second whenever
frames are not flowing.

Every frame is cut into fragments of at most **1390 bytes** of payload.
Each fragment's header is 8 bytes, big-endian:

```
 0        4        6        8
 +--------+--------+--------+----------------+
 | frameId| fragIdx|fragTot | payload...     |
 +--------+--------+--------+----------------+
   uint32   uint16   uint16
```

- `frameId` — monotonous frame counter, wrapping modulo 2³².
- `fragIdx` — fragment number, from 0.
- `fragTot` — total fragments in this frame.

An assembled frame is 4 bytes of length (uint32 BE), then an Annex-B stream:
NAL units separated by `00 00 00 01` start codes. A keyframe carries the
parameter sets first (SPS/PPS for AVC, VPS/SPS/PPS for HEVC).

One lost fragment loses the whole frame: the client drops the incomplete
frame when the next one starts and asks for a `keyframe`.

## 5. Stats (UDP 7879, client → server)

Once a second, binary, 18 bytes, all multi-byte fields big-endian:

```
 0    1     2      4      6      8      10     12        16   17
 +----+-----+------+------+------+------+------+---------+----+----+
 |0x04| fps | loss | net  |decode|render|total |throughput|bklg|busy|
 +----+-----+------+------+------+------+------+---------+----+----+
   u8   u8    u16    u16    u16    u16    u16     u32      u8   u8
```

- `fps` — frames the client **decoded** this second.
- `loss` — share of frames that could not be assembled, × 10000. Not packet
  loss: one lost fragment kills the whole frame, so this reads far above the
  link's real loss, and it also absorbs frames lost because the device did
  not drain its socket in time.
- `net` — frame assembly time (first packet to complete), in tenths of a
  millisecond.
- `decode` — submit-to-callback time, tenths of a millisecond. This is
  pipeline **depth**, not per-frame cost: the decoder is asynchronous, and a
  delay above the frame interval is normal for a device that keeps up.
- `render`, `total` — same scale, tenths of a millisecond.
- `throughput` — bytes per second over the last second.
- `bklg` (backlog) — the decoder queue's deepest point this second: how many
  frames were inside VideoToolbox at once.
- `busy` — share of the second with at least one frame in the decoder, in
  percent.

The last two answer the question none of the others do: **is the device
keeping up**. `fps` only says how many frames were sent to it (a 30 fps
clip yields 30 against a target of 60, and that is fine), and `decode` is
the pipe's length, not its capacity. A growing backlog at ~100 % busy means
frames arrive faster than they are consumed, and neither statistics nor
asynchrony can fake that.

The server uses these numbers to drive the bitrate, telling two causes
apart:

- **the link** — assembly time stretched, or there is loss while the device
  keeps up. The reaction is sharp (−30 %), because a congestion collapse
  feeds on itself;
- **the device** — noticeably fewer frames decoded than sent, a growing
  backlog, a decoder busy almost the whole second. The reaction is lazy:
  only after several such seconds in a row, in 5 % steps, and never below a
  quarter of the ceiling — a slow chip will not get faster, and there is no
  point driving the picture to zero.
