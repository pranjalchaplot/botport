# `BOTPACK.md` — Specification v0.1

A `BOTPACK.md` file is a single Markdown document that describes **one bot** well enough for another agent harness to rebuild it without guessing.

It serves two audiences in one file:

- **Machine** — the YAML frontmatter and structured Markdown tables. Parsed, validated, diffed, and imported.
- **Human** — the prose sections. Read, audited, edited, and trusted. If frontmatter and prose ever disagree, **the prose wins** and the importer must report a conflict.

---

## Rules

1. **One bot per file.** If a system has three bots, it has three packs linked by `handoff_targets` in frontmatter and `## 5. Handoffs` in the body.
2. **No secrets, ever.** Write `$GITHUB_TOKEN`, never `ghp_…`. An importer that encounters an unresolved `$VAR` must stop and ask.
3. **No transcripts.** Carry distilled conclusions and pointers, never conversation logs. A pack that quotes chat history is a pack nobody will read.
4. **Every routine needs a trigger and a definition of done.** "Check things daily" is not a routine.
5. **Declare your losses.** Section 7 (`Portability`) is mandatory. An empty section means "I checked and nothing is lost," never "I skipped looking."
6. **Under 400 lines.** Past 400 lines, split the bot or move reference material behind pointers. A pack that does not fit in one read is not portable.
7. **Frontmatter is the contract; prose is the nuance.** Keep frontmatter strictly to fields an importer can act on.

---

## Frontmatter

```yaml
---
botport: 0.1                     # format version, required
name: morning-brief              # unique slug, required
summary: >                       # one sentence, required
  Builds a 7am briefing from email, calendar, and overnight CI.
source:                          # required
  harness: claude-code
  version: 2.1.329
  captured: 2026-11-28
role: briefer                    # briefer | builder | reviewer | router | researcher
model:
  preferred: claude-opus-4-5     # advisory hint only
  class: fast                    # frontier | standard | fast | cheap
  reasoning: low                 # optional: low | medium | high
tags: [daily, email, calendar]   # optional
handoff_targets: [fixer]         # names of bots listed in ## 5. Handoffs
---
```

### Frontmatter fields

| Field | Required | Description |
|-------|----------|-------------|
| `botport` | Yes | Spec version (`0.1`). Importers must refuse unknown major versions. |
| `name` | Yes | Lowercase slug identifying the bot across handoffs and lineage. |
| `summary` | Yes | One-sentence description of what the bot does. |
| `source.harness` | Yes | Harness the pack was exported from (e.g. `claude-code`, `codex`, `gemini-cli`, `openclaw`). |
| `source.version` | Recommended | Harness version string at capture time. |
| `source.captured` | Yes | ISO date (`YYYY-MM-DD`) of export. |
| `role` | Recommended | Primary function: `briefer`, `builder`, `reviewer`, `router`, or `researcher`. |
| `model.class` | Yes | Portable capability tier (`frontier`, `standard`, `fast`, `cheap`). This is what actually ports. |
| `model.preferred` | Optional | Source model ID. Advisory only — the importer may ignore it. |
| `model.reasoning` | Optional | Reasoning effort hint (`low`, `medium`, `high`). |
| `tags` | Optional | Discovery and grouping tags. |
| `handoff_targets` | Optional | Bot slugs referenced in `## 5. Handoffs`. Should match the `To` column. |

---

## Body sections

Use numbered `##` headings in order so both humans and parsers can scan predictably.

### 1. Job

Three sentences maximum: what the bot is for, what it produces, and what it must never do. Humans read this first to decide whether to import the pack at all.

### 2. Operating rules

A bullet list of behavioral constraints — tone, timezone, escalation thresholds, and hard refusals. On import, these map into the target's instruction file (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`) or subagent instructions.

```markdown
- Use 24-hour time in `Asia/Kolkata`.
- Never send anything to an external channel without a draft-first pass.
- If a source is down, note it in one line and continue; retry at most twice.
```

### 3. Capabilities

Three subsections. List only what the bot **depends on** — not every tool the source harness happens to expose.

**Tools** — harness-native primitives required (`shell`, `read`, `write`, `web_fetch`, `browser`).

**MCP servers** — name, transport (`http` or `stdio`), required `$ENV_VAR` names, and tools used. Never include token values or credentials.

| Server | Transport | Needs | Tools used |
|--------|-----------|-------|------------|
| `gmail` | http | `$GMAIL_OAUTH` | `search`, `read` |
| `gh` | stdio | `$GITHUB_TOKEN` | `list_prs`, `ci_status` |

**Skills** — referenced by name, with the reason the bot depends on each.

| Skill | Why |
|-------|-----|
| `daily-brief` | Formats the digest consistently for a 30-second scan |

### 4. Routines

The section that separates a bot from a one-off prompt. Every row must specify a trigger and a concrete completion condition.

| Name | Trigger | Cadence | Action | Done when |
|------|---------|---------|--------|-----------|
| `morning` | schedule | `0 7 * * 1-5` | Collect email + calendar + CI, write digest | Digest posted to `#daily` |
| `ci-watch` | event | PR opened | Poll checks, summarize failures | Comment on PR or silence if green |
| `weekly` | schedule | `0 16 * * 5` | Roll up the week, flag drift | Summary posted in `#weekly` |

`Trigger` must be one of `schedule`, `event`, `manual`, or `heartbeat`. Cron expressions travel cleanly; the trigger *mechanism* does not — the importer maps it to whatever the target supports (cron job, cloud routine, heartbeat tick, or webhook).

### 5. Handoffs

Where cross-bot traceability lives. A bot that escalates or delegates must declare the contract in prose so another bot in any harness can act on it.

| When | To | Carry | Expect back |
|------|----|-------|-------------|
| CI red on `main` | `fixer` | failing job, log excerpt, commit range | PR link or "cannot reproduce" |
| Email needs a reply | `drafter` | thread id, tone constraint, deadline | unposted draft text |

- `Carry` is the **payload contract** — the minimum context the receiving bot needs.
- `Expect back` is the **return contract** — what the sender waits for before considering the handoff closed.

### 6. Memory

Carry distilled knowledge, not raw history.

**Durable facts** — stable operational conclusions worth carrying forward (roughly a dozen lines, never a log).

```markdown
- Standup is 09:30 IST; the digest must land before it.
- `#daily` is read by the whole team — no internal codenames.
- `integration-tests` on `main` is flaky; retry once before escalating.
```

**Pointers** — paths and URLs where detailed templates, runbooks, or trackers live. Never inline large reference files.

```markdown
- Runbook: `docs/oncall.md`
- Digest template: `skills/daily-brief/template.md`
```

### 7. Portability

Mandatory. Drafted by the exporter, verified by the importer. Make every non-trivial dependency or loss explicit.

```markdown
### Rebuild on import
- `daily-brief` skill — copy folder; format-compatible across harnesses
- `gmail` MCP server — re-add server and complete OAuth flow
- `morning` routine — cron expression is portable; trigger must be recreated

### Do not copy
- Auto-memory store — summarized into Durable facts above
- `PreToolUse` guard hook — no equivalent; enforce via `workspace-write` sandbox
- Session history — machine-local and private
```

### 8. Lineage

Append-only hop log. One row per creation or migration.

```markdown
| Date | From | To | Note |
|------|------|----|------|
| 2026-11-14 | — | claude-code | Built from scratch |
| 2026-11-28 | claude-code | codex | Initial port; hooks dropped, sandbox tightened |
```

A handoff chain you cannot trace is a handoff chain you cannot debug.

---

## Export & import contracts

### Export contract

Given an active bot in a source harness, an exporter must:

1. **Extract** instructions, dependent tools, MCP entries, skills, routines, and handoff contracts.
2. **Strip** all secret values, replacing them with `$ENV_VAR` references.
3. **Distill** durable facts from memory stores without copying raw transcripts.
4. **Populate** Section 7 (`Portability`) with known harness-specific gaps.
5. **Respect** the 400-line ceiling and append or initialize Section 8 (`Lineage`).

### Import contract

Given a valid `BOTPACK.md`, an importer must:

1. **Validate** frontmatter; refuse on an unknown `botport` major version and warn if frontmatter conflicts with body prose.
2. **Detect** the target harness and map each section to native files and conventions.
3. **Write** instructions, skills, MCP entries, and routine definitions into target locations.
4. **Stop** on any `$VAR` it cannot resolve and list every missing variable.
5. **Report** every loss from Section 7 plus any additional incompatibilities discovered during import.
6. **Never** overwrite existing configuration without showing a diff first.

Steps 4 and 5 are the point. **A silent import is a bug.**

---

## Open questions for `v0.2`

Contributions and discussion are especially welcome on these open design questions:

- Should routines carry an explicit `timezone` field separate from cadence (currently stated in Operating rules)?
- Should `handoff_targets` resolve to relative pack paths (e.g. `./fixer.BOTPACK.md`) in addition to slugs, so multi-bot graphs can be validated offline?
- Should `role` stay a closed enum or become free text with `tags` carrying classification?
- Where is the clean line between inline `Memory` (Section 6) and a linked `MEMORY.md` file?
