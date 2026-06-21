# AGENTS.md — Data Engineering 101

> This file instructs AI agents (Copilot, Claude Code, etc.) on how to work inside this Obsidian vault.
> It supplements `.github/copilot-instructions.md` with project-specific conventions for the Data Engineering 101 onboarding vault.

---

## 📌 Project Identity

- **Vault name:** `Data Engineering 101`
- **Purpose:** Intern Data Engineer onboarding roadmap — structured learning path from foundations to production-ready skills
- **Audience:** Interns with basic programming exposure; no prior data engineering experience assumed
- **Format:** Obsidian vault — all notes are `.md` files with Obsidian-flavored Markdown

---

## 🗂️ Vault Structure

```
Data Engineering 101/
├── AGENTS.md                              ← This file
├── 00 - Onboarding Roadmap.md             ← Master index; always check this first
├── D1 - Foundations & Tooling.md
├── D2 - SQL & Data Modeling.md
├── D3 - Data Storage & Formats.md
├── D4 - Batch Processing & ETL.md
├── D5 - Stream Processing.md
├── D6 - Cloud & Orchestration.md
├── D7 - AI-Ready Data Engineering.md      ← Optional / advanced
├── Checkpoints/
│   ├── CP1 - Tooling & Environment Ready.md
│   ├── CP2 - SQL Proficiency.md
│   ├── CP3 - Storage & Modeling.md
│   ├── CP4 - Batch Pipeline.md
│   ├── CP5 - Stream Pipeline.md
│   ├── CP6 - Cloud Deployment.md
│   └── CP7 - AI Data Engineering.md       ← Optional
├── .agents/
│   └── skills/                            ← Local agent skills for this vault
└── .github/
    └── copilot-instructions.md
```

Domain notes are named `D<N> - <Topic>.md`. Checkpoint notes are named `CP<N> - <Topic>.md` inside `Checkpoints/`.

---

## 📝 Obsidian Markdown Conventions

### Always use

| Convention | Syntax | Example |
|-----------|--------|---------|
| Internal links | `[[Note Name]]` | `[[D1 - Foundations & Tooling]]` |
| Callouts | `> [!TYPE] Title` | `> [!TIP] Quick Win` |
| Frontmatter | YAML between `---` delimiters | See template below |
| Tags | `tags:` in frontmatter | `tags: [DE101, domain-1]` |

### Callout types in use

| Type | Purpose |
|------|---------|
| `[!NOTE]` | Domain overview, context |
| `[!TIP]` | Learning tips, quick wins, memory aids |
| `[!WARNING]` | Common mistakes, anti-patterns |
| `[!IMPORTANT]` | Must-know concepts, foundational rules |
| `[!EXAMPLE]` | Code examples, real-world scenarios |
| `[!CHECKPOINT]` | Milestone verification criteria |

### Never use

- Standard Markdown links `[text](path)` for internal vault links — use `[[wikilinks]]` instead
- Hardcoded absolute file paths in note content

---

## 📄 Note Frontmatter Templates

### Domain notes (D1–D7)

```yaml
---
tags:
  - DE101
  - <domain-tag>        # e.g. domain-1, sql, python, etl, streaming, cloud
date: YYYY-MM-DD
status: not-started     # not-started | in-progress | complete
domain: "N of 7"        # e.g. "1 of 7"
---
```

### Checkpoint notes (CP1–CP7)

```yaml
---
tags:
  - DE101
  - checkpoint
date: YYYY-MM-DD
status: not-started     # not-started | in-progress | complete
domain: "N of 7"
verified_by: ""         # mentor name and date when passed
---
```

---

## ✍️ Content Rules

### When writing domain notes (D1–D6)

1. **Open with** a `[!NOTE]` callout summarizing what the intern will learn in this domain
2. **Include a "Back to" link** at the top: `**Back to:** [[00 - Onboarding Roadmap]]`
3. **Structure subdomains** as `## N.M — Subdomain Title` (e.g., `## 1.3 — Git Branching Strategies`)
4. **Use comparison tables** for concepts with trade-offs (e.g., ETL vs ELT, batch vs stream)
5. **Mark anti-patterns** with `[!WARNING]` callouts and ❌ / ✅ symbols
6. **Mark must-know concepts** with `[!IMPORTANT]` callouts
7. **Use code blocks** for all CLI commands, SQL queries, and code snippets
8. **End every domain note** with:
   - A `## ✅ Practice Checklist` section (checkboxes `- [ ]`)
   - A `## 🃏 Quick-Reference Flash Cards` section (Q&A pairs)
   - A checkpoint link: `*Checkpoint: [[Checkpoints/CP<N> - <Topic>|CP<N>]]*`

### When writing checkpoint notes (CP1–CP6)

- State the **pass criteria** as a concrete, observable checklist (`- [ ]` checkboxes)
- Include a `**Verified by:**` field for mentor sign-off
- Link back to the domain note it covers: `**Domain:** [[D<N> - <Topic>]]`
- Keep criteria unambiguous — "Write a SQL query joining 3 tables", not "Understands SQL joins"

### Code formatting rules

Always use **code formatting** for:
- CLI commands: `git commit`, `pip install pandas`, `docker run`
- SQL keywords in prose: `SELECT`, `JOIN`, `GROUP BY`, `PARTITION BY`
- File/format names: `.parquet`, `.json`, `.csv`, `requirements.txt`
- Tool names in technical context: `Airflow`, `Kafka`, `Spark`, `dbt`
- Config/env keys: `SPARK_HOME`, `spark.executor.memory`, `AIRFLOW__CORE__SQL_ALCHEMY_CONN`

---

## 🔎 Reference Sources

When generating or expanding content, **start with these baseline references** for accuracy and alignment:

| Source | Use For |
|--------|---------|
| https://github.com/andkret/cookbook | DE fundamentals, platform design, tooling overview |
| https://roadmap.sh/data-engineer | Skill progression, topic coverage roadmap |
| https://duckdb.org/docs/ | DuckDB — embedded analytical SQL engine |
| https://docs.getdbt.com | dbt — SQL transformation standard |
| https://iceberg.apache.org/docs/latest/ | Apache Iceberg table format |
| https://docs.delta.io/latest/index.html | Delta Lake table format |
| https://kafka.apache.org/documentation/ | Apache Kafka documentation |
| https://airflow.apache.org/docs/ | Apache Airflow orchestration |
| https://docs.docker.com | Docker and containerization |
| https://docs.python.org | Python language reference |
| https://spark.apache.org/docs/latest/ | Apache Spark documentation |
| https://docs.greatexpectations.io | Data quality framework |

Each domain note also includes a `## 📚 Domain References` section with topic-specific links.

**These are baselines, not an exclusive list.** You may also reference other legitimate, well-established sources — official tool docs, popular courses, reputable tutorials. Avoid paywalled, outdated, or unknown-quality resources.

---

## 🛠️ Confirmed Tech Stack

This is the agreed tool stack for hands-on exercises and checkpoints. Content should use these tools in examples and practice tasks.

| Layer | Tool | Notes |
|-------|------|-------|
| **Language** | Python 3.x | Primary language for all DE work |
| **Local SQL** | **DuckDB** | Zero-config, `pip install duckdb`, runs SQL on Parquet/CSV/JSON files directly. Use for D2 SQL exercises and D3 file format practice |
| **Transformation** | **dbt Core** | Free CLI, connects to DuckDB locally and Databricks on cloud. Primary tool for D4 |
| **Distributed Processing** | **Databricks Community Edition** (Azure) | Free, browser-based, no cluster setup. PySpark + SparkSQL included. Use for D4 Spark section |
| **Cloud Platform** | **Microsoft Azure** | Azure Blob Storage / ADLS Gen2 for object storage, Azure Data Factory for orchestration (visual, low setup) |
| **Orchestration** | **Azure Data Factory** | Azure-native, visual UI, minimal config. Covers D6 orchestration concepts |
| **Containerization** | **Docker Desktop** | Local container runtime for D6 |
| **Version Control** | **Git + GitHub** | Standard; required from D1 |
| **Streaming** | Kafka — **conceptual only** | No hands-on setup required at intern level; teach via diagrams and docs |

### Tool Setup Principle
> Intern setup time should be **< 30 minutes total**. Prefer hosted/browser-based tools (Databricks, ADF) over locally-installed servers. No manual cluster configuration, no local Kafka, no PostgreSQL server.

### PostgreSQL Note
PostgreSQL is **not a hands-on tool** in this roadmap, but interns should know it exists as the most common OLTP database they'll encounter in real jobs. Mention it as reference in D2 and D3; no setup required.

---

## 🔄 Status Sync Rules

Allowed `status` values across all vault notes: **`not-started` | `in-progress` | `complete`**

Whenever a domain note's `status` changes, **also update the roadmap table** in `00 - Onboarding Roadmap.md`:

| Domain note `status` | Roadmap Status column |
|----------------------|-----------------------|
| `not-started` | 🔴 Not started |
| `in-progress` | 🟡 In progress |
| `complete` | ✅ Complete |

**Rule:** After editing any domain note's frontmatter `status`, open `00 - Onboarding Roadmap.md` and update the matching row in the Learning Path table. Never let them diverge.

---

## ✅ Verified Tech Stack Behaviors

Confirmed behaviors from live testing and official sources across this vault's sessions. **Before testing a behavior yourself**, check here first, then follow the verification hierarchy below.

### How to Verify (all tools)

When a behavior is uncertain, check in this order:

1. **This section** — previously verified, saves redundant testing
2. **Official docs** — always the authoritative source (see links per tool below)
3. **Changelog / GitHub releases** — behaviors change across versions; confirm against the installed version
4. **GitHub Issues / Discussions** — for known bugs, workarounds, or planned support
5. **Reliable community sources** — Stack Overflow accepted answers, official engineering blogs, well-known courses
6. **Live test** — only if the above are inconclusive; document the result in this section afterward

> **When you discover a new confirmed behavior**, add it to the relevant tool subsection below with a `Notes` entry and source. This prevents future agents from re-testing the same thing.

---

### DuckDB

**Docs:** [duckdb.org/docs](https://duckdb.org/docs/sql/statements/overview) · **Changelog:** [github.com/duckdb/duckdb/releases](https://github.com/duckdb/duckdb/releases)

#### DDL & Constraints

| Feature | Supported? | Notes |
|---------|-----------|-------|
| `CREATE TABLE ... PRIMARY KEY` | ✅ Yes | Inline only |
| `CREATE TABLE ... UNIQUE` | ✅ Yes | Inline only |
| `CREATE TABLE ... CHECK (expr)` | ✅ Yes | Inline only |
| `CREATE TABLE ... FOREIGN KEY` | ✅ Yes | **Enforced** — FK violations raise an error |
| `ALTER TABLE ... ADD CONSTRAINT` | ❌ No | `Not implemented Error` — define all constraints inline at `CREATE TABLE` time |
| `TRUNCATE TABLE` | ✅ Yes | Faster than `DELETE` with no `WHERE` |
| `ALTER TABLE ... DROP COLUMN` | ✅ Yes | Supported |

#### Query & Functions

| Feature | Supported? | Notes |
|---------|-----------|-------|
| `range(n)` in `FROM` clause | ✅ Yes | `FROM range(1000)` or `FROM range(1000) t(i)` |
| `list_contains(arr, val)` | ✅ Yes | Works on `VARCHAR[]` and other array types |
| `unnest(arr)` | ✅ Yes | Flattens arrays to rows |
| `STRUCT` literal insert | ✅ Yes | `{'key': 'value'}` syntax |
| `VARCHAR[]` array type | ✅ Yes | Inline array literals like `['a', 'b', 'c']` |
| `FLOAT` vs `DECIMAL` precision | ✅ Confirmed | `SUM(9.99::FLOAT)` over 1000 rows ≠ 9990; `DECIMAL(10,2)` gives exact `9990.00` |

#### EXPLAIN Output

DuckDB renders `EXPLAIN` as a **visual tree** (not tabular). Filters are embedded inside scan nodes, not separate operators.

| Operator | Present? | Notes |
|----------|---------|-------|
| `SEQ_SCAN` | ✅ Yes | Includes inline `Filters:` for predicates |
| `HASH_GROUP_BY` | ✅ Yes | Grouping operator |
| `HASH_JOIN` | ✅ Yes | Join operator |
| `PROJECTION` | ✅ Yes | Column selection step |
| `INDEX_SCAN` | ❌ Not confirmed | Does not appear even with `PRIMARY KEY` + 10k rows |
| `FILTER` (standalone) | ❌ No | Embedded in `SEQ_SCAN`, not a separate node |

---

### dbt Core

**Docs:** [docs.getdbt.com](https://docs.getdbt.com) · **Changelog:** [github.com/dbt-labs/dbt-core/releases](https://github.com/dbt-labs/dbt-core/releases)

*No verified behaviors recorded yet. Add entries here as they are confirmed.*

---

### Apache Spark / Databricks

**Docs:** [spark.apache.org/docs/latest](https://spark.apache.org/docs/latest/) · [docs.databricks.com](https://docs.databricks.com) · **Changelog:** [spark.apache.org/news](https://spark.apache.org/news/)

*No verified behaviors recorded yet. Add entries here as they are confirmed.*

---

### Azure Data Factory

**Docs:** [learn.microsoft.com/azure/data-factory](https://learn.microsoft.com/en-us/azure/data-factory/) · **Changelog:** [azure.microsoft.com/updates — Data Factory](https://azure.microsoft.com/en-us/updates/?query=data+factory)

*No verified behaviors recorded yet. Add entries here as they are confirmed.*

---

## 🚫 Constraints

- **Do not** create planning or tracking files (todos, project notes) inside the vault — this is onboarding content only
- **Do not** create files outside the vault root (`/Users/thanghoangnguyen/Documents/Obsidian Vault/Data Engineering 101/`)
- **Do not** modify `.github/copilot-instructions.md` — that is managed separately
- **Keep notes intern-friendly** — clear language, practical examples, avoid unnecessary jargon
- **Do not** link to paywalled or unstable external resources
- **Do not** introduce tools outside the confirmed stack without vault owner approval

---

## 🗃️ Domain Map (Quick Reference)

| Domain | File | Subdomains | Checkpoint |
|--------|------|-----------|-----------|
| D1: Foundations & Tooling | `D1 - Foundations & Tooling.md` | Mindset shift · Python for DE · REST APIs · Git · Linux · Dev setup (incl. DuckDB) | `Checkpoints/CP1 - Tooling & Environment Ready.md` |
| D2: SQL & Data Modeling | `D2 - SQL & Data Modeling.md` | Window functions/CTEs · SQL for DE · Query perf · Normalization · Dimensional modeling | `Checkpoints/CP2 - SQL Proficiency.md` |
| D3: Data Storage & Formats | `D3 - Data Storage & Formats.md` | OLTP/OLAP · Relational/NoSQL · DWH/Lake/Lakehouse (Snowflake/BigQuery/Databricks) · Formats · Medallion arch · Iceberg/Delta Lake · Partitioning · DuckDB · Vector DBs (optional) | `Checkpoints/CP3 - Storage & Modeling.md` |
| D4: Batch Processing & ETL | `D4 - Batch Processing & ETL.md` | ETL vs ELT · dbt · Medallion in practice · Pipeline patterns · Data quality/contracts/observability · Distributed processing · Error handling | `Checkpoints/CP4 - Batch Pipeline.md` |
| D5: Stream Processing | `D5 - Stream Processing.md` | Batch vs stream · Message queues/Kafka · Lambda/Kappa arch · Delivery guarantees · Stateful processing | `Checkpoints/CP5 - Stream Pipeline.md` |
| D6: Cloud & Orchestration | `D6 - Cloud & Orchestration.md` | Cloud fundamentals · Docker · Orchestration (Airflow/Prefect/Dagster) · Scheduling/monitoring · Data observability · Governance & cost | `Checkpoints/CP6 - Cloud Deployment.md` |
| D7: AI-Ready DE *(optional)* | `D7 - AI-Ready Data Engineering.md` | AI pipeline overview · Embedding pipelines · Vector databases · LLM data flows · Data quality for AI | `Checkpoints/CP7 - AI Data Engineering.md` |
