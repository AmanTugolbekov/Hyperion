# Radio protocol

This is the contract between the ground station and the aircraft. The
controller app implements the client side; firmware must implement the server
side for the app to fly it unchanged.

Everything below was verified against real hardware — a commercial E88
quadcopter — unless a line says otherwise.

---

## 1. Link

The aircraft is a Wi-Fi access point. The phone joins it and is handed an
address by DHCP; the aircraft is the gateway.

| | |
|---|---|
| Aircraft address | `192.168.1.1` (read the gateway rather than assuming) |
| Control | UDP, port **7099** |
| Camera | RTSP, TCP port **7070**, path `/webcam` |
| Stills on board | HTTP `http://192.168.1.1/PHOTO/O/`, thumbnails `/PHOTO/T/` |
| File access | FTP, user `ftp`, password `ftp` |

Two behaviours matter for anyone writing a client:

**The aircraft never answers ICMP.** It does reply to ARP, so a ping timing out
says nothing about whether it is alive.

**Closed ports are silently dropped, not refused.** No RST comes back, so a TCP
connect attempt hangs until it times out. A port scanner that treats a timeout
as "closed" will report every port closed, including the ones that work. Ask
`/proc/net/tcp` what the official app actually connects to instead.

## 2. Control channel

A control frame is sent every **40 ms (25 Hz)**, continuously, whether or not
the pilot is touching anything. The aircraft stops responding if the stream
stops — it is a dead-man switch, not a command queue.

### Frame, 9 bytes

```
offset  0     1     2     3        4     5      6   7
       0x03  0x66  roll  pitch  throttle  yaw  flags  CS  0x99
```

| Field | Range | Neutral |
|---|---|---|
| roll, pitch, yaw | 1–255 | 128 |
| throttle | 1–255 | 128 — the aircraft holds altitude |
| flags | bit field, below | 0 |

`CS` is the XOR of bytes 2 through 6 — roll, pitch, throttle, yaw and flags.

### Command flags

| Bit | Command |
|---|---|
| `0x01` | Take off |
| `0x02` | Land |
| `0x04` | Emergency stop — cuts the motors immediately |
| `0x08` | 360° flip |
| `0x10` | Headless mode |
| `0x20` | Return to home |
| `0x80` | Gyro calibration |

A single command packet is routinely lost. Commands are therefore carried on
**ten consecutive control frames** rather than sent once. The receiving side
must treat a repeated flag as one command, not ten.

### Other packets

| Packet | Meaning |
|---|---|
| `01 01` | Heartbeat, once per second |
| `06 01` / `06 02` | Select front / bottom camera |
| `08 01` | Stop flight |
| `09 01` / `09 02` | Acknowledge a photo / video button press on the physical remote |

## 3. Board variants

Hardware sold under one model name is not one design. Three dialects are known;
the client detects which it is talking to rather than assuming.

| Variant | Port | Frame | Identified by |
|---|---|---|---|
| **Legacy** | UDP 7099 | 9 bytes, as above | Replies with a status packet |
| **GL** | UDP 7099 | 21 bytes, `03 66 14 …` | Device id in telemetry byte 0: 82, 85, 88, 103, or 90–101 |
| **WIFI_8K** | UDP 8090 | 8 bytes, no `0x03` prefix, checksum excludes flags | Never replies; needs the mode switch below |

The reference aircraft advertises the Wi-Fi network **`WIFI-UFO-8c4225`**.
Note that the name says "UFO" and nothing about E88 — a client matching network
names must not read "UFO" as the generic dialect, or it will drive this board
with the wrong frame format and the aircraft will ignore it entirely.

The reference aircraft used for this document reports device id **0x54 (84)**,
which is not in the GL list, so it runs the Legacy dialect on UDP 7099. Its
replies are described in section 7.

**WIFI_8K boards boot bound to their physical remote** and discard Wi-Fi
commands until told otherwise. Send `42 76` to UDP **8080** to take control, and
`42 77` to release it. Without this the aircraft looks like it is ignoring the
pilot entirely.

Hyperion firmware should implement the Legacy frame. It is the simplest, it
answers, and the controller app treats it as the default.

## 4. Camera

RTSP at `rtsp://192.168.1.1:7070/webcam`, MJPEG carried over RTP. The RTSP
session runs over TCP 7070; media arrives on a pair of UDP ports the client
chooses during SETUP.

The stream reports **240×320 — portrait** — because the camera is mounted
sideways. Clients should rotate based on the reported dimensions rather than
hard-coding an angle, so that a correctly mounted camera is not rotated wrongly.

Resolution is the camera's real output, not a client-side limit.

**Recording needs a client-side muxer.** The libmpv build shipped by media_kit
for Android carries no muxers at all — asking it to record answers
`recorder: Output format not found` and disables itself, whatever the file
extension. A client that wants clips has to assemble them itself; pulling a
JPEG frame from the player on a timer and writing MJPEG/AVI works, and costs
no re-encoding because the frames are already JPEG.

## 5. Platform traps

Two problems that look like protocol faults but are not.

**Android routes the traffic away.** The aircraft's access point offers no
internet, so Android marks the network unusable and sends every packet over
mobile data instead. Nothing reaches the drone, and nothing reports an error.
The client must pin its sockets with `ConnectivityManager.bindProcessToNetwork`.
The network request must *not* ask for `NET_CAPABILITY_INTERNET`, or it will
never match.

**The throttle stick means different things on different aircraft.** Where the
aircraft holds its own altitude, centre is hover and the stick should spring
back. Where throttle is raw motor power, the stick must stay where it is left
and start at the bottom — otherwise a centred stick means half power the moment
the motors arm.

## 6. How this was determined

Published reverse-engineering notes for this drone family contradict each other
on both the control port and the video transport, and their authors state they
never confirmed against hardware. Two implementations built from those notes did
not work.

What settled it was reading the manufacturer's own application:

```bash
adb shell pm path com.cooingdv.rcufo
adb pull <base.apk>
apkanalyzer dex code --class com.cooingdv.rcufo.socket.Config base.apk
```

The constants were in plain sight:

```
SERVER_IP             = "192.168.1.1"
PREVIEW_ADDRESS       = "rtsp://192.168.1.1:7070/webcam"
SEND_COMMAND_INTERVAL = 40
UdpComm.getInstance("192.168.1.1", 7099)
```

Then cross-checked against reality: while the official app was connected,
`/proc/net/tcp` showed an established session to `192.168.1.1:7070` and video
arriving over RTP on a pair of UDP ports.

**The lesson worth keeping:** when a device's protocol is undocumented, read the
software that already speaks it before trusting second-hand descriptions.

## 7. What the aircraft reports back

The Legacy board does answer, contrary to what a naive probe suggests. Over one
session it sent 422 replies, **every one of them identical**:

```
54 00 00 00 00        5 bytes
```

Byte 0 is the device id (0x54). Bytes 1–4 were zero throughout, with the
battery around half charge — so none of them carries a charge level.

**This board therefore has no telemetry.** The official app's parser reads its
battery and altitude fields from offsets 5 and 8, and only enters that branch
when the packet is at least 15 bytes long. A 5-byte reply never reaches it. The
stock app cannot show a charge level on this hardware either; the data is not
sent.

Treat the reply as a liveness signal and nothing more. A client should show no
battery indicator rather than a fabricated one.

**For Hyperion's own firmware this is a gap worth closing.** Sending real
battery and altitude in a longer packet costs nothing and makes the aircraft
safer to fly — a pilot who cannot see the charge finds out it was low by
watching the aircraft drop.

## 8. Not yet designed

- **Gripper commands.** Hyperion-specific. They should extend the flags byte
  rather than introduce a second channel.
- **Extended telemetry.** See above: the format is Hyperion's to define, since
  no donor board provides one worth copying.
