---
name: botport-validate
description: >-
  Validates a BOTPACK.md file against the v0.1 specification. Use this skill
  when the user asks to check, lint, or validate a BOTPACK.md — or before
  running botport-import to verify the pack is well-formed.
---

# botport-validate

Validate a `BOTPACK.md` file against the v0.1 specification and report all issues.

---

## Prerequisites

Read the [BOTPACK.md v0.1 Specification](../../docs/bot-spec.md) before validating.

---

## Procedure

### Step 1 — Read the file

Read the `BOTPACK.md` at the path specified by the user (default: `BOTPACK.md` in the project root).

### Step 2 — Parse structure

1. Extract the YAML frontmatter (between `---` delimiters).
2. Parse the body into sections by `##` headings.
3. Record the total line count.

### Step 3 — Run validation checks

Run every check below. Classify each finding as `error` (blocks import), `warning` (import proceeds with caveats), or `info` (advisory).

#### Frontmatter checks

| # | Check | Severity |
|---|-------|----------|
| F1 | `botport` field exists; major version is `0` (`error` if unknown major, `warning` if minor `!= 1`) | error / warning |
| F2 | `name` field exists and is a lowercase slug (`^[a-z0-9]+(?:-[a-z0-9]+)*$`) | error |
| F3 | `summary` field exists and is one sentence | error |
| F4 | `source.harness` field exists | error |
| F5 | `source.captured` exists and is a valid ISO date (`YYYY-MM-DD`) | error |
| F6 | `model.class` exists and is one of `frontier`, `standard`, `fast`, `cheap` | error |
| F7 | `role` exists and is one of `briefer`, `builder`, `reviewer`, `router`, `researcher` | warning |
| F8 | `source.version` exists | info |
| F9 | `model.preferred` is a recognized model ID (advisory) | info |
| F10 | `model.reasoning` if present is one of `low`, `medium`, `high` | warning |
| F11 | `handoff_targets` is a list of strings if present | warning |

#### Structure checks

| # | Check | Severity |
|---|-------|----------|
| S1 | All 8 sections present (`## 1. Job` through `## 8. Lineage`) | warning (error for Section 7) |
| S2 | Sections are numbered in order (1–8) | warning |
| S3 | Total file length ≤ 400 lines | warning |
| S4 | No duplicate section headings | error |

#### Content checks

| # | Check | Severity |
|---|-------|----------|
| C1 | Section 1 (Job) is 3 sentences or fewer | warning |
| C2 | Section 4 (Routines) — every row has `Trigger` and `Done when` values | warning |
| C3 | Section 4 — `Trigger` values are `schedule`, `event`, `manual`, or `heartbeat` | warning |
| C4 | Section 5 (Handoffs) — `To` column values (ignoring backticks) match `handoff_targets` in frontmatter | warning |
| C5 | Section 7 (Portability) has both "Rebuild on import" and "Do not copy" subsections | warning |
| C6 | Section 8 (Lineage) has at least one row | info |

#### Security checks

| # | Check | Severity |
|---|-------|----------|
| X1 | No GitHub PATs (`ghp_[A-Za-z0-9]{36,}`, `gho_[A-Za-z0-9]{36,}`) | error |
| X2 | No API keys (`sk-[A-Za-z0-9_-]{20,}`, `AIza[A-Za-z0-9_-]{35}`) | error |
| X3 | No Slack tokens (`xox[bpoa]-[A-Za-z0-9-]{10,}`) | error |
| X4 | No Bearer tokens (`Bearer [A-Za-z0-9._-]+`) | error |
| X5 | No PEM private keys (`-----BEGIN .* KEY-----`) | error |
| X6 | No inline URLs with tokens or credentials in query strings | error |
| X7 | No hardcoded IP addresses with ports that look like internal services | warning |

#### Consistency checks

| # | Check | Severity |
|---|-------|----------|
| K1 | Frontmatter `name` matches the `# <name>` heading (if present) | warning |
| K2 | Frontmatter `source.harness` appears in Section 8 Lineage | info |
| K3 | All `$ENV_VAR` references in Section 3 use uppercase with underscores | info |
| K4 | Frontmatter and prose do not contradict on bot name, role, or summary | error |

### Step 4 — Report results

Print a structured validation report:

```
botport-validate: <filename>
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Lines: <count>/400
Sections: <found>/8

Errors (<count>)
  ✗ <check-id>: <description>

Warnings (<count>)
  ⚠ <check-id>: <description>

Info (<count>)
  ℹ <check-id>: <description>

Verdict: <PASS | FAIL | PASS WITH WARNINGS>
```

- **PASS:** No errors, zero or more warnings/info.
- **FAIL:** One or more errors.
- **PASS WITH WARNINGS:** No errors, one or more warnings.

---

## Usage notes

- Run this skill **before** `botport-import` to catch issues early.
- Run this skill **after** `botport-export` as a final quality gate.
- The validation is purely structural — it does not check whether MCP servers actually exist or whether cron expressions are valid beyond format.
