# /flag-ops-cleanup

Safely remove a feature flag from the codebase.

## Usage

```
/flag-ops-cleanup <flag-name>
/flag-ops-cleanup <flag-name> --permanent-value=true
/flag-ops-cleanup <flag-name> --permanent-value=false
```

## Arguments

| Argument | Required | Description |
|---|---|---|
| `flag-name` | Yes | The exact flag identifier (e.g., `NEW_CHECKOUT_FLOW`, `darkMode`) |
| `--permanent-value` | No | `true` (keep the enabled branch) or `false` (keep the disabled branch). If omitted, the skill will ask before proceeding. |

## What this command does

Runs the `flag-cleanup` skill:

1. Finds every occurrence of `<flag-name>` in the codebase
2. Shows a full diff plan of all planned changes
3. Waits for your explicit "yes" before touching any file
4. Replaces `if (flag) { A } else { B }` with the appropriate permanent branch
5. Removes the flag's constant/definition and config entry
6. Runs the project's test suite to confirm nothing broke

## Example session

```
> /flag-ops-cleanup NEW_CHECKOUT_FLOW --permanent-value=true

Found 3 usages in 2 files:
  src/checkout.ts:14   if (isEnabled('NEW_CHECKOUT_FLOW')) { ... } else { ... }
  src/checkout.ts:42   const label = NEW_CHECKOUT_FLOW ? 'New' : 'Legacy';
  src/config.ts:88     const NEW_CHECKOUT_FLOW = process.env.FEATURE_NEW_CHECKOUT_FLOW === 'true';

Planned changes (permanent_value=true):
  src/checkout.ts:14   → keep THEN branch, remove else
  src/checkout.ts:42   → replace with: const label = 'New';
  src/config.ts:88     → delete line
  .flags.json:12       → remove "NEW_CHECKOUT_FLOW" key

Apply these changes? (yes/no): yes

Applying... done.
Running tests... 42 passed, 0 failed.
Flag NEW_CHECKOUT_FLOW successfully removed.
```

## Guardrails

- Will not proceed without your explicit "yes" to the diff plan.
- Will not commit changes — you control the git workflow.
- Stops and warns if tests fail after cleanup.
- Never removes comments that provide historical context.
