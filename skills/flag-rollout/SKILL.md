---
name: flag-rollout
description: Track and document feature flag rollout progress across environments
version: 0.1.0
trigger: explicit
args:
  - name: flag_name
    description: The flag identifier to document or update (optional — omit to show all)
    required: false
---

# Flag Rollout

Track, display, and document the rollout status of feature flags across environments. This skill reads and writes rollout metadata to `.flags.json` (or the project's flag config file) and renders a human-readable rollout table.

## Steps

### 1. Locate the flag config file

Check for a rollout config in this order:
1. `.flags.json` (project root) — preferred
2. `flags.yaml` / `flags.yml`
3. `config/flags.js` or `config/flags.ts`
4. `config/feature-flags.json`

If none exists, create `.flags.json` at the project root with an empty `{}` object and notify the user.

### 2. Read existing rollout records

Parse the config file. Each flag entry should conform to this schema:

```json
{
  "FLAG_NAME": {
    "description": "What this flag controls",
    "owner": "team-name or email",
    "created_date": "YYYY-MM-DD",
    "expected_removal_date": "YYYY-MM-DD",
    "environments": {
      "development": { "enabled": true, "rollout_pct": 100 },
      "staging":     { "enabled": true, "rollout_pct": 100 },
      "production":  { "enabled": true, "rollout_pct": 25  }
    },
    "notes": "Optional context about the rollout"
  }
}
```

Fields are optional except `description` and `owner` — prompt for those if missing when writing a new record.

### 3. If a specific flag_name is provided — update or create its record

Ask the user for any missing required fields:
- Description (if not already set)
- Owner (team or individual)
- Current rollout percentage per environment (dev / staging / production)
- Target rollout percentage (e.g., "100% production by next release")
- Expected removal date (when the flag should be cleaned up after full rollout)

Write the updated record back to the config file. Use 2-space JSON indentation. Do not disturb other entries.

### 4. Render the rollout status table

Display a summary table for all flags (or just the specified flag):

```
Flag Rollout Status — 2025-06-04
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Flag Name              Owner          Dev     Staging   Prod    Due Date    Status
─────────────────────────────────────────────────────────────────────────────────
NEW_CHECKOUT_FLOW      payments-team  100%    100%      25%     2025-07-01  ROLLING
DARK_MODE              ui-team        100%    100%      100%    2025-06-15  FULL — remove soon
EXPERIMENT_XYZ         growth-team    100%    0%        0%      2025-08-01  STAGING ONLY
OLD_AUTH_V1            platform-team  0%      0%        0%      2024-12-01  OVERDUE — cleanup
─────────────────────────────────────────────────────────────────────────────────
4 flags total  |  1 fully rolled out  |  1 rolling  |  1 staging  |  1 overdue
```

**Status logic:**
- `FULL — remove soon`: all environments at 100% and expected_removal_date is within 30 days
- `ROLLING`: production < 100% with staging ≥ 100%
- `STAGING ONLY`: staging enabled but production at 0%
- `DEV ONLY`: only development enabled
- `OVERDUE`: expected_removal_date is in the past
- `DISABLED`: all environments at 0% or disabled

### 5. Highlight action items

After the table, print an action list:

```
Action Items:
  [OVERDUE]       OLD_AUTH_V1 — removal date was 2024-12-01. Run /flag-ops-cleanup OLD_AUTH_V1
  [REMOVE SOON]   DARK_MODE — fully rolled out, due 2025-06-15. Run /flag-ops-cleanup DARK_MODE
  [NO OWNER]      EXPERIMENT_XYZ — assign an owner before continuing rollout
```

Only list flags that require action. If none, print "No action items."

### 6. Guardrails

- Never change rollout percentages in application code — only update the config file.
- Never set expected_removal_date to a date more than 12 months in the future without warning.
- If the same flag appears in multiple environments with conflicting values, surface the conflict explicitly.
- Do not invent rollout data — if a field is unknown, mark it as `null` and ask the user.
