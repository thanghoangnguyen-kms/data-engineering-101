---
name: obsidian-vault
description: Search, create, and manage notes in the Obsidian vault with wikilinks and index notes. Use when user wants to find, create, or organize notes in Obsidian.
---

# Obsidian Vault — Data Engineering 101

## Vault location

`/Users/thanghoangnguyen/Documents/Obsidian Vault/Data Engineering 101/`

## Structure

```
Data Engineering 101/
├── AGENTS.md                        ← AI agent instructions (do not modify)
├── 00 - Onboarding Roadmap.md       ← Master index
├── D1 - Foundations & Tooling.md
├── D2 - SQL & Data Modeling.md
├── D3 - Data Storage & Formats.md
├── D4 - Batch Processing & ETL.md
├── D5 - Stream Processing.md
├── D6 - Cloud & Orchestration.md
├── D7 - AI-Ready Data Engineering.md  (optional/advanced)
├── Checkpoints/
│   ├── CP1 - Tooling & Environment Ready.md
│   ├── CP2 - SQL Proficiency.md
│   ├── CP3 - Storage & Modeling.md
│   ├── CP4 - Batch Pipeline.md
│   ├── CP5 - Stream Pipeline.md
│   ├── CP6 - Cloud Deployment.md
│   └── CP7 - AI Data Engineering.md  (optional)
└── .github/
    └── copilot-instructions.md
```

## Naming conventions

- **Domain notes**: `D<N> - <Topic>.md` (e.g., `D1 - Foundations & Tooling.md`)
- **Checkpoint notes**: `CP<N> - <Topic>.md` inside `Checkpoints/` folder
- **Master index**: `00 - Onboarding Roadmap.md`

## Frontmatter

Every note starts with YAML frontmatter:

```yaml
---
tags:
  - DE101
  - <domain-tag>        # e.g. domain-1, sql, etl, streaming, cloud
date: YYYY-MM-DD
status: not-started     # not-started | in-progress | complete
domain: "N of 7"
---
```

Checkpoint notes add:
```yaml
verified_by: ""         # mentor name + date when passed
```

Status for checkpoints: `not-started | in-progress | passed | failed`

## Linking

- Use `[[wikilinks]]` for all internal links — never `[text](path)` Markdown links
- Link to roadmap at top: `**Back to:** [[00 - Onboarding Roadmap]]`
- Navigation at bottom: `*Previous: [[D<N-1> - ...]] | Next: [[D<N+1> - ...]]*`
- Checkpoint link at bottom: `*Checkpoint: [[Checkpoints/CP<N> - <Topic>|CP<N>]]*`

## Callout types

| Type | Purpose |
|------|---------|
| `[!NOTE]` | Domain overview, context |
| `[!TIP]` | Learning tips, quick wins |
| `[!WARNING]` | Anti-patterns, common mistakes |
| `[!IMPORTANT]` | Must-know concepts |
| `[!EXAMPLE]` | Code examples |
| `[!CHECKPOINT]` | Milestone criteria |

## Domain note structure

Every domain note must end with:
1. `## ✅ Practice Checklist` — checkboxes `- [ ]`
2. `## 📚 Domain References` — table of authoritative sources
3. `## 🃏 Quick-Reference Flash Cards` — Q&A pairs
4. Checkpoint wikilink + Prev/Next navigation

## Workflows

### Search for notes

```bash
# Search by filename
find "/Users/thanghoangnguyen/Documents/Obsidian Vault/Data Engineering 101/" -name "*.md" | grep -v ".agents" | grep -i "keyword"

# Search by content
grep -rl "keyword" "/Users/thanghoangnguyen/Documents/Obsidian Vault/Data Engineering 101/" --include="*.md"
```

Or use Grep/Glob tools directly on the vault path.

### Find a domain note

```bash
find "/Users/thanghoangnguyen/Documents/Obsidian Vault/Data Engineering 101/" -name "D*.md"
```

### Find checkpoint notes

```bash
find "/Users/thanghoangnguyen/Documents/Obsidian Vault/Data Engineering 101/Checkpoints/" -name "*.md"
```

### Find backlinks to a note

```bash
grep -rl "[[D3 - Data Storage" "/Users/thanghoangnguyen/Documents/Obsidian Vault/Data Engineering 101/" --include="*.md"
```

### Create a new domain note

1. Use `D<N> - <Topic>.md` filename format
2. Add frontmatter (see template above)
3. Open with `[!NOTE]` callout + `**Back to:** [[00 - Onboarding Roadmap]]`
4. Structure subdomains as `## N.M — Subdomain Title`
5. End with Practice Checklist + Domain References + Flash Cards + navigation links
6. Update `[[00 - Onboarding Roadmap]]` to include the new domain

### Create a new checkpoint note

1. Use `CP<N> - <Topic>.md` inside `Checkpoints/` folder
2. Add frontmatter with `verified_by: ""`
3. Include `[!CHECKPOINT]` callout with observable pass criteria as checkboxes
4. Add `**Verified by:**` and `**Domain:**` fields

