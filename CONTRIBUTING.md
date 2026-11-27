# Contributing to Botport

Botport is early and intentionally small. The goal right now is getting the **`BOTPACK.md` format**, **cross-harness portability matrix**, and **export/import skills** sharp through real-world use.

---

## Ways to contribute

1. **Stress-test the spec (`docs/bot-spec.md`)**
   - Try hand-writing a `BOTPACK.md` for one of your own bots and note where the format feels cramped or ambiguous.
   - Weigh in on the [open questions for v0.2](docs/bot-spec.md#open-questions-for-v02) by opening a spec-feedback issue.

2. **Expand or correct the feature scan (`FEATURES.md`)**
   - Harnesses evolve quickly. If a primitive changed, a version behavior shifted, or you run a harness not yet covered (Cursor, Copilot, Windsurf, opencode, Amp), open a PR or harness-gap issue with links to primary documentation.

3. **Add worked examples (`examples/`)**
   - Real, anonymized packs for different bot roles (`builder`, `reviewer`, `router`, `researcher`) help shape the export/import contracts.
   - Name files `<bot-name>.BOTPACK.md` (or keep `BOTPACK.example.md` as the primary reference).

4. **Help build the skills (`skills/`)**
   - The next milestone is `botport-export` and `botport-import` as portable `SKILL.md` skills.
   - Before writing code or skill prompts, make sure your design follows the [Export & import contracts](docs/bot-spec.md#export--import-contracts).

---

## Contribution rules

- **Never commit secrets.** Examples must only reference `$ENV_VAR` names, never real tokens, keys, emails, or private URLs.
- **Keep packs under 400 lines.** If an example grows past 400 lines, trim prose or move detail behind a pointer.
- **Cite primary sources.** Changes to `FEATURES.md` should reference official vendor docs or verifiable practitioner writeups.
- **Keep README and spec aligned.** If you change a section name or frontmatter field in `docs/bot-spec.md`, update `README.md` and `examples/BOTPACK.example.md` in the same PR.

---

## Pull requests

Keep PRs focused on one change — a spec refinement, a harness update, a worked example, or a skill. Check the [PR template](.github/PULL_REQUEST_TEMPLATE.md) before submitting.
