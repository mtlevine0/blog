---
title: "Building a Six-Tube IV-9 Numitron Clock - My Second Attempt at PCB Design"
date: 2026-08-08
draft: false
tags: ["hardware", "esp32", "kicad", "pcb", "numitron", "firmware", "clock", "display", "esoteric-display"]
---

A Numitron is what you get if you build a seven-segment display out of light bulbs. Each segment is a tungsten filament in an evacuated glass envelope, and it glows for the same reason a lamp does: you run current through it and it gets hot. No high voltage or gas discharge involved.

RCA introduced the Numitron in [1970](https://en.wikipedia.org/wiki/Numitron) as a low-voltage alternative to the [Nixie tube](https://en.wikipedia.org/wiki/Nixie_tube), which had ruled numeric displays since the 1950s on a cold-cathode neon discharge that needed on the order of 170V to strike — awkward to drive from the 5V TTL logic that was becoming standard by the late 1960s. A Numitron's seven filament segments ran directly off a few volts and needed [far fewer switching elements than a Nixie's ten cathodes](https://hackaday.com/2024/07/08/plight-of-the-lowly-numitron-tube/), making it the simpler thing to drive from digital logic at the time. That combination made it a popular stopgap through the 1970s — [everything from gas pumps to aircraft cockpit gauges](https://hackaday.com/2024/07/08/plight-of-the-lowly-numitron-tube/) used one — right up until cheap [7-segment LED displays](https://en.wikipedia.org/wiki/Seven-segment_display) erased its one remaining advantage: an LED could also run straight off logic-level voltage, at a fraction of the current and without an incandescent filament's short service life, so there was no longer a reason to reach for a light bulb.

The IV-9 is a Soviet-era version of that idea in a small side-view tube, and unlike a nixie it will happily run off the same low-voltage rail as the microcontroller sitting next to it. This style of display — a light bulb standing in for a modern display panel — is exactly the kind of esoteric display tech that I find interesting.  I plan to explor a few other types of display technologies including the nixie along with other gas discharge displays.

I also wanted the clock to be multi-functional, with an ambient temperature/humidity display alongside the time.

![The finished six-tube IV-9 Numitron clock on the bench, tubes lit](/images/numitron-clock/iv9-clock.JPG)

# EDA
### What's on the board

The design splits into a main sheet and a display sheet:

- **ESP32-S3-WROOM-1**, WiFi for NTP and weather, and enough GPIO to bit-bang everything else
- **Six chained TLC5916 constant-current shift registers** (U5–U10, one per tube) driving 48 segment channels
- **DS3231M RTC** with a coin cell, so the clock survives a power cut without a network
- **HDC2010** temperature/humidity sensor for the indoor readings
- **AZ1084-3.3 LDO**, USB-C input, and a USBLC6-2SC6 for ESD on the data lines
- **One addressable RGB LED** (Harvatek B3DK3BRG) as a status indicator, plus BOOT and SEL buttons

Fabricated at OSH Park on their standard 2-layer service: 1.6mm FR408, 1oz copper, ENIG finish, 6mil minimum trace and clearance.

<!-- IMAGE: bare PCB, top side -->
![KiCad PCB layout for the IV-9 Numitron Clock, showing the six tube footprints, driver ICs, and antenna keep-out zone](/images/numitron-clock/iv9-pcb.png)

I couldn't find an existing verified IV-9 KiCad footprint online, so I made my own from scratch: measured the tube with digital calipers and built the KiCad footprint pad-by-pad. It was my first time hand-building a footprint rather than pulling one from a library.  When the board came back from the fab, the tube leads aligned perfectly with the display face oriented in parallel to the board edge.

![Custom KiCad footprint for the IV-9 Numitron tube, drawn from digital caliper measurements](/images/numitron-clock/numitron-kicad-footprint.png)
{.compact}

The tubes are wired common-anode to +3.3V, with the TLC5916s sinking each segment. That's the detail that makes the constant-current driver worth using rather than a plain shift register and 48 resistors: the tube's filament resistance changes as it heats, and a current sink doesn't care. A single external resistor (R-EXT, 1k here) sets the current for all eight of a chip's outputs, about 19mA per segment with the high-current multiplier engaged.

### Assembling a two-sided board

This was my second attempt at [PCB design and fabrication](/posts/pcb-design/), and the board came out about 3x larger than my first one. Even at that size, space was tight enough that I had to place components on both sides of the board: a hot plate reflowed the top, where most of the components live, a hot air station handled the RTC on the bottom, and the backup battery got hand-soldered in afterward.  To my surprise I was able to one-shot the tiny temperature & humidity sensor, it responded immediately with no solder bridges.  Soldering the USB-C socket also went smoothly, there were bridges but I simply dragged some soldering wick across the leads with my soldering iron.  

![Bottom side of the board, showing the hand-soldered coin cell battery holder next to the hot-air-reflowed RTC](/images/numitron-clock/iv9-clock-bottom.jpg)

### The board did nothing

The first assembled board did nothing. No tube segments lit, and the status LED didn't either: the least useful failure mode available, because it's consistent with almost any hardware bug you can name. I went over the schematic by hand eventually concluding that my common anode displays were accidentally wired as common cathode.  This was easy enough to fix by rerouting the tubes common wire to +3v.  A manual schematic review also identified that the hasty last minute addition of the status LED had left its serial input unconnected to the MCU.  A simple bodge wire fixed this.

### Ambient sensors need their space

The HDC2010 comes in a package under 1.5mm², small enough to tuck into whatever gap is available without stopping to think about what's radiating heat nearby, which is exactly what I did. I placed it roughly 2 inches from the LDO regulator, which turned out to be insufficient spacing: the sensor reads ~90°F in a room sitting at ~70°F ambient. The main culprit is the ground plane covering the entire backside of the board — it acts as a heatsink for the LDO and carries that heat straight across the board to the sensor. Next revision, I'll keep the ground pour clear around temperature-sensitive components, and cut isolation slots to put the sensor on its own thermally isolated island.

### AI assisted EDA

The defects were fixable but frustrating.  I decided to try asking Claude Opus to review my (yet to be repaired) schematic for issues.  It came back with two critical issues along with some more trivial noise.

Both fixes are already in the board — the tube commons route to the +3.3V rail, and the status LED's data line lands on a real GPIO — and the KiCad layout shown earlier, under "What's on the board," is the revision that came back after that fix, not the one that came back unlit.

It turns out I'm hardly the only one attempting to apply AI to EDA.  I had previously come across ongoing [MCP integration](https://github.com/mixelpixx/Konnect) work with KiCad.  I tried using it but at least at the time (July '26) the MCP was only working on Windows and I'm working solely on Linux.  I recently came across a [Hacker News article](https://news.ycombinator.com/item?id=49569366) discussing [evaluations being done by EE Bench](https://eebench.org/blog/can-ai-design-circuit-boards-yet/).  Their evaluations have Claude in the lead with the conclusion that yes current LLM technology is already making effective contributions in the EDA space.  I have begun another display project where I'm taking an AI-first approach to its EDA component.

# Software

### Reading the tube's pinout by lighting one segment at a time

I took a [pin-swap-and-back-annotate](https://docs.kicad.org/10.0/en/pcbnew/pcbnew.html#back-annotation) approach to routing the tube segments: route the traces in whatever way was most convenient, then update the schematic's net assignments to match, rather than routing to match a net assignment I'd already fixed. That leaves the actual segment-to-pin mapping unknown until you check — so once the board came back, I wrote a small bring-up utility that lit one output at a time, noted which physical segment lit, and built the digit patterns in software from that table. The results:

```
  Bit (OUTn) | U5 pin | V1 pin | Segment
  -----------+--------+--------+--------
       0     |   5    |   9    |  DP
       1     |   6    |   8    |  B
       2     |   7    |   7    |  C
       3     |   8    |   6    |  A
       4     |   9    |   5    |  F
       5     |  10    |   4    |  G
       6     |  11    |   3    |  D
       7     |  12    |   2    |  E
```

### Brightness, why a linear scale looks broken

All six drivers share CLK, LE, and OE. That means there is no per-tube or per-segment brightness in hardware: anything you set applies to all 48 channels at once.

The obvious way to dim a display like this is PWM. The TLC5916 offers something better: a "Special Mode" in which you clock in an 8-bit Configuration Code that sets a digital current gain, independent of R-EXT.

Getting into that mode is a five-clock dance on OE and LE straight out of the datasheet's Figure 16, which was fun to bit-bang correctly the first time:

```c
void switchMode(bool toSpecial) {
  static const int oeSeq[5] = {1, 0, 1, 1, 1};
  int leSeq[5] = {0, 0, 0, toSpecial ? 1 : 0, 0};
  digitalWrite(PIN_SDI, LOW);
  for (int i = 0; i < 5; i++) {
    digitalWrite(PIN_OE, oeSeq[i]);
    digitalWrite(PIN_LE, leSeq[i]);
    digitalWrite(PIN_CLK, HIGH);
    digitalWrite(PIN_CLK, LOW);
  }
}
```

Then came the part I didn't anticipate. The setup portal offers brightness as a percentage, and the first implementation mapped that percentage straight onto the current gain. **At "50%" the tubes looked essentially off.**

Incandescent output is not linear in current. The rule of thumb for tungsten is that luminous output goes roughly as I^3.5, so halving the current doesn't halve the brightness: it takes it down by around a factor of ten. Cutting current to 50% lands you somewhere around 9% of the light.

The fix is to invert that curve, so the number in the portal tracks how bright the clock *looks* rather than what the driver is doing:

```c
#define BRIGHTNESS_GAMMA 3.5f

float actualCurrentFraction(int percent) {
  percent = constrain(percent, 1, 100);
  return powf(percent / 100.0f, 1.0f / BRIGHTNESS_GAMMA);
}
```

With that in place, the five offered settings (100/75/50/25/10% perceived) work out to roughly 100/92/82/67/52% of actual current. Every one of those stays at or above the TLC5916's documented "suitable" floor for the high-current mode, which a linear scale would have blown straight through at the low end.

### Time, and what's actually authoritative

The RTC is the display's source of truth, always. NTP only *corrects* the DS3231, periodically, when there's a network. The clock works completely offline on its coin cell, and everything renders from the last RTC read regardless of what the network is doing.

### Outdoor weather, a fallback for poor sensor placement

The HDC2010's indoor reading turned out effectively useless, skewed by the LDO's heat as covered above. But the board already carries a WiFi radio for NTP, and that same connectivity can just as easily pull real conditions from outside — half-salvaging the ambient-display feature even though the onboard sensor can't be trusted indoors.

Outdoor conditions come from two free, keyless HTTPS hops: `api.zippopotam.us` turns a US ZIP into lat/lon, and `api.open-meteo.com` turns lat/lon into current temperature and humidity. The geocoding result is cached in NVS keyed by the ZIP, so a reboot doesn't re-resolve it.

### The setup portal

Press BOOT during the boot window and the clock brings up a SoftAP and captive portal for entering WiFi credentials, timezone (as POSIX TZ strings, so DST transitions handle themselves), 12/24-hour format, °F/°C, ZIP code, brightness, and how often the display should interrupt itself to show a reading.

The page is built at request time rather than served from a PROGMEM literal, so it can reflect the settings currently in effect: SSID prefilled, saved timezone selected, current brightness checked. It costs a bit of RAM per request and it's worth it.

Every option is whitelist-validated against the values actually offered. That's not paranoia about attackers; it's that a hand-crafted POST could otherwise write a setting no menu can reach, and the only recovery on this board would be a reflash.

![Numitron Clock Setup captive portal on a phone, showing network, timezone, ZIP code, clock format, and temperature unit fields](/images/numitron-clock/iv9-clock-setup.png)
{.compact}

### What's next

I'm working on a few display related projects: a display using another soviet tube (IN-9) this time of the gas discharge variety.  Additionally I'm working on a large format LED video wall acting as a heads up display in my office.  I'm reverse engineering some of the proprietary software that drives commodity hardware I bought to power the display.