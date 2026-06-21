# Copilot Instructions

This is an **Obsidian vault** named `Data Engineering 101`, used for onboarding Intern Data Engineers through a structured 6-domain learning roadmap.

> **See `AGENTS.md` in the vault root for the full agent context, conventions, and constraints.**

## Vault Structure

- Notes are written in **Markdown** (`.md`)
- Obsidian supports **wikilinks** (`[[Note Name]]`), tags, and frontmatter YAML metadata
- Main entry point: `[[00 - Onboarding Roadmap]]`
- Domain notes follow the pattern: `D<N> - <Topic>.md` (D1 through D6; D7 is optional/advanced)
- Checkpoint notes: `CP<N> - <Topic>.md` inside the `Checkpoints/` folder

## Conventions

- Use `[[wikilink]]` syntax for internal links, not standard Markdown links
- Frontmatter goes at the top between `---` delimiters:
  ```yaml
  ---
  tags: [DE101, domain-1]
  date: YYYY-MM-DD
  status: not-started   # not-started | in-progress | done | needs-review
  domain: "1 of 7"
  ---
  ```
- Avoid hardcoded absolute file paths; use relative wikilinks

## Content Focus

This vault covers 7 data engineering domains for interns:
1. Foundations & Tooling (Git, Linux, Python, dev environment)
2. SQL & Data Modeling
3. Data Storage & Formats
4. Batch Processing & ETL
5. Stream Processing
6. Cloud & Orchestration
7. AI-Ready Data Engineering *(optional/advanced)*

**Confirmed Tech Stack:** Python · DuckDB (local SQL) · dbt Core · Databricks Community Edition (Azure, PySpark/SparkSQL) · Azure (Blob/ADLS Gen2, Data Factory) · Docker · Git/GitHub. Kafka is conceptual-only — no hands-on setup.

When helping with content:
- Use Markdown headings (`#`, `##`, `###`) to structure notes
- Callouts: `> [!NOTE]`, `> [!TIP]`, `> [!WARNING]`, `> [!IMPORTANT]`, `> [!EXAMPLE]`
- Use ❌ / ✅ for anti-pattern vs correct-pattern comparisons
- Use **code formatting** for all CLI commands, SQL keywords, and tool names
- Keep language intern-friendly — avoid jargon, include practical examples
