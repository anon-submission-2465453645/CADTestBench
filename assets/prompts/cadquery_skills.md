# CadQuery Coding Skill for LLMs

> **Purpose**: This document distills the CadQuery documentation into actionable, dense guidance for LLMs generating CadQuery code. Read this fully before writing any CadQuery script.

---

## 1. What is CadQuery?

CadQuery is a Python library for parametric 3D CAD modelling built on top of the OpenCASCADE (OCCT) kernel. It uses a **fluent (method-chaining) API** inspired by jQuery. All dimensions are in millimetres by default.

```python
import cadquery as cq
result = cq.Workplane("XY").box(10, 20, 5).faces(">Z").hole(3)
```

---

## 2. Core Mental Model

### The Four API Layers (use the highest level that works)

| Layer | When to use | Entry point |
|---|---|---|
| **Fluent API** | Most models — default choice | `cq.Workplane(...)` |
| **Sketch API** | Complex 2D profiles, constrained geometry | `cq.Sketch()` |
| **Direct API** | Face-level work, things fluent can't do | `cq.Shape`, `cq.Solid`, etc. |
| **OCCT (OCP) API** | Maximum control, very verbose | `import OCP` |

**Always start with the Fluent API.** Drop down only when you need something it cannot express.

### The Workplane Stack

The `Workplane` object is a **stack machine**:
- Every method call returns a **new** `Workplane` (the old one becomes its `.parent`).
- The stack holds the current selection: Shapes, Vectors, or Locations.
- Methods consume objects on the stack and/or push new ones.
- Chain is searched upward (via `.parent`) to find the "context solid" for boolean operations.

```python
part = cq.Workplane("XY").box(1, 2, 3).faces(">Z").vertices().circle(0.5).cutThruAll()
```

**Always chain calls inline.** The chain is the model — each method returns a new `Workplane` and the whole expression is the result. Use parentheses to wrap long chains across lines without breaking the chain:

```python
part = (
    cq.Workplane("XY")
    .box(1, 2, 3)
    .faces(">Z").vertices().circle(0.5).cutThruAll()
)

---

## 3. Workplane Construction

```python
# Standard named planes: "XY", "YZ", "XZ", or aliases:
# "front"="XY", "back"="-XY", "top"="XZ", "bottom"="-XZ", "left"="YZ", "right"="-YZ"
cq.Workplane("XY")
cq.Workplane("front")

# With explicit origin:
cq.Workplane("XY", origin=(0, 0, 5))

# From a plane object:
cq.Workplane(cq.Plane(origin=(0,0,0), xDir=(1,0,0), normal=(0,0,1)))

# Pivot to a new workplane from the top face of existing solid:
result.faces(">Z").workplane()

# Workplane offset from a face:
result.faces(">Z").workplane(offset=2)

# Rotated workplane:
result.faces(">Z").workplane().transformed(offset=cq.Vector(0, -1.5, 1.0), rotate=cq.Vector(60, 0, 0))

# Copy workplane from another object (useful for cross-axis extrusions):
result.copyWorkplane(cq.Workplane("right", origin=(-5, 0, 0)))
```

**Key rule**: When you call `.faces(...).workplane()`, the origin of the new workplane is the **projection of the previous origin onto that face** — NOT necessarily the face center. Use `centerOption="CenterOfMass"` or `centerOption="ProjectedOrigin"` to control this.

---

## 4. Selector Reference

Selectors filter Faces, Edges, Vertices, or Wires. Pass them as strings to `.faces()`, `.edges()`, `.vertices()`, `.wires()`.

### Direction / Min-Max Selectors

| String | Meaning |
|---|---|
| `>Z` | Face/edge with center of mass at **maximum Z** |
| `<Z` | Face/edge with center of mass at **minimum Z** |
| `>X`, `<X`, `>Y`, `<Y` | Same for X and Y |
| `>Z[-2]` | Second-to-last face sorted by Z (Nth selector) |
| `<Z[0]` | First from bottom |

### Parallel / Perpendicular / Aligned

| String | Meaning |
|---|---|
| `\|Z` | Parallel to Z axis (vertical faces/edges) |
| `\|X`, `\|Y` | Parallel to X or Y |
| `#Z` | Perpendicular to Z |
| `+Z` | Faces with normal pointing in +Z direction |
| `-Z` | Faces with normal pointing in -Z direction |

### Type Selector

| String | Meaning |
|---|---|
| `%PLANE` | Planar faces |
| `%CYLINDER` | Cylindrical faces |
| `%CONE`, `%SPHERE`, `%TORUS` | Other face types |
| `%LINE` | Straight edges |
| `%CIRCLE` | Circular edges |
| `%ELLIPSE` | Elliptical edges |

### Logical Combinations

```python
.edges("|Z or <Z")          # parallel-to-Z OR bottom edges
.edges("not(<X or >X or <Y or >Y)")  # exclude four side edges
.faces(">Z[-2]")            # second face from top
```

### Custom Vector Direction

```python
result.edges(">(-1, 1, 0)").chamfer(1)   # edge in custom direction
```

### Topological Relations

```python
# Select all faces sharing edges with the current selection:
result.faces(">Z").edges("<Y")
result.ancestors("Face")    # all faces containing current selection
result.siblings("Edge")     # all edges connected to current selection
```

### NthSelector with Area

```python
from cadquery import selectors
result.faces(">Z").wires(selectors.AreaNthSelector(0))   # smallest wire
result.faces(">Z").wires(selectors.AreaNthSelector(-1))  # largest wire
```

### Nearest to Point

```python
from cadquery import selectors
result.vertices(selectors.NearestToPointSelector((0, 1, 0)))
```

### Tagging (reference earlier stack states by name)

```python
result = (
    cq.Workplane("XY")
    .box(4, 4, 1)
    .faces(">Z").tag("top")      # tag this workplane state
    .workplane()
    .circle(1).extrude(2)
    .workplaneFromTagged("top")  # jump back to tagged state
    .circle(0.5).extrude(1)
)
```

---

## 5. 2D Sketch Operations (Fluent API)

These create **pending wires** that are later consumed by 3D operations.

```python
# Primitives (centered on current workplane origin by default):
.rect(width, height)
.circle(radius)
.polygon(nSides, diameter)
.ellipse(x_radius, y_radius)
.slot2D(length, diameter, angle=0)

# Point and movement:
.center(x, y)           # move workplane origin in 2D
.moveTo(x, y)           # move pen
.move(dx, dy)           # move pen relative

# Lines and arcs:
.lineTo(x, y)
.line(dx, dy)
.hLine(distance)        # horizontal line by distance
.vLine(distance)        # vertical line by distance
.hLineTo(xCoord)        # horizontal line to absolute x
.vLineTo(yCoord)        # vertical line to absolute y
.threePointArc((x1, y1), (x2, y2))
.sagittaArc((x, y), sag)
.radiusArc((x, y), radius)
.tangentArcPoint((x, y))
.spline([(x1,y1), (x2,y2)], tangents=None)
.close()                # close the current wire back to start

# Construction geometry (not turned into solids):
.rect(w, h, forConstruction=True)
.circle(r, forConstruction=True)

# Multiple points / arrays:
.pushPoints([(x1,y1), (x2,y2)])   # push a list of points onto the stack
.rArray(xSpacing, ySpacing, xCount, yCount)  # rectangular array
.cskArray(radius, startAngle, angle, count)   # polar/circular array

# Offset 2D wires:
.offset2D(distance, kind="arc")   # kind: "arc", "intersection", "tangent"

# Mirror:
.mirrorX()   # mirror about X axis
.mirrorY()   # mirror about Y axis
```

---

## 6. 3D Primitives (Fluent API)

```python
# All placed relative to the current workplane origin:
.box(length, width, height, centered=True)
.sphere(radius)
.cylinder(height, radius, direct=(0,0,1), angle=360)
.cone(height, radius1, radius2)
.wedge(dx, dy, dz, xmin, zmin, xmax, zmax)
.text("Hello", fontsize, distance)       # 3D extruded text

# Centered parameter controls placement:
# centered=True  → box centered at origin (default)
# centered=False → box starts at origin, extends in +X +Y +Z
# centered=(True, True, False) → per-axis control
```

---

## 7. 3D Operations (Fluent API)

### Extrude

```python
.extrude(distance)
.extrude(distance, taper=10)        # tapered extrude (degrees)
.extrude(distance, combine=True)    # fuse with existing solid (default True)
.extrude(distance, combine="cut")   # subtract instead
.twistExtrude(distance, angleDegrees)
```

### Cut / Remove

```python
.cutBlind(distance)                 # cut downward into solid
.cutThruAll()                       # cut through entire solid
.hole(diameter, depth=None)         # simple through-hole
.cboreHole(diameter, cboreDiameter, cboreDepth, depth=None)   # counterbored
.cskHole(diameter, cskDiameter, cskAngle, depth=None)          # countersunk
```

### Revolve

```python
.revolve(angleDegrees=360, axisStart=(0,0,0), axisEnd=(0,1,0))
```

### Sweep

```python
# Wire on stack is profile, path defined separately:
.sweep(path, multisection=False, makeSolid=True, isFrenet=False)

# path can be a wire from the workplane
path = cq.Workplane("XZ").spline([(0,0,0),(0,5,5),(0,10,0)])
profile = cq.Workplane("XY").circle(1)
result = profile.sweep(path)
```

### Loft

```python
# Multiple cross-sections (push each via placeSketch or pendingWires):
result = cq.Workplane("XY").circle(5).workplane(offset=10).circle(3).loft()
result = cq.Workplane("XY").circle(5).workplane(offset=10).rect(4,4).loft(ruled=True)
```

### Shell / Offset

```python
.shell(thickness)           # hollow out, positive = outward, negative = inward
.shell(thickness, faceSelector)  # open shell (removes selected face first)
.offset2D(distance)         # offset a 2D wire
```

### Boolean Operations

```python
.union(other_workplane)
.cut(other_workplane)
.intersect(other_workplane)

# Combine with add():
result = part_a.add(part_b)
```

### Edge Modifications

```python
# Fillet (rounds edges):
.fillet(radius)                     # fillet all edges on stack
.faces(">Z").edges().fillet(0.5)    # fillet edges of top face

# Chamfer (angled cuts):
.chamfer(length)
.chamfer(length, length2)           # asymmetric chamfer
.faces(">Z").edges().chamfer(0.5)
```

### Mirror and Transform

```python
.mirror("XY")                               # mirror about XY plane
.mirror(mirrorPlane="XY", basePointVector=(0,0,5))
.rotateAboutCenter((1,0,0), 90)             # rotate about axis through bounding box center
.translate((x, y, z))
.rotate((0,0,0), (0,0,1), 45)              # rotate: start, end, degrees
```

### Split

```python
.split(keepTop=True, keepBottom=False)   # split solid at current workplane
```

---

## 8. The Sketch API (for complex profiles)

Use `cq.Sketch()` when the Fluent 2D tools are insufficient.

```python
import cadquery as cq

s = (
    cq.Sketch()
    .trapezoid(4, 3, 90)           # width, height, angle
    .vertices()
    .circle(0.5, mode="s")         # subtract circles at each vertex
    .reset()                        # clear selection
    .vertices()
    .fillet(0.25)
    .reset()
    .rarray(0.6, 1, 5, 1)          # rectangular array
    .slot(1.5, 0.4, mode="s", angle=90)  # slotted holes
)

result = cq.Workplane("XY").placeSketch(s).extrude(2)
```

### Sketch Modes

Every face operation accepts `mode=`:
- `"a"` — fuse (default, additive)
- `"s"` — subtract (cut)
- `"i"` — intersect
- `"r"` — replace
- `"c"` — construction (store only; must provide `tag=`)

### Sketch Operations

```python
.segment((x1,y1), (x2,y2), tag="s1")     # line segment
.arc((x1,y1), (xm,ym), (x2,y2), tag="a1") # 3-point arc
.rect(w, h, angle=0)
.circle(r)
.ellipse(a, b)
.trapezoid(w, h, angle)
.polygon(n, d)
.hull()                                    # convex hull of current geometry
.fillet(r)
.chamfer(l)
.offset(d)
.assemble(tag="face")                      # convert edges to face-based representation
.constrain("s1", "a1", "Angle", 45)       # geometric constraint (experimental)
.solve()                                   # solve constraints
.face(other_sketch, mode="s")             # boolean with another sketch
```

### Sketch for Loft

```python
s1 = cq.Sketch().trapezoid(3, 1, 110).vertices().fillet(0.2)
s2 = cq.Sketch().rect(2, 1).vertices().fillet(0.2)
result = cq.Workplane("XY").placeSketch(s1, s2.moved(z=3)).loft()
```

---

## 9. Stack Navigation and Access

```python
# Get the last shape (exits fluent chain to get a Shape object):
shape = result.val()             # last item on stack
shapes = result.vals()           # all items on stack

# Get the context solid (the main body being built):
solid = result.findSolid()       # returns Solid or Compound

# Stack navigation:
result.first()                   # first item on stack
result.last()                    # last item on stack
result.end(n=1)                  # go up n levels in parent chain
result.all()                     # all objects in a list

# Size:
result.size()                    # number of items on stack

# Parent access:
result.parent                    # the previous Workplane in chain
```

---

## 10. Common Patterns and Idioms

### Bolt Hole Pattern

```python
result = cq.Workplane("XY").box(40, 20, 5).faces(">Z").workplane().rect(30, 10, forConstruction=True).vertices().hole(3)
```

### Array of Features

```python
result = cq.Workplane("XY").box(50, 10, 5).faces(">Z").workplane().rarray(10, 1, 5, 1).hole(2)
```

### Counterbored and Countersunk Holes

```python
# Counterbored:
result = cq.Workplane("XY").box(40, 20, 5).faces(">Z").workplane().rect(30, 10, forConstruction=True).vertices().cboreHole(2.4, 4.4, 2.1)

# Countersunk:
result = cq.Workplane("XY").box(40, 20, 5).faces(">Z").workplane().rect(30, 10, forConstruction=True).vertices().cskHole(3.2, 6.4, 82)
```

### Shell (Hollow Box)

```python
result = cq.Workplane("XY").box(20, 20, 20).faces(">Z").shell(-2)  # 2mm wall, open top
```

### Tapered Extrude / Draft Angle

```python
result = cq.Workplane("XY").rect(20, 20).extrude(10, taper=5)   # 5° draft angle
```

### Using `.each()` for Custom Operations

```python
def make_boss(loc):
    return cq.Workplane("XY").circle(2).extrude(3).val().located(loc)

result = cq.Workplane("XY").box(40, 40, 5).faces(">Z").workplane().rarray(15, 15, 2, 2).each(make_boss, useLocalCoordinates=True)
```

### Parametric Model Template

```python
import cadquery as cq

# --- Parameters ---
length = 80.0
width  = 60.0
height = 10.0
hole_d = 4.0
margin = 8.0

# --- Model ---
result = cq.Workplane("XY").box(length, width, height).faces(">Z").workplane().rect(length - 2*margin, width - 2*margin, forConstruction=True).vertices().hole(hole_d)
```

---

## 11. Free Function API (`cadquery.func`)

An alternative functional API for lower-level direct shape construction:

```python
from cadquery.func import *

r = rect(10, 5)
c = circle(3)

# Extrude, sweep, loft, revolve:
s1 = extrude(r, (0, 0, 10))
s2 = sweep(r, spline([(0,0,0),(0,5,5)], [(0,0,1),(0,1,1)]))
s3 = loft(r, c.moved(z=10))
s4 = revolve(fill(r), (5, 0, 0), (0, 1, 0), 90)

# Placement:
s = sphere(5).moved([(0,-10,0), (0,10,0)])   # two spheres at given locs
result = compound(s1, s2)

# Text on surfaces:
result = text("Hello", size=1, spine=some_edge)
```

---

## 12. Common Mistakes and Pitfalls

### Mistake 1: Forgetting `.workplane()` after face selection

```python
# WRONG — .hole() needs a workplane, not just a face selection:
result.faces(">Z").hole(3)

# CORRECT:
result.faces(">Z").workplane().hole(3)
```

### Mistake 2: Wrong selector for cylindrical faces

```python
# This selects based on NORMAL direction — doesn't apply to cylinders:
result.faces(">Z")

# For a cylinder's curved face:
result.faces("%CYLINDER")
```

### Mistake 3: Not resetting selection in Sketch API

```python
# WRONG — selectors accumulate without reset:
s = cq.Sketch().rect(4,4).vertices().fillet(0.5).vertices().chamfer(0.2)

# CORRECT — call .reset() between selections:
s = cq.Sketch().rect(4,4).vertices().fillet(0.5).reset().edges().chamfer(0.2)
```

### Mistake 4: Using `.val()` too early breaks chaining

```python
# WRONG — val() exits the chain, .extrude() won't work:
result = cq.Workplane("XY").circle(5).val().extrude(10)

# CORRECT:
result = cq.Workplane("XY").circle(5).extrude(10)
```

### Mistake 5: Assuming `.box()` is always centered

```python
# box() is centered by default:
cq.Workplane("XY").box(10, 10, 10)  # center at (0,0,0)

# To start at origin corner, use centered=False:
cq.Workplane("XY").box(10, 10, 10, centered=False)
```

### Mistake 6: Confusing distance vs coordinate in line operations

```python
.hLine(5)        # move 5mm right (distance)
.hLineTo(5)      # move to x=5 (coordinate)
.vLine(3)        # move 3mm up (distance)
.vLineTo(3)      # move to y=3 (coordinate)
```

### Mistake 7: Non-planar wires in `.loft()`

All cross-sections for loft must be on separate, parallel workplanes. Use `.workplane(offset=N)` to create each section:

```python
result = cq.Workplane("XY").rect(10, 10).workplane(offset=5).circle(4).workplane(offset=10).rect(6, 6).loft()
```

---

## 13. Quick Selector Cheatsheet

```
>Z  <Z  >X  <X  >Y  <Y     → min/max in direction (faces & edges)
>Z[-2]                      → 2nd from top
|Z  |X  |Y                  → parallel to axis
#Z  #X  #Y                  → perpendicular to axis
+Z  -Z  +X  ...             → faces with normal aligned to direction
%PLANE %CYLINDER %SPHERE    → face geometry type
%LINE %CIRCLE %ELLIPSE      → edge geometry type
not(...) or ... and ...     → logical combinations
>(-1,1,0)                   → custom direction vector
```

---