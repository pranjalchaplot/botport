# Botport

**A passport for your bot.** One Markdown file that carries a bot's job, rules, tools, routines, and handoff contracts out of one agent harness and into another.

[Spec (`BOTPACK.md` v0.1)](docs/bot-spec.md) · [Feature Scan](FEATURES.md) · [Worked Example](examples/BOTPACK.example.md) · [Contributing](CONTRIBUTING.md)

> **Status:** `v0.1` specification and cross-harness feature scan complete. Export and import skills are in development.

```
Claude Code  ──┐                                ┌──▶  Codex
Codex        ──┤   BOTPACK.md (portable)        ├──▶  Claude Code
Gemini CLI   ──┼──▶  + explicit loss report  ───┼──▶  Gemini CLI
OpenClaw     ──┘   + append-only lineage        └──▶  Cursor / Copilot / opencode
```

---

## The problem

Nobody runs just one agent anymore. You might run a Claude Code bot for code review, a Codex bot for implementation, an OpenClaw bot on Telegram for your morning briefing, and a Gemini CLI bot for long-context reads.

Each bot has its own instruction file, skill folder, MCP servers, permissions, and scheduled routines. **None of it moves cleanly between harnesses.** When you switch tools — or want your review bot to hand a failing build to your implementation bot — you rebuild the setup by hand and lose the routines, the routing rules, and the accumulated operational context.

Existing migration tools (`swik`, `agentsync`, `claude-code-migrate`, `conforme`) treat this as a **CLI dotfile translation problem**: read JSON/TOML/Markdown from one local directory, write JSON/TOML/Markdown into another. That works if you have both CLIs installed on the same machine and trust a script to rewrite your config.

**Botport treats it as a document problem.** The bot writes its own passport in plain Markdown. A human can read it, edit it, version-control it, and drop it into any harness that can read a file — no binary required on either side.

---

## Document over CLI

| | CLI config copiers | Botport (`BOTPACK.md`) |
|---|---|---|
| **Unit of portability** | Raw config files | One self-contained bot |
| **Requires both tools installed** | Yes | No — plain Markdown file |
| **Routines & heartbeats** | Usually dropped | First-class, with triggers and "done when" |
| **Bot-to-bot handoffs** | Not modeled | Explicit contracts (`When` / `To` / `Carry` / `Expect back`) |
| **Silent data loss** | Common when features don't map | Forbidden — Section 7 declares every loss |
| **Traceability across hops** | None | Append-only `Lineage` log |
| **Human review** | Diffing generated JSON/TOML | Under 400 lines of structured prose and tables |

---

## How it works

1. **Export** — Inside the source harness, the bot (via the upcoming `botport-export` skill) inspects its instructions, skills, MCP servers, schedules, and durable memory, then emits a `BOTPACK.md` under 400 lines with all secret values stripped to `$ENV_VAR` names.
2. **Review & edit** — Read the pack like any Markdown doc. Tighten operating rules, verify required environment variables, and inspect Section 7 (`Portability`) to see what will and won't survive the move.
3. **Import** — Hand `BOTPACK.md` to a new harness (via `botport-import`). The importer maps portable primitives to native locations, stops on any unresolved `$VAR`, shows a diff before overwriting, and reports every loss.
4. **Trace** — Every migration appends a hop to `## 8. Lineage`, and `## 5. Handoffs` keeps multi-bot escalation chains intact even when each bot lives in a different harness.

---

## Anatomy of a `BOTPACK.md`

One file per bot: YAML frontmatter for the machine, structured Markdown for the human. If frontmatter and prose ever disagree, **the prose wins** and the importer reports a conflict.

| Part | What it carries |
|---|---|
| **Frontmatter** | `botport` version, `name`, `summary`, `source` harness/version, `role`, portable `model.class`, `tags`, `handoff_targets` |
| **1. Job** | Three sentences max: what it is for, what it produces, what it must never do |
| **2. Operating rules** | Behavioral constraints, timezone, tone, and refusal boundaries |
| **3. Capabilities** | Required native tools, MCP servers (with `$ENV_VAR` names, never values), and `SKILL.md` dependencies |
| **4. Routines** | Scheduled tasks, event triggers, and heartbeats — each with an explicit `Done when` |
| **5. Handoffs** | Escalation contracts: `When`, `To`, `Carry` payload, and `Expect back` return contract |
| **6. Memory** | Distilled durable facts and file/URL pointers — never raw transcripts |
| **7. Portability** | Mandatory **Rebuild on import** and **Do not copy** checklists |
| **8. Lineage** | Append-only hop table (`Date`, `From`, `To`, `Note`) for debugging multi-hop chains |

See [docs/bot-spec.md](docs/bot-spec.md) for the complete `v0.1` specification and export/import contracts.

---

## Example at a glance

Here is the shape of a pack for a weekday briefing bot that hands CI failures to a `fixer` bot and email replies to a `drafter` bot (excerpted from [examples/BOTPACK.example.md](examples/BOTPACK.example.md)):

```markdown
---
botport: 0.1
name: morning-brief
summary: Builds a 7am briefing from email, calendar, and overnight CI, then hands failures to the fixer bot.
source:
  harness: claude-code
  version: 2.1.329
  captured: 2026-11-28
role: briefer
model:
  preferred: claude-opus-4-5
  class: fast
  reasoning: low
tags: [daily, email, calendar, ci]
handoff_targets: [fixer, drafter]
---

## 4. Routines

| Name | Trigger | Cadence | Action | Done when |
|------|---------|---------|--------|-----------|
| `morning` | schedule | `0 7 * * 1-5` | Collect email, calendar, CI; write digest | Posted to `#daily` |
| `ci-watch` | event | PR opened or pushed | Poll checks until settled, summarize failures | Comment on PR, or silence if green |
```

Read the full worked example: [examples/BOTPACK.example.md](examples/BOTPACK.example.md).

---

## What ports — and what doesn't

Every harness shares the same ten core primitives, yet implementation details diverge sharply. Botport's rule is simple: **a handoff that hides losses is worse than no handoff.**

| Surface | Portability | What happens on import |
|---|---|---|
| Prose instructions & `SKILL.md` | Clean / mostly clean | Merged into target instruction file (`AGENTS.md`, `CLAUDE.md`, `GEMINI.md`); Claude-only frontmatter flagged |
| MCP servers & subagents | Structural / rewrite | Transports and `$ENV_VAR` names carry over; OAuth flows, tool filters, and YAML/TOML schemas are rebuilt |
| Routines & heartbeats | Rewrite | Cron expressions travel; trigger mechanisms (heartbeat vs. cloud routine vs. webhook) must be recreated |
| Model names & hooks | Does not port directly | Specific model IDs map via `model.class` (`frontier` / `standard` / `fast` / `cheap`); extra hook events become sandbox rules or instructions |
| Secrets & transcripts | Never copied | Env var names only; agent memory is distilled into durable facts |

Read the full evidence base across Claude Code, Codex CLI, Gemini CLI, and OpenClaw in [FEATURES.md](FEATURES.md).

---

## Design rules

1. **Markdown is the interface.** If a human cannot read it, audit it, and fix it in a text editor, it is not portable.
2. **Secrets never travel.** Reference `$ENV_VAR` names, never token or key values.
3. **Losses are declared, not hidden.** Every pack ends with what the target must rebuild and what it must not copy. A silent import is a bug.
4. **Routines are first-class.** A bot without its triggers, cadences, and definitions of done is just a prompt.
5. **Lineage is append-only.** A handoff chain you cannot trace is a handoff you cannot debug.

---

## Repo layout

```text
botport/
├── README.md                              Project overview and architecture
├── FEATURES.md                            Cross-harness primitive scan & portability matrix
├── CONTRIBUTING.md                        How to contribute spec feedback, examples, and skills
├── docs/
│   └── bot-spec.md                        BOTPACK.md v0.1 format specification
├── examples/
│   └── BOTPACK.example.md                 Complete worked example (morning-brief)
├── .github/
│   ├── PULL_REQUEST_TEMPLATE.md           PR checklist
│   └── ISSUE_TEMPLATE/
│       ├── spec-feedback.md               Spec proposals and v0.2 open questions
│       └── harness-gap.md                 Missing harness primitives or portability quirks
└── skills/
    ├── botport-export/
    │   └── SKILL.md                       Reads active harness config, emits a valid BOTPACK.md
    ├── botport-import/
    │   └── SKILL.md                       Parses BOTPACK.md, maps to native config, reports losses
    └── botport-validate/
        └── SKILL.md                       Lints a BOTPACK.md against the v0.1 spec
```

---

## Roadmap

- [x] Feature scan across Claude Code, Codex CLI, Gemini CLI, OpenClaw, and prior art
- [x] `BOTPACK.md` v0.1 format specification and worked example
- [x] `botport-export` skill — reads the active harness and emits a valid `BOTPACK.md`
- [x] `botport-import` skill — reads `BOTPACK.md`, writes native config, stops on missing `$VAR`s, and reports losses
- [x] `botport-validate` skill — lints a `BOTPACK.md` against the v0.1 spec
- [ ] Round-trip portability tests across at least two real harnesses
- [ ] `v0.2` spec refinement (routine timezones, offline resolvable handoffs, linked memory boundaries)

---

## Contributing

The spec is intentionally small and early. If you run multi-agent setups across harnesses and hit an edge case—a missing primitive, a trigger mechanism that won't map, or a clean real-world `BOTPACK.md`—open an issue or PR. See [CONTRIBUTING.md](CONTRIBUTING.md) to get started.
