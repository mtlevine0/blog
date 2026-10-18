---
title: "IN-9 Spectrum Analyzer, Part 3: Agentic Assisted CAD"
cover:
    image: "render-lit-hero.jpg"
    relative: true
    alt: Render of the ten-tube IN-9 display in its MakerBeam frame with the tubes lit
date: 2026-10-17
draft: true
tags: ["hardware", "freecad", "cad", "3d-printing", "python", "in-9", "claude", "ai", "unconventional-displays", "product-design"]
description: "Scripting a parametric FreeCAD assembly for the ten-tube IN-9 display: two-tube modules, a MakerBeam frame, and a friction-fit tube bracket."
---

[Part two]({{< ref "posts/in-9-spectrum-analyzer/part-2-ai-pcb-design" >}}) ended with a dual-channel driver board small enough to tile. This post covers the mechanical side: the enclosure that holds ten IN-9 tubes, five driver boards, and five boost PSUs in a row, designed in FreeCAD.

<!-- TODO: why FreeCAD, and why this needed real CAD rather than eyeballing it like the part 1 frame. -->

# The design
Ten tubes stand in a row at 22 mm pitch, in five printed two-tube modules. Each module is one C-shaped print: the boost PSU on its floor, a dual driver board on its back plane, and two tubes in grommets in its roof. A MakerBeam 10 × 10 aluminium frame with printed corner cubes is the structure. The modules hold onto it with T-key ribs on their back face and front posts that slide along the beams' T-slots, so nothing is bolted. A printed tube bracket near the tips keeps the tubes aligned, gripping the glass with crush ribs, and clear acrylic panels close the front and sides.

![The v5 assembly in FreeCAD: ten IN-9 tubes in five two-tube modules inside a MakerBeam frame](assembly-v5-front.png "The final assembly: five two-tube modules in the MakerBeam frame.")

The rest of the post covers how Claude built it, then how the design got here and each part in detail.

# Parametric from scripts
None of the parts were drawn by hand. Every part is a FreeCAD Python script, and the `.FCStd` files and STLs are generated from them:

- `tube_module.py` builds one two-tube module: the printed body, grommets, tubes, the driver board (imported as a STEP straight from KiCad), and the PSU.
- `psu_model.py` models the 30 × 21 × 12 mm boost PSU from a photo of the listing.
- `makerbeam.py` models the MakerBeam profile, a printable corner cube, and the frame hardware.
- `in9_assembly.py` places five modules in the frame, adds the tube bracket and acrylic panels, prints the panel cut sizes, and exports the STLs.

The assembly derives every number from the module's parameters, so changing the module width or height carries through to the beam lengths and cut list.

Scripts are also what made it practical to hand the CAD work to Claude. A FreeCAD document is a binary blob of features and placements, but a Python script is something an agent can read, diff, and edit like any other code.

# Claude driving FreeCAD
As with the PCB in part two, Claude did most of the modelling and I steered. The tooling is a bit different, though.

## FreeCAD MCP
[FreeCAD MCP](https://github.com/neka-nat/freecad-mcp) has two halves: an addon that runs inside FreeCAD and serves an XML-RPC API on `localhost:9875`, and an MCP server that wraps that API as tools for Claude: creating and editing objects, inspecting the document, and running arbitrary Python in the FreeCAD GUI.

## From photos and drawings to models
The other parts had no STEP files, so Claude modelled them from a product photo, a dimensioned drawing, or a caliper reading. Anything guessed is marked as an estimate in the code.

- **Boost PSU:** outline and height from the listing, components placed by reading the photo.
- **MakerBeam:** the slot profile traced off MakerBeam's drawing and extruded.
- **Corner cube:** sized from the stock MakerCube STL, then redrawn to print without supports.
- **Grommets:** from the listing's drawing. The 3 mm groove sets the module's roof thickness.
- **IN-9 tube:** a revolved profile at the 9.3 mm diameter I measured with calipers.

The pattern: start from the listing, then swap in measurements as real parts arrive. A new measurement is a one-line change and a rebuild, and the clash checks and asserts flag anything it broke.

## Checking its own work
The weak spot of an agent doing CAD is that it can't see what it built unless you give it a way to. Every change went through the same loop before Claude reported back:

1. **Clash check.** Boolean intersections between every pair of solids in global placement, reporting `common().Volume`. Some overlaps are expected, like the cube screws in the beam bores, or the bracket's crush ribs biting into the glass. Any other non-zero volume is a real clash.
2. **Asserts.** The scripts assert the geometry rules that are easy to break: the front posts sit beside the PSU and below its plugs, and the T-key ribs land on the beam centrelines and fit the slot opening. A failing assert means the design needs fixing, not the assert.
3. **Screenshot.** A snippet run through the same RPC saves a 1600 × 1100 view of the model, and Claude reads the PNG back to look at it.
4. **Overhang audit.** After a change to a printed part, Claude scanned the exported STL for downward-facing facets steeper than 45°.

The screenshot step caught the kind of thing numbers don't, like a part that was technically clash-free but sitting in the wrong place.

## An AGENTS.md for the mechanical design
A long CAD session outlives Claude's context, so the design knowledge had to live in the repo. `assembly/AGENTS.md` is the quick start: the design in one paragraph, what each script does, the coordinate system and object naming, how to run FreeCAD live or headless, the verification loop, and the design rules to keep (printable without supports, nothing bolted, the frame is the structure).

It also collects the FreeCAD scripting gotchas Claude hit along the way, so the next session doesn't hit them again:

- `shape.translate()` and `rotate()` on a `Part.Shape` only change its placement. Assigning `.Placement` later quietly throws the move away. Baking it in needs `transformGeometry()`.
- Parts repeated with `App::Link` only report their true position through `getGlobalPlacement()`, which matters a lot when the clash check is the safety net.
- The FreeCAD GUI caches imported modules, so every rebuild needs `importlib.reload`.
- Front is the driver-board side, +Y. That got flipped once and had to be corrected across every script, so now it's written down.

## What it couldn't do
Claude is good at keeping a parametric model consistent: changing one module parameter and chasing it through the frame, the cut list, the asserts, and the STLs is exactly the tedious work it doesn't mind. It's much less good with dimensions nobody has measured. The PSU model's mounting hole and part positions are estimated from a photo of the listing, and the MakerBeam slot cavity is only approximate, so the guide tells Claude not to build tight fits on them. That's why the T-key ribs have clearance in the slots rather than a snug fit.

The tube bracket proved the point. The first one was designed before I'd measured the tubes, around the nominal 10 mm diameter, and its crush ribs never touched the 9.3 mm glass. The fix was a printed grip test with a ring at each candidate diameter, labelled, to tune the fit against real tubes.

![The grip test gauge: a strip of ten tube-bracket rings, each with three crush ribs, labelled with grip diameters from 8.8 to 9.7 mm](tube-bracket-grip-test.png "The grip test: the bracket's ring, spine and crush ribs, one ring per grip diameter from 8.8 to 9.7 mm.")

Like the board constraints in part two, the inputs that matter still come from me, and from calipers.

# How the design evolved
<!-- TODO: trim or expand. v1–v3 had no screenshots kept. -->
The design went through five versions before it settled:

1. **v1:** a first tube module, a 50 × 30 mm PSU, and a ten-tube chassis.
2. **v2:** tube modules stacked on a tier of upright PSUs, with each module split into a spine and a cap.
3. **v3:** a smaller 30 × 21 mm boost PSU lying flat in a printed tray under each pair of tubes.
4. **v4:** a MakerBeam 10 × 10 frame with corner cubes around one-tube modules joined by lap joints, all bolted to the beams.
5. **v5:** two-tube modules, each carrying its own PSU and dual driver board, held by nothing but the frame.

![v4 assembly: ten single-tube modules on PSU trays in a MakerBeam frame](assembly-v4-front.png "v4: one module per tube, on PSU trays, bolted to the frame.")

v4 worked on paper, but it was clear the tube modules needed an overhaul now that an external frame was available to hold everything together.

# The two-tube module
Each module is one C-shaped print, 44 × 37 × 65.9 mm, two tube pitches wide:

- **Floor:** the boost PSU lies flat mounted by a single M2 screw.
- **Back plane:** one dual driver board stands centred on two M2 bosses. The board has parts on its back face, so the rest pads and boss tips are placed to clear them. The trimmers' leads clear the back plane by 0.08 mm.
- **Roof:** two tubes in grommets, with a 16.4 mm gap above the board for wiring the tube leads into the screw terminals.
- **Front lip:** a 4 mm rib under the roof's front edge that stiffens it to carry the tubes.
- **Front posts:** two short posts at the floor's front corners, beside the PSU and below its plugs.
- **Sides:** open, so the 12V and PWM wiring runs along the row as one tunnel.

The driver boards' power headers land 21.5 mm from the next board's, so neighbouring modules link with short leads. Those links should carry only 12V and GND: each module has its own PSU driving just its two tubes, and paralleling five 150V outputs isn't a good idea.

![One assembled two-tube module out of the frame: the PSU on the floor, the driver board on the back plane, and two IN-9 tubes in grommets in the roof](two-tube-module.png "One two-tube module, out of the frame.")

# The frame
The frame is an exoskeleton of MakerBeam 10 × 10 aluminium, sized to sit flush with the modules' top, bottom, front and back. Four 220 mm long beams run along the row, front and rear, top and bottom. Each end has two crossbars and two uprights, and a printed corner cube joins the beams at each of the eight corners, with an M3 button head through the cube into each beam's tapped end. With the frame on, the whole display is 240 × 57 × 65.9 mm, not counting the tubes.

# Modular assembly with minimal hardware
The modules hold onto the frame with T-key ribs that slide along the long beams' inner T-slots: the back face's top and bottom ribs in the rear beams, and the lip and front post ribs in the front beams. The rib heads sit under the slot lips, so a module can only come out along the slots. Assembly is: take one end of the frame off (its uprights, crossbars and cubes come away as one piece), slide the five modules in, put the end back on. The end crossbars hold the row in place.

![Cross-section of a tube module in the frame: T-key ribs on the module's four corners slide in the MakerBeam T-slots, with a close-up of one rib's neck and head inside a slot](module-t-key-section.png "Section through a module at its front post. Each rib's head sits under the slot's lips, so it can only slide along the beam.")

That took the hardware list from v4's couple hundred fasteners to 24 M3 button heads for the corner cubes and 15 M2 self-tappers for the boards and PSUs.

The acrylic panels and the tube bracket follow the same rule. The front and two side openings are closed by 2 mm clear acrylic, its edges sitting in the beam slots that face each opening, with a notch at each corner to clear the cubes. The front panel slides in along the long beams with the modules, and each side panel goes in as its end of the frame goes back on. The assembly script prints the cut sizes.

![The front panel and two side panels laid out on an acrylic sheet, each with notched corners](acrylic-panels-sheet.png "The three acrylic panels nested on a sheet for cutting.")

The tube bracket keeps the tops of the tubes at the same 22 mm pitch as the grommets, and parallel. It sits at the tube tops, just above the glowing column: a ring around each tube, joined by a thin spine behind the tubes so as little as possible stands in front of them. Three crush ribs in each ring grip the glass, and nothing else holds it: it presses down over the tube tips last.

# Printable without supports
Every printed part had to print without supports:

- The module prints lying on its left face, so the C-profile is flat and every wall and rib grows straight up.
- Any face that would overhang is bevelled at 45°, and the screw bosses print as tapered teardrops.
- The stock MakerCube corners were replaced with a printed cube: 1.2 mm walls, a chamfered ceiling, and teardrop holes.
- The 220 mm frame beams can also be printed, as a 2 × 2 cluster with snap-off ties holding them straight during the print.

![The tube module on the print bed, lying on its left face: the C-profile flat, with the screw bosses, rest pads and T-key ribs all rising straight up](module-print-orientation.png "The module as it prints: left face down, teardrop bosses and grommet holes pointing up.")

# What's next
So far the only thing built is the quick prototype frame: a 3D print on aluminium extrusion, holding seven tubes to see them standing in a row, with no driver boards, PSUs, or wiring.

![Seven IN-9 tubes standing in a 3D-printed frame on aluminum extrusion](frame-seven-tubes-front.jpg "The prototype frame: a first test fit of the tubes.")

Next is replacing it with the real thing: printing the five modules and the corner cubes, getting the acrylic cut, and assembling and wiring the full ten-tube row.

Until then, the closest look at the finished display is a mock-up. FreeCAD screenshots are fine for checking clearances, but not for seeing what the display will look like lit. For that, Claude drove Blender through [Blender MCP](https://github.com/ahujasid/blender-mcp), which, like the FreeCAD addon, runs Python inside the live app.

1. **Export and import:** the assembly went out of FreeCAD as meshes and into Blender with materials by part: silver aluminium, dark PLA, rubber grommets, and the boards.
2. **Tubes:** rebuilt as clear glass with the exhaust tip and a nickel anode. That profile later went back into `tube_module.py`.
3. **Stills:** a low-sample test render before every change, which Claude checked before rendering the finals in Cycles.
4. **Glow:** after looking up photos of lit IN-9s, an orange-red core and halo per tube, with bloom in the compositor.
5. **Animation:** a sine wave keyframed along the row with an orbiting camera, rendered headless on the GPU and encoded with `ffmpeg`.

{{< video src="blender-sine-wave-orbit.mp4" poster="blender-sine-wave-orbit-poster.jpg" loop="true" caption="The Blender mock-up: a sine wave running along the ten tubes while the camera orbits the display." >}}

It isn't engineering, but a lit mock-up from the real CAD geometry is a cheap way to see if it's worth building before printing anything.

<!-- TODO: printing and assembling it for real, controller/firmware, wiring. -->
