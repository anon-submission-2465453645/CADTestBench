Your task now is to refine CAD test assertions using dual feedback: execution failures on ground truth AND mutation testing results.

==============================================================================
TASK: Add or Update Assertions
==============================================================================
Assertions were tested against:
1. **Ground Truth Code**: The correct implementation
2. **Mutations**: Code variants that intentionally violate requirements

**IMPORTANT**: 
- Mutations are tested exclusively with assertions that pass on ground truth.
- Any assertion failing on ground truth is buggy and automatically excluded from mutation testing.
- If an assertion passes on ground truth but errors on a mutation, the mutation IS caught (the mutation broke the assertion).
- A mutation is "caught" if ANY assertion fails OR errors on it (since all assertions pass on ground truth).

These evaluations reveal two distinct classes of issues:

1. Failed Assertions on Ground Truth
  - These are assertions that flag an error on valid, correct geometry.
  - This indicates the assertion itself is flawed.

  Your job for these cases:
    - Diagnose the specific technical and conceptual problem
    - Revise the assertion without contradicting the prompt

2. Uncaught Mutations
  - These are mutation variants that pass all assertions.
  - This MAY mean the test suite is incomplete, OR the mutation doesn't violate actual prompt requirements.
  
  Common causes:
    - Assertion is too weak or too narrow
    - Mutation changed geometry in a way not covered by current checks
    - Assertion checks the wrong metric or attribute
    - Assertion relies on superficial properties that remain unchanged under mutation
    - Mutation affects higher level semantics (for example symmetry, connectivity) that were never asserted
    - **OR: The mutation doesn't actually violate any requirement from the prompt** (false positive)
  
  Your job for these cases:
    - **First**: Determine if the mutation actually violates a requirement stated in the prompt
    - **If YES**: Propose new assertions or modify existing ones to catch it
    - **If NO**: The mutation is acceptable - mark relevant assertions as "unchanged" and ignore this mutation
    - Only add assertions for prompt requirements that are explicitly stated or strongly implied
    - Do NOT add assertions for variations that don't contradict the prompt text

==============================================================================
INPUT INFORMATION
==============================================================================
You will receive:
1. **Original User Prompt**: The CAD design requirements
2. **Current Assertions**: Existing test assertions
3. **Ground Truth Execution**: Which assertions failed on correct code
4. **Mutation Testing**: Which mutations weren't caught

==============================================================================
OUTPUT FORMAT
==============================================================================
First, provide analysis explaining:
- Why each ground truth assertion failed (if any)
- Why each mutation wasn't caught (if any)
- Your refinement strategy for both issues

Then output JSON array with all assertions (updated, new, or unchanged).

**JSON Structure:**

For UNCHANGED assertions:
```json
{
  "assertion_id": <int>,
  "status": "unchanged"
}
```

For UPDATED assertions:
```json
{
  "assertion_id": <int>,
  "status": "updated",
  "assertion_description": <string>,
  "prompt_justification": <string>,
  "assertion_code": <string>,
  "update_reason": "Why this was updated (ground truth failure or mutation gap)"
}
```

For NEW assertions:
```json
{
  "assertion_id": <int>,
  "status": "new",
  "assertion_description": <string>,
  "prompt_justification": <string>,
  "assertion_code": <string>,
  "targets_mutation": <int or list of ints>,  // Which mutation(s) this should catch
  "reason": "Addresses uncaught mutation(s)"
}
```

**Requirements:**
- Include ALL assertions (even unchanged ones)
- For new assertions, use sequential IDs starting after highest existing ID
- For assertions failing on ground truth try to update them
- Updated assertions can address either ground truth failures or mutation gaps
