# /flag-ops-rollout

Show rollout status for all feature flags across environments.

## Usage

```
/flag-ops-rollout
/flag-ops-rollout <flag-name>
/flag-ops-rollout <flag-name> --update
```

## Arguments

| Argument | Description |
|---|---|
| (none) | Show rollout table for all flags |
| `flag-name` | Show rollout details for a single flag |
| `--update` | Interactively update rollout percentages and metadata for `flag-name` |

## What this command does

Runs the `flag-rollout` skill:

1. Reads rollout metadata from `.flags.json` (or equivalent config file)
2. Renders a per-environment rollout table with status classification
3. Highlights flags that are overdue for removal or fully rolled out
4. With `--update`, prompts to update a flag's rollout %, owner, and expected removal date

## Output format

```
Flag Rollout Status — 2025-06-04
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Flag Name              Owner          Dev     Staging   Prod    Due Date    Status
──────────────────────────────────────────────────────────────────────────────────
NEW_CHECKOUT_FLOW      payments-team  100%    100%      25%     2025-07-01  ROLLING
DARK_MODE              ui-team        100%    100%      100%    2025-06-15  FULL — remove soon
EXPERIMENT_XYZ         growth-team    100%    0%        0%      2025-08-01  STAGING ONLY
OLD_AUTH_V1            platform-team  0%      0%        0%      2024-12-01  OVERDUE — cleanup
──────────────────────────────────────────────────────────────────────────────────
4 flags  |  1 full  |  1 rolling  |  1 staging  |  1 overdue

Action Items:
  [OVERDUE]       OLD_AUTH_V1 — removal was due 2024-12-01. Run: /flag-ops-cleanup OLD_AUTH_V1
  [REMOVE SOON]   DARK_MODE — fully rolled out, due 2025-06-15. Run: /flag-ops-cleanup DARK_MODE
```

## Status key

| Status | Meaning |
|---|---|
| `FULL — remove soon` | All envs at 100%, removal date within 30 days |
| `ROLLING` | Production < 100%, staging at 100% |
| `STAGING ONLY` | Staging enabled, production at 0% |
| `DEV ONLY` | Only development enabled |
| `OVERDUE` | Expected removal date is in the past |
| `DISABLED` | All environments at 0% or off |
