---
name: de-handbook-writing
description: Use when writing, adding, or expanding content in the Data Engineering 101 Obsidian vault — filling placeholders, adding sections, creating domain or checkpoint notes.
---

# DE Handbook Writing

## Overview

Writing handbook content follows a **5-phase loop**. Never skip to Draft — agents that do produce unreviewed content that doesn't fit the intern audience, skips structural alignment, and bypasses user approval before touching files.

## The 5-Phase Loop

```
Research → Outline → Draft → Verify → Review (→ iterate)
```

**You must stop and wait for user input at phases marked ⏸️.**

---

### Phase 1 — Research & Scope

Before writing a single sentence:

1. **Read the current note** — understand existing structure, depth, and adjacent sections
2. **Define the concept** — what is the core idea, in one sentence?
3. **Set intern depth** — what prior knowledge can be assumed? What is out of scope?
4. **Identify prerequisite links** — what domain notes or concepts should be linked?
5. **Consult authoritative sources** — official docs, well-known references. Good sources include:
   - Official tool docs (duckdb.org, docs.getdbt.com, spark.apache.org, etc.)
   - Established references listed in AGENTS.md (roadmap.sh/data-engineer, Kimball Group, andkret/cookbook)
   - Popular courses, reputable tutorials, well-known blogs
   - Avoid: paywalled resources, unknown-quality sites, AI-generated summaries as primary source

---

### Phase 2 — Outline ⏸️

**Write the outline and present it to the user before drafting.**

Format:
```
## <Section heading>
  - <Subsection or key concept>
  - <Intended code snippet: what it demonstrates>
  - <Callout type planned: [!NOTE] / [!WARNING] / etc.>
```

> ❌ "I planned it mentally" — not acceptable. The outline is a user checkpoint.
> ✅ Present the outline, wait for feedback, adjust if needed, then proceed.

---

### Phase 3 — Draft

Write content following vault conventions:

**Obsidian formatting rules:**
- Internal links: `[[wikilinks]]` only — never `[text](path)` Markdown links
- Callouts: `> [!TYPE] Title` — see types below
- Code: fenced ` ```sql ``` `, ` ```python ``` `, ` ```bash ``` `
- Anti-patterns: use ❌ / ✅ pairs inside `[!WARNING]` callouts

**Callout types:**

| Type | Use For |
|------|---------|
| `[!NOTE]` | Section overview, context |
| `[!IMPORTANT]` | Must-know concepts, foundational rules |
| `[!TIP]` | Quick wins, memory aids |
| `[!WARNING]` | Anti-patterns, common mistakes |
| `[!EXAMPLE]` | Code walkthroughs, real-world scenarios |

**Tech stack for examples:**
- SQL exercises → **DuckDB** (zero-config, `pip install duckdb`)
- Transformations → **dbt Core** (connected to DuckDB locally)
- Distributed processing → **Databricks Community Edition** (PySpark / SparkSQL)
- Cloud storage → **Azure Blob / ADLS Gen2**
- Orchestration → **Azure Data Factory**
- Streaming → **conceptual only** — no Kafka setup, no local broker
- ❌ Do NOT use PostgreSQL for hands-on examples (reference only)
- ❌ Do NOT describe local Kafka setup

**Required sections at end of every domain note:**

```markdown
## ✅ Practice Checklist
- [ ] <observable, concrete task>

## 📚 Domain References
| Resource | Use For |
|----------|---------|

## 🃏 Quick-Reference Flash Cards
**Q:** ...
**A:** ...

*Checkpoint: [[Checkpoints/CP<N> - <Topic>|CP<N>]]*

*Previous: [[D<N-1> - ...]] | Next: [[D<N+1> - ...]]*
```

**Frontmatter: update `status` when working:**
```yaml
status: in-progress   # set when you start adding content
status: complete      # set only after content is complete and reviewed
```

---

### Phase 4 — Technical Verification

Before presenting to user:

- [ ] All SQL runs on DuckDB/SparkSQL — no dialect-specific syntax without a note
- [ ] Code snippets are complete and runnable (not pseudocode unless labeled)
- [ ] Architectural claims are accurate (e.g., "ELT not ETL in cloud" — correct)
- [ ] Intern readability: no undefined jargon, no assumed expert knowledge
- [ ] All internal links use `[[wikilinks]]` — grep for `](` patterns
- [ ] Required end-sections are present

---

### Phase 5 — Review ⏸️

**Present the draft to the user. Do NOT save to file until approved.**

Show:
1. Full draft (or key sections if very long)
2. What you verified in Phase 4
3. What's intentionally out of scope

Wait for feedback. If changes are requested, loop back to Phase 3 or 4.

**Only edit the file after user approves the draft.**

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Writing prose before outline | Stop — present outline first |
| Committing file before review | Stop — show draft, get approval |
| `status` left as `not-started` | Update to `in-progress` at start, `done` at finish |
| Using PostgreSQL in DuckDB examples | Replace with DuckDB-compatible SQL |
| Using `[link](path)` for internal links | Replace with `[[wikilink]]` |
| Mentioning Kafka setup steps | Keep Kafka conceptual — diagrams only |
| Practice checklist items that aren't observable | "Understand X" → "Write a query that does X" |

## Red Flags — Stop and Go Back

- "I'll outline it in my head and just start writing"
- "The user said to be quick, I'll skip the outline"
- "This section is simple — the outline is obvious, I don't need to write it"
- "I'll save the file and show them what I wrote"
- "I'll verify the SQL later"
- "The user will see it when I'm done — that counts as review"
- "I already researched this topic — I can skip Phase 1"

All of these mean: **stop, go back to the correct phase**.
