# Prompt audit: claw-hwp

**Date:** 2026-09-28 · **Proposed diff:** [`proposed.diff`](proposed.diff). It is proposed only; nothing in the plugin was changed. To apply it: `git apply prompt-audit/proposed.diff`. Each finding is its own hunk, so you can take them one at a time.

## Assumptions (Step 0)

- **Scope:** everything in the repository that reaches the model as text, i.e. the `hwp` skill: `plugins/claw-hwp/skills/hwp/SKILL.md`, `references/*.md`, `themes/*.md`, `scripts/README.md`, plus the plugin and marketplace manifests. The repo has no `CLAUDE.md`, `AGENTS.md`, rule files, commands, subagents or `.claude/` settings. No file outside the repo was read.
- **Target model:** this is agent-configuration text (a Claude Code / Codex plugin skill) and it pins no model, so it was audited against the model running the audit, **Claude Opus 5.5**. `SKILL.md:88` also documents installing through **Codex** (a non-Anthropic agent). That was recorded as context only. No change to the provider is proposed.
- **Provenance:** the clone is shallow (every line blames to `^e6620dc`), so `git blame` could not show which passage is newer. Every conflict below is instead resolved **against the scripts themselves**, which are the ground truth for what the ops do.
- **Language:** JS/Node + Python. Request code: none. The plugin never calls the Claude API.

## Summary

The skill's prompting is in good shape for current models. Its emphatic rules nearly all come with reasons, and the SECURE FILL prohibitions protect real data, so they belong on the keep list. The trigger-heavy frontmatter `description` is routing text and is kept too. The real problem is **stale facts: the skill has outgrown parts of its own text.** The three findings with the most impact:

1. **`SKILL.md:635` says "Default to `.hwpx` output."** The same file says four times (379, 385, 396, 454) that `.hwp` ↔ `.hwpx` conversion was removed and that output always keeps the input's format. A model that reads the pitfalls list last may try a conversion that no longer exists.
2. **`SKILL.md:464` tells the model to call `replace_text {find, replace}` for *both* formats.** The `.hwp` path only reads `query`/`replacement` (`create.js:1648`, `cell-patch.js:1510`) and throws `'query' is required` when they are missing. Only `hwpx-edit.js:4672` reads `find`/`replace`. As written, every `.hwp` form fill that follows this section will fail on its first `replace_text`.
3. **`SKILL.md:482` says large `.hwp` forms can take "text/cells only".** In fact `insert_image` and `place_seal` are raw-patch ops (`create.js:3640`, `3649`; `SKILL.md:298`, `433`) and are Hancom-safe at any size. Only `append_image` has the size limit. The text as written makes the model refuse seal and image placement on large application forms.

**Counts by group:** Group 1 (dated prompt text): 3 · Group 2 (brittle skill/config files): 10 · Group 3 (tool descriptions): not applicable (no API tool definitions; the skill `description` is trigger text) · Group 4 (request config / architecture): not applicable (no request-building code, no sub-agents). Flag-only items: 5.

## Findings

| # | Location | Evidence | Pattern | Why obsolete | Confidence | Action |
|---|---|---|---|---|---|---|
| 1 | `SKILL.md:635` | "HWP 5.0 lossy round-trip: `.hwp` → `.hwpx` → `.hwp` may drop formatting. **Default to `.hwpx` output.** Only round-trip back to `.hwp` on explicit user request" | G2: instruction files that contradict each other (same file) / volatile specifics | Contradicts `SKILL.md:379/385/396/454` ("There is no `.hwp ↔ .hwpx` conversion — it was removed… `.hwp` in → `.hwp` out"). No conversion path exists in the scripts. | High | `remove` |
| 2 | `SKILL.md:464`, `469-470` | "`replace_text {find, replace}` for inline placeholder text", followed by "every occurrence of `find`" (under "Common (both `.hwp` and `.hwpx`)") | G2: volatile specifics (the API claim doesn't match the code) | `.hwp` `replace_text` requires `query`/`replacement` (`create.js:1648-1649`, `cell-patch.js:1510-1511`). Only `.hwpx` uses `find`/`replace` (`hwpx-edit.js:4672`). The op table at `SKILL.md:267` already says `query`/`replacement`. | High | `rewrite`: "`replace_text` for inline placeholder text (`.hwp`: `{query, replacement}`; `.hwpx`: `{find, replace}`)"; make the two gotcha bullets format-neutral |
| 3 | `SKILL.md:482` | "Adding new objects (images, etc.) to a large form (50+ pages) via the rhwp round-trip isn't 한컴-safe — on big forms, fill text/cells only." | G2: volatile specifics | `insert_image` / `place_seal` are raw-patch (`create.js:3640`, `3649`; listed in `RAW_PATCH_OPS` at `SKILL.md:433`; "Hancom-Docs compatible raw-patch" at `SKILL.md:298`). Only `append_image` has the size limit (`SKILL.md:244`). | High | `rewrite`: scope the warning to `append_image` and point to `insert_image` / `place_seal` |
| 4 | `SKILL.md:250` | "All character decorations (… super/subscript, **emphasis dot**) are supported." | G2: contradiction | `SKILL.md:261` and `create.js:489-493`, `1893-1895` say `emphasis_dot` was removed and is silently ignored. | High | `rewrite`: drop "emphasis dot" from the list |
| 5 | `SKILL.md:117` | "Conversion to PDF / DOCX is **out of scope for v0**. Will be added in a later release via LibreOffice headless." | G2: volatile specifics / time-sensitive content | The plugin is at 1.5.58 (`plugin.json`), so "v0" is stale. It also disagrees with `SKILL.md:542`, which says "v2 roadmap… not in v1". The roadmap promise encourages the model to tell users the feature is coming. | High | `rewrite`: "Conversion to PDF / DOCX is not supported." |
| 6 | `references/README.md:5-13` | "🚧 v0 in progress." · `hwpx-format.md` = "not started" · `rhwp-api.md` = "not started" | G2: volatile specifics | Both files exist and are substantial (308 and 388 lines). The table also leaves out 5 of the 8 reference files. This README says "Claude consults" these docs, so if the model reads it, it learns that real references are empty. | High | `rewrite`: a current file/purpose table |
| 7 | `SKILL.md:659-665` | References list names 4 files | G2: volatile specifics (incomplete index) | `hwp-internals.md`, `hwp-object-placement.md`, `hwpx-object-placement.md` and `hwpx-style-spacing.md` exist and are cited inline (e.g. `:255`, `:295`) but are missing from the index. `equation-syntax.md` is described as covering only `append_equation`, but `insert_equation` (`:300`) uses it too. | Medium | `add`: the four missing entries, plus the `insert_equation` mention |
| 8 | `SKILL.md:641-652`, `656` | Bundled-scripts table; "Python 3.9+ … (standard library only)" | G2: volatile specifics | `secure-fill.mjs` and `make_seal.py` are central to the skill (SECURE FILL, 서명·날인) but missing from the table. `make_seal.py` imports Pillow (`make_seal.py:25,53`; the skill says as much at `:59`), which contradicts "standard library only". | Medium | `add` two rows; `rewrite` the dependency line |
| 9 | `SKILL.md:448` | "…or write a tiny probe that calls rhwp's `getCellInfo(sec, para, ctrl, idx)` until it errors." | G2: wrong degrees of freedom / superseded method | `extract_text.js --inspect --with-cell-text` (`:108`, `:148-159`) now dumps exactly these coordinates. Telling the model to hand-write a WASM probe sends it down a slower path that is easier to get wrong. | Medium | `rewrite`: point to `--inspect --with-cell-text` |
| 10 | `SKILL.md:542` | "Our plugin's `.hwp → .pdf` conversion is on the v2 roadmap (LibreOffice headless / Hop CLI), not in v1." | G2: time-sensitive content | A roadmap promise in an instruction file goes stale on its own, and it conflicts with `:117`. | Medium | `rewrite`: "(this plugin has no PDF conversion)" |
| 11 | `SKILL.md:261` | "**Removed in 1.5.x** — `emphasis_dot`: previously documented as a prop, … confirmed in repeated 한컴독스 verification cycles… Op no longer accepts…" | G1d: migration-relative phrasing; G2: history narrative | The text reads as a diff against an old version the model never saw. The rule that still matters is short: the prop is ignored, so use another cue. | Medium | `rewrite` into a present-tense rule |
| 12 | `SKILL.md:625` | "…(24 failed iterations before fix landed)" | G2: history narrative | Development history gives the rule no authority; the behaviour it prescribes (suggest verification for lists) stands on its own. | Medium | `remove` the parenthetical |
| 13 | `SKILL.md:243` | "…the earlier blanket 'in-place = Hancom Docs ✗' assumption came from testing large forms only." | G1d: migration-relative phrasing; G2: history narrative | It refers to a previous rule the model never saw ("earlier assumption"). The current rule is fully stated without it. | Medium | `remove` the clause |

### Flag only (not in the diff)

| # | Location | Note | Confidence |
|---|---|---|---|
| F1 | `scripts/README.md` | Maintainer doc with a v0-era status table: it lists 6 of the scripts and says `extract_text.js` uses an "in-memory rhwp conversion". It is stale, but SKILL.md never points the model at it. Refresh it when convenient. | Low |
| F2 | `SKILL.md:99` | "Don't tell the user 'preview is not supported' — that's only true on cowork". Cowork *does* get a preview path (the hosted drop-in viewer, `:563`), so the aside is slightly wrong. It's harmless, because the row that follows handles cowork correctly. | Low |
| F3 | `SKILL.md:259` | "(Same 91-font set + caveats as the `.hwp` track.)" appears *inside* the `.hwp` section. It looks copied from the `.hwpx` reference. | Low |
| F4 | `SKILL.md:256`, `:298`, `:300` | Roadmap asides ("planned extension", "still pending", "aligned with the HWPX team"). These are the same kind of volatile claim as #5/#10. They weren't verified against code, so they are left unedited. | Low |
| F5 | `SKILL.md` overall (664 lines) | Several facts appear 3-4 times (PK magic-byte detection, "no conversion", preview surface rules). Once #1 is fixed the copies all agree, so under keep-list item 8 this is working redundancy and is not flagged for removal. | Low |

## Kept on purpose (examples)

- **Frontmatter `description` (`:3`)**: the ⚠️ SECURITY / "LOAD THIS SKILL FIRST" urgency is routing text whose job is to make the skill load before the model runs `ls` or `cat` on a PII file. That calibrated urgency is allowed (keep-list item 6).
- **SECURE FILL absolute rules (`:17-22`, `:43-51`)**: real data-protection constraints, each with a reason ("실측됨" = measured), including prompt-injection resistance (keep-list items 3 and 5).
- **"Batch every edit into ONE call" (`:115`)**: emphatic, but it gives measured reasons.
- **The "don't substitute markdown" rule (`:257`)**, **vocabulary discipline (`:608`)**, and **don't drive 한컴독스 with a generic browser (`:619`)**: prohibitions against demonstrated failures, with reasons.
- **Op tables and the `set_cell_text` mega-row (`:268`)**: dense, but it is tool-contract detail (keep-list item 4).

## Verify (Step 7)

Findings 1-10 are fact fixes. They were checked against the scripts cited above; no model behaviour probe is needed. Findings 11-13 only remove narrative and don't change any rule. After applying the diff, run one `.hwp` form fill that uses `replace_text` to confirm the model now sends `query`/`replacement`.
