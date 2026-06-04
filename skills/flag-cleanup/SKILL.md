---
name: flag-cleanup
description: Safely remove a named feature flag from the codebase by replacing conditionals with their permanent branch
version: 0.1.0
trigger: explicit
args:
  - name: flag_name
    description: The exact flag identifier to remove (e.g., NEW_CHECKOUT_FLOW or process.env.FEATURE_DARK_MODE)
    required: true
  - name: permanent_value
    description: "The permanent state of the flag: 'true' (feature is enabled) or 'false' (feature is disabled/rolled back)"
    required: true
---

# Flag Cleanup

Safely remove a feature flag from the codebase. This skill replaces all conditional branches that depend on the flag with the appropriate permanent code path, then removes the flag's definition.

## Steps

### 1. Search all usages

Run a comprehensive search for every occurrence of `{{flag_name}}` in the codebase:
- Exact string matches (case-sensitive and case-insensitive variants)
- String interpolation forms (e.g., `` `process.env.${FLAG}` ``)
- Object property access (e.g., `flags.newCheckout`, `featureFlags['newCheckout']`)
- Comment references (mark as informational, do not auto-edit)

Collect full file paths and line numbers. Group by file.

If zero usages are found, report "Flag `{{flag_name}}` not found in codebase. Nothing to remove." and stop.

### 2. Determine the permanent value

Confirm `{{permanent_value}}` is either `true` or `false`. If the user did not provide it, ask:
> "Is `{{flag_name}}` being removed because it is fully enabled (true) or fully disabled/rolled back (false)?"

Do not proceed until the permanent value is confirmed. This is a non-negotiable checkpoint.

### 3. Show the diff plan

Before modifying any file, generate a dry-run summary showing every planned change:

```
File: src/checkout.ts
  Line 14: if (isEnabled('NEW_CHECKOUT_FLOW')) { ... } else { ... }
  → Remove if/else, keep the THEN branch (permanent_value=true)

File: src/config.ts
  Line 88: const NEW_CHECKOUT_FLOW = process.env.FEATURE_NEW_CHECKOUT_FLOW === 'true';
  → Delete this line (definition removal)

File: .flags.json
  Line 12: "NEW_CHECKOUT_FLOW": { "default": true, "owner": "payments-team" }
  → Delete this key from the JSON object
```

Print the full diff plan and ask: "Apply these changes? (yes/no)"

Do NOT proceed without explicit "yes" from the user.

### 4. Replace conditional blocks with the permanent branch

For each code file in the diff plan, apply the following transformations based on `{{permanent_value}}`:

**If permanent_value = true (feature is ON):**
- `if (flag) { A } else { B }` → replace with the contents of block `A`
- `flag ? A : B` → replace with `A`
- `unless flag` blocks → remove entirely (condition is never true)
- Guard clauses `if (!flag) return;` → remove entirely

**If permanent_value = false (feature is OFF):**
- `if (flag) { A } else { B }` → replace with the contents of block `B`
- `flag ? A : B` → replace with `B`
- `if (flag)` blocks with no else → remove the entire block
- Guard clauses `if (flag) return;` → remove entirely

Preserve all surrounding code. Do not touch unrelated logic. Maintain indentation.

### 5. Remove the flag constant/definition

After all conditional blocks are resolved:
- Delete the constant or variable declaration (e.g., `const FEATURE_X = ...`, `FEATURE_X = os.environ.get(...)`)
- Remove the flag entry from any config file (`.flags.json`, `flags.yaml`, etc.)
- Remove any import that becomes unused as a direct result of this change
- Remove environment variable declarations in `.env.example`, `docker-compose.yml`, CI config if the variable is flag-specific and no longer needed

### 6. Run the existing test suite

Execute the project's test command. Detect it from:
1. `package.json` `scripts.test` field
2. `Makefile` `test` target
3. `pytest` if `pytest.ini` or `pyproject.toml` with `[tool.pytest]` exists
4. `go test ./...` if `go.mod` exists
5. `cargo test` if `Cargo.toml` exists

Run the tests and show the output summary (pass/fail count). If tests fail, show which tests failed and their error messages.

If tests fail, do NOT commit or report success. Report: "Cleanup complete but tests failing — review the test output above before committing."

If tests pass, report: "Flag `{{flag_name}}` successfully removed. All tests passing. Staged changes are ready to commit."

## Guardrails

- Never delete a flag without the user typing "yes" to the diff plan in Step 3.
- Never touch files outside the flag's usage set (no opportunistic cleanup).
- Never remove comments that provide historical context — mark them with `// flag-ops: removed {{flag_name}}` instead.
- If a file has merge conflicts or is in a dirty git state, skip it and warn the user.
- If unsure whether a code path belongs to the flag's conditional, leave it untouched and note the uncertainty.
