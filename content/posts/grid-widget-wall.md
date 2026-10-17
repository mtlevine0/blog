---
title: "An Open Source LED Video Wall from Cheap Commodity Hardware (Part 2: GRID)"
date: 2026-09-08T10:00:00-05:00
draft: true
tags: ["hardware", "python", "pygame", "led", "video-wall", "home-lab", "reverse-engineering"]
---

![now-playing widget on the wall, mid-song](/images/led-wall/spotiamp-frame.png)
{.pixelated}

[Part one]({{< ref "led-video-wall.md" >}}) of this series covered `colorlightpy`: a driver that will take any well-formed stream of raw RGB frames and put it reliably on a wall-mounted 384×192 LED display. That's necessary, but it says nothing about *what* to show, or how to switch between things without SSHing in and restarting a process. This post is about the layer working on turning "a pipe that accepts pixels" into something closer to an actual product: [GRID](https://github.com/mtlevine0/grid), a wall of independent live widgets with a physical remote.

## What GRID is

GRID — **G**lanceable **R**egistry of **I**ndependent **D**isplays, a backronym landed on after the project outgrew its working name, `jumbo-widget` — is `gridtron`, an orchestrator that:

- runs every widget as its own subprocess
- decides which one's frames actually go to the wall
- turns a physical macro pad into the remote for all of it

```bash
./grid --headless | colorlightpy stream -i eth0 -W 384 -H 192 --pixel-format bgr
```

That's the whole seam between the two projects. `gridtron` writes exactly the frame contract `colorlightpy` expects — `width * height * 3` bytes, no header, straight to stdout, one black frame on exit — so from the driver's point of view, nothing changed between part one and this post. It's still just reading raw pixels off a pipe. What's upstream of that pipe got a lot more interesting.

## Why: one widget became nine

The project started as a single widget — a now-playing display, `spotiamp` — and outgrew that almost immediately into wanting several things on one wall, on demand, without a keyboard involved. Nine widgets are installed as of this writing: live sports scores, a system monitor, now-playing, weather, home-network stats, NVR camera alerts, a home energy monitor, an interactive emoji wall, and — a little self-referentially — a widget that reads Claude Code's own local transcripts to show token/cost burn against the current session.

The decision that made all of that tractable was making every widget **its own OS process**, started in its own directory exactly as if launched by hand. That started as necessity, not design: the existing apps each defined top-level `layouts`, `screen`, `control` and `main` modules, so no two could be imported into a single interpreter without a rename, and each had its own virtualenv with its own dependencies. Living inside GRID doesn't change any of that — a widget is a subprocess like any other, unaware it's being orchestrated. It just runs headless and writes frames the same way it always did standalone, which is what makes "install a widget" as cheap as `cp -r` plus three lines of TOML.

That necessity turned into a resilience property for free: a crashed widget can't take the wall down. It just stops updating; `gridtron` keeps running, the pad keeps working, and switching to something else is one press away.

![GRID's home page, rendered off the real manifest and render code](/images/led-wall/pad-home.png)
![weather widget, real conditions at capture time](/images/led-wall/weather-frame.png)
![Frigate NVR alerts pulled from a live instance](/images/led-wall/frigate-frame.png)
![home energy monitor: whole-panel wattage, today's cost, and panel load](/images/led-wall/emporia-frame.png)

Inactive widgets keep running by default (about a tenth of a core each) so switching to one is instant, with live data already on screen rather than a blank page while it spins up.

The newest of the nine, a home energy monitor pulling from an Emporia Vue panel sensor, is the most data-dense widget on the wall so far: one page for real-time whole-panel wattage against a 200A/240V ceiling with a five-minute sparkline, one for today's and this month's cost against the account's actual per-kWh rate, one ranking the top individual circuits by draw, and one shaped like a day-long load curve. None of it is a novel UI idea — it's the same kind of thing a real energy-monitoring product shows — but it's a good demonstration of the wall doing something other than glanceable status: numbers you'd actually study for a few seconds, not just glance past.

## Not just a display: the emoji wall

Every other widget on the wall is read-only — it shows you something. `emoji-wall` is the first one built the other way around: post an emoji from anywhere on the box and it drops onto the wall and falls, with the poster's name riding along.

![emoji wall mid-fall, three people posting at once](/images/led-wall/emoji-wall-frame.png)

Three things land on the same `emoji.post` command underneath: the pad's eight favorite-emoji keys on its main page, a dial-browse-then-fire pair (rotate through a category, press to drop whatever's browsed — a chip in the corner of the wall shows what's queued so you're not looking away from the wall to see what you're about to post), and `emojictl post <glyph> [from-name]` from anywhere else on the box, which is the one meant for a phone shortcut or a chat-bot relay to hook into. (The control socket is local-only, so "remote" means something else on the host or over SSH, not the open internet.) Repost the same glyph inside about 0.6 seconds and it escalates — bigger, extra copies past a couple of repeats, a screen-shake past five in a row — which is the kind of thing that turns "post an emoji" into people actually competing to spam the wall.

It's a small widget, but it's the one that made the control bus's remote-posting story real: every other widget is driven from the pad or `gridctl` sitting at the same machine, and `emoji-wall` is the first thing anyone's actually going to want to trigger from a phone.

## The control surface: reverse-engineering a "Stream Deck" that isn't one

The physical remote is a Razer Stream Controller, and the first thing worth knowing about it is that none of the Stream Deck tooling on the internet applies to it, because it isn't one. Under the branding it's a **Loupedeck Live**: it enumerates as USB CDC-ACM — a plain serial port, `/dev/ttyACM*` — plus two USB audio interfaces for its onboard mixer, not as a HID device at all. `python-elgato-streamdeck`, `hidapi`, every Stream Deck recipe — none of it applies; there isn't even a `hidraw` node to point them at. The actual protocol is websocket-style binary frames over that serial port: a 2-byte action code, a transaction id echoed back in the reply, then a payload — button presses, knob rotation, and framebuffer writes in little-endian RGB565.

The device also has one gotcha that's a good illustration of why reverse-engineering by observation is slower and less trustworthy than it looks. Some Loupedeck hardware exposes three separate displays — left, centre, right — and that three-display model is what most reference material describes. This particular device does not: it has exactly **one** display, a single 480×270 surface covering the key grid and both side strips together. But writes addressed to the nonexistent left/right/centre ids don't error — the firmware **acks every framebuffer write, including ones sent to ids that don't exist**, `CONFIRM_FRAMEBUFF` and `CONFIRM_DRAW` and all, with the panel just quietly not updating. There is no way in software to tell a discarded write from a successful one; only looking at the actual panel tells you, and the three-display model survived a full round of testing here before anyone thought to look at the hardware. It's the same shape of problem as `colorlightpy`'s protocol work in part one — a device that never tells you when you're wrong, so the failure mode of a bad hypothesis isn't an error, it's silence.

Once that geometry was sorted out: **12 LCD keys** (90×90, rendered live with pygame — a key can show the current layout, a live score, CPU percent, or the reply to what you just pressed), **6 dial/encoders**, **8 RGB buttons**, and **2 side strips**. The home page is one key per installed widget, so any widget is always a single press away.

## A key never lies

Every widget speaks the same dotted-command protocol over its own control socket — `gridctl widget.next`, `gridctl send game.next` — deliberately readable as `<widget>ctl`. Press a key on the pad and the command's actual reply lands on that key for a moment: `ok layout: cpu` right where you were already looking, instead of on some log file nobody's watching. A command that fails toasts red on the key. A widget with no socket renders its command keys disabled rather than pretending they'll work. It's a small rule — a key never lies about what just happened — but it's the kind of thing that matters more on a device with no way to type back at you.

There's a good, slightly embarrassing anecdote behind why that rule is now enforced by a test rather than left to code review. `rotate.toggle` — the button that turns automatic widget rotation on and off — originally raised `no dwell set; use rotate.every <seconds> first` if the process had started without `--rotate`. In a terminal, that reads as helpful validation: you just type the argument it asks for. That's exactly why it survived review. On the pad, it meant the Rotate key was permanently dead from a cold start, because a button cannot type an argument back. The fix wasn't just patching that one command — it was `test_button_commands.py`, which now asserts that every no-argument command bindable to a button never raises, never returns a bare `ok`, and works pressed twice in a row so a toggle gets exercised in both directions. The bug was specific; the lesson generalized into something that runs on every future command.

## Installing a widget

A widget doesn't need to know GRID exists. Drop a TOML manifest pointing at an existing app's directory and its command, and it starts appearing as a page on the pad:

```toml
[widget]
name    = "my-widget"
title   = "My Widget"
icon    = "📊"
accent  = "#5ac8fa"
path    = "/home/matt/Dev/somewhere/my-widget"
command = [".venv/bin/python", "main.py", "--headless", "--pixel-format", "rgb"]
socket  = "$XDG_RUNTIME_DIR/my-widget.sock"
```

Or, for a widget that lives inside the GRID tree itself — a **bundled** widget — most of that manifest is inferred from the directory: name from the directory name, path from the directory itself, the command defaults to `python main.py --headless --pixel-format rgb`, and `python` resolves to the widget's own `.venv` if it has one, or the orchestrator's if it doesn't. `grid-new` scaffolds a new one from scratch — the pygame loop, the frame contract, a control bus with three commands already on it, a pad manifest, and a README — which is the fast path from "I want a new thing on the wall" to a widget that already runs.

## Where it's headed

Standardization across widgets is the honest rough edge right now. A media-shaped view contract (`set_track`, `set_artwork`, `show_volume`, `hide_volume`) grew up around the now-playing widget and leaked into the others, which implement all four as no-ops just to stay droppable onto the same interface. The layout-registry code is duplicated near-identically between two widgets and absent from a third. And the now-playing widget has quietly become a de-facto shared component library for the others, imported by a hardcoded path rather than packaged as one. None of it blocks anything — every shipped widget is fully drivable from the pad — but it's the next thing to clean up before a tenth widget makes it worse.

That's the two-part shape of the project so far: `colorlightpy` gets pixels onto cheap commodity hardware reliably, and GRID is what's turning that pixel pipe into something that's starting to feel like a finished product — instant switching, a physical remote, widgets that fail independently instead of taking the wall down with them. There's a real gap left between "starting to feel like" and actually commercial-grade — the standardization work above is part of that, and it'll keep showing up in future posts. Both halves are on [GitHub](https://github.com/mtlevine0) if you want to build on either one.
