# /flag-ops

List all feature flags in this project with their current status.

## What this command does

1. Runs the `flag-scan` skill in summary mode (no orphan/stale analysis).
2. Cross-references found flags against the project's flag config file (`.flags.json`, `flags.yaml`, etc.) if one exists.
3. Prints a concise table of all flags, their type, usage count, and whether they are defined in config.

## Output format

```
Feature Flags — <project-root>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Flag Name              Type          Usages   In Config?   Last Modified
──────────────────────────────────────────────────────────────────────────
NEW_CHECKOUT_FLOW      env-var       3        Yes          2025-03-10
darkMode               config-prop   2        Yes          2025-05-01
BETA_DASHBOARD         function      1        No           2025-01-14
OLD_AUTH_V1            constant      5        Yes          2024-09-01
──────────────────────────────────────────────────────────────────────────
4 flags found  |  3 in config  |  1 undefined
```

For a deeper analysis including orphaned and stale flags, run `/flag-ops-scan`.
For rollout percentages and due dates, run `/flag-ops-rollout`.
