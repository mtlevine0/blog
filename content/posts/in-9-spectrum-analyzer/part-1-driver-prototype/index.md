---
title: "IN-9 Spectrum Analyzer, Part 1: Prototyping the Tube Driver"
cover:
    image: "breadboard-prototype-top-down.jpg"
    relative: true
    alt: Top-down view of the breadboard prototype driving three lit IN-9 tubes
date: 2026-09-11T00:04:00-04:00
draft: false
tags: ["hardware", "kicad", "pcb", "esp8266", "firmware", "python", "audio", "in-9", "display", "unconventional-displays", "claude", "product-design"]
description: "Validating a PWM-controlled current sink for IN-9 neon bar-graph tubes, and driving three of them from a PC as a 3-band spectrum display."
---

This is part one of a series about building an audio spectrum analyzer out of IN-9 neon bar-graph tubes. The end goal is a finished piece that sits on a shelf as functional art: a row of glowing neon columns dancing to whatever's playing. Before committing an enclosure and PCB fabrication, I wanted to verify my driver design. This post covers that feasibility work: a small analog driver board, firmware that turns serial commands into PWM, and a host script on the PC that turns system audio into a simple 3-band display.

# Background
### The IN-9
The IN-9 is a Soviet-era linear gas-discharge indicator: a long, thin glass tube filled with neon, with a straight anode wire running alongside a cathode. When the tube strikes, the cathode glows orange-red, and how far up the tube the glow reaches is set by how much current you push through it. More current, longer bar. It was built to be a bar-graph meter, so it's a natural fit for a spectrum analyzer, and it has the same warm neon glow as a Nixie tube. Elektor's [Nixie bargraph thermometer](https://www.elektormagazine.com/labs/nixie-bargraph-thermometer-with-arduino) project has a good introduction to the tube's construction and its quirks.

Like a Nixie, it needs high voltage to strike, so the system runs off a 150V boost supply. The difference is in how you control it. A Nixie is digital: a cathode is either lit or not. The IN-9 is analog: the bar length tracks the tube current, from a few milliamps to around 12mA for a full-length bar ([datasheet](https://xdevs.com/doc/Nixie_Tubes/russian_nixie_tubes/IN-9/in9.pdf)). So the driver isn't a switch, it's a programmable current source that can hold the tube at any point in between.

# The driver board
### From PWM to current
A microcontroller can't output an analog current, but it can output PWM. The driver converts that PWM into a regulated tube current in three stages:

1. **Filter.** A 10k / 100nF RC low-pass (R1, C1) averages the 3.3V PWM into a DC voltage proportional to duty cycle. Its corner is about 160Hz (a 1ms time constant), which smooths the 20kHz carrier into a steady level while still following the music.
2. **Op-amp.** An LM358 compares that voltage against the voltage at the bottom of the transistor and drives the transistor's base until they match.
3. **Current sink.** An MPSA42, a 300V NPN transistor, sits below the tube's cathode with a 270Ω resistor (R3) under its emitter.

![Original single-channel driver schematic: RC filter, LM358, and MPSA42 current sink under the tube](driver-v1-schematic.png "The original single-channel driver. PWM comes in on J1, the tube connects to J2.")

The trick is that the op-amp's feedback comes from the emitter, not from its own output. The op-amp forces the emitter voltage to equal the filtered input voltage, so the current through R3 is `Vin / 270Ω`. That same current flows up through the collector and through the tube, so 3.3V at 100% duty gives about 12.2mA, right around a full-length bar.

Putting the transistor inside the feedback loop is what makes this work. If the op-amp just drove the base open-loop, the transistor's ~0.65V base-emitter drop would eat the bottom fifth of the 3.3V range, and the bar would drift as the transistor warmed up. With the transistor inside the loop, the op-amp quietly adjusts for both, and the tube current tracks the input.

### The board
I laid it out as a tiny single-channel board in KiCad: an SOIC LM358, the TO-92 MPSA42, a handful of 0805 passives, a 3-pin header for power, PWM, and GND, and a header for the tube. The +150V and tube nets carry 150V, so they got their own netclass and a custom DRC rule enforcing 0.4mm of clearance from everything else.

![3D render of the single-channel IN-9 driver board](driver-v1-pcb-render.png "First single-channel layout, rendered in KiCad. By this point the op-amp ran from 12V and the op-amp input had a 100k pulldown (R4).")
{.half}

For the prototype, three of these boards went onto a breadboard alongside the MCU and a small 150V boost converter module, one board per tube.

![Three single-channel driver boards on a breadboard with an MCU and 150V boost module, driving three lit IN-9 tubes](breadboard-driver-boards-closeup.jpg "The breadboard prototype: boost module on the left, MCU at the bottom, and three driver boards each wired to a tube.")

# Firmware
### PWM over serial
The firmware is simple. It accepts a line of text with one percentage per channel (`42.5 60 80`), converts each to a 10-bit duty cycle, writes it to a 20kHz PWM pin, and replies with an acknowledgement.

It started with one channel on D0, grew to two (D0, D1), and then to three (D0, D1, D2) for the 3-band display. The serial protocol stayed the same throughout:

```text
> 25 50 75
< PWM set to 25.0 50.0 75.0.
> flicker 85 50
< Flicker on (85.0 Hz, 50% on).
```

The host waits for each acknowledgement before sending the next command, so the board never gets flooded and a missed reply shows up as a timeout instead of a silently dropped frame.

### Flicker
The firmware also gates all of the PWM outputs on and off at 85Hz, with a 50% on-time by default. Both the rate and the on-time can be changed over serial with `flicker <Hz> [on%]`, or turned off entirely.

IN-9s were [often driven with half-wave rectified current](https://www.elektormagazine.com/labs/nixie-bargraph-thermometer-with-arduino) rather than DC, so the tube saw pulses at mains frequency rather than a steady current. Gating the drive at 85Hz reproduces that pulsed operation.  Introducing the flicker helps ensure the tubes glow starts at its base and ends at the point as dictated by the current.  I found without it the glow is more prone to "jumping" where only a portion of the filament is glowing.

# Host software
### From audio to bars
The PC side is a single Python script, `pwm_test.py`. It started as a test harness (sweeps, sawtooths, and fixed levels to check that each tube responds linearly across its range) and grew real modes once that worked.

It captures system audio with `parec` from the monitor of the default PulseAudio/PipeWire sink, so whatever is playing on the PC drives the tubes with no extra cabling. Two meter modes sit on top of that:

- **`vu`**: a classic level meter. Left channel to tube A, right to B, the mix to C.
- **`eq`**: the 3-band spectrum display.

```bash
./pwm_test.py -p /dev/ttyUSB0 eq
```

### Three bands
For `eq`, the script mixes left and right to mono and runs an FFT every 10ms over a sliding 2048-sample (46ms) Hann window. The FFT bins are summed into log-spaced bands, each covering the same number of octaves, so pink noise (roughly the average tilt of music) reads the same in every band. With the default 50Hz to 16kHz span and three bands, that works out to about:

| Tube | Band |
| --- | --- |
| A | 43 – 345 Hz (bass) |
| B | 345 Hz – 2.3 kHz (mids) |
| C | 2.3 – 16 kHz (highs) |

Each band's power is smoothed with separate attack and release time constants (30ms up, 150ms down), so bars jump up on a transient and fall back gracefully instead of jittering. Levels are converted to dBFS and mapped onto bar length through an auto-ranging window. The window snaps out to new extremes instantly and relaxes back over a few seconds, so the display stays lively whether the music is quiet or loud. By default the three bands share one window, which preserves their relative heights so you can actually see the shape of the spectrum. `--per-band` gives each band its own window if you'd rather have every tube fill its range.

```mermaid
flowchart TB
    subgraph pc["PC"]
        a["System audio<br/><small>parec monitor</small>"] --> f["FFT → 3 bands<br/><small>smoothing · auto-range</small>"]
    end
    f -- "USB serial<br/><small>A% B% C% @ 50 Hz</small>" --> m["MCU<br/><small>20 kHz PWM · flicker gate</small>"]
    m -- PWM --> d["Driver ×3<br/><small>RC · op-amp · MPSA42</small>"]
    d --> t["IN-9 ×3"]
    hv["150 V boost"] --> t
```

### Calibrating the tubes
One thing that only showed up with real tubes is that they don't all reach the top at 100% PWM, and they don't all stop at the same place. To compensate, the script takes a `--tube-max` per channel (for example `--tube-max 80,75,90`) describing how far each tube actually lights at full PWM. Every value the script sends is a bar length rather than a duty cycle, rescaled so that "50%" really means half of what that tube can show.

Here's the whole chain running: system audio on the PC, through the `eq` mode, out over serial to the three tubes.

{{< video src="three-band-demo.mp4" poster="three-band-demo-poster.jpg" class="compact" caption="The 3-band display running to music. Left to right: bass, mids, highs." >}}

# What's next
The prototype answered the question it was built for. A PWM pin, an RC filter, and an op-amp current sink give smooth, stable, linear control of an IN-9's bar length, and a PC can feed three of them a live spectrum over serial at 50 updates a second.

It also showed where three tubes stop scaling. The ESP8266 has about seven clean PWM pins and generates PWM in software, so ten tubes at 20kHz is beyond it. Ten tubes at 12mA each also draw about 120mA from the 150V rail, around 18W, which is more than many small boost modules are rated for. Splitting the load so each boost module drives just two tubes brings that down to about 24mA, or 3.6W per module. Part two covers the first step toward ten: a dual-channel driver board designed almost entirely by AI. Later parts cover scaling the PWM side up to ten bands and designing a physical assembly to hold the tubes.
