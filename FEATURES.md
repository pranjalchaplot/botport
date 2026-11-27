# What agent harnesses actually ship

Scan date: **2026-11-28**. Versions checked: Claude Code 2.1.329, Codex CLI 0.171.0, Gemini CLI 0.70.0, OpenClaw v2026.11.x.

This is the evidence base for Botport. It answers two questions: *what do people actually configure?* and *what survives a move between harnesses?*

---

## 1. The ten primitives

Every serious harness converged on the same stack. The names and file formats differ; the concepts don't.

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

**Notes that matter for portability**

- `AGENTS.md` is now an open standard (Linux Foundation, Agentic AI Foundation) read natively by Codex, Cursor, Copilot, Windsurf, Amp, and Gemini CLI. It is the closest thing to a lingua franca, and it's still only *instructions* — nothing else travels.
- `SKILL.md` is the other open standard. Skills are the one primitive that genuinely moves between harnesses unchanged (modulo Claude-only frontmatter like `context: fork`).
- **Slash commands are gone as a primitive.** In 2026 they were merged into skills. Old `.claude/commands/*.md` files still work; new work should be a skill.
- Codex has **no agent-written memory**. Everything durable is either in `AGENTS.md` or gone. Claude Code writes its own `MEMORY.md` (first 200 lines / 25 KB load each session). This asymmetry is the single biggest loss in a Claude → Codex move.
- Hook coverage is 17 events vs 3. `PreToolUse` validation logic has no Codex equivalent — it has to become sandbox policy.

---

## 2. What people actually use, ranked

Ranked by how often it shows up in a real, working setup — not by what vendors market. Derived from vendor docs plus practitioner writeups; there is no public telemetry for this.

| Rank | Feature | Why it sticks |
|------|---------|---------------|
| 1 | Instruction file (`AGENTS.md` / `CLAUDE.md`) | The only thing guaranteed to be there next week. Every correction ends up here. |
| 2 | Skills | "Turn anything you repeat twice into a skill." Replaced slash commands, prompts, and half of subagents. |
| 3 | MCP servers | One protocol, every harness. The reason a setup is worth porting at all. |
| 4 | Subagents | Keeps the main context clean; parallel review/explore/test. |
| 5 | Scheduled tasks / routines / heartbeat | The difference between a chatbot and a bot. Morning briefings, nightly triage, drift checks. |
| 6 | Permissions + sandbox | The thing people configure once and never touch again — until they switch machines and get burned. |
| 7 | Hooks | Formatting, linting, audit trails, blocking dangerous commands. |
| 8 | Memory | Agent-written notes (Claude) vs nothing (Codex). High value, low portability. |
| 9 | Plugins / marketplaces | The install unit for everything above. |
| 10 | Chat channels | Slack, WhatsApp, Telegram. Where the bot actually lives. |

**The pattern:** people build up items 1–5, then hit a wall — a second harness, a new machine, a teammate who uses a different tool — and items 1–5 are exactly the ones with no export button.

---

## 3. What does not port

This is the table Botport exists to make explicit. A handoff that hides these losses is worse than no handoff.

| Surface | Ports? | What actually happens |
|---------|--------|----------------------|
| Prose instructions | ✅ clean | Concept is identical; merge into the target's instruction file |
| Skills (`SKILL.md`) | ✅ mostly | Same file format. Audit Claude-only frontmatter (`context: fork`, `user-invocable`). |
| MCP server definitions | ⚠️ structural | `command`/`args`/`env` and HTTP `url` translate. OAuth, headers, and tool filters need redoing. |
| Subagent definitions | ⚠️ rewrite | Markdown+YAML ↔ TOML. Fields differ; `tools`/`model` may not exist on the target. |
| Routine schedules | ⚠️ rewrite | Cron expression travels; the *trigger model* doesn't (heartbeat vs cloud routine vs GitHub trigger). |
| Model names | ❌ | `claude-opus-4-5` means nothing to Codex. Carry a *class* (frontier / fast / cheap) plus a preference. |
| Hooks | ❌ mostly | 17 → 3 events. Validation hooks become sandbox config or instructions. |
| Agent-written memory | ❌ | Machine-local. Can be *summarized* into durable facts; cannot be moved as a store. |
| Secrets / OAuth tokens | 🚫 never | Reference env var names. Never values. |
| Sessions / transcripts | 🚫 never | Private, machine-local, and mostly noise. |
| Permission state | ⚠️ | Approval modes map loosely; sandbox policy does not. |

---

## 4. Prior art, and the gap

| Tool | Approach | Gap |
|------|----------|-----|
| `swik` | CLI, Claude ↔ Codex, dry-run + backups | Two harnesses only; no routines, no handoff graph, no document |
| `agentsync` | CLI, 6 harnesses, per-artifact migrate | Config-shaped, not bot-shaped; no identity or routines |
| `claude-code-migrate` | CLI, one direction | Narrow; no target beyond Codex |
| `conforme` | Format equivalence tables | Reference material, not an artifact |
| Vendor `/init` commands | Draft `AGENTS.md` from a repo | Describes the *repo*, not the *bot* |

**The gap:** every existing tool is a **translator for developers who have both tools installed**. None produces a portable, human-readable artifact that describes a *bot* — its job, its schedule, who it hands off to — and none is packaged as a skill, so the agent itself can do the export.

That is the wedge: **a bot that writes its own passport.**

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
