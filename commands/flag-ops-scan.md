# /flag-ops-scan

Find orphaned and stale feature flags in this project.

## What this command does

Runs the full `flag-scan` skill including:
- Orphan detection: flags defined but never used in code, or used in code but never defined in config
- Staleness detection: flags whose code was last modified more than 90 days ago with no open PR/issue referencing them

## Usage

```
/flag-ops-scan
```

No arguments required. The scan runs against the current working directory.

## Output format

```
Flag Scan Results — <project-root>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ORPHANED FLAGS (2)
──────────────────
Flag Name        Reason                              Locations
OLD_PAYMENT_V1   In config, never used in code       .flags.json:12
BETA_DASHBOARD   Used in code, not in config         src/dashboard.ts:88, src/routes.ts:14

STALE FLAGS (1)  [last modified >90 days ago, no open PR/issue]
────────────────────────────────────────────────────────────────
Flag Name        Last Modified   Days Old   Files
EXPERIMENT_XYZ   2024-09-01      276        src/experiment.ts

No orphaned or stale flags = clean codebase.
```

## Next steps

To remove a flag: `/flag-ops-cleanup <flag-name>`
To view rollout status: `/flag-ops-rollout`

Note: This command never modifies files. It is read-only.
