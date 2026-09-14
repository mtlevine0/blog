---
title: "An Open-Source LED Video Wall from Cheap Commodity Hardware (Part 1: The Driver)"
date: 2026-09-08T09:00:00-04:00
draft: true
tags: ["hardware", "python", "led", "video-wall", "networking", "reverse-engineering", "claude", "display", "esoteric-display"]
---

![Finished Product](/images/led-wall/led-wall-hero.JPG)

This is part one of a series on a wall-mounted LED video wall I've been building: cheap commodity LED panel hardware, driven entirely by open-source software, aiming for output quality similar to commercially available video walls for a fraction of the cost. This post covers the initial build, hardware iterations, and the low-level driver, [`colorlightpy`](https://github.com/mtlevine0/colorlightpy), that pushes pixels to the panels.

# Hardware
### The initial prototype

I started playing with HUB75 LED panels about years ago, bit-banging the protocol directly from an Arduino. I later found an off-the-shelf HAT from Electrodragon for a Raspberry Pi and started using hzeller's popular [rpi-rgb-led-matrix](https://github.com/hzeller/rpi-rgb-led-matrix). That worked well for small displays, but I found 192×192 pixels to be about the ceiling of acceptable performance from that setup. By then I'd already acquired enough P3 64×64 LED modules for a 384×192 display, so I split the load across two Raspberry Pis and sent UDP frames to each. It worked, but far from optimally: frames were easily dropped, producing occasional artifacts on the display.

![Electrodragon RGB Matrix HAT for the Raspberry Pi, with three HUB75 IDC headers and an onboard RTC coin cell](/images/led-wall/electrodragon-raspberry-pi-panel-driver.png)

FPGAs seemed like an obvious next step with their ability to excel at bridging digital interfaces.  I came across commercial products that used FPGAs to convert Gigibit Ethernet-based video frames into HUB75 frames. The products came with zero english based support, but they were cheap enough (~$15) that I decided to try them.

### The panel and the receiver card

The panel itself consists of six chained 64×64 HUB75 "P3" LED panels, the same commodity indoor LED tile you'll find behind commercial video walls, wired as three parallel chains for a combined 384×192 resolution. HUB75 is a dumb parallel interface: no framebuffer or timing controller of its own, just rows of shift registers that need to be fed continuously by something upstream.

![Colorlight 5A-75B receiver card, with the eight HUB75 output headers driving the chained panels](/images/led-wall/Colorlight-5A-75B-LED-Receiving-Card.webp)

### Frame and power

The frame holding it all together is 3D-printed parts bolted to 2020 T-slot extruded aluminum.  A collection of LED modules held together by a frame is referred to as a cabinet in commercial LED walls.  The 2020 T-slot makes it easy to combine multiple cabinets together to build a larger display in the future.

![Back of the LED wall, showing the 3D-printed frame brackets on 2020 T-slot extrusion, the power bus bars, and wiring to the LED modules and driver cards](/images/led-wall/led-wall-back.JPG)

Power comes from a single 200W Meanwell 5V supply, delivered to two bus bars that distribute it out to all sixteen LED modules and both driver cards.  This power supply can reliably power the display up to about 80% full brightness on an all white screen.  I'll have to run additional bus wires to achieve 100% brightness with this setup.

### Replacing the vendor software

The card is meant to be driven by LEDVISION, the vendor's Windows app: point it at a video source or a static layout and it pushes pixels out over the card's Ethernet port.

![LEDVISION, the vendor's Windows software for driving Colorlight receiver cards](/images/led-wall/colorlight-ledvision.jpg)

LEDVISION works, but it's Windows-only and closed source, with no public protocol documentation and no changelog to speak of. There's nothing to file an issue against if it breaks. Getting the panel off of it entirely, so the wall runs on Linux with no proprietary software anywhere in the path, was the actual point of `colorlightpy`. Nobody involved in that supply chain publishes a spec for the wire protocol LEDVISION speaks to the card, so I leveraged Claude to reverse-engineer it: using Wireshark, I captured a video stream sent from LEDVISION to the Colorlight receivers and asked Claude to work out the protocol from the capture.

# Software
### Talking to the card

The 5A-75B doesn't speak IP. It listens for raw Ethernet Layer 2 frames on whatever link it's plugged into, addressed to a fixed destination MAC (`11:22:33:44:55:66`) that has nothing to do with the card's actual hardware address. There's no ARP or handshake, and no acknowledgment. You send frames, and if the EtherType and payload layout are exactly right, pixels change. If they're not, nothing happens and there's no error to read back. That absence of feedback made the whole process slow: every hypothesis about the protocol had to be tested by watching the panel, not by reading a response.

The protocol that fell out of that process, per row of the panel:

- **Row data**: EtherType `0x5500 | (row >> 8)` encodes the row's high byte right into the EtherType field, with the low byte, a column offset, a pixel count, and a 2-byte magic value (`0x0888`, purpose still unknown, just required) in the payload ahead of the packed RGB bytes.
- **Brightness**: EtherType `0x0A00 + brightness`, no payload needed; the value is encoded directly in the EtherType.
- **Sync**: EtherType `0x0107`, sent once per frame after all the row data, latches the whole pixel buffer to the display atomically so you never see a frame torn mid-scan.

A standard Ethernet MTU caps a single frame at about 493 pixels' worth of RGB data, so a 384-pixel-wide row fits in one packet but a wider chain has to be split across several packets, all reassembled correctly on the receive side, since it does none of the reassembling for you.

None of this needs `scapy` to construct. [`protocol.py`](https://github.com/mtlevine0/colorlightpy/blob/master/colorlight/protocol.py) builds the raw frame bytes with `struct.pack` and nothing else, specifically so it can be unit-tested and profiled without a live NIC or root privileges. Only the send path touches `scapy`, for the raw L2 socket itself, which does need root (or `CAP_NET_RAW`) on account of writing directly to the network device below the IP layer.

### Streaming architecture

The CLI has a couple of built-in test-pattern generators (vectorized NumPy, no per-pixel Python loops), useful for confirming the panel is wired up correctly before trusting it with anything real. But the mode that actually matters is `stream`: read raw RGB bytes from stdin (or a named pipe) and push them to the panel as fast as they arrive.

```bash
python main.py stream -i eth0 -W 384 -H 192 --fps 30 --pixel-format bgr
```

That's the entire interface. Feed it `width * height * 3` bytes per frame (no header or framing, just raw pixels) and it drives the panel. It doesn't care what produced those bytes. `ffmpeg` decoding a video file into `rawvideo` works:

```bash
ffmpeg -re -stream_loop -1 -i video.mp4 -f rawvideo -pix_fmt rgb24 -vf scale=384:192 - \
  | python main.py stream -i eth0 -W 384 -H 192
```

The one thing worth engineering into a live pipe is what happens when the upstream producer dies or stalls: an `ffmpeg` process crashing, a pipe closing mid-frame. Left alone, that's a frozen or corrupted image sitting on a wall-mounted display, which is a bad failure mode for something meant to look like a finished product. `stream` watches for exactly that: if stdin goes quiet past a configurable threshold, it falls back to a "no signal" test pattern instead of leaving stale pixels up, and switches back to the live stream the moment data resumes, without the driver process ever needing to restart. Small feature, but it's the difference between "occasionally glitches" and "just works," and it only showed up after living with the first version for a while and watching it fail.

### Surviving suspend and shutdown

Everything above assumes the host driving the panel stays up and the link stays alive. That's not true on a laptop, or on anything else with power management enabled, and running into that gap turned into its own piece of work: teaching `colorlightpy` to notice when Linux is about to suspend, and to recover cleanly once it wakes back up.

The same fire-and-forget quality of the Colorlight protocol that made the reverse-engineering slow also makes this harder than it sounds. A raw socket send can report success while the receiver stays completely unresponsive, because the NIC hasn't finished renegotiating its link speed yet. Some Realtek NICs come back from suspend at 100 Mbps half duplex instead of gigabit; packets go out fine, but the receiver just isn't listening at that speed. There's no way to ask the hardware whether a frame actually landed, so the driver has to assume it didn't and act accordingly.

Before suspend, `colorlightpy` now subscribes to systemd-logind over D-Bus and listens for its `PrepareForSleep` signal. On its own that signal buys no time: by the time the callback runs, the kernel could already be well into suspending. So the driver also acquires a logind delay inhibitor at startup, a file descriptor that logind waits on before it lets suspend proceed. When the sleep signal arrives, the driver stops ordinary stream frames from going out, waits briefly for a usable link, and sends an all-black frame three times a tenth of a second apart before finally releasing the inhibitor. The repetition and spacing both came from watching it fail in practice: a single send sometimes just doesn't reach the receiver, and retries sent too close together can all land in the same brief link outage.

Resume is trickier, since there's no reliable event to trigger recovery on. The process isn't running inside a desktop session that would normally surface a wake notification, so instead it compares two clocks on every frame: one that keeps ticking through suspend and one that doesn't. A gap between them means the host was asleep, and recovery kicks in on the next frame: reopen the raw socket, wait for carrier, force a gigabit renegotiation with `ethtool` if the link came back slow, then replay the current frame a few times to make sure it actually sticks.

Ordinary shutdown needed the same kind of care. Stopping the process with a plain `SIGTERM` used to land mid-frame, some rows sent and the rest cut off, leaving a torn image latched on the panel until something else happened to overwrite it. The driver now defers the signal until it's between frames, so a stop or restart always ends with either a complete frame or a deliberate black one, never a half-sent one. One case snuck through even after that fix: named-pipe mode can block indefinitely inside the system call that opens the pipe, waiting for a writer that may never show up, with no frame in flight to protect during that wait. A `systemctl restart` reproduced it exactly, the process sitting in `deactivating (stop-sigterm)` for the full 90-second systemd timeout before getting killed outright. That call site now gets its own narrow exception: an interrupt handler installed just around the blocking open, so it can still be killed immediately there without giving up the frame-boundary protection everywhere else.

### Where this leaves off

At this point there's a driver that will take any well-formed stream of raw RGB frames and put it on the wall reliably, degrading gracefully instead of freezing or tearing when the source misbehaves. That's necessary but not sufficient for a wall you'd actually want to look at day to day. It says nothing about *what* to show, or how to switch between things without SSHing in and restarting a process.

That's [GRID](https://github.com/mtlevine0/grid): an orchestrator that runs a handful of independent widgets (scoreboard, system monitor, weather, now-playing, NVR alerts), each as its own subprocess speaking `colorlightpy`'s exact frame contract, and a physical macro pad to switch between them. Two frames actually captured off the running wall, ahead of the full writeup:

![system monitor widget on the wall](/images/led-wall/sys-monitor-frame.png)
![scoreboard widget on the wall](/images/led-wall/scoreboard-frame.png)

More on how that's built in the next post.

# Appendix
### Hot-air rework

One of the modules turned out to have a bad blue-channel driver IC in one quadrant, a failure that's easy to diagnose (a dead color channel, localized to one block of pixels) and looks intimidating to fix if you've never done surface-mount rework before. It turned out to be a good excuse to finally learn hot-air rework: desoldering and replacing a leadless IC with a hot-air gun instead of an iron. The process is a lot more approachable than it looks, and the hobbyist-level tooling (a decent hot-air station, some flux, solder paste) is cheap enough that there's no real excuse not to have it on hand.

![Hot-air rework on a HUB75 LED module PCB, reflowing the driver IC identified as the source of a bad blue channel in one quadrant](/images/led-wall/led-module-driver-repair.jpg)
