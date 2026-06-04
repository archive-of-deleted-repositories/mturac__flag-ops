---
name: flag-scan
description: Scan codebase for feature flag usage, detect orphaned and stale flags
version: 0.1.0
trigger: explicit
---

# Flag Scan

Scan the entire codebase for feature flag usage patterns, cross-reference with configuration files, and identify orphaned or stale flags that represent technical debt.

## Steps

### 1. Scan codebase for feature flag usage

Search the codebase for common feature flag patterns. Run searches across the project root, excluding `node_modules/`, `dist/`, `build/`, `.git/`, `vendor/`, `__pycache__/`, `*.min.js`, `*.map`.

**Function-call patterns:**
- `isEnabled(`, `is_enabled(`
- `flagEnabled(`, `flag_enabled(`
- `featureFlag.get(`, `feature_flag.get(`
- `getFeatureFlag(`, `get_feature_flag(`
- `hasFeature(`, `has_feature(`
- `useFeatureFlag(`, `useFeature(`

**Environment variable patterns:**
- `process.env.FEATURE_`
- `process.env.FF_`
- `process.env.FLAG_`
- `import.meta.env.VITE_FEATURE_`
- `ENV[` or `os.environ` containing `FEATURE_`, `FF_`, `FLAG_`

**Constant/config patterns:**
- Top-level constants prefixed `FEATURE_` or `FF_`
- `flags.` or `featureFlags.` property access
- `feature_enabled?(` (Rails)
- `LaunchDarkly.variation(`, `ldclient.variation(`
- `Statsig.checkGate(`

File types to include: `*.js`, `*.ts`, `*.jsx`, `*.tsx`, `*.py`, `*.rb`, `*.go`, `*.rs`, `*.java`, `*.kt`, `*.php`, `*.swift`, `*.vue`, `*.svelte`.

### 2. Parse found flags

For each match, extract:

| Field | How |
|---|---|
| **Flag name** | The identifier — constant name, env var suffix, method argument, or property key |
| **Flag type** | `env-var`, `constant`, `config-property`, `function-call`, `third-party-sdk` |
| **File locations** | Every file:line where the flag appears |
| **Usage count** | Total occurrences across all files |
| **Last modified** | `git log -1 --format="%ai" -- <file>` for each file containing the flag |

Build an internal index keyed by flag name before proceeding.

### 3. Check against flag configuration file

Look for a flag config file in this order:
1. `.flags.json` (project root)
2. `flags.yaml` or `flags.yml` (project root)
3. `config/flags.js` or `config/flags.ts`
4. `config/feature-flags.json`
5. `src/feature-flags/config.json`
6. `lib/feature_flags.rb` (Rails)
7. `settings/flags.py` (Django)

If found, parse the file and extract: flag name, default value, description, owner, created date, environments. Cross-reference with the in-code index from Step 2.

If no config file exists, note it in output and proceed — all in-code flags lack a canonical definition.

### 4. Identify orphaned flags

A flag is **orphaned** if any of the following is true:
- It appears in the config file but has zero matches in any code file (defined but never used).
- It appears in code but has zero matches in the config file and no definition anywhere (checked but never defined).
- Its only code references are in dead code branches (e.g., always-false conditionals, unreachable paths).

List each orphaned flag with: name, reason for classification, locations.

### 5. Identify stale flags

A flag is **stale** if:
- The most recent `git log` date across all its files is more than 90 days before today's date.
- No open pull request title or issue title (from `gh pr list` / `gh issue list` if GitHub CLI is available) references the flag name.

List stale flags sorted ascending by last-modified date (oldest first).

### 6. Output

Print three sections:

**Active Flags**

| Flag Name | Type | Usages | Files | Last Modified | In Config? |
|---|---|---|---|---|---|
| `NEW_CHECKOUT_FLOW` | env-var | 3 | 2 | 2025-03-10 | Yes |
| `darkMode` | config-property | 2 | 2 | 2025-05-01 | Yes |

**Orphaned Flags** (if any)

| Flag Name | Reason | Locations |
|---|---|---|
| `OLD_PAYMENT_V1` | In config, never checked in code | `.flags.json:12` |
| `BETA_DASHBOARD` | Checked in code, not in config | `src/dashboard.ts:88` |

**Stale Flags** (if any, sorted oldest-first)

| Flag Name | Last Modified | Days Old | Files |
|---|---|---|---|
| `EXPERIMENT_XYZ` | 2024-09-01 | 276 | `src/experiment.ts` |

If no orphaned or stale flags are found, print "No orphaned flags detected." and "No stale flags detected." respectively.

### 7. Guardrails

- NEVER auto-delete or auto-modify any flag.
- NEVER remove a flag definition or conditional block without explicit user confirmation.
- If asked to clean up, redirect to `/flag-ops-cleanup [flag-name]`.
- Flag all findings as informational only — the user decides next steps.
