# FrameForge Python API

This page describes how to drive FrameForge from Python (the FreeCAD Python console, macros or other workbenches):
creating profiles, editing them, trimming, mitering, cutouts, links, IDs and BOMs.

FrameForge has no separate "public API" module. The API *is* the set of FeaturePython proxy classes the GUI commands use
(`Profile`, `TrimmedProfile`, `ExtrudedCutout`, ...). Every recipe below mirrors what the corresponding command does,
so objects created from Python behave exactly like objects created from the toolbar.

> **GUI required.** The proxies import `FreeCADGui`/`pivy` and their `execute()` methods touch `ViewObject`
> (e.g. to hide the trimmed body). Run these scripts inside the FreeCAD GUI (console or macro), or with
> `QT_QPA_PLATFORM=offscreen` for automation. `FreeCADCmd` alone is not supported.

## Contents

- [Object model](#object-model)
- [Profile catalog](#profile-catalog)
- [Creating profiles](#creating-profiles)
- [Editing profiles](#editing-profiles)
- [Custom profiles (from a sketch section)](#custom-profiles)
- [End miter](#end-miter)
- [End trim](#end-trim)
- [Extruded cutout](#extruded-cutout)
- [Parametric line](#parametric-line)
- [Links](#links)
- [IDs, BOM and cut list](#ids-bom-and-cut-list)
- [Helper functions](#helper-functions)
- [Complete example](#complete-example)

## Object model

All FrameForge objects are `Part::FeaturePython` documents objects with a Python `Proxy`:

| Object | Proxy / ViewProvider | Module | Created by command |
|---|---|---|---|
| Profile (beam) | `Profile` / `ViewProviderProfile` | `freecad.frameforge.profile` | `FrameForge_CreateProfiles` |
| Custom profile | `Profile` / `ViewProviderCustomProfile` | `freecad.frameforge.profile` | `FrameForge_CreateCustomProfiles` |
| Trimmed profile (trim or miter) | `TrimmedProfile` / `ViewProviderTrimmedProfile` | `freecad.frameforge.trimmed_profile` | `FrameForge_TrimProfiles`, `FrameForge_EndMiter` |
| Extruded cutout | `ExtrudedCutout` / `ViewProviderExtrudedCutout` | `freecad.frameforge.extruded_cutout` | `FrameForge_AddExtrudeCutout` |
| Parametric line | `ParametricLine` / `ParametricLineViewProvider` | `freecad.frameforge.parametric_line` | `FrameForge_ParametricLine` |
| Link | `App::Link` + extra props | `freecad.frameforge.create_link` | `FrameForge_Link` |

Operations stack: a `TrimmedProfile` wraps a `Profile` (or another `TrimmedProfile`/`ExtrudedCutout`), and an
`ExtrudedCutout` wraps any of these. The wrapped object is hidden and the wrapper carries the resulting shape. Each
wrapper copies the profile metadata (`Family`, `SizeName`, `Material`, `Width`, `Height`, `Length`, `Price`,
`ApproxWeight`, `CuttingAngleA/B`), so BOM tools can work on the outermost object.

The type helpers in `freecad.frameforge._utils` detect objects by their properties:
`is_profile(obj)`, `is_trimmedbody(obj)`, `is_extrudedcutout(obj)`, `is_link(obj)`, `is_part(obj)`, `is_group(obj)`,
`is_fusion(obj)`.

## Profile catalog

Standard sections are stored as JSON in `freecad/frameforge/resources/profiles/` (`metal.json`, `wood.json`,
`aluminium_extrusion.json`). The material name is the capitalized file name (`Metal`, `Wood`,
`Aluminium_extrusion`).

```json
{
  "Square Hollow": {
    "norm": "EN10210/10219",
    "unit": "Metric Units",
    "fillet": true,
    "sizes": {
      "20x2.0": {"Height": "20", "Width": "20", "Thickness": "2", "Radius1": "4", "Radius2": "2", "Weight": "1.05"}
    }
  }
}
```

The size name is the key in `sizes`. Possible size fields are `Height`, `Width`, `Thickness`, `Flange Thickness`,
`Radius1`, `Radius2` and `Weight` (linear weight in kg/m). Fields that are missing mean 0.

Loading it:

```python
import json, os
from freecad.frameforge.ff_tools import PROFILESPATH

def load_catalog():
    catalog = {}
    for f in os.listdir(PROFILESPATH):
        if f.endswith(".json"):
            with open(os.path.join(PROFILESPATH, f)) as fd:
                catalog[os.path.splitext(f)[0].capitalize()] = json.load(fd)
    return catalog

catalog = load_catalog()
print(list(catalog["Metal"]))                           # families
print(list(catalog["Metal"]["Square Hollow"]["sizes"]))  # sizes
```

Supported `Family` values (these drive the cross-section drawn in `Profile.execute`):

- Metal: `Equal Leg Angles`, `Unequal Leg Angles`, `Flat Sections`, `Square`, `Square Hollow`, `Rectangular Hollow`,
  `UPE`, `UPN`, `HEA`, `HEB`, `HEM`, `IPE`, `IPN`, `Round Bar`, `Pipe`
- Wood: `Flat Sections`, `Square`
- Aluminium extrusion: `V-Slot`, `T-Slot`, `T-Slot 3-Slots`, `T-Slot 2-Slots`, `T-Slot 2-Slots Opp`, `T-Slot 1-Slot`
  (`ProfileWidth`/`ProfileHeight` select the variant, e.g. 20x20, 20x40)
- `Custom Profile` (see [Custom profiles](#custom-profiles))

## Creating profiles

A profile is a `Part::FeaturePython` with the `Part::AttachExtensionPython` extension, attached `NormalToEdge` to a
sketch edge (or any edge, e.g. a `ParametricLine`), and the `Profile` proxy:

```python
import FreeCAD as App
from freecad.frameforge.profile import Profile, ViewProviderProfile

def make_profile(edge_owner, edge_name, material, family, size_name, name="Profile_000",
                 anchor_x=1, anchor_y=1, make_fillet=True, unit_price=0.0, length=0.0,
                 mirror_h=False, mirror_v=False, rotation=0.0, offset_a=0.0, offset_b=0.0):
    doc = App.ActiveDocument
    size = load_catalog()[material][family]["sizes"][size_name]
    val = lambda key: float(size.get(key, 0))

    obj = doc.addObject("Part::FeaturePython", name)
    obj.addExtension("Part::AttachExtensionPython")

    link_sub = None
    if edge_owner is not None:
        # put the profile next to its sketch (same Part / Body / Group)
        if edge_owner.Parents:
            edge_owner.Parents[-1][0].addObject(obj)
        obj.MapMode = "NormalToEdge"
        obj.AttachmentSupport = (edge_owner, edge_name)  # obj.Support on FreeCAD <= 0.21
        link_sub = (edge_owner, edge_name)
    obj.MapPathParameter = 1

    Profile(
        obj,
        val("Width"), val("Height"),
        val("Thickness"), val("Flange Thickness"),
        val("Radius1"), val("Radius2"),
        length,              # only used when there is no target edge
        val("Weight"),       # linear weight, kg/m
        unit_price,          # price per metre
        make_fillet,
        anchor_x, anchor_y,  # 0/1/2, see below
        material, family, size_name,
        False,               # bevels_combined, keep False
        link_sub,
        init_mirror_h=mirror_h, init_mirror_v=mirror_v,
        init_rotation=rotation,
        init_offset_a=offset_a, init_offset_b=offset_b,
    )
    ViewProviderProfile(obj.ViewObject)
    return obj

sketch = App.ActiveDocument.getObject("Sketch")
beams = [make_profile(sketch, f"Edge{i + 1}", "Metal", "Square Hollow", "20x2.0", name="Beam")
         for i in range(len(sketch.Shape.Edges))]
App.ActiveDocument.recompute()
```

To create a profile on every edge of a sketch, do what the GUI does when no sub-element is selected and iterate
`Edge1 … EdgeN`.

### `Profile(...)` arguments

| Positional | Property | Meaning |
|---|---|---|
| `init_w`, `init_h` | `ProfileWidth`, `ProfileHeight` | section size (mm) |
| `init_mt` | `Thickness` | wall/web thickness |
| `init_ft` | `ThicknessFlange` | flange thickness (I/H/U sections) |
| `init_r1`, `init_r2` | `RadiusLarge`, `RadiusSmall` | fillet radii |
| `init_len` | `ProfileLength` | length when the profile has no `Target` edge |
| `init_wg` | `LinearWeight` | kg/m (`ApproxWeight` is computed from it) |
| `init_unit_price` | `UnitPrice` | price per metre (`Price` is computed from it) |
| `init_mf` | `MakeFillet` | draw the fillets |
| `init_anchor_x` | `AnchorX` | 0 = `Left`, 1 = `Center`, 2 = `Right` (a legacy bool also works) |
| `init_anchor_y` | `AnchorY` | 0 = `Bottom`, 1 = `Center`, 2 = `Top` |
| `material`, `fam`, `size_name` | `Material`, `Family`, `SizeName` | catalog keys / labels |
| `bevels_combined` | — | `False` adds `BevelACutX/Y`, `BevelBCutX/Y`. `True` adds the unused `BevelStart*/End*` set. Use `False` |
| `link_sub` | `Target` | `(object, "EdgeN")`. Path the profile follows and length source |
| `custom_profile` | `CustomProfile` | sketch section for custom profiles |

| Keyword | Property | Meaning |
|---|---|---|
| `init_mirror_h`, `init_mirror_v` | `MirrorH`, `MirrorV` | flip the section in X / Y |
| `init_rotation` | `RotationAngle` | degrees around the path axis (bound by expression to `AttachmentOffset.Rotation.Angle`) |
| `init_offset_a`, `init_offset_b` | `OffsetA`, `OffsetB` | extend the beam at its start / end (mm, may be negative) |

The *Pre-extend* option in the dialog sets both offsets to `max(width, height)`. This helps when the beams will be
mitered or trimmed afterwards, because then the profiles overlap at the corners.

### Read-only/computed properties

`ProfileLength` (target edge length + offsets), `ApproxWeight`, `Price`, `Width`, `Height`, `Length`,
`CuttingAngleA`, `CuttingAngleB`, `Cutout`, plus `PID` (part ID) and `FrameforgeVersion`.

### A profile without an edge

Pass `edge_owner=None` / `link_sub=None` and a non-zero `init_len`. The profile is then a straight beam of that length
along the local Z axis, which you place with `obj.Placement`.

## Editing profiles

**Single properties:** set them and recompute. The geometry is rebuilt in `Profile.execute()`:

```python
b = App.ActiveDocument.getObject("Beam001")
b.AnchorX = "Left"        # "Left" | "Center" | "Right"
b.AnchorY = "Top"         # "Bottom" | "Center" | "Top"
b.RotationAngle = 90
b.MirrorH = True
b.OffsetA = 20            # extend start
b.OffsetB = -5            # shorten end
b.BevelACutY = 30         # manual bevels, clamped to [-60, 60] degrees
b.MakeFillet = False
b.UnitPrice = 12.5
App.ActiveDocument.recompute()
```

**Change the section (family/size):** use `Profile.set_properties`. The *Edit Profile* dialog uses it too:

```python
size = load_catalog()["Metal"]["Rectangular Hollow"]["sizes"]["40x20x2"]
v = lambda k: float(size.get(k, 0))
b.Proxy.set_properties(
    b,
    v("Width"), v("Height"), v("Thickness"), v("Flange Thickness"), v("Radius1"), v("Radius2"),
    b.ProfileLength, v("Weight"), b.UnitPrice, b.MakeFillet,
    1, 1,                                    # anchor_x, anchor_y
    "Metal", "Rectangular Hollow", "40x20x2",
    init_mirror_h=b.MirrorH, init_mirror_v=b.MirrorV,
    init_rotation=b.RotationAngle,
    init_offset_a=b.OffsetA, init_offset_b=b.OffsetB,
)
App.ActiveDocument.recompute()
```

`set_properties` overwrites **all** listed properties, and its keyword arguments default to "no mirror / no rotation
/ no offset". Pass the current values, as above, if you want to keep them. It also runs the compatibility migrations
for profiles from older files.

The target edge is `Target` (and `AttachmentSupport`). To move a profile to another edge, set both:

```python
b.AttachmentSupport = (sketch, "Edge3")
b.Target = (sketch, "Edge3")
```

To open the GUI editor from code, run `b.ViewObject.Proxy.edit()` (or `Gui.ActiveDocument.setEdit(b)`).

Wrap scripted edits in a transaction so that Undo works:

```python
doc = App.ActiveDocument
doc.openTransaction("Script edit")
...  # changes
doc.commitTransaction()
doc.recompute()
```

## Custom profiles

A custom profile extrudes a closed sketch (the section) along an edge. The section sketch should be drawn around its
origin. Width and height come from the sketch bounding box:

```python
from freecad.frameforge.profile import Profile, ViewProviderCustomProfile

def make_custom_profile(edge_owner, edge_name, section_sketch, material="Metal", name="Custom_000",
                        linear_weight=0.0, unit_price=0.0, length=0.0):
    obj = App.ActiveDocument.addObject("Part::FeaturePython", name)
    obj.addExtension("Part::AttachExtensionPython")
    if edge_owner.Parents:
        edge_owner.Parents[-1][0].addObject(obj)
    obj.MapMode = "NormalToEdge"
    obj.AttachmentSupport = (edge_owner, edge_name)
    obj.MapPathParameter = 1
    Profile(obj, 0, 0, 0, 0, 0, 0, length, linear_weight, unit_price, False, 1, 1,
            material, "Custom Profile", "None", False, (edge_owner, edge_name), section_sketch)
    ViewProviderCustomProfile(obj.ViewObject)
    return obj
```

## End miter

A miter is a `TrimmedProfile` with `TrimmedProfileType = "End Miter"`. Each profile of the corner needs its own
miter object, with the *other* profile as boundary:

```python
from freecad.frameforge.trimmed_profile import TrimmedProfile, ViewProviderTrimmedProfile

def make_end_miter(body, other):
    """body / other: Profile, or an already trimmed/mitered profile."""
    doc = App.ActiveDocument
    tp = doc.addObject("Part::FeaturePython", f"{body.Name}_Mt")
    if body.Parents:
        body.Parents[-1][0].addObject(tp)
    TrimmedProfile(tp)
    ViewProviderTrimmedProfile(tp.ViewObject)
    tp.TrimmedBody = body
    tp.TrimmingBoundary = [(other, ["Face1"])]  # the sub-element is not used for miters
    tp.TrimmedProfileType = "End Miter"
    return tp

a, b = beams[0], beams[1]
make_end_miter(a, b)
make_end_miter(b, a)
App.ActiveDocument.recompute()
```

The command class has the same helper:
`CreateEndMiterCommand().make_end_miter_profile(a, [(b, ["Face1"])])` from
`freecad.frameforge.create_end_miter_tool`.

How it works and what it needs:

- The two profiles' target edges must **share an endpoint**. Otherwise the recompute fails with
  `End Miter: edges not aligned`.
- The cut plane bisects the angle between the two edges. With identical anchors it passes through the common
  vertex. With different `AnchorX`/`AnchorY` it is placed at the center of the profiles' intersection, so the profiles
  must overlap (use offsets or *pre-extend*). Otherwise you get `End Miter: profiles don't intersect`.
- A beam with corners at both ends is mitered twice by **chaining**: miter the first miter object at the other end.

```python
# Rectangle of 4 beams, miter every corner
n = len(beams)
ends = list(beams)                  # current outermost object of each beam
for i in range(n):
    j = (i + 1) % n
    ends[i] = make_end_miter(ends[i], beams[j])
    ends[j] = make_end_miter(ends[j], beams[i])
App.ActiveDocument.recompute()
```

The boundary can be the raw profile. Only its target edge and its anchors are used.

After recompute, `tp.CuttingAngleA` / `tp.CuttingAngleB` contain readable angles (e.g. `"@ 45.0"`) and `tp.Length`
the length of the cut part.

## End trim

An end trim is a `TrimmedProfile` with `TrimmedProfileType = "End Trim"`. It cuts the trimmed body against one or
more boundaries:

```python
def make_end_trim(body, boundaries, cut_type="Perfect fit"):
    """boundaries: list of (object, ["FaceN", ...])"""
    tp = App.ActiveDocument.addObject("Part::FeaturePython", f"{body.Name}_Tr")
    if body.Parents:
        body.Parents[-1][0].addObject(tp)
    TrimmedProfile(tp)
    ViewProviderTrimmedProfile(tp.ViewObject)
    tp.TrimmedBody = body
    tp.TrimmingBoundary = boundaries
    tp.TrimmedProfileType = "End Trim"
    tp.CutType = cut_type
    return tp

tp = make_end_trim(beams[3], [(beams[2], ["Face1"])])
App.ActiveDocument.recompute()
```

| `CutType` | Behaviour |
|---|---|
| `Perfect fit` | splits the body with the **whole shape** of each boundary object and removes the pieces that do not contain the body's center of gravity (coped cut). Sub-elements are ignored |
| `Simple fit` | for each listed **planar face**, removes the half-space on the outside of the face (straight cut) |

`TrimmingBoundary` is a `PropertyLinkSubList`. To change it, assign a new list. Appending in place does not work:

```python
tp.TrimmingBoundary = list(tp.TrimmingBoundary) + [(beams[0], ["Face3"])]
```

## Extruded cutout

Extrude a sketch through a profile (holes, slots…). `baseObject` is the face that gives the cut direction (cuts go
along the negative face normal):

```python
from freecad.frameforge.extruded_cutout import ExtrudedCutout, ViewProviderExtrudedCutout

def make_cutout(target, face_name, sketch, cut_type="Through All", length=None):
    obj = App.ActiveDocument.addObject("Part::FeaturePython", f"{target.Name}_Ex")
    obj.addExtension("Part::AttachExtensionPython")
    if target.Parents:
        part = target.Parents[-1][0]
        part.addObject(obj)
        part.addObject(sketch)
    ExtrudedCutout(obj, sketch, [target, face_name])
    ViewProviderExtrudedCutout(obj.ViewObject)
    obj.CutType = cut_type  # "Through All" | "Distance"
    if length is not None:
        obj.ExtrusionLength = length  # only used with "Distance"
    obj.recompute()
    return obj
```

The target can be a profile, a trimmed profile or another cutout. Each closed wire of the sketch becomes a face of the
cut.

## Parametric line

A straight edge between two vertices that follows them when they move. Use it as a profile path where there is no
sketch:

```python
from freecad.frameforge.parametric_line import ParametricLine, ParametricLineViewProvider

line = App.ActiveDocument.addObject("Part::FeaturePython", "ParametricLine")
ParametricLine(line)
ParametricLineViewProvider(line.ViewObject)
line.Vertex1 = (beams[0], "Vertex1")
line.Vertex2 = (beams[2], "Vertex5")
App.ActiveDocument.recompute()

make_profile(line, "Edge1", "Metal", "Square", "20x20", name="Brace")
```

## Links

`freecad.frameforge.create_link.makeLink(source)` creates an `App::Link` to `source` with an attach extension
(`MapMode = "Deactivated"`) and `PID` / `FrameforgeVersion` properties, so that the link shows up in IDs and the BOM:

```python
from freecad.frameforge.create_link import makeLink
link = makeLink(App.ActiveDocument.getObject("Part"))
link.Placement.Base = App.Vector(0, 1000, 0)
```

## IDs, BOM and cut list

```python
from freecad.frameforge._utils import get_profiles_and_links_from_document
from freecad.frameforge.populate_ids import populate_ids

profiles, links = get_profiles_and_links_from_document()
populate_ids(
    profiles, links,        # objects to number
    profiles, links,        # all objects of the document (to avoid duplicates)
    numbering_type="all_numbers",   # | "all_letters" | "number_for_profiles_letters_for_links"
                                    # | "letters_for_profiles_number_for_links"
    allow_duplicated=False,
    group_ids_for_identical=True,   # identical parts share one ID
    include_part_count_in_pid=False,
    reset_existing=True,
    numbering_scheme="fill_document",  # | "fill_selection" | "continue_document" | "start_at"
    start_number="1",
    start_letter="A",
)
```

`populate_ids` writes the `PID` property of each object. Older links may lack `PID`. `PopulateIDsTaskPanel.accept()`
adds it before numbering.

BOM and cut list (what the *Create BOM* dialog does):

```python
from collections import defaultdict
from freecad.frameforge.create_bom import traverse_assembly, group_profiles, group_links, make_bom, make_cut_list
from freecad.frameforge.best_fit import CutPart, best_fit_decreasing

root = App.ActiveDocument.getObject("Part")
profiles_data, links_data = [], []
traverse_assembly(profiles_data, links_data, root, full_parent_path=False)

make_bom(group_profiles(profiles_data), group_links(links_data), bom_name="Frame_BOM")

stock_length, kerf = 6000.0, 3.0
groups = defaultdict(list)
for p in profiles_data:
    groups[(p["family"], p["material"], p["size_name"])].append(p)
stocks = {
    f"{mat}_{fam}_{size}": best_fit_decreasing(stock_length, [CutPart(p["label"], float(p["length"]), kerf, p) for p in g])
    for (fam, mat, size), g in groups.items()
}
make_cut_list(stocks, cutlist_name="Frame_CutList")
App.ActiveDocument.recompute()
```

`traverse_assembly` accepts Parts, Groups, Part Fusions, profiles, trimmed profiles, cutouts and links. Both
`make_bom` and `make_cut_list` accept an existing `spreadsheet=` to overwrite.

## Helper functions

In `freecad.frameforge._utils`:

| Function | Purpose |
|---|---|
| `get_profiles_and_links_from_document()` | `(profiles, links)`: every `Profile` and link object of the active document |
| `get_profiles_and_links_from_object(profiles, links, obj)` | collects recursively below `obj` (Part, Group, Fusion). Inside a Part only visible objects count, i.e. the outermost trimmed/cutout object |
| `get_profile_from_trimmedbody(obj)` | the base `Profile` under a trimmed profile chain |
| `get_profile_from_extrudedcutout(obj)` | the base `Profile` under a cutout |
| `get_trimmedprofile_from_extrudedcutout(obj)` | the trimmed profile under a cutout |
| `get_trimmed_profile_all_cutting_angles(tp)` | all cut angles of a trimmed profile chain |
| `length_along_normal(obj)` | cut length of a (trimmed) profile along its axis |
| `normalize_anchor(val)` | bool/int → anchor index 0–2 |

GUI commands (for `Gui.runCommand(name)`, which works on the current selection like a toolbar click):
`FrameForge_CreateProfiles`, `FrameForge_CreateCustomProfiles`, `FrameForge_TrimProfiles`, `FrameForge_EndMiter`,
`FrameForge_AddExtrudeCutout`, `FrameForge_ParametricLine`, `FrameForge_Link`, `FrameForge_PopulateIDs`,
`FrameForge_ResetIDs`, `FrameForge_CreateBalloons`, `FrameForge_RefreshBalloons`, `FrameForge_CreateBOM`,
`FrameForge_RecomputeFrameForgeObjects`, `FrameForge_ExportTechDraw`.

## Complete example

A mitered 1000 × 500 rectangular frame, which you can paste into the FreeCAD Python console. It uses `load_catalog`,
`make_profile` and `make_end_miter` from above:

```python
import FreeCAD as App, Part, Sketcher

doc = App.newDocument("Frame")
sk = doc.addObject("Sketcher::SketchObject", "Skeleton")
V = App.Vector
pts = [V(0, 0, 0), V(1000, 0, 0), V(1000, 500, 0), V(0, 500, 0)]
for i in range(4):
    sk.addGeometry(Part.LineSegment(pts[i], pts[(i + 1) % 4]))
doc.recompute()

doc.openTransaction("Scripted frame")
beams = [make_profile(sk, f"Edge{i + 1}", "Metal", "Square Hollow", "20x2.0", name=f"Beam{i + 1}")
         for i in range(4)]
ends = list(beams)
for i in range(4):
    j = (i + 1) % 4
    ends[i] = make_end_miter(ends[i], beams[j])
    ends[j] = make_end_miter(ends[j], beams[i])
doc.commitTransaction()
doc.recompute()

for e in ends:
    print(e.Label, e.Length, e.CuttingAngleA, e.CuttingAngleB)
```
