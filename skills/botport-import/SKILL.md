---
name: botport-import
description: >-
  Imports a BOTPACK.md file into the current agent harness, mapping portable
  bot configuration to native files and conventions. Use this skill when the
  user wants to onboard a bot from a BOTPACK.md passport, migrate a bot to
  this harness, or reconstruct a bot from its portable specification.
---

# botport-import

Import a `BOTPACK.md` v0.1 file into the active harness, mapping every section to native configuration.

> **Contract:** Follow the [import contract](../../docs/bot-spec.md#export--import-contracts) exactly. A silent import is a bug — every loss must be reported.

---

## Prerequisites

Before starting, read:

- [BOTPACK.md v0.1 Specification](../../docs/bot-spec.md) — the format you're parsing
- [Cross-harness feature scan](../../FEATURES.md) — portability matrix
- [Worked example](../../examples/BOTPACK.example.md) — reference input

---

## Procedure

### Step 1 — Read and parse the BOTPACK.md

1. Read the file specified by the user (default: `BOTPACK.md` in the project root).
2. Parse the YAML frontmatter. Extract all fields.
3. Parse the body sections by `##` headings (numbered 1–8).
4. If frontmatter and prose disagree on any point, **the prose wins** — report the conflict.

### Step 2 — Validate the pack

Run the following checks before proceeding:

| Check | Action on failure |
|-------|-------------------|
| `botport` version | Refuse import on unknown major version (`!= 0`); warn if minor version `> 1` |
| Required frontmatter fields present (`botport`, `name`, `summary`, `source.harness`, `source.captured`, `model.class`) | Stop and list missing fields |
| All 8 body sections present | Warn on missing optional sections; stop on missing Section 7 (Portability) |
| No literal secrets detected | Stop and list findings — the pack is not safe to import |
| Unresolved `$ENV_VAR` references in Section 3 | Stop and list every missing variable before writing MCP configs |
| Every routine has `Trigger` and `Done when` | Warn — incomplete routines may not be recreatable |
| `handoff_targets` matches Section 5 `To` column | Warn on mismatches (ignore backticks around slugs) |

If validation fails with a stop condition, report all issues and do not proceed.

### Step 3 — Detect the target harness

Identify the current harness using the same detection logic as `botport-export`:

| Harness | Target instruction file | Config locations |
|---------|------------------------|------------------|
| Claude Code | `CLAUDE.md` | `.claude/settings.json`, `.claude/agents/*.md`, `.mcp.json` |
| Codex CLI | `AGENTS.md` | `.codex/config.toml`, `.codex/agents/*.toml` |
| Gemini CLI | `GEMINI.md` | `.gemini/settings.json` |
| Antigravity | `AGENTS.md` / `GEMINI.md` | `.agents/`, MCP configs |
| OpenClaw | Workspace docs | Agent config, heartbeat config |

### Step 4 — Map model & instructions (Frontmatter + Sections 1–2)

Map `model.class` (`frontier` / `standard` / `fast` / `cheap`) and optional `model.reasoning` (`low` / `medium` / `high`) to the target harness's closest equivalent (e.g., Codex `[profiles.*]` + `model_reasoning_effort`). Treat `model.preferred` as an advisory hint only — report if it cannot be matched on the target.

Write the bot's Job and Operating Rules into the target's instruction file (watch harness size limits such as Codex's 32 KiB `AGENTS.md` cap).

**Claude Code → `CLAUDE.md`:**
```markdown
# <bot-name>

<Section 1: Job content>

## Operating rules

<Section 2: Operating rules content>
```

**Codex CLI → `AGENTS.md`:**
```markdown
# <bot-name>

<Section 1: Job content>

## Operating rules

<Section 2: Operating rules content>
```

**Gemini CLI → `GEMINI.md`:**
```markdown
# <bot-name>

<Section 1: Job content>

## Operating rules

<Section 2: Operating rules content>
```

If the target instruction file already exists, **show a diff before overwriting**. Offer to merge or replace.

### Step 5 — Map capabilities (Section 3)

**MCP servers:**

| Source format | Target format | Action |
|---------------|---------------|--------|
| Table rows | `.mcp.json` (Claude) | Create entries with `command`/`args`/`env` stubs |
| Table rows | `config.toml` (Codex) | Add `[mcp_servers.<name>]` sections |
| Table rows | `settings.json` (Gemini) | Add server entries |

For each server:
1. Check if every required `$ENV_VAR` (skip `none` / `—`) is set in the environment. If any are missing, **stop and list every missing variable**.
2. Create the config entry referencing `$ENV_VAR` names — **never inline secret values** into tracked config files.
3. Note any OAuth flows, headers, or tool filters that need manual re-authorization or reconfiguration.

**Skills:**
- If the source pack references skills, check if they exist in the target workspace.
- If a skill folder is provided alongside the pack, copy it to the target's skills directory.
- If a skill is missing, add it to the "Rebuild" report.

**Tools:**
- Verify that required native tools (`shell`, `read`, `write`, `web_fetch`, `browser`) are available in the target harness.
- If a tool is not available, add it to the loss report.

### Step 6 — Map routines (Section 4)

| Source trigger | Claude Code | Codex CLI | Gemini CLI | OpenClaw |
|----------------|-------------|-----------|------------|----------|
| `schedule` (cron) | `/schedule` or routine | Scheduled task | — (manual) | Cron job |
| `event` | Hook or routine | Hook (`UserPromptSubmit`) | — | Heartbeat + condition |
| `heartbeat` | Routine | — | — | Native heartbeat |
| `manual` | Skill | Skill | Skill | Manual trigger |

For each routine:
1. Carry the cron expression as-is (it's portable).
2. Map the trigger mechanism to the target's native equivalent.
3. If the target has no equivalent (e.g., Gemini CLI has no scheduled tasks), report it as a loss and suggest a workaround (e.g., external cron, CI workflow).
4. Write the routine definition in the target's format.

### Step 7 — Map handoffs (Section 5)

Document all handoff contracts (`When` / `To` / `Carry` / `Expect back`) in the target instruction file. If a `To` target represents a local subagent rather than an external bot, also scaffold a subagent stub when supported:

- **Claude Code:** `.claude/agents/<target-bot>.md` with the handoff contract as instructions.
- **Codex CLI:** `.codex/agents/<target-bot>.toml` with the contract.
- **Gemini CLI / Antigravity / OpenClaw:** Document the handoff contract in the instruction file (or workspace agent defs if used).

### Step 8 — Map memory (Section 6)

| Target harness | Action |
|----------------|--------|
| Claude Code | Write durable facts to `MEMORY.md` (keep within first 200 lines / 25 KB); add pointers to `CLAUDE.md` |
| Codex CLI | Add durable facts and pointers to `AGENTS.md` (Codex has no agent-written memory) |
| Gemini CLI / Antigravity | Add durable facts and pointers to `GEMINI.md` or a linked reference file |
| OpenClaw | Add to workspace docs |

### Step 9 — Process Portability section (Section 7)

Read the "Rebuild on import" and "Do not copy" lists:

1. For each "Rebuild" item, check if it's been handled in Steps 4–8. If not, add it to the final report.
2. For each "Do not copy" item, verify it was not accidentally copied.
3. Add any **additional** incompatibilities discovered during import that weren't in the source pack's Section 7.

### Step 10 — Update lineage (Section 8)

Append a hop to the lineage table in the imported `BOTPACK.md`:

| Date | From | To | Note |
|------|------|----|------|
| (today) | (source harness) | (target harness) | Imported via botport-import; (summary of losses) |

Write this updated `BOTPACK.md` to the target workspace so the lineage travels forward (showing a diff first if `BOTPACK.md` already exists there).

### Step 11 — Final report

Print a structured import report:

```
botport-import: <bot-name>
━━━━━━━━━━━━━━━━━━━━━━━━━━

Source:  <source-harness> <version> (captured <date>)
Target:  <target-harness>

✓ Written
  - Instruction file: <path>
  - MCP servers: <count> entries
  - Skills: <count> mapped
  - Routines: <count> created
  - Handoff contracts: <count> documented
  - Memory: <count> durable facts

⚠ Requires manual action
  - <list of items needing manual setup (OAuth, missing skills, etc.)>

✗ Losses
  - <list of features that could not be mapped>

⚠ Missing environment variables
  - <list of $ENV_VARs not found>
```

**Never overwrite files without showing a diff first.**

---

## Error conditions

| Condition | Behavior |
|-----------|----------|
| Unknown `botport` major version | Refuse import |
| Literal secrets in pack | Refuse import, list findings |
| Unresolved `$ENV_VAR` | Stop, list all missing variables |
| Target file exists | Show diff, ask before overwriting |
| Frontmatter ↔ prose conflict | Warn, use prose value |
