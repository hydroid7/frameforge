# FrameForge Tools Reference

This page lists every FrameForge tool: what to select, the dialog options and what the tool creates. For a step-by-step
walkthrough see the [tutorial](tutorial.md). To do the same from Python, see the [Python API](python-api.md).

Most tools check the current selection. A toolbar button stays greyed out until the selection matches the
**Selection** line of that tool.

| Toolbar | Tool | Shortcut | Command name |
|---|---|---|---|
| Drawing Primitives | [Parametric Line](#parametric-line) | — | `FrameForge_ParametricLine` |
| Frameforge | [Create Profile](#create-profile) | `Shift+S` | `FrameForge_CreateProfiles` |
| Frameforge | [Create Custom Profile](#create-custom-profile) | `Shift+C` | `FrameForge_CreateCustomProfiles` |
| Frameforge | [Trim Profile](#trim-profile) | `M, C` | `FrameForge_TrimProfiles` |
| Frameforge | [Create Miter Ends](#create-miter-ends) | `M, M` | `FrameForge_EndMiter` |
| Frameforge | [Extruded Cutout](#extruded-cutout) | `E, C` | `FrameForge_AddExtrudeCutout` |
| Part Primitives | [Attached Link](#attached-link) | — | `FrameForge_Link` |
| Frameforge output | [Populate IDs / Reset IDs](#populate-ids--reset-ids) | `M, I` / — | `FrameForge_PopulateIDs`, `FrameForge_ResetIDs` |
| Frameforge output | [Create / Refresh Balloons](#create--refresh-balloons) | `M, L` / — | `FrameForge_CreateBalloons`, `FrameForge_RefreshBalloons` |
| Frameforge output | [Create BOM](#create-bom) | `M, B` | `FrameForge_CreateBOM` |
| Frameforge utilities | [Recompute FrameForge objects](#recompute-frameforge-objects) | `M, R` | `FrameForge_RecomputeFrameForgeObjects` |
| Frameforge utilities | [Export TechDraw pages](#export-techdraw-pages) | `M, T` | `FrameForge_ExportTechDraw` |

The toolbars also contain the standard FreeCAD *New Sketch*, *Group*, *Part*, *Part Fuse*, *Part Cut* and
*PartDesign Body* commands.

## Parametric Line

A straight line between two vertices that follows them when they move. Use it as a profile path where you don't have
a sketch, e.g. a diagonal brace between two existing profiles.

- **Selection:** exactly two vertices (from any objects).
- **Result:** a `ParametricLine` object. Select its edge as the path in *Create Profile*.

## Create Profile

Creates standard beams (metal, wood, aluminium extrusion) along edges.

- **Selection:** edges of sketches or parametric lines, or whole sketches (then every edge is used). You can also
  change the selection while the dialog is open. With nothing selected, the profile is a free beam of the given
  *Length*.
- **Profile:** choose *Material → Family → Size* from the catalog. The *Norm* and *Unit* fields and the picture
  update with the choice. The *Dimensions* (height or diameter, width, thicknesses, radii, weight in kg/m) are filled
  from the catalog, and you can still change them. *Unit Price* is per metre and is used for the BOM price.
- **Profiles Options**
  - *Make Fillets*: draw the rolled radii (only for families that have them).
  - *Pre-extend*: extends both ends by `max(W, H)` (`OffsetA`/`OffsetB`), so that beams overlap at corners before you
    miter or trim them.
  - *Path alignment*: a 3×3 anchor grid that sets where the edge goes through the section (`AnchorX`/`AnchorY`).
  - *Mirror horizontally / vertically*, *Rotation (°)*: orient the section around the path.
  - *Combined Bevels*: legacy bevel properties. Leave it unchecked.
- **Settings**
  - *In place*: the profiles go next to their sketch (same Part/Body).
  - *Create a Part*: puts the sketch(es) and the new profiles in a new `Part`.
  - *Create a Group*: not implemented yet, behaves like *In place*.
  - Naming: build the object name from a `Profile` prefix, the sketch name, the family and the size.
- **Result:** one `Profile` object per edge. The preview updates live, and *Cancel* removes it.

**Editing:** double-click a profile in the tree. This opens the same dialog, but the attachment cannot be changed.
For finer control you can edit the properties directly in the property editor: anchors, offsets, rotation, mirror,
and the manual bevels `BevelACutY/X`, `BevelBCutY/X` (±60°).

## Create Custom Profile

Like *Create Profile*, but the cross-section is your own sketch.

- **Selection:** the path edges or sketches (as above).
- **Dialog:** press *Select Profile* and pick the section sketch (a closed sketch drawn around its origin). Then set
  *Material*, *Weight (kg/m)*, *Unit Price*, and *Length* if there is no path.
- **Result:** profiles with `Family = "Custom Profile"`. Width and height are taken from the section's bounding box.
  In the BOM the family column shows the section sketch's label.

## Trim Profile

Cuts the end of a profile against other profiles (T-joints, butt joints).

- **Selection:** nothing, or a face of the profile to trim, optionally followed by faces of the boundary profiles.
  Only profiles and trimmed profiles can be selected, so to trim a cutout, trim before you cut.
- **Dialog**
  - *Trim Type*
    - *Perfect fit* (coped cut): the profile is split by the *whole* boundary shape and the parts away from its
      center are removed. Use it for profiles that fit around a round or shaped tube.
    - *Simple fit*: straight cut along the selected planar face(s).
  - *Trimmed Object → Add*: set the profile to trim from the current selection.
  - *Trimming Boundaries → Add / Remove*: add the selected faces as boundaries, or remove the highlighted entries.
- **Result:** a `<name>_Tr` trimmed profile that replaces the original in the tree (the original is hidden).
  Double-click it to edit. The cut length and angles appear in its `Length` and `CuttingAngleA/B` properties.

## Create Miter Ends

Miters two profiles that meet at a corner.

- **Selection:** one face on each of two profiles (or trimmed/mitered profiles). Cutouts can only be mitered from
  Python, see the [Python API](python-api.md#end-miter).
- **Result:** two `<name>_Mt` objects, one for each profile, each cut on the bisector plane.
- **Notes**
  - The two path edges must share an endpoint.
  - If the profiles use different path alignments, the cut goes through the center of their overlap. They must then
    overlap, so use *Pre-extend* or offsets.
  - For a profile with corners at both ends, run the tool again on its `_Mt` object and the next profile. The
    miters then chain up.

## Extruded Cutout

Cuts holes and slots through a profile using a sketch.

- **Selection:** a face of the profile (or trimmed profile, or another cutout) and a closed sketch, in either order.
- **Dialog:** *Cut type* is *Through All* or *Distance* (with a *Length*). The cut goes into the selected face, along
  the reverse of its normal.
- **Result:** a `<name>_Ex` object. The BOM marks the part with *Cutout = Yes*.

## Attached Link

Creates an `App::Link` with an attachment, to reuse a part or sub-frame several times.

- **Selection:** any object(s). The link points to the top-level container (root object) of each selected object.
- **Result:** a `<label>_Link` object, then the attachment editor opens so that you can place it. Links get a `PID`
  and appear in IDs and the BOM.

## Populate IDs / Reset IDs

Gives each profile and link a part ID (`PID`). These IDs are used by balloons and the BOM.

- **Selection:** profiles, trimmed profiles, cutouts, links, or Parts/Groups/Fusions that contain them.
- **Options:** *Numbering Type* (numbers, letters, or mixed for profiles and links), *Allow duplicated*, *Group
  profiles ID if identical* (identical parts share one ID), *Include Part count in PID*, *Reset existing selection
  numbering*, and *Numbering Scheme*: fill gaps in the selection or in the document, continue after the highest ID, or
  start at a given number/letter (*First indexes*). The *Profiles Prefix* / *Links Prefix* fields are not used yet.
- *Reset IDs* clears the `PID` of the selected objects.

## Create / Refresh Balloons

Places TechDraw balloons with the `PID` of each part.

- **Create — selection:** exactly one TechDraw view (`DrawProjGroupItem`) and the profiles/links (or their
  containers) to annotate. You get one balloon per part, at its center of gravity.
- **Refresh — selection:** existing FrameForge balloons. Their anchor points and texts are updated after design
  changes.

## Create BOM

Writes a bill of materials spreadsheet and, optionally, a cut list.

- **Selection:** Parts, Groups, Fusions, profiles or links. To update existing spreadsheets instead of creating new
  ones, first select the BOM sheet, then the cut-list sheet, then the objects.
- **BOM options:** *BOM Name*, *Full Parent Path* (show the whole container path), *Include Links*, *Group Profiles /
  Links* (one line per identical part, with a quantity).
- **Cut List options:** *Generate Cut List*, *Stock Material Length* and *Kerf*. The parts are packed into stock bars
  (best-fit decreasing), per material/family/size.
- **Notes:** only *visible* objects inside a Part are counted, so hide what should not be in the BOM. Columns: ID,
  label, family, size, material, length, cut angles, cutout, weight, price, quantity.

## Recompute FrameForge objects

Recomputes every profile, trimmed profile and cutout, dependencies first. This helps after opening older files or
when a chain of trims or miters is out of date.

## Export TechDraw pages

Exports every TechDraw page of the document as a PDF into a folder that you choose. For unsaved documents the
FreeCAD user data folder is used.
