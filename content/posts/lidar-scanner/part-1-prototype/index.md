---
title: "3D Lidar Scanner, Part 1: Low-Resolution Point Cloud"
cover:
    image: "main-floor-multiscan.png"
    relative: true
    alt: Colored point cloud of a house's main floor, merged from several scans and viewed from above with the ceiling cropped away
date: 2026-01-10
draft: false
tags: ["hardware", "lidar", "3d-scanning", "point-cloud", "python", "esp32", "stepper", "3d-printing"]
description: "A low-resolution prototype 3D room scanner: a surplus robot-vacuum lidar on a stepper-driven turntable, with color from a camera."
---

I've been fascinated by the "dollhouse" views commonly used on real estate listings. When I saw cheap 2D lidar scanners from robot vacuums for sale as surplus, I decided to build my own 3D scanner. This is part one of a series on building a 3D interior scanner from a 2D lidar.

This post covers the first prototype, built over a few evenings in January. It's deliberately simple: a cheap lidar on a 3D-printed turntable, scanning a room one slice at a time. The resolution is low, but it was enough to scan real rooms, color them from a camera, and show what later versions would need to fix.

# The hardware
The mechanical side is four 3D-printed parts:

- a **tripod adapter**, a cradle that holds a NEMA 17 stepper on top of a tripod,
- a **turntable**, a disc on the stepper's shaft with a grid of mounting holes,
- a **lidar mount**, a plate with four standoffs for the lidar and a flange that stands it on its side, so its scan plane is vertical,
- and a **camera mount**, an arm that holds the camera alongside the lidar.

![The January scanner assembled from its printed parts](january-scanner-assembly.png)
{.half}

The stepper is driven from an ESP32 through an ordinary STEP/DIR driver. The firmware is a small serial command interpreter: `ANGLE 42.5` moves to an absolute angle, `SPEED` sets the RPM, `HOME` zeroes the position, and every command ends with `OK` or `ERROR`. There's no homing switch, so 0° is wherever the platform was at power-up.

## The lidar: HLS-LFCD2
The lidar is an HLS-LFCD2, the unit sold as the LDS-01 on the TurtleBot3. It's cheap and well documented, and it's about as simple as a lidar protocol gets. You send `b` to start it and `e` to stop it. While it runs, it streams 42-byte frames at 230400 baud. Each frame starts with `0xFA`, then an index byte that says which six degrees it covers, then two bytes of motor speed, then six 6-byte readings, with a two-byte checksum at the end. The distance sits two bytes into each reading:

```python
base_angle = (frame[1] - 0xA0) * 6
for m in range(6):
    offset = 6 * (m + 1)
    distance = struct.unpack('<H', frame[offset:offset + 2])[0]  # mm
```

That's one reading per degree, 360 per revolution, at about 5 revolutions a second. A full slice is 60 frames.

# Scanning
The prototype scans with a step-and-settle loop, which is as direct as it sounds:

1. Move the platform to the next angle.
2. Wait a second for the vibration to settle.
3. Capture one full revolution from the lidar.
4. Rotate that slice into 3D by the platform angle and append it to the cloud.

Each lidar reading is an angle φ within the scan plane and a distance d. In the scan plane that's a 2D point, `(d·sin φ, d·cos φ)`, with the second coordinate pointing up. Turning the platform by θ rotates the scan plane about the vertical axis, so the up coordinate is unchanged and the horizontal one is swung around:

```python
x_2d = distance * np.sin(phi)
y_2d = distance * np.cos(phi)    # up

x = -x_2d * np.cos(theta)
y = y_2d
z = x_2d * np.sin(theta)
```

Because the lidar sees a full circle, half a turn of the platform would be enough to cover every direction. The camera only looks one way, though, so a scan turns the full 360° in 0.5° steps to let it see the whole room. With a one-second settle at every stop, that takes about fifteen minutes.

Each scan is saved as an ASCII PLY file, which opens directly in MeshLab or CloudCompare. I tested it at home, starting with my office and then the rest of the main floor.

![Uncolored point cloud of a small office, shaded by height, viewed from above with the ceiling cropped away](hls-lfcd2-office-scan-uncolored.png "An HLS-LFCD2 scan of my office with no color. Ceiling cropped away.")

That's the resolution this prototype works at. With one reading per degree from the lidar and half a degree between slices, walls, doorways, and large furniture are clear, but anything smaller is already breaking up.

# Color
A bare point cloud shows geometry but not much else. You can recognize a room, but not what's in it. To color it, I mounted a camera alongside the lidar and took a frame at every stop.

The coloring relies on the fact that the camera turns with the lidar. With the camera on its side, the center column of each frame is a vertical line of pixels looking in nearly the same direction as that slice. For each lidar point, its elevation angle as seen from the camera picks a row in that column:

```python
y_from_camera = y_3d - camera_vertical_offset
vertical_angle = np.degrees(np.arctan2(y_from_camera, horizontal_dist))
# vertical angle → pixel row, via the camera's vertical field of view
```

The camera's field of view is much narrower than the lidar's full circle, so only part of each slice gets real color. Everything outside it is painted a placeholder yellow. Getting the colors to land on the right points took a few rounds of hand-tuning. The camera's tilt and its height above the lidar each needed a correction.

![Colored point cloud of a small office seen from above, with the ceiling and the two nearest walls cut away](hls-lfcd2-office-scan.png "The same office scan with color from the camera, seen from a different angle with the ceiling and the two nearest walls cut away. Most of the yellow placeholder points were on the ceiling.")

# Stitching rooms together
One scan only sees what's in line of sight from one spot. To capture more than one room and reduce blind spots, I took several scans from different places and aligned them by hand in MeshLab, then merged them into one cloud. The image at the top of this post is the whole main floor, merged this way. But lining up each scan by eye was slow and never quite exact.

{{< video src="kitchen-three-scan-pan.mp4" poster="kitchen-three-scan-pan-poster.jpg" loop="true" caption="The kitchen, merged by hand from three HLS-LFCD2 scans taken from different spots, panning across the room from about eye height. Ceiling cropped away." >}}

{{< video src="main-floor-multiscan-orbit.mp4" poster="main-floor-multiscan-orbit-poster.jpg" loop="true" caption="The merged main floor from the cover image, orbited from above with the ceiling cropped away." >}}

# What's next
This prototype answered the question it was built for. A surplus 2D lidar on a 3D-printed turntable can capture a recognizable, colored 3D room, and a few scans stitched together start to look like the dollhouse views I was after. It also showed how far this version was from that goal:

- **Resolution.** The HLS-LFCD2 takes one reading per degree, so at 3 meters neighboring points are about 5 cm apart. Walls and large furniture come through, but anything smaller turns into noise.
- **Speed.** Every stop waits a second for the platform to settle and keeps just one revolution of data. A single room takes about fifteen minutes.
- **Color.** One column of pixels per frame colors only part of each slice, and every alignment correction was a number tuned by eye.
- **Merging.** Aligning scans by hand doesn't scale past a few rooms.

The rest of the series works through those one at a time: a denser lidar and a scanner that sweeps continuously instead of stopping, coloring from full camera frames with a fitted calibration, and automatic registration and meshing of a whole house.

# Appendix
### Getting power and data into a spinning head

I'm a bit amazed at how cheap these commodity lidars are, given the performance they provide. To capture a full circle, the laser head has to spin continuously, so I assumed there was a slip ring inside: a set of rings with a brush riding on each one. Those work fine for power and are used widely, but I wondered how well they'd hold up for high-speed data. I asked Claude to compile some research, and it turns out most of these lidars don't use a slip ring at all:

{{< claude-research >}}
These commodity 2D lidars all share the same basic layout. A ranging core, a laser and a detector, sits in a head that spins five to ten times a second. The head measures one distance at a time and tags each one with the angle it was pointing at. Spinning the core is what turns a single-beam rangefinder into a 360° scanner, but it also raises an awkward problem. The core needs power going up into it and has to send a steady stream of readings back down, and it does this while rotating continuously for years.

**Slip rings.** The traditional answer is a slip ring: metal brushes on the stationary side rubbing against rings on the rotating side. Neato's XV-11 robot vacuum, whose lidar started the cheap-lidar era around 2010, used one. Its head connects with four wires labeled TX, RX, GND and VCC, and its patent describes power fed "through a 2-wire slip ring on the rotation center". Slip rings work, but they wear. Slamtec says they "only have thousands of hours of life," and that adds up quickly: at 5 revolutions a second, 1,000 hours is 18 million turns. A patent from Carnavicom, a Korean lidar maker, adds cost and electrical noise to the list.

**Power through a transformer.** Most current cheap lidars have no electrical contact between the head and the base at all. Power crosses the joint through a pair of coils that act as a transformer split in two. The base drives a stationary coil with AC, which sets up a changing magnetic field. A second coil on the head sits inside that field, separated by a small air gap, and the field induces a voltage in it. The head rectifies that, smooths it and regulates it down to the supply its electronics need. It's the same principle as a wireless phone charger. The coils are wound concentrically around the rotation axis, so the coupling is the same at every angle and rotation makes no difference to the transformer.

Teardowns show this in practice. A Delta-2G from a recovered robot vacuum has its two coils "mounted vertically like tubes," one inside the other, with the drive circuit on the base board and "AC rectification, smoothing and LDO" on the head. A Xiaomi LDS01RR has a "transformer coil for powering head" at the center of its base board. Efficiency isn't great, but the power involved is small. The STL-27L draws about 1.45 W in total.

![Cross-section of a spinning lidar's rotary coupler. A rotating head holding the laser and detector sits on a stationary base. Two concentric tube-shaped coils, the inner one turning with the head and the outer one fixed to the base, pass power across an air gap. An infrared LED on the underside of the head shines down the hollow center of the coils onto a phototransistor on the base board](rotary-coupler-diagram.svg "A simplified cross-section of the coupler in a typical cheap lidar. Power crosses between the concentric coils, and data goes down the hollow center as infrared light. The motor and bearings are left out.")

**Data on a beam of light.** The middle of the coil assembly is hollow, and that's where the data goes. An infrared LED on the underside of the head, right on the rotation axis, shines straight down onto a phototransistor in the base. Because both parts sit on the axis, the LED looks the same to the receiver at every angle, and the link never breaks as the head turns. The base board amplifies the phototransistor's output back into clean logic levels and passes it out as a UART. In the Delta-2G, that's 115200 baud. Slamtec's RPLIDAR A2 uses the same pairing, which Slamtec brands OPTMAG: "wireless power and optical communication technology." LDROBOT's LD06 datasheet likewise lists a "wireless transmission unit" and a "wireless communication unit" in place of a slip ring.

That light link carries a surprising amount of data. The HLS-LFCD2 sends 300 frames of 42 bytes a second, about 126 kbit/s on its 230400-baud line. The STL-27L sends about 1,800 packets of 47 bytes a second, about 850 kbit/s, which fills over 90% of its 921600-baud line. That's close to a megabit a second through a single LED.

**Mostly one way.** In the cheapest designs, the light link only goes down. The Delta-2G's connector has no RX line at all, so it can't receive commands. That works because the head doesn't need to be told much. It starts ranging as soon as it has power, and motor speed is controlled from the base by varying the voltage or a PWM signal, which never needs to reach the head. The HLS-LFCD2 does take commands, `b` to start and `e` to stop, and ROBOTIS lists a "Full Duplex option" for it. No public teardown of the HLS-LFCD2 turned up, so it's unclear whether those commands reach the head over a second light path or whether the base simply switches the motor and the head's power.

More expensive scanners go both ways. A patent from Ouster, which makes lidars for cars and industrial robots, sends data down the hollow shaft and control signals up through a second optical channel arranged in a ring around it, with power through a coil set in a ferrite channel at the edge of the board. A Google patent, now owned by Waymo, uses inductive power too but moves data with pairs of concentric conductive rings that form capacitors across the gap, enough for a 100 Mbit Ethernet link.

**Knowing the angle.** One consequence of this design is that the base can't see where the head is pointing. The head has to work out its own angle and send it along with the distances. In the HLS-LFCD2, that's the index byte at the start of each frame, which says which six degrees the frame covers. In the STL-27L, every packet carries a start and end angle in hundredths of a degree. The XV-11 used an optical encoder for the same job, giving it 1° accuracy. When a lidar reports angles, the angle is measured on the head itself, and the base only passes it along.

### Sources
- [Not Black Magic: LiDAR Modules](https://notblackmagic.com/bitsnpieces/lidar-modules/): a teardown and protocol analysis of a Delta-2G.
- [iliasam/LDS01RR_lidar](https://github.com/iliasam/LDS01RR_lidar): a teardown of a Xiaomi LDS01RR.
- [RECESSIM: Xiaomi Roborock LIDAR](https://wiki.recessim.com/view/Xiaomi_Roborock_LIDAR): photos of the coils and the wireless receiver board.
- [SparkFun: Neato Robotics XV-11 Tear-down](https://news.sparkfun.com/490) and [Hizook: Ultra-Low-Cost Laser Rangefinders](https://hizook.com/ultra-low-cost-laser-rangefinders-actualized-neato-robotics/).
- [US8996172B2](https://patents.google.com/patent/US8996172B2/en): Neato's distance sensor patent.
- [Slamtec RPLIDAR A2](https://www.slamtec.com/en/lidar/a2): OPTMAG and slip-ring lifetime.
- [ROBOTIS LDS-01 specifications](https://emanual.robotis.com/docs/en/platform/turtlebot3/appendix_lds_01/).
- [US20220291351A1](https://patents.google.com/patent/US20220291351A1/en): Carnavicom, inductive power and wireless data for a rotary lidar.
- [US11340336B2](https://patents.google.com/patent/US11340336B2/en): Ouster, optical uplink and downlink channels.
- [US20150091374A1](https://patents.google.com/patent/US20150091374A1/en): Google/Waymo, contactless electrical coupling for a rotating lidar.
{{< /claude-research >}}
