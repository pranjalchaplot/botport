---
name: botport-export
description: >-
  Exports the current bot's configuration into a portable BOTPACK.md file.
  Use this skill when the user asks to export, package, or create a passport
  for their bot — or when a bot needs to write its own BOTPACK.md for migration
  to another agent harness.
---

# botport-export

Export the active bot's configuration into a valid `BOTPACK.md` v0.1 file.

> **Contract:** Follow the [export contract](../../docs/bot-spec.md#export--import-contracts) exactly. A silent export that hides losses is a bug.

---

## Prerequisites

Before starting, read the following references:

- [BOTPACK.md v0.1 Specification](../../docs/bot-spec.md) — the format you're emitting
- [Cross-harness feature scan](../../FEATURES.md) — what ports and what doesn't
- [Worked example](../../examples/BOTPACK.example.md) — reference output

---

## Procedure

### Step 1 — Detect the source harness

Identify which harness you're running in and gather its version:

| Harness | Detection signal |
|---------|-----------------|
| Claude Code | `CLAUDE.md` exists, or `.claude/` directory present |
| Codex CLI | `AGENTS.md` with `.codex/` directory, or `config.toml` with `[profiles]` |
| Gemini CLI | `GEMINI.md` exists, or `.gemini/` directory present |
| Antigravity | Running inside Antigravity / T3 Code environment |
| OpenClaw | OpenClaw workspace markers or heartbeat config present |
| Other (`cursor`, `copilot`, `opencode`) | `.cursor/`, `.github/copilot-instructions.md`, or `opencode.json` present |

Record the harness name and version for `source.harness` and `source.version` in frontmatter.

### Step 2 — Gather the bot's identity

Ask the user (or infer from existing config) for:

- **`name`** — a lowercase slug identifying this bot (e.g., `morning-brief`, `code-reviewer`)
- **`summary`** — one sentence: what the bot does
- **`role`** — one of: `briefer`, `builder`, `reviewer`, `router`, `researcher`
- **`model.class`** — the portable tier: `frontier`, `standard`, `fast`, or `cheap`
- **Optional hints** — `model.preferred` (current model ID), `model.reasoning` (`low` | `medium` | `high`), and `tags`

If the bot already has an instruction file, extract the job description from it. If ambiguous, ask the user.

### Step 3 — Extract instructions and operating rules

Read the harness instruction file(s):

| Harness | Files to read |
|---------|--------------|
| Claude Code | `CLAUDE.md`, `.claude/settings.json` |
| Codex CLI | `AGENTS.md`, `AGENTS.override.md`, `.codex/config.toml` |
| Gemini CLI | `GEMINI.md`, `.gemini/settings.json` |
| Antigravity | `AGENTS.md`, `GEMINI.md`, `.agents/rules/*.md` |
| OpenClaw | Workspace docs, agent instructions |

Distill into:

- **Section 1 (Job):** Three sentences max — purpose, output, hard refusal.
- **Section 2 (Operating rules):** Bullet list of behavioral constraints, timezone, tone, escalation thresholds.

### Step 4 — Inventory capabilities

**Tools:** List only tools the bot *depends on* (not every available tool). Common portable names: `shell`, `read`, `write`, `web_fetch`, `browser`.

**MCP servers:** For each server the bot uses:

| Server | Transport | Needs | Tools used |
|--------|-----------|-------|------------|

- Transport: `http` or `stdio`
- **Needs:** `$ENV_VAR` names only (or `none` if unauthenticated) — **never include secret values**
- Tools used: only the ones the bot actually calls

**Skills:** List dependent skills by name and why they're needed.

### Step 5 — Document routines

For every scheduled task, event trigger, heartbeat, or recurring workflow:

| Name | Trigger | Cadence | Action | Done when |
|------|---------|---------|--------|-----------|

- `Trigger` must be one of: `schedule`, `event`, `manual`, `heartbeat`
- `Cadence` holds the cron expression (for `schedule`), event condition (for `event`), interval (for `heartbeat`), or `on demand` (for `manual`)
- Every routine must have a concrete "Done when" condition
- Cron expressions are portable; trigger mechanisms are not — note this in Section 7
- If the bot has no routines, keep the `## 4. Routines` heading with `None.`

### Step 6 — Document handoffs

If this bot escalates or delegates to other bots:

| When | To | Carry | Expect back |
|------|----|-------|-------------|

Record the bot slugs in `handoff_targets` in frontmatter. If the bot has no handoffs, keep the `## 5. Handoffs` heading with `None.` and omit `handoff_targets`.

### Step 7 — Distill memory

**Durable facts:** 10–15 bullet points of stable operational knowledge. Never copy raw transcripts or conversation logs.

**Pointers:** File paths and URLs to templates, runbooks, trackers.

Sources to check:
- Claude Code: `MEMORY.md` (first 200 lines)
- Codex: No agent-written memory — check `AGENTS.md` for inline notes
- Gemini CLI / Antigravity: Check for memory files or persisted context

### Step 8 — Populate the Portability section

This section is **mandatory** and must be honest (an empty section means "I checked and nothing is lost," never "I skipped looking"):

**Rebuild on import:**
- List every skill, MCP server, routine trigger, and integration that must be recreated on the target.

**Do not copy:**
- Agent memory stores (already distilled above)
- Hooks with no equivalent on other harnesses (cite which)
- Session history
- Permission state that won't map

Cross-reference with [FEATURES.md](../../FEATURES.md) Section 3 ("What does not port") to ensure nothing is missed.

### Step 9 — Initialize lineage

Create or append to the lineage table:

| Date | From | To | Note |
|------|------|----|------|
| (today) | — | (source harness) | Exported via botport-export |

If the bot was previously imported from another harness, preserve existing rows and append.

### Step 10 — Assemble and validate

1. Assemble frontmatter + all 8 body sections into a single `BOTPACK.md`.
2. Verify the file is **under 400 lines**. If over, trim prose or move reference material behind pointers.
3. Run validation checks (or invoke `botport-validate`):
   - All required frontmatter fields present
   - No literal secrets (scan for patterns like `ghp_`, `sk-`, `gho_`, `Bearer `, API keys)
   - Every routine has both a trigger and a "Done when"
   - `handoff_targets` matches the `To` column in Section 5 (ignoring surrounding backticks)
   - Sections are numbered 1–8 in order
4. Write the file to the project root as `BOTPACK.md` (or a user-specified path).
5. Report what was exported and what was flagged in Portability.

---

## Secret scanning patterns

Before writing the final file, scan for and reject any of these patterns:

```
ghp_[A-Za-z0-9]{36,}          # GitHub PAT
gho_[A-Za-z0-9]{36,}          # GitHub OAuth
sk-[A-Za-z0-9_-]{20,}         # OpenAI / Anthropic keys
xox[bpoa]-[A-Za-z0-9-]{10,}   # Slack tokens
AIza[A-Za-z0-9_-]{35}         # Google API key
Bearer [A-Za-z0-9._-]+        # Bearer tokens
-----BEGIN .* KEY-----         # PEM keys
```

If any match is found, replace with `$ENV_VAR_NAME` and warn the user.

---

## Output

The skill produces one file: `BOTPACK.md` at the specified path, conforming to the v0.1 spec. It also prints a summary:

```
✓ Exported: <bot-name>
  Harness: <source-harness> <version>
  Sections: 8/8
  Lines: <count>/400
  Routines: <count>
  Handoffs: <count>
  Portability warnings: <count>
  Secrets found and stripped: <count>
```
