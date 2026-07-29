You will receive a CAD design request written in natural language.

==============================================================================
CRITICAL SCOPE RULE
==============================================================================
- Only assert properties that are EXPLICITLY constrained by the user's text.
- Do NOT test anything that is not stated or strictly implied in the request.
- If a property is not mentioned in the user's prompt, do NOT create assertions for it.

==============================================================================
CONTEXT
==============================================================================
Assume the CAD model will be available as `final_result` from the code structure below.

```python
import sys, os
import math
import numpy as np
import cadquery as cq

def create_cad() -> cq.Workplane:
    # --- Implementation code will be here (contents unknown) ---
    return final_result

final_result = create_cad()

# Your generated assertions will go here
```
==============================================================================
WHAT TO GENERATE
==============================================================================
Your response MUST have two parts, in this exact order:

**1. Reasoning (short paragraph)**
Before any JSON, write a brief reasoning block (3-6 sentences) explaining:
- What geometric shape/object the prompt describes
- Which explicit constraints you identified in the prompt
- The overall assertion strategy you will follow (e.g. topology checks, dimension ratios, absolute sizes, volume, symmetry, etc.)

**2. JSON array**
After the reasoning, return a JSON array where each element is one test case object.

**JSON Structure:**
```json
[
  {
    "assertion_id": 1,
    "assertion_description": "...",
    "prompt_justification": "...",
    "assertion_code": "...",
  },
  ...
]
```
**Each test case MUST include:**

1. **assertion_id** (integer):
   - Unique identifier for this assertion (1, 2, 3, ...)
   - Must be sequential starting from 1

2. **assertion_description** (string):
   - A concise explanation of what geometric property is being tested
   - Focus on the technical aspect (e.g., "Verifies the model has 6 planar faces")

3. **prompt_justification** (string):
   - Explain WHY this assertion is required based on the user's prompt
   - Quote or reference the specific part of the prompt that justifies this test
   - Make explicit the connection between the user's words and this assertion
   - Example: "The prompt states 'create a cube', which by definition requires 6 equal square faces"

4. **assertion_code** (string):
   - Python code that operates on the `final_result` variable
   - Keep each test focused on a single measurable requirement
   - Store measured values in descriptive variables before the check
   - **MUST use the `check()` helper** (available at runtime) instead of bare `assert`:
     `check(condition, pass_msg, fail_msg)`
     - `condition`: the boolean expression to test
     - `pass_msg`: an f-string describing what was verified and the actual measured values
       (printed only when the check passes)
     - `fail_msg`: an f-string showing expected vs actual values
       (used as the AssertionError message when the check fails)
   - The pass message should be informative and read naturally, e.g.
     `f"Face count is {face_count}, matches expected 6 for a rectangular box"`
   - Use \\n for multi-line code in the JSON string

==============================================================================
COMPLETE EXAMPLE
==============================================================================
For a user prompt: "Create a rectangular box with 10mm x 10mm base and 20mm height"

**Reasoning:**
The prompt asks for a rectangular box with fully specified dimensions: a square 10mm x 10mm base and a 20mm height. The explicit constraints are: the shape must be a box (6 planar faces, 12 edges, 8 vertices), the base must be square with 10mm sides, the height must be 20mm, and the volume should be 10×10×20 = 2000 mm³. My strategy will cover topology (face/edge/vertex counts), absolute dimension checks using sorted bounding-box dims with relative tolerances, volume verification, and shape-factor validation.

```json
[
  {
    "assertion_id": 1,
    "assertion_description": "Verifies the model has exactly 6 planar faces",
    "prompt_justification": "The prompt requests a 'rectangular box', which by geometric definition must have 6 rectangular faces",
    "assertion_code": "face_count = final_result.faces('%Plane').size()\\ncheck(face_count == 6, f'Face count is {face_count}, matches expected 6 for a rectangular box', f'Rectangular box must have 6 planar faces, found {face_count}')"
  },
  {
    "assertion_id": 2,
    "assertion_description": "Verifies the largest dimension is 20mm (the height)",
    "prompt_justification": "The prompt specifies '20mm height', which is the largest dimension of this box",
    "assertion_code": "bbox = final_result.val().BoundingBox()\\ndims = sorted([bbox.xlen, bbox.ylen, bbox.zlen])\\ncheck(abs(dims[2] - 20) / 20 < 0.01, f'Largest dimension is {dims[2]:.2f}mm, within 1% of expected 20mm height', f'Largest dimension (height) should be 20mm, found {dims[2]:.2f}mm')"
  },
  {
    "assertion_id": 3,
    "assertion_description": "Verifies the two smaller dimensions are 10mm each (the square base)",
    "prompt_justification": "The prompt specifies '10mm x 10mm base', so the two smallest bounding box dimensions must both be 10mm",
    "assertion_code": "bbox = final_result.val().BoundingBox()\\ndims = sorted([bbox.xlen, bbox.ylen, bbox.zlen])\\ncheck(abs(dims[0] - 10) / 10 < 0.01 and abs(dims[1] - 10) / 10 < 0.01, f'Base dimensions are {dims[0]:.2f}x{dims[1]:.2f}mm, both within 1% of expected 10mm', f'Base should be 10x10mm, found {dims[0]:.2f}x{dims[1]:.2f}mm')"
  },
  {
    "assertion_id": 4,
    "assertion_description": "Verifies the volume matches a 10x10x20mm box",
    "prompt_justification": "A box with 10mm x 10mm base and 20mm height must have volume = 10 * 10 * 20 = 2000 mm³",
    "assertion_code": "volume = final_result.val().Volume()\\ncheck(abs(volume - 2000) / 2000 < 0.01, f'Volume is {volume:.2f} mm³, within 1% of expected 2000 mm³', f'Volume should be 2000 mm³, found {volume:.2f} mm³')"
  }
]
```

**All assertions MUST include:**
1. Use the **`check()` helper** instead of bare `assert`:
   `check(condition, pass_msg, fail_msg)`
2. A **pass_msg** (f-string) that reads naturally and includes measured values, e.g.:
   `f"Volume is {volume:.2f} mm³, within 1% of expected 2000 mm³"`
   This prints when the assertion passes, providing insight into the model's actual geometry.
3. A **fail_msg** (f-string) showing expected vs actual values, e.g.:
   `f"Volume should be 2000 mm³, found {volume:.2f} mm³"`
4. **Descriptive variable names** to store measured values before the check

==============================================================================
TRANSFORM & SCALE INVARIANCE
==============================================================================
Assertions will be tested against augmented versions of the model that have been
randomly rotated (90° around X/Y/Z) and translated. Assertions MUST pass
regardless of the model's position and orientation in space.

Additionally, models may be built at different scales when the prompt does not
constrain absolute dimensions. A prompt like "create a cylinder" does not specify
a size, so a valid model could be 1mm tall or 100mm tall. Your assertions must
handle this:

**If the prompt specifies exact dimensions** (e.g. "10mm x 10mm base, 20mm height"):
  - You CAN assert absolute values like `abs(dims[2] - 20) < 0.01`
  - Use tolerances relative to the expected value, not hardcoded:
    `abs(dims[2] - 20) < 20 * 0.01` rather than `abs(dims[2] - 20) < 0.01`

**If the prompt does NOT specify dimensions** (e.g. "create a box"):
  - Do NOT assert any absolute dimension values
  - Assert only ratios, proportions, counts, and topology
  - Example: "a cube" → assert `dims[0] / dims[2] > 0.99` (all sides equal)
  - Example: "a cylinder" → assert volume matches `math.pi * r**2 * h` using
    measured dims, NOT hardcoded sizes

**General tolerance rules:**
  - NEVER hardcode small absolute tolerances like `< 0.01` or `< 0.001`
  - Scale tolerances relative to the measured geometry:
    `abs(a - b) < max(abs(a), abs(b)) * 0.01` (1% relative tolerance)
  - For equality checks between two measured values:
    `abs(dim_a - dim_b) / max(dim_a, dim_b, 1e-9) < 0.01`
  - For volume/area checks, use relative comparison:
    `abs(actual - expected) / max(expected, 1e-9) < 0.01`

**DO** use (invariant under rotation, translation, & scale):
- Face/edge/vertex counts: `.faces().size()`, `.edges().size()`, `.vertices().size()`
- Type-based selectors: `faces("%Plane")`, `faces("%Cylinder")`, `edges("%Circle")`
- Dimension ratios: `dims[2] / dims[0]`, `dims[0] / dims[1]`
- Volume-to-bounding-box ratio (shape factor): `volume / (dims[0] * dims[1] * dims[2])`
- Topology: number of solids, shells, wires

**DO** use (invariant under rotation & translation, but NOT scale):
- Volume: `obj.Volume()` — only assert absolute values when size is specified
- Total surface area: `obj.Area()` — only assert absolute values when size is specified
- Sorted bounding-box dimensions for size checks (only when dimensions are given):
  `dims = sorted([bbox.xlen, bbox.ylen, bbox.zlen])`
  then assert against `dims[0]`, `dims[1]`, `dims[2]` (smallest → largest)

**DO NOT** use (breaks under rotation or translation):
- Axis-specific dimensions: `bbox.zlen` as "height", `bbox.xlen` as "width"
- Hardcoded absolute positions: `bbox.xmin`, `bbox.ymin`, `bbox.zmax`
- Hardcoded coordinates as arguments: `shape.isInside((0, 0, 5))`, `obj.Center()`
- Direction-based selectors: `faces(">Z")`, `faces("<Z")`, `faces("|Z")`, `edges("|X")`
- Axis-aligned normals or direction checks

NOTE: APIs like `isInside`, `facesIntersectedByLine`, `CenterOfBoundBox`, and
`centerOfMass` ARE useful -- but their arguments must be derived from the model's
own geometry (e.g. its center, bounding box, etc.), never hardcoded.

**How to convert dimension checks:**
Instead of `assert abs(bbox.zlen - 20) < 0.01` (assumes height is along Z, hardcoded tolerance),
use sorted dimensions with relative tolerance:
```python
dims = sorted([bbox.xlen, bbox.ylen, bbox.zlen])
assert abs(dims[2] - 20) / 20 < 0.01  # largest dimension is the height, 1% tolerance
```

Instead of `assert abs(bbox.xlen - bbox.ylen) < 0.01` (assumes square base in XY),
use:
```python
dims = sorted([bbox.xlen, bbox.ylen, bbox.zlen])
assert abs(dims[0] - dims[1]) / max(dims[0], dims[1]) < 0.01  # two smallest dims are equal
```

When no dimensions are given, assert only the shape:
```python
dims = sorted([bbox.xlen, bbox.ylen, bbox.zlen])
# "cube" → all three dimensions are equal
assert dims[0] / dims[2] > 0.99, f'Cube sides should be equal, ratio={dims[0]/dims[2]:.4f}'
```

==============================================================================
REQUIREMENTS
==============================================================================
- Include 8-20 meaningful assertions depending on the design complexity and specificity of the user request
- Generate as many assertions as possible to thoroughly test all explicit requirements
- Cover ALL explicit requirements expressed in the user's request
- Do NOT invent constraints not stated in the prompt
- Each assertion should be independently testable
- Every assertion MUST include an informative error message with actual measured values
- All assertions MUST be invariant to rotations and translations (see TRANSFORM & SCALE INVARIANCE above)
- NEVER hardcode absolute tolerances; always scale tolerances relative to measured geometry
- When the prompt does not specify dimensions, assert only ratios, proportions, counts, and topology

==============================================================================
CADQUERY QUERY / INSPECTION REFERENCE (USE THIS, DO NOT INVENT APIS)
==============================================================================
You must only use the following CadQuery querying/inspection APIs and selector syntax.

------------------------------------------------------------------------------
A) Core selector methods (query geometry from a Workplane)
------------------------------------------------------------------------------
Use these to select geometry subsets from `final_result`:

- final_result.faces(selector=None)
- final_result.edges(selector=None)
- final_result.vertices(selector=None)
- final_result.solids(selector=None)
- final_result.shells(selector=None)

Useful stack helpers in the Workplane API object:
- final_result.size()  -> number of items currently on the Workplane stack
- final_result.val()   -> single selected object (expects exactly one)
- final_result.vals()  -> list of selected objects 
- final_result.solids() -> list of solids (if any) in the current selection
- these methods can be used to get the shape(s) for further inspection, the methods above are not present on Shape objects.

Selection size/count:
- .faces(), .edges(), .vertices(), .solids(), .shells() all return a Workplane object, so to get the count of selected items use:
- final_result.faces(...).size(), final_result.edges(...).size(), final_result.solids().size() etc.

------------------------------------------------------------------------------
B) Selector string modifiers (how to filter)
------------------------------------------------------------------------------
Axis strings: X, Y, Z, and combined axes: XY, XZ, YZ.

Modifiers:
- %  Type selector (TypeSelector)              ← INVARIANT, prefer this
- |  Parallel to axis/plane direction           ← NOT invariant, avoid
- #  Perpendicular to axis/plane direction      ← NOT invariant, avoid
- +  Positive direction (DirectionSelector)     ← NOT invariant, avoid
- -  Negative direction (DirectionSelector)     ← NOT invariant, avoid
- >  Maximum in a direction                     ← NOT invariant, avoid
- <  Minimum in a direction                     ← NOT invariant, avoid

INVARIANT examples (use these):
- faces("%Plane")    -> planar faces (count is rotation-invariant)
- faces("%Cylinder") -> cylindrical faces (count is rotation-invariant)
- edges("%Circle")   -> circular edges (count is rotation-invariant)
- faces()            -> all faces (count is invariant)
- edges()            -> all edges (count is invariant)
- vertices()         -> all vertices (count is invariant)

NOT invariant (avoid in assertions):
- faces(">Z"), faces("<Z"), faces("|Z"), faces("#Z"), edges("|X"), etc.
  These depend on the model's orientation and will break under rotation.

------------------------------------------------------------------------------
C) Counting and topology (invariant)
------------------------------------------------------------------------------
These are always safe to use in assertions:
- final_result.faces().size()            -> total face count
- final_result.faces("%Plane").size()    -> planar face count
- final_result.faces("%Cylinder").size() -> cylindrical face count
- final_result.edges().size()            -> total edge count
- final_result.edges("%Circle").size()   -> circular edge count
- final_result.vertices().size()         -> total vertex count
- final_result.solids().size()           -> solid count
- final_result.shells().size()           -> shell count

------------------------------------------------------------------------------
D) Geometry inspection APIs (operate on selected shapes)
------------------------------------------------------------------------------
Workplane / solid:
- obj = final_result.val()
- bbox = obj.BoundingBox()
  - bbox.xlen, bbox.ylen, bbox.zlen  (use sorted for invariance, see below)
  - bbox.xmin, bbox.xmax, etc.       (NOT invariant, avoid)
- obj.Volume()                        ← INVARIANT
- obj.Center()                        ← NOT invariant (position changes)

INVARIANT bounding-box pattern:
```python
bbox = final_result.val().BoundingBox()
dims = sorted([bbox.xlen, bbox.ylen, bbox.zlen])
# dims[0] = smallest, dims[1] = middle, dims[2] = largest
```

Shape: Base class for all geometric objects (Face, Edge, Solid, Wire, Vertex, Shell)
- shape = final_result.val()

- shape.Area() -> float                    ← INVARIANT
  Total surface area of all faces.

- shape.Volume(tol=None) -> float          ← INVARIANT
  Volume of the shape.

- shape.CenterOfBoundBox(tolerance=None) → Vector
- static cq.Shape.centerOfMass(obj) → Vector
  These return absolute positions, so do NOT compare against hardcoded coordinates.
  Instead, use them relative to each other or to other model-derived points.
  Example (symmetry check): the center of mass should coincide with the bounding box center:
  ```python
  com = cq.Shape.centerOfMass(shape)
  cob = shape.CenterOfBoundBox()
  dist = (com - cob).Length
  bbox_diag = (bbox.xlen**2 + bbox.ylen**2 + bbox.zlen**2)**0.5
  assert dist / bbox_diag < 0.01, f'Center of mass deviates from bbox center by {dist:.4f}'
  ```

- shape.isInside(point, tolerance=1e-06) → bool
  Useful for containment checks. Derive the point from the model, never hardcode it.
  Example: verify a hole goes through the center:
  ```python
  center = shape.CenterOfBoundBox()
  assert shape.isInside(center), 'Center of bounding box should be inside the solid'
  ```

- shape.facesIntersectedByLine(point, axis, ...) → List[Face]
  Useful for checking holes and through-features. Derive point and axis from geometry.
  Example: count how many faces a line through the center intersects:
  ```python
  center = shape.CenterOfBoundBox()
  dims = sorted([bbox.xlen, bbox.ylen, bbox.zlen])
  # Use the longest bbox axis as the line direction
  if bbox.xlen == dims[2]: axis = (1, 0, 0)
  elif bbox.ylen == dims[2]: axis = (0, 1, 0)
  else: axis = (0, 0, 1)
  hit_faces = shape.facesIntersectedByLine(center, axis)
  ```

