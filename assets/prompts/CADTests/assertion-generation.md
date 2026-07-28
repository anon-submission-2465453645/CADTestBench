You will receive a CAD design request written in natural language.

==============================================================================
CRITICAL SCOPE RULE
==============================================================================
• Only assert properties that are EXPLICITLY constrained by the user's text.
• Do NOT test anything that is not stated or strictly implied in the request.
• If a property is not mentioned in the user's prompt, do NOT create assertions for it.

==============================================================================
CONTEXT
==============================================================================
Assume the CAD model will be available as `final_result` from the code structure below.
The base plane for all CAD models is assumed to be the XY plane, with Z as the vertical axis.

```python
import sys, os
import math
import numpy as np
import cadquery as cq
from cad_assert_helpers import *

def create_cad() -> cq.Workplane:
    # --- Implementation code will be here (contents unknown) ---
    return final_result

final_result = create_cad()

# Your generated assertions will go here
```

==============================================================================
WHAT TO GENERATE
==============================================================================
Return a JSON array where each element is one test case object.

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
   - A Python assert statement (or small block of assertions)
   - Operate on the `final_result` variable
   - Keep each test focused on a single measurable requirement
   - **MUST include informative error messages with f-strings showing expected vs actual values**
   - Store measured values in descriptive variables before asserting
   - Use \n for multi-line code in the JSON string

==============================================================================
COMPLETE EXAMPLE
==============================================================================
For a user prompt: "Create a rectangular box with 10mm x 10mm base and 20mm height"

test cases:
```json
[
  {
    "assertion_id": 1,
    "assertion_description": "Verifies the model has exactly 6 planar faces",
    "prompt_justification": "The prompt requests a 'rectangular box', which by geometric definition must have 6 rectangular faces",
    "assertion_code": "face_count = final_result.faces('%Plane').size()\nassert face_count == 6, f'Rectangular box must have 6 planar faces, found {face_count}'"
  },
  {
    "assertion_id": 2,
    "assertion_description": "Verifies the box height is 20mm in the Z direction",
    "prompt_justification": "The prompt specifies '20mm height', which means the box should have a 20mm dimension perpendicular to the XY base plane",
    "assertion_code": "bbox = final_result.val().BoundingBox()\nheight = bbox.zmax - bbox.zmin\nassert abs(height - 20) < 0.01, f'Box height should be 20mm, found {height:.2f}mm'"
  }

  # more assertions here
]
```

**All assertions MUST include informative error messages using f-strings that display:**
- What was expected
- What was actually found (actual measured values)
- Use descriptive variable names to store measured values before asserting

==============================================================================
REQUIREMENTS
==============================================================================
• Include 8-20 meaningful assertions depending on the design complexity and specificity of the user request
• Generate as many assertions as possible to thoroughly test all explicit requirements
• Cover ALL explicit requirements expressed in the user's request
• Do NOT invent constraints not stated in the prompt
• Each assertion should be independently testable
• Every assertion MUST include an informative error message with actual measured values
• Assume the base plane is XY with Z as the vertical axis when testing dimensions and orientations
