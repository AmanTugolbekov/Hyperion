# Hyperion

A smart drone that picks up, carries and releases lightweight objects, so that
small items can be moved without a person present.

Built at **Ala-Too International University**.

---

## The idea

Most consumer drones observe. Hyperion is meant to *act*: a gripping mechanism
lets the aircraft collect a small object, fly it somewhere, and put it down
again. The target uses are the ordinary ones that are tedious or awkward for a
person — delivering a small package across a site, retrieving something from a
roof or a ravine, moving parts between benches in a workshop.

The project is deliberately split into three working parts, each with its own
repository, so that the mobile app, the flight software and the physical
airframe can be developed and reviewed independently.

## Repositories

| Repository | Contents | Status |
|---|---|---|
| [Hyperion_drone_controller_app](https://github.com/AmanTugolbekov/Hyperion_drone_controller_app) | Flutter ground station: FPV video, sticks, telemetry | **Flying** — control and camera verified on hardware |
| [Hyperion_drone_firmware](https://github.com/AmanTugolbekov/Hyperion_drone_firmware) | Flight controller and gripper firmware | Planned — not started |
| [Hyperion_drone_hardware](https://github.com/AmanTugolbekov/Hyperion_drone_hardware) | Airframe, gripper, electronics, BOM | In progress — frame models published |
| **Hyperion** (this one) | System overview, protocol specification | — |

## How the parts fit together

```
┌──────────────────┐   Wi-Fi    ┌──────────────────┐
│  Ground station  │ ---------> │   Drone          │
│  (Android phone) │   UDP      │                  │
│                  │            │  flight control  │
│  sticks ─────────┼──commands─>│  gripper         │
│  video  <────────┼──RTSP──────┤  camera          │
│  telemetry <─────┼──UDP───────┤  sensors         │
└──────────────────┘            └──────────────────┘
```

The phone and the aircraft share no code. They agree only on a wire protocol,
which is written down in [docs/PROTOCOL.md](docs/PROTOCOL.md). That document is
the contract between the controller repository and the firmware repository: if
the firmware honours it, the existing app flies the aircraft unchanged.

## Current state

The ground station exists and works. It is being developed against a commercial
E88 quadcopter, which stands in for the Hyperion airframe while that airframe is
being built. This is deliberate: it keeps the app honest about real radio
conditions — dropped packets, a silent aircraft, a camera that streams sideways
— instead of against a simulator that always behaves.

The protocol that commercial drone speaks was recovered by reading its own
application, not guessed, and verified against live traffic. The write-up is in
[docs/PROTOCOL.md](docs/PROTOCOL.md); it doubles as a worked example of how to
approach an undocumented aircraft.

## What is still ahead

- Flight controller firmware, and the gripper servo control alongside it
- Mechanical design of the gripper and its mounting
- Telemetry decoding, so battery and altitude reach the pilot
- Autonomy: object detection and assisted positioning over a target

## License

Coursework at Ala-Too International University. Ask before reuse.
