# Botport

**A passport for your bot.** One Markdown file that carries a bot's identity, tools, routines, and handoff contracts out of one harness and into another.

```
Claude Code  ──┐
Codex        ──┤   BOTPACK.md   ┌──▶  Codex
Gemini CLI   ──┼──▶  (portable) ──┼──▶  Claude Code
OpenClaw     ──┘                └──▶  Cursor / Copilot / opencode
```

## The problem

People don't run one agent anymore. They run a Claude Code bot for review, a Codex bot for implementation, an OpenClaw bot on Telegram for their morning briefing, and a Gemini bot for the big-context reads. Each one has its own instruction file, its own skill folder, its own agent definitions, its own cron jobs.

Nothing moves between them. When you switch harness — or want your review bot to hand a fix to your implementation bot — you rebuild the setup by hand and lose the routines, the routing rules, and the accumulated context.

Existing tools (`swik`, `agentsync`, `claude-code-migrate`, `conforme`) solve this as a **CLI config copier**: read JSON/TOML/MD, write JSON/TOML/MD. That works if you're a developer with both tools installed and you trust a translator with your dotfiles.

Botport solves it as a **document**. The bot describes itself in plain Markdown, a human can read and edit it, and any harness that can read a file can adopt it. No install on the source side. No binary on the target side.

## The artifact

A single `BOTPACK.md` — YAML frontmatter for the machine, prose for the human, tables for the boring parts:

- **Identity** — role, voice, what it refuses to do
- **Capabilities** — tools, MCP servers, skills it depends on
- **Routines** — schedules, triggers, what "done" looks like
- **Handoffs** — when this bot escalates, to whom, carrying what
- **Memory** — durable facts plus pointers, never the raw transcript
- **Portability** — what the target must rebuild, what it must not copy
- **Lineage** — where this bot came from, so a chain stays traceable

See [docs/bot-spec.md](docs/bot-spec.md) for the format and [examples/BOTPACK.example.md](examples/BOTPACK.example.md) for a filled one.

## Repo layout

```
botport/
├── FEATURES.md                 the scan: what harnesses ship today, and what ports
├── docs/
│   └── bot-spec.md             BOTPACK.md v0.1 format
├── examples/
│   └── BOTPACK.example.md      a complete worked example
└── skills/                     (next) the agent-facing skill that writes these
```

## Status

- [x] Feature scan across Claude Code, Codex, Gemini CLI, OpenClaw and the long tail
- [x] `BOTPACK.md` v0.1 spec
- [ ] Export skill — reads the current harness, emits `BOTPACK.md`
- [ ] Import skill — reads `BOTPACK.md`, writes native config, reports losses
- [ ] Round-trip test against two real harnesses

## Design rules

1. **Markdown is the interface.** If a human can't read it and fix it, it's not portable.
2. **Secrets never travel.** Reference env var names, never values.
3. **Losses are declared, not hidden.** Every pack ends with what the target must rebuild.
4. **Routines are first-class.** A bot without its schedule is a prompt, not a bot.
5. **Lineage is append-only.** A handoff chain you can't trace is a handoff you can't debug.
