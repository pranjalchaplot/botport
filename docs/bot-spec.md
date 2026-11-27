# `BOTPACK.md` — format v0.1

A botpack is a single Markdown file that describes one bot well enough for another harness to rebuild it.

Two audiences, one file:

- **Machine** — the YAML frontmatter, plus the tables. Parsed, diffed, imported.
- **Human** — the prose. Read, edited, trusted. If the frontmatter and the prose disagree, the prose wins and the import reports a conflict.

---

## Rules

1. **One bot per file.** If you have three bots, you have three packs and a `handoffs:` section linking them.
2. **No secrets, ever.** Write `$GITHUB_TOKEN`, not `ghp_…`. An import that needs a value stops and asks.
3. **No transcripts.** Carry conclusions, not conversations. A pack that quotes a chat log is a pack nobody will read.
4. **Every routine needs a trigger and a definition of done.** "Check things daily" is not a routine.
5. **Declare your losses.** Section 7 is mandatory. An empty section means "I checked and nothing is lost", not "I didn't look".
6. **Under 400 lines.** Past that, split the bot. A pack that doesn't fit in one read isn't portable.
7. **Frontmatter is the contract, prose is the nuance.** Keep frontmatter to what an importer can act on.

---

## Frontmatter

```yaml
---
botport: 0.1                     # format version, required
name: morning-brief              # slug, required, unique
summary: >                       # one sentence, required
  Builds a 7am briefing from email, calendar, and overnight CI.
source:                          # required
  harness: claude-code
  version: 2.1.329
  captured: 2026-11-28
role: briefer                    # briefer | builder | reviewer | router | researcher
model:
  preferred: claude-opus-4-5     # advisory only
  class: fast                    # frontier | standard | fast | cheap
  reasoning: low
tags: [daily, email, calendar]
handoff_targets: [fixer]         # names of bots this one escalates to
---
```

`model.class` is the field that actually ports. `model.preferred` is a hint the importer may ignore.

---

## Body

### 1. Job

Three sentences maximum. What it is for, what it produces, what it must never do. This section is what a human reads to decide whether to import the pack at all.

### 2. Operating rules

Bullet list of behavioural constraints. Tone, escalation threshold, things it refuses. These become the target's instruction file or the subagent's `developer_instructions`.

```markdown
- Reply in the user's timezone, 24h clock.
- Never send anything to a channel without a draft-first pass.
- If a source is down, say so in one line and continue; do not retry more than twice.
```

### 3. Capabilities

Three subsections. List only what the bot *depends* on — not everything the harness offers.

**Tools** — harness-native tools it needs (`shell`, `read`, `web_fetch`, `browser`).

**MCP servers** — name, transport, and the env vars required. No values.

| Server | Transport | Needs | Tools used |
|--------|-----------|-------|------------|
| `gmail` | http | `$GMAIL_OAUTH` | `search`, `read` |
| `gh` | stdio | `$GITHUB_TOKEN` | `list_prs`, `ci_status` |

**Skills** — by name, with the trigger that made them necessary.

| Skill | Why |
|-------|-----|
| `daily-brief` | Formats the digest consistently |

### 4. Routines

The section most packs will get wrong, and the one that makes a bot a bot.

| Name | Trigger | Cadence | Action | Done when |
|------|---------|---------|--------|-----------|
| `morning` | schedule | `0 7 * * 1-5` | Collect email + calendar + CI, write digest | Digest posted to `#daily` |
| `ci-watch` | event | PR opened | Poll checks, summarize failures | Comment on PR or silence |
| `weekly` | schedule | `0 16 * * 5` | Roll up the week, flag drift | Summary in `#weekly` |

`Trigger` is one of `schedule`, `event`, `manual`, `heartbeat`. The cron expression travels; the trigger *mechanism* does not — the importer maps it to whatever the target has (cron job, cloud routine, heartbeat tick, webhook).

### 5. Handoffs

Where traceability lives. A bot that escalates must say so in a form another bot can act on.

| When | To | Carry | Expect back |
|------|----|-------|-------------|
| CI red on `main` | `fixer` | failing job, log excerpt, commit range | PR link or "cannot reproduce" |
| Email needs a reply | `drafter` | thread id, tone constraint | draft text |

`Carry` is the payload contract — the minimum the receiving bot needs. `Expect back` is the return contract. Both are prose so they survive any transport.

### 6. Memory

**Durable facts** — conclusions worth carrying. A dozen lines, not a log.

```markdown
- Standup is 09:30 IST; the digest must land before it.
- `#daily` is read by the whole team — no internal codenames.
- CI on `main` is flaky on `integration-tests`; retry once before escalating.
```

**Pointers** — where the rest lives. Files, URLs, repo paths. Never inline the content.

```markdown
- Runbook: `docs/oncall.md`
- Digest template: `skills/daily-brief/template.md`
```

### 7. Portability

Mandatory. Written by the exporter, read by the importer.

```markdown
### Rebuild on import
- `daily-brief` skill — copy the folder, it is format-compatible
- `gmail` MCP server — re-add and re-auth OAuth
- `morning` routine — cron expression is portable; trigger must be recreated

### Do not copy
- Auto-memory store — summarized into Durable facts above
- `PreToolUse` guard hook — no equivalent; becomes `workspace-write` sandbox
- Session history — machine-local and private
```

### 8. Lineage

Append-only. One line per hop.

```markdown
| Date | From | To | Note |
|------|------|----|------|
| 2026-11-28 | claude-code | codex | Initial port; hooks dropped, sandbox tightened |
```

A chain you can't trace is a chain you can't debug.

---

## Import contract

Given a valid pack, an importer must:

1. **Validate** frontmatter; refuse on unknown `botport` major version.
2. **Detect** the target harness and report which sections map cleanly.
3. **Write** instructions, skills, MCP entries, and routines into native locations.
4. **Stop** on any `$VAR` it cannot resolve, and list them.
5. **Report** every loss from section 7 plus anything it discovered.
6. **Never** overwrite without a diff shown first.

Steps 4 and 5 are the point. A silent import is a bug.

---

## Open questions for v0.2

- Should routines carry a `timezone` field separate from cadence? (Currently implied by prose.)
- Do we need `handoff_targets` to be resolvable *files* rather than names, so a chain can be walked offline?
- Is `role` a closed enum, or should it be free text with `tags` doing the work?
- How much of section 6 belongs in the pack vs a linked `MEMORY.md`?
