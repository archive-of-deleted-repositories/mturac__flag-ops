# flag-ops

A Claude Code plugin for managing feature flags: scan for orphaned flags, track rollout status, and safely remove flags when they're done.

## Install

```bash
claude plugin install flag-ops
```

Or clone and load locally:

```bash
git clone https://github.com/mturac/flag-ops
claude plugin install ./flag-ops
```

## Commands

| Command | Description |
|---|---|
| `/flag-ops` | List all feature flags with type, usage count, and config status |
| `/flag-ops-scan` | Find orphaned and stale flags (read-only) |
| `/flag-ops-cleanup <name>` | Safely remove a flag — shows diff, waits for confirmation, runs tests |
| `/flag-ops-rollout` | Show per-environment rollout table with action items |

## Example: flag table

```
Feature Flags — ./my-app
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Flag Name              Type          Usages   In Config?   Last Modified
──────────────────────────────────────────────────────────────────────────
NEW_CHECKOUT_FLOW      env-var       3        Yes          2025-03-10
darkMode               config-prop   2        Yes          2025-05-01
BETA_DASHBOARD         function      1        No           2025-01-14
OLD_AUTH_V1            constant      5        Yes          2024-09-01
──────────────────────────────────────────────────────────────────────────
4 flags found  |  3 in config  |  1 undefined
```

## Example: rollout status

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

## How orphan detection works

`/flag-ops-scan` compares two sources of truth:

1. **Code scan** — grep across `*.js`, `*.ts`, `*.py`, `*.go`, `*.rb`, `*.rs`, and more for patterns like `isEnabled(`, `process.env.FEATURE_`, `flags.`, `FF_` constants, and third-party SDK calls (`LaunchDarkly.variation`, `Statsig.checkGate`).

2. **Config file** — reads `.flags.json`, `flags.yaml`, or `config/flags.js` if present.

A flag is **orphaned** when it exists in one source but not the other:
- Defined in config, never referenced in code — likely safe to remove from config.
- Referenced in code, absent from config — undocumented flag, risk of inconsistent behavior.

A flag is **stale** when its code files haven't been touched in more than 90 days and no open PR or issue references the flag name.

The scan is always read-only. No files are modified until you explicitly run `/flag-ops-cleanup`.

## Flag config schema

Store rollout metadata in `.flags.json` at the project root:

```json
{
  "NEW_CHECKOUT_FLOW": {
    "description": "New checkout page with address autocomplete",
    "owner": "payments-team",
    "created_date": "2025-01-15",
    "expected_removal_date": "2025-07-01",
    "environments": {
      "development": { "enabled": true, "rollout_pct": 100 },
      "staging":     { "enabled": true, "rollout_pct": 100 },
      "production":  { "enabled": true, "rollout_pct": 25 }
    },
    "notes": "Ramping prod 25% → 100% by June"
  }
}
```

## Supported flag patterns

| Pattern | Example |
|---|---|
| Environment variable | `process.env.FEATURE_NEW_UI` |
| Constant | `const FF_DARK_MODE = true` |
| Function call | `isEnabled('checkout-v2')` |
| Config property | `featureFlags.darkMode` |
| LaunchDarkly | `ldclient.variation('flag-key', user, false)` |
| Statsig | `Statsig.checkGate(user, 'gate_name')` |
| Rails | `feature_enabled?(:new_checkout)` |

## License

MIT
