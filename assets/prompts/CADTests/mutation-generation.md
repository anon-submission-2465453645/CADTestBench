You are an expert at generating test cases for CAD code by creating intentional mutations.

==============================================================================
TASK
==============================================================================
Given:
- A natural language CAD design prompt, and
- A reference CADQuery implementation that satisfies this prompt,

produce 5-15 mutation variants of the code. These variants must intentionally break one or more geometric requirements from the design prompt.

Very important:
- Only the natural language prompt defines requirements.
- The reference code does NOT add any requirements. Code parameters, literal values, or implementation choices that are not mentioned in the prompt are NOT requirements.
- Mutations that only change unconstrained implementation details or numbers that the prompt does not specify are NOT valid and must NOT be generated.
- Avoid generating mutations that are too similar to each other. Each mutation must break a different requirement or break the same requirement in a different way.
- Do NOT generate mutations that change the plane orientation, position, or scale of the entire model unless the prompt specifically requires it.
- Do NOT use CAD conventions or common sense to infer requirements that are not explicitly stated in the prompt.
The following are NOT valid mutations:
- Changing extrusion direction when the prompt does not specify direction
- Changing sign of a dimension when magnitude is not specified
- Violations based on "default behavior", "expected orientation", or "typical modeling conventions" when the prompt does not specify these aspects.

Each mutation should:
1. Be syntactically valid Python and valid CADQuery code that executes without runtime errors.
2. Modify only the modeling operations that affect the final geometry.
3. Intentionally break one or more requirements from the design prompt.
4. Produce final geometry that violates the prompt in some way, so that a purely geometry based evaluation would mark it as incorrect.

Do not introduce changes that affect only comments, variable names, logging, or other non geometric behavior.

==============================================================================
OUTPUT FORMAT
==============================================================================
Return a JSON array with mutation objects. Each object must have:

```json
[
  {
    "mutation_id": 1,
    "mutation_description": "Complete geometric failure description with all details needed to write an assertion",
    "modified_code": "Complete working CADQuery code for this mutation",
  },
  ...
]
```
**Field Requirements:**
1. **mutation_id** (integer): Unique sequential identifier
2. **mutation_description** (string): A detailed explanation of the generated mutation.
This description must include:
   - Which requirement from the prompt is violated. Only the prompt defines requirements. Changing something in the reference code is not a violation unless that change contradicts the prompt text.
   - The geometric effect of this change.
   - How the resulting geometry differs from the correct one.
   - It must refer ONLY to requirements that exist in the prompt text or that are semantically implied by it (for example "cube" means equal edges).
3. **prompt_requirement_connection** (string): A  statement showing precisely how the mutation breaks a specific requirement from the prompt.
   - It must have the form: "The prompt requires X, but the mutated geometry has Y."
4. **modified_code** (string): The complete mutated CADQuery code
   - Must be syntactically valid and executable
   - Should follow the same structure as the reference code
