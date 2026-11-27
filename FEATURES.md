# What agent harnesses actually ship

Scan date: **2026-11-28**. Versions checked: Claude Code 2.1.329, Codex CLI 0.171.0, Gemini CLI 0.70.0, OpenClaw v2026.11.x.

This document is the evidence base behind [Botport](README.md) and the [`BOTPACK.md` v0.1 specification](docs/bot-spec.md). It answers two practical questions: *what do people actually configure in a working bot?* and *what survives a move between harnesses?*

---

## 1. The ten primitives

Every serious agent harness has converged on the same ten-part stack. The file names and schemas differ; the underlying concepts do not.

| # | Primitive | What it does | Claude Code | Codex CLI | Gemini CLI | OpenClaw |
|---|-----------|--------------|-------------|-----------|------------|----------|
| 1 | **Instruction file** | Always-loaded rules and conventions | `CLAUDE.md`, falls back to `AGENTS.md` | `AGENTS.md`, 32 KiB cap, `AGENTS.override.md` | `GEMINI.md` (name configurable) | workspace docs |
| 2 | **Skills** | Reusable procedures, loaded on demand | `.claude/skills/<name>/SKILL.md` | `.agents/skills/` or `~/.agents/skills` | enabled by default | bundled/managed/workspace skills |
| 3 | **Subagents** | Isolated workers with own model + tools | `.claude/agents/*.md` (YAML) | `.codex/agents/*.toml` (TOML) | agent defs | session-keyed subagents |
| 4 | **MCP servers** | External tools over a standard protocol | `.mcp.json`, OAuth, per-agent scoping | `config.toml`, per-agent scoping | `settings.json`, `includeTools`/`excludeTools` | MCP client |
| 5 | **Hooks** | Deterministic side effects on lifecycle events | ~17 events, JSON over stdin | 3 events (`SessionStart`, `Stop`, `UserPromptSubmit`) | limited | task/hook follow-ups |
| 6 | **Permissions** | What it may do without asking | permission modes + OS sandbox | `read-only` / `workspace-write` / `danger-full-access` | opt-in sandbox | approvals |
| 7 | **Model config** | Model, reasoning effort, profiles | per-agent `model:` | `[profiles.*]`, `model_reasoning_effort` | per-agent | per-agent |
| 8 | **Scheduled work** | Runs without a live session | scheduled tasks / routines (`/schedule`), API + GitHub triggers | scheduled tasks | — | **heartbeat** (~30 min) + cron jobs |
| 9 | **Distribution** | Shipping a bundle of the above | plugins + marketplaces | plugins (`agents/openai.yaml`) | extensions | skill self-install |
| 10 | **Sessions** | Resume, fork, rewind | JSONL, `/rewind`, ~30 day checkpoints | JSONL, `codex resume`, fork | session files | session lanes |

### Notes that matter for portability

- **`AGENTS.md` is the lingua franca — and still incomplete.** Now an open standard under the Agentic AI Foundation (Linux Foundation), `AGENTS.md` is read natively by Codex, Cursor, Copilot, Windsurf, Amp, and Gemini CLI. Yet it only carries *instructions*: no tools, no schedules, no handoffs.
- **`SKILL.md` is the one genuinely portable unit.** Skills move between harnesses mostly unchanged, aside from harness-specific frontmatter such as Claude's `context: fork` or `user-invocable`.
- **Slash commands are gone as a separate primitive.** In 2026 they were folded into skills. Legacy `.claude/commands/*.md` files still resolve, while new reusable workflows belong in `SKILL.md`.
- **Memory is deeply asymmetric.** Claude Code writes and loads its own `MEMORY.md` (first 200 lines / 25 KB per session). Codex has **no agent-written memory** — anything durable lives in `AGENTS.md` or gets lost. That asymmetry is the single largest hidden loss in a Claude → Codex move.
- **Hook coverage diverges by an order of magnitude.** Claude Code exposes ~17 lifecycle events; Codex exposes 3. Deterministic `PreToolUse` validation hooks have no Codex equivalent and must be translated into sandbox policy or explicit operating rules.

---

## 2. What people actually use, ranked

Ranked by how often each primitive shows up in real, long-lived setups — derived from vendor documentation and practitioner writeups (there is no public usage telemetry).

| Rank | Feature | Why it sticks |
|------|---------|---------------|
| 1 | Instruction file (`AGENTS.md` / `CLAUDE.md`) | The only surface guaranteed to be loaded next week. Every user correction eventually lands here. |
| 2 | Skills | "Turn anything you repeat twice into a skill." Absorbed slash commands, prompt snippets, and half of subagents. |
| 3 | MCP servers | One protocol across harnesses — and usually the reason a bot setup is worth porting at all. |
| 4 | Subagents | Keeps the main context window clean via parallel review, exploration, and testing. |
| 5 | Scheduled tasks / routines / heartbeat | The line between a chatbot and a bot: morning briefings, nightly triage, drift checks. |
| 6 | Permissions + sandbox | Configured once and forgotten — until you move machines or harnesses and get burned. |
| 7 | Hooks | Formatting, linting, audit trails, and blocking dangerous commands before execution. |
| 8 | Memory | High operational value, low portability (agent-written notes in Claude vs. none in Codex). |
| 9 | Plugins / marketplaces | The distribution wrapper around everything above. |
| 10 | Chat channels | Slack, WhatsApp, Telegram — where autonomous bots actually deliver work. |

**The pattern:** People accumulate items 1–5 over weeks, then hit a wall — adopting a second harness, setting up a new machine, or sharing a workflow with a teammate on another tool. Those exact five items have no built-in export button.

---

## 3. What does not port

Botport exists to make this table explicit. A migration that hides these losses creates silent regressions on the target harness.

| Surface | Ports? | What actually happens |
|---------|--------|----------------------|
| Prose instructions | ✅ Clean | Identical concept; merge into the target's instruction file |
| Skills (`SKILL.md`) | ✅ Mostly | Same file format; audit Claude-only frontmatter (`context: fork`, `user-invocable`) |
| MCP server definitions | ⚠️ Structural | `command`/`args`/`env` and HTTP `url` translate; OAuth flows, headers, and tool filters must be redone |
| Subagent definitions | ⚠️ Rewrite | Markdown+YAML ↔ TOML; field names differ and specific `tools`/`model` values may not exist |
| Routine schedules | ⚠️ Rewrite | Cron expressions travel; trigger mechanisms do not (heartbeat vs. cloud routine vs. GitHub trigger) |
| Model names | ❌ Indirect | `claude-opus-4-5` means nothing to Codex; port a capability *class* (`frontier` / `standard` / `fast` / `cheap`) plus a preference hint |
| Hooks | ❌ Mostly | ~17 events → 3; `PreToolUse` guards become sandbox config or prose rules |
| Agent-written memory | ❌ Summarize | Machine-local store; distill into durable facts and pointers, never copy raw store |
| Secrets / OAuth tokens | 🚫 Never | Reference `$ENV_VAR` names only — never values |
| Sessions / transcripts | 🚫 Never | Private, machine-local, and mostly noise |
| Permission state | ⚠️ Loose | Approval modes map loosely; OS sandbox boundaries do not |

---

## 4. Prior art, and the gap

| Tool | Approach | Gap |
|------|----------|-----|
| `swik` | CLI, Claude ↔ Codex, dry-run + backups | Two harnesses only; no routines, no handoff graph, no portable doc |
| `agentsync` | CLI, 6 harnesses, per-artifact sync | Config-shaped, not bot-shaped; no identity, routines, or lineage |
| `claude-code-migrate` | CLI, one-directional | Narrow scope; no targets beyond Codex |
| `conforme` | Format equivalence reference | Reference tables, not a transferable artifact |
| Vendor `/init` commands | Draft `AGENTS.md` from a codebase | Describes the *repo*, not the *bot* |

**The gap:** Every existing tool is a **dotfile translator for developers who have both tools installed locally**. None produces a portable, human-readable artifact that describes a *bot* — its job, its schedules, its escalation contracts, and its lineage — and none is packaged as a skill so the bot itself can author its own handoff.

That is Botport's wedge: **a bot that writes its own passport.**

---

## 5. How this shapes `BOTPACK.md`

Every design decision in [docs/bot-spec.md](docs/bot-spec.md) traces directly back to this scan:

- **Prose + tables over raw JSON/TOML** because instructions and `SKILL.md` are the most portable surfaces, while schemas break across vendors.
- **`model.class` alongside `model.preferred`** because literal model IDs fail across providers.
- **Dedicated `Routines` and `Handoffs` sections** because scheduled work and cross-bot escalation are what turn a prompt into an operational system.
- **Distilled `Memory` + pointers** because agent memory stores are asymmetric and transcripts are noise.
- **Mandatory `Portability` and `Lineage` sections** because every non-trivial port is lossy, and undeclared losses are impossible to debug two hops later.

---

## Sources

- [Claude Code — Hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Code — Scheduled tasks / routines](https://docs.claude.com/en/docs/claude-code/routines)
- [Claude Code — Subagents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- [Codex — Custom instructions with AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- [Codex — Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents)
- [Codex — Customization overview](https://learn.chatgpt.com/docs/customization/overview)
- [Agentic Primitives Compared: Codex vs Claude vs Gemini](https://codex.danielvaughan.com/2026/03/26/agentic-primitives-codex-claude-gemini/)
- [Migrating a Workflow from Claude Code to Codex CLI](https://codex.danielvaughan.com/2026/03/26/migrating-claude-code-to-codex-cli/)
- [Claude Code vs Codex vs Gemini CLI (2026)](https://drydock.si/guides/claude-code-vs-codex-vs-gemini-cli)
- [Coding Agent Memory Compared 2026](https://iadecider.com/articles/agent-memory-and-persistence-reference-2026)
- [OpenClaw — Heartbeat & cron](https://open-claw.bot/guide/heartbeat-automation)
- [swik](https://github.com/m3252/swik) · [agentsync](https://github.com/chrisleekr/agentsync) · [conforme](https://github.com/maxgfr/conforme)
