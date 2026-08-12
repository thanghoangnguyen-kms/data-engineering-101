# CLAUDE.md

> Claude Code loads this file automatically at session start. **[AGENTS.md](AGENTS.md) is the authoritative source** for this vault — it covers structure, Markdown/frontmatter conventions, content rules per track, tech stack, status-sync rules, and constraints, and is shared with other AI tools (Copilot, etc.). This file only adds Claude-Code-specific notes that don't belong in a cross-tool file. Don't duplicate `AGENTS.md` content here — if a convention applies to all agents, it belongs there, not here.

## Start here

Read `AGENTS.md` before editing any note. It is not optional context — it defines naming, frontmatter, and content structure that every domain/checkpoint note must follow.

## Token-efficient search

Use [`rtk`](https://github.com/rtk-ai/rtk) instead of raw `grep`/`find` when searching or locating files in this vault — it filters, dedupes, and groups output, cutting token usage by roughly 60-90%:

```bash
rtk grep "pattern" .   # instead of grep -r "pattern" .
rtk find "*.md" .      # instead of find . -name "*.md"
```

Fall back to plain `grep`/`find` only if `rtk` isn't installed locally.

## Skills

Project skills are kept in two places and must stay in sync:
- `.claude/skills/` — Claude Code's native skill directory (invoked via the Skill tool / `/skill-name`)
- `.agents/skills/` — read by other agent tools via the `AGENTS.md` convention

`skills-lock.json` tracks only *vendored* skills pulled from external repos; locally-authored skills need no lock entry.

## Constraints

All constraints in `AGENTS.md § 🚫 Constraints` apply here too (no files outside the vault root, don't modify `.github/copilot-instructions.md`, no planning/tracking files in the vault, etc.) — see that file for the full list rather than re-checking it here.
