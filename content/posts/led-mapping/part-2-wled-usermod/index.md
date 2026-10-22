---
title: "3D LED Mapping, Part 2: Effects on the Controller"
# TODO: cover image once the on-device media exists
# cover:
#     image: ""
#     relative: true
#     alt: ""
date: 2026-10-19
draft: true
tags: ["leds", "wled", "esp32", "c++", "python", "platformio", "testing"]
description: "Moving the mapped effects off the laptop and onto the ESP32 as a WLED usermod, with a test that keeps the C++ port in step with the Python originals."
---

<!-- TODO: cover. An effect running on the tripod with the laptop closed or out of frame, plus a poster frame. Also set the cover in the front matter.
{{< video src="tripod-on-device.mp4" poster="tripod-on-device-poster.jpg" loop="true" caption="..." >}}
-->

In [part one](../part-1-proof-of-concept/), a webcam and a few handheld views produced a 3D map of an LED strip wound around a tripod. The effects that use the map ran on my laptop. Every frame was computed in Python and streamed to the ESP32 over Wi-Fi, 40 times a second. That was fine for testing, but it isn't how lights should work. Wi-Fi adds jitter, and the lights stop when the laptop goes to sleep.

This post moves the effects onto the controller. You capture the map once with the laptop, push it to the ESP32, and the effects run on the ESP32 from then on. They show up in WLED's effect list next to its built-in ones.

# Is the ESP32 fast enough?
I expected this to be the hard part. It isn't. The rough numbers for 200 LEDs:

- **The map** is three floats per LED, about 2.4 KB. The board has almost a megabyte of free filesystem space.
- **The math.** The heaviest effect, `plasma`, does about six `sin` and `exp` calls per LED. The estimate was roughly a millisecond per frame for the whole strip.
- **The strip itself.** A WS2812B takes 30 µs per LED to update, so 200 LEDs take 6 ms. That caps the frame rate at around 150 fps no matter how fast the effects are.

The strip is the bottleneck, not the math. There's one trap. The ESP32 has a hardware floating-point unit for single precision only, and it does `double` math in software. NumPy uses doubles everywhere. The port has to use `float` and `sinf`/`expf` in every per-frame calculation, or it gets much slower.

# A WLED usermod
There were two ways to do this. One was to write standalone firmware with FastLED and add my own Wi-Fi setup, map upload and effect selection. The other was to build on WLED, which was already running on the board.

WLED supports usermods, add-ons compiled into the firmware that can register their own effects. Going that route keeps everything WLED already does: Wi-Fi setup, over-the-air updates, the web UI and phone app, presets, schedules and power limiting. The calibration from part one keeps working too, since it only needs WLED's realtime DDP input. The mapped effects become ordinary WLED effects with names like "OSLM radar", and they can go in a preset or a schedule like any other effect.

<!-- TODO: screenshot of the WLED effect list showing the OSLM effects, and/or the Info panel with the "OSLM map" and "OSLM effect fps" rows.
![WLED's effect list with the OSLM effects](wled-effect-list.png)
{.half}
-->

## Getting the map onto the board
The map file is as simple as I could make it. It starts with the bytes `OSLM`, a version number and the LED count. Then come three little-endian 32-bit floats per LED. An LED the camera never found is stored as NaN and stays dark.

```python
def encode_map(m: LedMap) -> bytes:
    return b"OSLM" + struct.pack("<HH", MAP_VERSION, m.n) + np.asarray(m.positions, "<f4").tobytes()
```

Uploading the map needed no firmware code. WLED's file editor already posts files to `/upload`, so `oslm push` sends `/oslm.bin` the same way. Then it sends a small JSON command that tells the usermod to reload the map, and it can also switch the strip to an effect:

```sh
uv run oslm push captures/tree/map.json --effect radar
```

The usermod draws map LED *i* to pixel *i* of the WLED segment. That's only the right LED when one segment covers the whole strip, with no reverse, mirroring or grouping. Before uploading, `push` reads the device's segment setup and warns if any of that is off.

## Work done once, at load
Some effects depend on the whole map, not just one LED. The rotating effects (`radar`, `pinwheel`, `barber` and the rest) measure each LED's angle around the median x and z of the whole map. `twinkle` gives each LED a fixed random value made from a hash of its position. NumPy recomputes all of this every frame, which costs nothing on a laptop.

On the ESP32 those values are computed once, when the map loads, and stored with each LED. That makes the per-frame work simple. Here's `radar` in C++, next to the Python version from part one:

```cpp
inline void radar(const Scene& s, float t, Rgb* out, Rgb color = {0.1f, 1.0f, 0.3f},
                  float period = 2.0f, float tail = 0.15f) {
  float turn = fract(t / period);
  for (size_t i = 0; i < s.leds.size(); i++) {
    float behind = fract(turn - s.leds[i].az); // turns since the beam passed
    out[i] = scale(color, expf(-behind / tail));
  }
}
```

The load-time code is the one place that uses `double`. The position hash multiplies by 43758.5453 and keeps only the fractional part, which amplifies rounding error. In single precision the result would drift away from NumPy's. It runs once per LED at boot, so the slow software math doesn't matter.

# One effect, two implementations
This is the real cost of running on the device. All 21 effects now exist twice, once in Python and once in C++. The Python versions are still needed. `oslm play` uses them, and so does the 3D viewer. Two copies of anything drift apart.

The fix was to make the C++ testable without hardware. The effects live in one header, `oslm_effects.h`, with no Arduino or WLED code in it. The usermod includes it, and so does a small command-line program built with `g++` on the laptop. That program reads a map file, renders an effect at the times you give it, and writes the raw RGB bytes to stdout.

A pytest test builds that program and compares it against the Python effects. It renders every effect on two maps, a flat random scatter and the simulated helix from part one, each with a few unmapped LEDs. It checks six times from 0 to 3599.9 seconds. Every LED has to match Python to within 2 out of 255 in each channel:

```python
for k, t in enumerate(TIMES):
    want = effects.render(effects.EFFECTS[name], P, known, t).astype(int)
    diff = np.abs(got[k] - want).max(axis=1)
    assert (got[k][~known] == 0).all()
    assert diff.max() <= 2, f"t={t}: LEDs {np.flatnonzero(diff > 2)} differ by up to {diff.max()}"
```

The test compiles with `-std=c++11 -Wall -Wextra -Werror`, since the ESP32 toolchain compiles in C++11 mode. Another test checks that the C++ effect table lists the same effects, in the same order, as the Python dictionary. Adding an effect in Python and forgetting to port it fails the tests. The whole suite, 76 tests, runs in about 25 seconds.

# Fitting into WLED
A WLED effect is a C function that takes no arguments. It draws into the current segment and returns how long to wait before the next frame. The OSLM effects all go through one render function, which needs to know which effect to run. A template gives each entry in the effect table its own no-argument function, and a second template registers all of them at startup:

```cpp
template <int I> static uint16_t mode_oslm() { return oslmRender(oslm::EFFECTS[I].fn); }
```

Each effect's registration string tells the WLED UI which controls to show. All of them use the Speed slider. A Speed of 128 is real time, the same as `oslm play`. Each segment keeps its own effect clock, advanced each frame by the elapsed time scaled by Speed, so moving the slider doesn't make the effect jump. The clock wraps every hour so float precision doesn't wear down after the lights have been on all evening.

Each effect also has a **Custom colors** checkbox. It's off by default, so effects look the way they did in part one. With it on, single-color effects like `radar` and `snow` use WLED's first color picker. `pinwheel`, `helix` and `barber` paint their three blades with the three color pickers. Rainbow-style effects take their hues from the selected WLED palette. The UI only shows the pickers or palette list an effect actually uses. Presets save those settings like any other effect's.

<!-- TODO: short clip of changing Custom colors / palette from the phone app while an effect runs. -->

The usermod also adds two rows to WLED's Info panel. One shows the map status, for example "124/145 LEDs mapped", with a warning if the strip length doesn't match the map. The other shows how many frames per second the effects are actually rendering.

# Building and flashing
The usermod targets WLED 0.15.2, the version already on the board. WLED changed how usermods are registered after 0.15, so the version is pinned. A setup script clones that WLED tag, links the usermod into its `usermods` folder, adds it to WLED's usermod list and writes a PlatformIO build environment for it:

```sh
firmware/setup.sh
cd firmware/wled && pio run -e esp32dev_oslm
```

The resulting image is about 1.4 MB. It goes onto the board through WLED's own update page over Wi-Fi. USB is the fallback if a bad build won't boot. WLED's settings and presets live in the filesystem, so they survive the update, but it's still worth backing them up first.

# Results
<!-- TODO: measured on-device numbers. The "OSLM effect fps" row in Info for a light effect (hline) and the heaviest (plasma). How does it compare to the ~150 fps strip limit and the 40 fps that oslm play streamed? -->

<!-- TODO: side-by-side or sequence clip of several effects on the tripod running from the ESP32 alone.
{{< video src="tripod-effects-on-device.mp4" poster="tripod-effects-on-device-poster.jpg" loop="true" caption="..." >}}
-->

# What's next
<!-- TODO: confirm or replace with your actual plans. -->
- **Write each effect once.** The equivalence test keeps the two versions honest, but every new effect still has to be written twice. Generating the viewer's frames from the C++ build would remove one dependency on Python. A small effect language, like Pixelblaze uses, would remove the duplication entirely.
- **A real object.** The tripod has done its job. Next is something worth lighting.
