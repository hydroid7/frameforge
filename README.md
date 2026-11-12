# FrameForge

Frameforge is a FreeCAD workbench that aids in designing beams and frames, with cut, mitter joins and so on.

Features:

- Create Beams (metal, wood) from a sketch or Edges
- Trim, offset, cut, mitter cut
- Cutout (make holes) from a sketch
- Export a BOM with cut angles, lenght, material


**Even if this project is used in production, mainly for [Microforge](https://www.microforge.fr/en) (bike trailers), please note that:**
- I am spending as much time as I can, but it is not enough
- **You can help me improving it by contributing via Ko-Fi**
- Also, I provide help, training, courses via my company [Inductive Brain](https://www.inductivebrain.fr/en)

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/L3L41KKMJR)

## Prerequisite

- FreeCAD ≥ v0.21.x

## Installation

Frameforge workbench can be installed via the [Addon Manager](https://wiki.freecad.org/Std_AddonMgr)

WARNING: When migrating from 0.1.x to 0.2.x, be careful to back-up your projects ! 
Internal data structures were modified, should be able to migrate, but in case...

## Quick Start

### Create the skeleton

Beams are mapped onto Edges or ParametricLine (from a Sketch for instance)

For a start, we are going to create a simple frame.

1. In a new file, switch to the Frameforge workbench.
2. Create a [sketch](https://wiki.freecad.org/Sketcher_NewSketch)  
3. A dialog will open asking you to 'Select orientation'. Choose the XY for instance.
3. Draw a simple square in the sketch... this will be our skeleton

![Create Skeleton](docs/images/02-create-frame-skeleton.png)

5. Close the Sketch edit mode.

### Create the frame

1. Launch the FrameForge Profile tool

![profile](docs/images/10-profiles.png)

2. It opens a FrameForge dialog (with options)

![profile](docs/images/10-profiles-task.png)
![profile](docs/images/10-profiles-task-2.png)

3. Select a profile from the lists (Material / Family / Size).  
*Note*: Size can be adjusted just below the family, the tool has a lot of predefined profiles. Same for parameters.

![profiles choice](docs/images/11-profiles-family.png)

4. In the 3D View, select edges to apply the profile creation:

![Edge Selection](docs/images/13-edge-selection.png)

5. Press OK in the Create Profile Task.  
**Result**: now there are four profiles!

![Profiles](docs/images/14-profiles-done.png)

![Zoom in profile](docs/images/14-zoom-on-profiles.png)

**Voila!** You have your first frame! For more information, follow the [tutorial](docs/tutorial.md)

## Tools

Select what the *Selection* column says, then launch the tool. Buttons stay greyed out until the selection matches.
Double-click a profile, trimmed profile or cutout in the tree to edit it later.

| Tool | Shortcut | Selection | Usage |
|---|---|---|---|
| Parametric Line | — | two vertices | Line that follows both vertices, usable as a profile path (e.g. a brace) |
| Create Profile | `Shift+S` | sketch edges, whole sketches, or nothing | Choose Material / Family / Size, alignment, rotation, mirror and pre-extend. Creates one beam per edge, or a free beam of the given length |
| Create Custom Profile | `Shift+C` | path edges or sketches | Press *Select Profile* and pick a closed section sketch. The beam is extruded along the path |
| Trim Profile | `M, C` | a face of the profile to trim, then faces of the boundary profiles | *Perfect fit* (coped cut around the boundary shape) or *Simple fit* (straight cut along the faces) |
| Create Miter Ends | `M, M` | one face on each of two profiles | Miters both profiles on the bisector. Their edges must share an endpoint. Run it again on the `_Mt` object for the other end |
| Extruded Cutout | `E, C` | a face of a profile + a closed sketch | Holes and slots, *Through All* or to a *Distance* |
| Attached Link | — | any object | Link to the top-level part, placed with the attachment editor. Counted in IDs and the BOM |
| Populate IDs / Reset IDs | `M, I` / — | profiles, links, Parts, Groups, Fusions | Numbers parts (`PID`): numbers or letters, identical parts can share an ID |
| Create Balloons / Refresh Balloons | `M, L` / — | one TechDraw view + parts / existing balloons | Balloons showing each part's ID. Refresh them after design changes |
| Create BOM | `M, B` | Parts, Groups, Fusions, profiles, links (optionally existing BOM and cut-list sheets first) | Spreadsheet with length, cut angles, weight and price, plus an optional cut list (stock length, kerf) |
| Recompute FrameForge objects | `M, R` | — | Recomputes all profiles, trims and cutouts, dependencies first |
| Export TechDraw pages | `M, T` | — | Exports every TechDraw page as a PDF |

Details for every option are in the [tools reference](docs/tools.md).

## Python API

To script FrameForge (create and edit profiles, trims, miters, cutouts, BOM...) from the FreeCAD console or macros, see the [Python API documentation](docs/python-api.md).

## Maintainer

Vivien HENRY
vivien.henry@inductivebrain.fr


## Credits

This workbench is based on [MetalWB](https://framagit.org/Veloma/freecad_metal_workbench)

Special thanks to:

- Vincent B
- Quentin Plisson
- rockn
- Jonathan Wiedemann

And others people that I don't know but they should be in this [thread](https://forum.freecad.org/viewtopic.php?style=5&t=72389)


## LICENSE

FrameForge is licensed under the [GPLv3](LICENSE)
