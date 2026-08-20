# A working example

This is the whole house in one compose file — the real setup, with addresses and
passwords replaced. Five rooms, two transports, one place to press play.

```bash
cp .env.example .env      # set HOST_IP, PUID and a password
docker compose up -d
```

Then open Music Assistant at `http://<host>:8095` and the Bluetooth panel at
`http://<host>:8088`.

## What is in it

| Service | Role | Reached over |
| :-- | :-- | :-- |
| `music-assistant-server` | the brain: library, streaming, and the servers the players connect to | host network |
| `squeezelite-dx5` | a USB DAC, bit-perfect | Squeezelite (SlimProto, 3483) |
| `squeezelite-dx3` | a second USB DAC | Squeezelite |
| `bluetooth-web` | every Bluetooth speaker, paired from a browser | Snapcast (1704) |
| `ledfx` | WLED strips reacting to the music | both, feeding LedFx |
| `bubbleupnpserver` | optional: makes it all visible to UPnP/DLNA devices | host network |

## Why two transports

**Squeezelite for wired DACs.** It hands the audio over untouched, so the DAC
follows the track's own rate — 44.1, 96, 192 kHz, 16- or 24-bit — instead of
everything being resampled to one format. That is the whole point of owning a
good DAC.

**Snapcast for Bluetooth and LedFx.** Snapcast's job is keeping rooms on one
clock. Bluetooth is not bit-perfect anyway (A2DP re-encodes), so sync is what
matters there, and LedFx only needs to hear the music.

Both are Music Assistant providers, so every player shows up in the same list.

## Before it works

**The host must be a working PipeWire machine.** Every player here attaches to
the host's audio session rather than running its own daemon. That setup —
installing PipeWire, enabling lingering so it survives logout, and checking the
socket exists — is written out once in
[pipewire-snapclient](https://github.com/shuricksumy/pipewire-snapclient#-host-setup-preparation)
and applies to all of them.

**Find your own node names.** `PIPEWIRE_NODE` values here are examples. On your
host:

```bash
pw-cli ls Node | grep -E 'node.name|node.description'
```

**Check your uid.** `PUID` must own `/run/user/<uid>/pipewire-0`:

```bash
id -u
```

**Give each Squeezelite player its own MAC.** It is how the server tells them
apart; two players sharing one will fight over a single slot. The values here
start with `02:`, which marks them locally administered — safe to invent.

## Three things that will bite you

**One player per output device.** Each service above owns exactly one sink, and
that is deliberate. If you copy a block to add a room, change `PIPEWIRE_NODE` as
well as the name — two players pointed at the same device do not share it, they
fight over it, and the symptom is distorted or silent audio with nothing useful
in either log.


**On Ubuntu, keep `security_opt: [apparmor=unconfined]` on `bluetooth-web`.**
Docker's default AppArmor profile grants no D-Bus rules, so the container is
refused before it can talk to `bluetoothd`, and the error it produces is not
obvious. Debian and Alpine hosts do not need it.

**Bluetooth rooms arrive late.** A2DP adds roughly 150–250 ms. Playing alongside
a wired room, set a negative latency offset on the Bluetooth player in the panel
and tune by ear until they line up.
