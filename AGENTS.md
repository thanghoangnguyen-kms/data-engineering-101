# AGENTS.md — Data Engineering 101

> This file is the **authoritative** source instructing AI agents (Copilot, Claude Code, etc.) on how to work inside this Obsidian vault — structure, conventions, content rules, tech stack, and constraints.
> It supplements `.github/copilot-instructions.md` with project-specific conventions for the Data Engineering 101 onboarding vault.
> `CLAUDE.md` supplements this file with Claude-Code-only notes (e.g. using `rtk` for token-efficient search) and defers here for everything else — don't duplicate vault conventions there; add them here instead.

---

## 📌 Project Identity

- **Vault name:** `Data Engineering 101`
- **Purpose:** Intern onboarding vault with two parallel tracks — Backend Engineering and Data Engineering — each a structured learning path from foundations to production-ready skills
- **Audience:** Interns with basic programming exposure; no prior backend or data engineering experience assumed
- **Format:** Obsidian vault — all notes are `.md` files with Obsidian-flavored Markdown
- **Tracks:**
  - 🖥️ **Backend Track** (`Backend/`) — Python, FastAPI, databases, auth, testing, async, containers (B1–B8)
  - 🗄️ **Data Engineering Track** (`DataEngineering/`) — SQL, storage formats, batch ETL, streaming, cloud orchestration (D1–D7)

---

## 🗂️ Vault Structure

```
Data Engineering 101/
├── AGENTS.md
├── 00 - Onboarding Roadmap.md             ← two-track launcher
├── 00.1 - How to Use This Vault.md
├── README.md
├── Backend/
│   ├── 00 - Backend Track Roadmap.md
│   ├── B1 - Foundations & Dev Setup.md
│   ├── B2 - Web & API Fundamentals.md
│   ├── B3 - Databases & ORM.md
│   ├── B4 - Authentication & Security.md
│   ├── B5 - Testing & Code Quality.md
│   ├── B6 - Async, Queues & Background Jobs.md
│   ├── B7 - Microservices & Containers.md
│   ├── B8 - Capstone Project.md
│   └── Checkpoints/
│       ├── CB1 - Dev Environment Ready.md
│       ├── CB2 - API Built & Documented.md
│       ├── CB3 - DB & ORM Proficiency.md
│       ├── CB4 - Auth & Security Verified.md
│       ├── CB5 - Test Suite Complete.md
│       ├── CB6 - Queue & Workers Running.md
│       ├── CB7 - Service Containerised.md
│       └── CB8 - Capstone Complete.md
├── DataEngineering/
│   ├── 00 - DE Track Roadmap.md
│   ├── D1 - Foundations & Tooling.md
│   ├── D2 - SQL & Data Modeling.md
│   ├── D3 - Data Storage & Formats.md
│   ├── D4 - Batch Processing & ETL.md
│   ├── D5 - Stream Processing.md
│   ├── D6 - Cloud & Orchestration.md
│   ├── D7 - AI-Ready Data Engineering.md
│   └── Checkpoints/
│       ├── CP1 - Tooling & Environment Ready.md
│       ├── CP2 - SQL Proficiency.md
│       ├── CP3 - Storage & Modeling.md
│       ├── CP4 - Batch Pipeline.md
│       ├── CP5 - Stream Processing.md
│       ├── CP6 - Cloud Deployment.md
│       └── CP7 - AI Data Engineering.md
└── docs/                                  ← git-ignored, internal only
    └── specs/
```

Domain notes are named `B<N> - <Topic>.md` (Backend) or `D<N> - <Topic>.md` (Data Engineering). Checkpoint notes are named `CB<N>` / `CP<N>` inside each track's `Checkpoints/` folder.

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

### Never use

- Standard Markdown links `[text](path)` for internal vault links — use `[[wikilinks]]` instead
- Hardcoded absolute file paths in note content

---

## 📄 Note Frontmatter Templates

### Backend domain notes (B1–B8)

```yaml
---
tags:
  - BE101
  - <topic-tag>         # e.g. backend-1, api, fastapi, auth, testing
date: YYYY-MM-DD
status: not-started     # not-started | in-progress | complete
domain: "N of 8"        # e.g. "2 of 8"
track: backend
---
```

### Backend checkpoint notes (CB1–CB8)

```yaml
---
tags:
  - BE101
  - checkpoint
  - backend
date: YYYY-MM-DD
status: not-started     # not-started | in-progress | complete
domain: "N of 8"
track: backend
verified_by: ""         # mentor name and date when passed
---
```

### Data Engineering domain notes (D1–D7)

```yaml
---
tags:
  - DE101
  - <domain-tag>        # e.g. domain-1, sql, etl, streaming, cloud
date: YYYY-MM-DD
status: not-started     # not-started | in-progress | complete
domain: "N of 7"        # e.g. "1 of 7"
track: data-engineering
---
```

### Data Engineering checkpoint notes (CP1–CP7)

```yaml
---
tags:
  - DE101
  - checkpoint
date: YYYY-MM-DD
status: not-started     # not-started | in-progress | complete
domain: "N of 7"
track: data-engineering
verified_by: ""         # mentor name and date when passed
---
```

---

## ✍️ Content Rules

### When writing Backend domain notes (B1–B8)

1. **Open with** a `[!NOTE]` callout summarizing what the intern will learn in this domain
2. **Include a "Back to" link** at the top: `**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]`
3. **Structure subdomains** as `## N.M — Subdomain Title` (e.g., `## 2.3 — Pydantic Schemas & Validation`)
4. **Use comparison tables** for concepts with trade-offs (e.g., async def vs def, REST vs gRPC)
5. **Mark anti-patterns** with `[!WARNING]` callouts and ❌ / ✅ symbols
6. **Mark must-know concepts** with `[!IMPORTANT]` callouts
7. **Use code blocks** for all CLI commands, Python snippets, and config examples
8. **End every domain note** with:
   - A `## ✅ Practice Checklist` section (checkboxes `- [ ]`, concrete and observable)
   - A `## 📚 Domain References` section (table of links)
   - A `## 🃏 Quick-Reference Flash Cards` section (Q&A pairs)
   - Navigation: `*Checkpoint: [[Backend/Checkpoints/CB<N> - <Topic>|CB<N>]]*`
   - Navigation: `*Previous: [[Backend/B<N-1> - ...]] | Next: [[Backend/B<N+1> - ...]]*`

### When writing Backend checkpoint notes (CB1–CB8)

- State the **pass criteria** as a concrete, observable checklist (`- [ ]` checkboxes)
- Include a `**Verified by:**` field for mentor sign-off
- Link back to the domain note: `**Domain:** [[Backend/B<N> - <Topic>]]`
- Keep criteria unambiguous — "Build a FastAPI route that returns 201", not "Understands FastAPI"

### When writing Data Engineering domain notes (D1–D7)

1. **Open with** a `[!NOTE]` callout summarizing what the intern will learn in this domain
2. **Include a "Back to" link** at the top: `**Back to:** [[DataEngineering/00 - DE Track Roadmap|DE Track Roadmap]]`
3. **Structure subdomains** as `## N.M — Subdomain Title` (e.g., `## 1.3 — Git Branching Strategies`)
4. **Use comparison tables** for concepts with trade-offs (e.g., ETL vs ELT, batch vs stream)
5. **Mark anti-patterns** with `[!WARNING]` callouts and ❌ / ✅ symbols
6. **Mark must-know concepts** with `[!IMPORTANT]` callouts
7. **Use code blocks** for all CLI commands, SQL queries, and code snippets
8. **End every domain note** with:
   - A `## ✅ Practice Checklist` section (checkboxes `- [ ]`)
   - A `## 📚 Domain References` section (table of links)
   - A `## 🃏 Quick-Reference Flash Cards` section (Q&A pairs)
   - Navigation: `*Checkpoint: [[DataEngineering/Checkpoints/CP<N> - <Topic>|CP<N>]]*`

- State the **pass criteria** as a concrete, observable checklist (`- [ ]` checkboxes)
- Include a `**Verified by:**` field for mentor sign-off
- Link back to the domain note: `**Domain:** [[DataEngineering/D<N> - <Topic>]]`
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

### Backend Track References

| Source | Use For |
|--------|---------|
| https://fastapi.tiangolo.com | FastAPI — primary web framework |
| https://docs.pydantic.dev | Pydantic v2 — validation and schemas |
| https://docs.sqlalchemy.org | SQLAlchemy — async ORM |
| https://alembic.sqlalchemy.org | Alembic — DB migrations |
| https://docs.python.org/3/ | Python language reference |
| https://docs.astral.sh/uv/ | uv — Python package manager |
| https://docs.astral.sh/ruff/ | Ruff — linter and formatter |
| https://mypy.readthedocs.io | mypy — static type checker |
| https://docs.docker.com | Docker and containerization |
| https://grpc.io/docs/languages/python/ | gRPC Python |
| https://redis.io/docs | Redis — caching and queues |

### Data Engineering Track References

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
| https://spark.apache.org/docs/latest/ | Apache Spark documentation |
| https://docs.greatexpectations.io | Data quality framework |

Each domain note also includes a `## 📚 Domain References` section with topic-specific links.

**These are baselines, not an exclusive list.** You may also reference other legitimate, well-established sources — official tool docs, popular courses, reputable tutorials. Avoid paywalled, outdated, or unknown-quality resources.

---

## 🛠️ Confirmed Tech Stack

### 🖥️ Backend Track Stack

| Layer | Tool | Notes |
|-------|------|-------|
| **Language** | Python 3.11+ | Primary language for all backend work |
| **Web Framework** | **FastAPI** | Async, auto OpenAPI docs, Pydantic-native |
| **Server** | **Uvicorn** | ASGI server for FastAPI |
| **Validation** | **Pydantic v2** | Request/response schemas, settings management |
| **ORM** | **SQLAlchemy 2.0 (async)** | Async ORM with `asyncpg` driver |
| **Migrations** | **Alembic** | Schema versioning alongside SQLAlchemy |
| **Database** | **PostgreSQL** | Primary relational DB for hands-on exercises |
| **Caching** | **Redis** | Covered in B3; used in B6 for queues too |
| **Package Manager** | **uv** | Replaces pip/venv/pip-tools; use from B1 onward |
| **Linting/Formatting** | **Ruff** | Replaces flake8 + black + isort |
| **Type Checking** | **mypy** | Static analysis; configured via `pyproject.toml` |
| **Testing** | **pytest** | Unit, integration, e2e; introduced in B5 |
| **Containers** | **Docker Desktop** | Local container runtime; introduced in B7 |
| **Version Control** | **Git + GitHub** | Required from B1 |

### 🗄️ Data Engineering Track Stack

| Layer | Tool | Notes |
|-------|------|-------|
| **Language** | Python 3.x | Primary language for all DE work |
| **Local SQL** | **DuckDB** | Zero-config, `pip install duckdb`, runs SQL on Parquet/CSV/JSON files directly. Use for D2 SQL exercises and D3 file format practice |
| **Transformation** | **dbt Core** | Free CLI, connects to DuckDB locally and Databricks on cloud. Primary tool for D4 |
| **Distributed Processing** | **Databricks Free Edition** (Azure) | Free, browser-based, **serverless-only** — no cluster creation at all. PySpark + Spark SQL included (R and Scala are not). Unity Catalog on by default. Use for D4 Spark section. Replaced Community Edition, which is retired. *Full constraint table under Verified Tech Stack Behaviors — no cache APIs, no Spark UI, no RDDs, DBFS disabled, restricted outbound internet* |
| **Cloud Platform** | **Microsoft Azure** | Azure Blob Storage / ADLS Gen2 for object storage, Azure Data Factory for orchestration |
| **Orchestration** | **Azure Data Factory** | Azure-native, visual UI, minimal config. Covers D6 orchestration concepts |
| **Containerization** | **Docker Desktop** | Local container runtime for D6 |
| **Version Control** | **Git + GitHub** | Standard; required from D1 |
| **Streaming** | Kafka — **conceptual only** | No hands-on setup required at intern level; teach via diagrams and docs |

### Tool Setup Principles

**Backend Track:** Interns install tools locally (Python, uv, Docker Desktop, VS Code). PostgreSQL runs in Docker — no native install needed. Setup time < 30 minutes.

**Data Engineering Track:** Prefer hosted/browser-based tools (Databricks, ADF) over locally-installed servers. No manual cluster configuration, no local Kafka. Setup time < 30 minutes.

### PostgreSQL Note (DE Track)
PostgreSQL is **not a hands-on tool** in the DE roadmap, but interns should know it exists as the most common OLTP database they'll encounter in real jobs. Mention it as reference in D2 and D3; no setup required.

---

## 🔄 Status Sync Rules

Allowed `status` values across all vault notes: **`not-started` | `in-progress` | `complete`**

Whenever a domain note's `status` changes, **also update the matching track roadmap**:

| Track | Roadmap file to update |
|-------|----------------------|
| Backend | `Backend/00 - Backend Track Roadmap.md` |
| Data Engineering | `DataEngineering/00 - DE Track Roadmap.md` |

| Domain note `status` | Roadmap Status column |
|----------------------|-----------------------|
| `not-started` | 🔴 Not started |
| `in-progress` | 🟡 In progress |
| `complete` | ✅ Complete |

**Rule:** After editing any domain note's frontmatter `status`, open its track roadmap and update the matching row. Never let them diverge. The top-level `00 - Onboarding Roadmap.md` shows overall track progress — update it only when an entire track moves from one phase to another.

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

#### File Formats & Extensions

Verified live on **DuckDB v1.4.5 LTS** while writing D3.

| Behaviour | Result | Notes |
|-----------|--------|-------|
| `avro` extension origin | ✅ **Core** | `INSTALL avro;` — **not** `FROM community`. `installed_from = 'core'` |
| `avro` write support | ⚠️ **Partial** | `COPY t TO 'x.avro' (FORMAT avro)` works for `INTEGER`, `BIGINT`, `VARCHAR`, `DOUBLE`, `BOOLEAN`, `LIST`, `STRUCT` |
| `avro` write — unsupported types | ❌ Hard error | `DATE`, `TIMESTAMP`, `TIMESTAMPTZ`, `DECIMAL(p,s)`, `UUID` all raise `Not implemented Error: Can't convert logical type '<T>' to Avro type` |
| `iceberg` extension | ✅ Core, read | Reads standalone; **writes require attaching a REST catalog** (the catalog *is* the commit) |
| `delta` extension | ✅ Core | Read plus limited blind-insert only |
| `icu` extension | **Required** for named timezones | Without `INSTALL icu; LOAD icu;`, `SET TimeZone='Asia/Ho_Chi_Minh'` fails (Python client raises a `pytz` import error) |
| `hive_partitioning` | Auto-detected | Passing the flag is unnecessary; `y=2026/m=3/` prunes automatically |
| `filename` | Automatic since v1.3.0 | Added as a virtual column; `filename=true` is a no-op |
| `ROW_GROUP_SIZE` default | **122,880 rows** | Why a 1,000,000-row file has 9 row groups. Distinct from `FILE_SIZE_BYTES` (bytes) |
| Parquet codecs accepted | `uncompressed`, `snappy`, `gzip`, `zstd`, `brotli`, `lz4`, `lz4_raw` | Default `snappy`. Measured on 1M rows: 19.47 / 10.06 / 4.90 / **2.61** MB for none/snappy/gzip/zstd |
| `DECIMAL(p,s)` → Parquet physical type | **Precision-dependent** | `INT32` (p≤9), `INT64` (p≤18), `FIXED_LEN_BYTE_ARRAY` (p≥19). Not always `FIXED_LEN_BYTE_ARRAY` |
| `DECIMAL` through JSON | ❌ Precision lost | 1,000 × `9.99::DECIMAL(10,2)` sums to `9990.00` via Parquet but `9989.999999999829` (as `DOUBLE`) via JSON |
| Parquet positional multi-file read | ⚠️ Silently drops columns | `read_parquet([f1, f2])` where `f2` has an extra column returns only `f1`'s columns, **no error**. Use `union_by_name=true` |
| `union_by_name` + a renamed column | ⚠️ Produces orphan columns | A rename yields **both** old and new columns, each half-`NULL`. Only table-format column IDs prevent this |
| CSV reader on added column | ✅ Fails loudly | Raises `Error when sniffing file` rather than shifting values (better than most tools) |
| CSV reader on **reordered** same-count columns | ❌ Silently wrong | Values land in the wrong columns and parse cleanly |
| Over-partitioning | ❌ Can OOM the write | `PARTITION_BY (customer_id)` with 50,000 distinct values raised `OutOfMemoryException` (12.7 GiB) |
| Sorting effect on file size | ⚠️ **Often larger** | `ORDER BY order_date` grew the file 2.61 → 6.83 MB: sort column 25× smaller, but `category` 198× larger, `customer_id` 3.6× larger. Sort for **skipping**, not size |

#### Ingestion & Deduplication (execution-verified, DuckDB 1.4.5, during D4)

| Feature | Supported? | Notes |
|---------|-----------|-------|
| `read_csv('glob*.csv')` | ✅ Yes | Glob patterns work for CSV, JSON, Parquet |
| `filename = true` | ✅ Yes | Adds a `filename` column with the source path. **Note the tension with the File Formats row above** ("automatic since v1.3.0, flag is a no-op"): on 1.4.5 a read *without* the option returned no `filename` column under `SELECT *`, so the flag does affect the projection. Verify before relying on either claim |
| `filename = '_source_file'` | ✅ Yes | **Names the column directly** — cleaner than aliasing, which otherwise leaves a duplicate `filename` column under `SELECT *` |
| `all_varchar = true` | ✅ Yes | Reads every column as text — the correct Bronze-layer setting; prevents inference failing the load on one bad value |
| `union_by_name = true` | ✅ Yes | Tolerates drifted schemas across files |
| `SELECT * EXCLUDE (col)` | ✅ Yes | Useful when a helper column would otherwise duplicate |
| `QUALIFY` | ✅ Yes | Filters on window-function results; `WHERE` cannot (it runs before window evaluation) |
| `TRY_CAST` | ✅ Yes | Returns `NULL` instead of raising. Plain `CAST('N/A' AS DECIMAL)` raises `Conversion Error` — so a hard cast kills the model *before* any `not_null` test can flag the row |
| `to_json(table_alias)` | ✅ Yes | Serialises a whole row — handy for quarantine tables |
| `date_diff('second', a, b)` | ✅ Yes | |
| `DATE - interval 3 day` | ✅ Yes | Returns `TIMESTAMP` |
| `CHECK` constraint enforcement | ✅ Enforced | `Constraint Error: CHECK constraint failed on table ...` |
| `INSERT ... ON CONFLICT DO UPDATE` | ✅ Idempotent | Verified: row count stable across repeated identical upserts |

> **Deduplication gotcha (verified):** `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY _ingested_at DESC)` is **non-deterministic** for duplicates arriving in the same batch — `_ingested_at` is one value per statement and `_source_file` is identical, so the tiebreak is arbitrary and may keep the *older* record. Order by a timestamp from the **source data** first, then use ingestion metadata as a tiebreaker.

#### EXPLAIN Output

DuckDB renders `EXPLAIN` as a **visual tree** (not tabular). Filters are embedded inside scan nodes, not separate operators.

**Partition-pruning evidence** — `EXPLAIN ANALYZE` on a Hive-partitioned glob shows these lines inside the `PARQUET_SCAN` node. This is the observable proof of pruning; wall-clock is not (a pruned query can be *slower* on small data):

```text
File Filters: (m = 3)
Scanning Files: 1/12
Total Files Read: 1
```

| Operator | Present? | Notes |
|----------|---------|-------|
| `SEQ_SCAN` | ✅ Yes | Includes inline `Filters:` for predicates |
| `HASH_GROUP_BY` | ✅ Yes | Grouping operator |
| `HASH_JOIN` | ✅ Yes | Join operator |
| `PROJECTION` | ✅ Yes | Column selection step |
| `INDEX_SCAN` | ❌ Not confirmed | Does not appear even with `PRIMARY KEY` + 10k rows |
| `FILTER` (standalone) | ❌ No | Embedded in `SEQ_SCAN`, not a separate node |

---

### FastAPI / Pydantic

**Docs:** [fastapi.tiangolo.com](https://fastapi.tiangolo.com) · [docs.pydantic.dev](https://docs.pydantic.dev) · **Changelog:** [github.com/tiangolo/fastapi/releases](https://github.com/tiangolo/fastapi/releases)

| Decision | Outcome | Notes |
|----------|---------|-------|
| Where to introduce `pydantic-settings` | **B3** (not B2) | First concrete need is DB URL in B3. Introducing in B2 without a real config value is confusing for interns. |

---

### dbt Core

**Docs:** [docs.getdbt.com](https://docs.getdbt.com) · **Changelog:** [github.com/dbt-labs/dbt-core/releases](https://github.com/dbt-labs/dbt-core/releases)

Verified during D4 authoring by **running a real `dbt-duckdb` project** (dbt-core 1.10.23, dbt-duckdb 1.10.0, DuckDB 1.4.5) unless marked *(docs)*.

#### Test & YAML syntax

| Behaviour | Result | Notes |
|-----------|--------|-------|
| `data_tests:` vs `tests:` | Both work | `tests:` is a deprecated alias. **Cannot use both on one resource.** |
| Test arguments nesting | `arguments:` required | Top-level args emit `[WARNING][MissingArgumentsPropertyInGenericTestDeprecation]`. Nested form parses clean. `arguments:` needs **v1.10.5+** *(docs)* |
| Test `config:` placement | Same level as `arguments:` | `severity`, `error_if`, `warn_if`, `store_failures` all live under `config:` |
| `store_failures` output | Schema `<schema>_dbt_test__audit` | One table per test, e.g. `main_dbt_test__audit.unique_stg_orders_order_id` |
| Source `freshness` / `loaded_at_field` | Must nest under `config:` | Flat form deprecated. Both keys moved into `config:` in recent releases |
| `identifier:` on a source table | Works | Maps logical `source('raw','orders')` → real table `bronze_orders` |

#### Snapshots

| Behaviour | Result | Notes |
|-----------|--------|-------|
| Snapshotting an **all-VARCHAR** source | ❌ **Fails** | `Binder Error: Cannot mix values of type VARCHAR and TIMESTAMP WITH TIME ZONE in COALESCE operator`. The `timestamp` strategy needs a real timestamp in `updated_at` |
| Fix | Snapshot a **typed model** | `relation: ref('stg_customers')`, not `source('raw','customers')` |
| `dbt_valid_to_current` cast | Must match the column type | `cast('9999-12-31' as timestamptz)`. `as date` reproduces the COALESCE error. dbt's docs example uses `to_date()` — written for another warehouse |
| Meta columns added | `dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id`, `dbt_updated_at` | `dbt_is_deleted` appears **only** with `hard_deletes: new_record` *(docs)* |

#### Contracts

| Behaviour | Result | Notes |
|-----------|--------|-------|
| Partial column list under `enforced: true` | ❌ **Fails** | Reports `missing in contract` per undeclared column. Every column needs `name` + `data_type` |
| Type mismatch | ❌ **Compilation Error** | Fails *before* the table is created. Output is a `\| column_name \| definition_type \| contract_type \| mismatch_reason \|` table |
| `incremental` + `on_schema_change: ignore` | ❌ **Parse-time error** | `Invalid value for on_schema_change: ignore. Models materialized as incremental with contracts enabled must set on_schema_change to 'append_new_columns' or 'fail'` |
| `view` materialization | Columns/types only | **`constraints` not supported** *(docs)* |
| dbt-duckdb constraint enforcement | Enforced at DB level | Unusual — many adapters treat `constraints` as metadata only |

#### Commands & build behaviour

| Behaviour | Result | Notes |
|-----------|--------|-------|
| `dbt build` on test failure | Skips downstream | `SKIP relation main.dim_customers ... [SKIP]`. Acts as a circuit breaker |
| Run-end summary line | `Done. PASS=n WARN=n ERROR=n SKIP=n NO-OP=n TOTAL=n` | `NO-OP` is included; always the last line |
| Failure detail block | `Completed with 1 error, 0 partial successes, and 0 warnings:` then `Got n results, configured to fail if != 0` plus a `compiled code at target/compiled/...` path | |
| `packages.yml` present but `dbt deps` not run | ❌ Hard error | `dbt found 1 package(s) specified in packages.yml, but only 0 package(s) installed` |
| Incremental `delete+insert` + `unique_key` | Idempotent | Verified stable row counts across 3 consecutive runs |
| Incremental strategies (dbt-duckdb) | `append`, `delete+insert`, `merge`, `microbatch` | `merge` needs **DuckDB 1.4.0+**; `external` materialization does **not** support incremental strategies *(docs)* |

#### Version currency *(docs)*

dbt Core **1.12** is current (Jul 2026); **1.10 and 1.9 are deprecated**. `pip` resolves the newest dbt the local **Python** version allows — Python 3.9 pins you to a deprecated 1.10.x. Install on Python 3.10+ for a supported release.

---

### Apache Spark / Databricks

**Docs:** [spark.apache.org/docs/latest](https://spark.apache.org/docs/latest/) · [docs.databricks.com](https://docs.databricks.com) · **Changelog:** [spark.apache.org/news](https://spark.apache.org/news/)

#### Databricks Free Edition constraints

Community Edition is **retired**; Free Edition replaced it. Verified against [Free Edition](https://learn.microsoft.com/en-us/azure/databricks/getting-started/free-edition) and [Free Edition limitations](https://learn.microsoft.com/en-us/azure/databricks/getting-started/free-edition-limitations) on 2026-08-12. **Check these before writing D4's Spark section or D6.**

| Constraint | Detail | Impact on this vault |
|-----------|--------|----------------------|
| **Serverless compute only** | "Custom compute configurations are not supported" | ❌ **No cluster-creation steps.** Any "spin up a cluster / choose a runtime / set `spark.executor.memory`" instruction is dead — do not write one |
| PySpark + Spark SQL | ✅ Supported | D4's Spark section is viable as planned |
| **R and Scala** | ❌ Not supported | Keep all Spark examples in **Python or SQL** |
| Unity Catalog | On by default; one metastore per account | Interns meet **Delta-in-Unity-Catalog**, not raw Delta on DBFS — matters for D3 §3.3 framing and D4 §4.4 |
| SQL warehouses | One, capped at `2X-Small` | Fine for teaching; don't promise concurrency |
| Jobs | Max **5 concurrent job tasks** per account | Keep D4/D6 orchestration examples small |
| Lakeflow pipelines | One active pipeline per type | Don't design multi-pipeline exercises |
| **Outbound internet** | Restricted to trusted domains unless LinkedIn-verified | ⚠️ Interns may **not** be able to call arbitrary REST APIs from a Databricks notebook. Do ingestion exercises locally (DuckDB/Python) as D1 §1.3 already does |
| Quota exceeded | Compute shut down for the rest of the day | Keep exercises short; warn interns not to leave jobs running |
| Account scope | One workspace, no account console/APIs, non-commercial use only | No admin/governance hands-on — keep D6 governance conceptual |

#### Serverless Spark API restrictions

Free Edition runs only serverless compute, which removes Spark APIs a normal cluster provides. Verified against [serverless limitations](https://docs.databricks.com/aws/en/compute/serverless/limitations) and [supported Spark properties](https://docs.databricks.com/aws/en/spark/conf).

| Feature | Available? | Notes |
|---------|:----------:|-------|
| DataFrame/SQL **cache APIs** | ❌ No | *"Dataframe and SQL cache APIs are not supported on serverless compute."* `cache()`, `persist()`, `CACHE TABLE` all raise |
| **Spark UI** | ❌ No | *"use the query profile to view information about your Spark queries"* |
| **RDD APIs** | ❌ No | *"Only Spark Connect APIs are supported"* — `sparkContext` unavailable |
| **DBFS root** | ❌ Disabled | Path writes like `/tmp/...` fail. Use UC managed tables (`saveAsTable`) or UC volumes (`/Volumes/<cat>/<schema>/<vol>/`) |
| `dbfs:/databricks-datasets/` | ✅ Readable | Reserved read-only namespace; survives DBFS disablement |
| Settable Spark properties | **Only 6** | `spark.databricks.execution.timeout`, `spark.sql.legacy.timeParserPolicy`, `spark.sql.session.timeZone`, `spark.sql.shuffle.partitions` (**default `auto`**, not 200), `spark.sql.ansi.enabled` (**default `true`**), `spark.sql.files.maxPartitionBytes` (128 MB) |
| `spark.sql.autoBroadcastJoinThreshold` | ❌ Not settable | The `broadcast()` hint still works; AQE converts joins at runtime anyway |

> ⚠️ The engine sections below are **documentation-verified, not execution-verified** — no Java or Databricks access existed while writing D4, so no PySpark was run. Upgrade an entry once someone runs it in a notebook.

#### Structured Streaming on serverless *(documentation-verified, during D5)*

Confirmed against [serverless limitations](https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/limitations), [streaming on serverless](https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/streaming) and [Auto Loader options](https://learn.microsoft.com/en-us/azure/databricks/ingestion/cloud-object-storage/auto-loader/options) on 2026-08-19. **No notebook was run** — upgrade these once someone executes them.

| Behaviour | Result | Notes |
|-----------|--------|-------|
| Structured Streaming on serverless | ✅ **Supported** | So D5's hands-on section is viable on Free Edition |
| Supported triggers | **Only two** | `Trigger.AvailableNow()` (**recommended**, Spark 3.3+) and `Trigger.Once()` (supported but **deprecated** since Spark 3.4 / DBR 11.3 LTS) |
| `Trigger.ProcessingTime(interval)` / `Trigger.Continuous(interval)` | ❌ Not supported | In serverless **notebooks and jobs** |
| No explicit trigger | ❌ **Hard error** | Spark defaults to `Trigger.ProcessingTime("0 seconds")` → raises `INFINITE_STREAMING_TRIGGER_NOT_SUPPORTED`. **Always set `availableNow=True`** |
| Lakeflow pipelines | ✅ **Exempt** | Serverless pipelines support triggered, continuous, and real-time modes — the trigger restriction does not apply. Free Edition still caps at one active pipeline per type |
| `cloudFiles.schemaLocation` | **Required for inference** | It is what *enables* schema inference and evolution — omit it and Auto Loader does not infer a schema at all |
| Auto Loader type inference on JSON/CSV/XML | ⚠️ **Everything is `String`** | These formats encode no types, so `withWatermark`/`window` on a timestamp column fails until `cloudFiles.schemaHints` pins it (or `inferColumnTypes` is set) |
| `cloudFiles.schemaEvolutionMode` default | **Conditional** | `addNewColumns` when no schema is supplied; `none` when you supply one. `addNewColumns` **fails the stream** with `UnknownFieldException`, records the new schema, and succeeds on restart |
| `_rescued_data` column | Added automatically | Whenever Auto Loader infers a schema. Captures values that did not fit — so a type mismatch is never silently dropped |
| Both trigger limits together | **Differs by source** | Auto Loader: `cloudFiles.maxFilesPerTrigger` (default **1000**) and `cloudFiles.maxBytesPerTrigger` **can both be set** — it consumes to whichever is hit first. Spark's plain file source: they **cannot** both be set |
| `maxOffsetsPerTrigger` | **Kafka source only** | Setting it on a file source is a silent no-op |
| Streaming `checkpointLocation` on a UC volume | ✅ Fine | Distinct from `DataFrame.checkpoint()`, which **is** banned on serverless. Do not conflate the two |
| Writing files to a UC volume | ⚠️ Single sequential write only | Direct-append and random writes are unsupported — `open(path, "a")` fails; one `open(path, "w")` + write is fine |
| Delta streaming sink output modes | `append` + `complete` only | **No `update` mode.** Upsert-style streaming writes go through `foreachBatch` + `MERGE` |

> ⚠️ **`KAFKA` is listed as a supported serverless data source** (read and write). This does **not** license a Kafka hands-on exercise — this vault has no broker and `AGENTS.md § Confirmed Tech Stack` keeps Kafka **conceptual only**. Free Edition also restricts outbound internet to trusted domains.

#### Azure Event Hubs *(documentation-verified, during D5)*

| Behaviour | Result | Notes |
|-----------|--------|-------|
| **Log compaction** | ✅ **Supported (GA)** | Corrects a plausible wrong assumption: it *is* an Event Hubs feature. Enabled per event hub via its cleanup policy (**not** Kafka's `cleanup.policy`); tombstones work the same way — a `null` payload on an existing key; the partition key is the compaction key. **Not on Basic tier.** Microsoft names CDC as a primary use case |
| Kafka endpoint | Standard / Premium / Dedicated | **Basic tier has none** |
| Kafka transactions · Kafka Streams | Public **preview** | Premium / Dedicated only |
| Compression | Premium / Dedicated only | `gzip` |
| Concept mapping | Cluster→Namespace, Topic→event hub, Partition→Partition, Consumer group→Consumer group, Offset→Offset | |

> The general lesson recorded for future agents: on "Kafka-compatible" services, gaps are usually **tier or preview restrictions rather than missing concepts**, and the *configuration surface* often differs even where behaviour matches. Verify per service **and tier**; do not infer absence from one overview page.

#### Spark engine defaults (OSS)

| Config | Default | Notes |
|--------|---------|-------|
| `spark.sql.shuffle.partitions` | `200` | `auto` on Databricks serverless |
| `spark.sql.autoBroadcastJoinThreshold` | `10485760` (10 MB) | |
| `spark.sql.adaptive.enabled` | `true` | **AQE default-on since Spark 3.2**: coalesce shuffle partitions, sort-merge→broadcast conversion, skew-join splitting |
| `spark.sql.adaptive.skewJoin` thresholds | factor `5.0`, `256 MB` | Advisory partition size 64 MB |
| `spark.sql.ansi.enabled` | `true` in Spark 4.0 / DBR 17.0+ | Blocks implicit STRING→numeric coercion — an aggregation over a wrongly-inferred string column **errors** instead of returning null |
| RDD `cache()` | `MEMORY_ONLY` | Overflow partitions **recomputed** |
| DataFrame/Dataset `cache()` | `MEMORY_AND_DISK` | Overflow partitions **spill to disk**. Same method, different default |

#### CSV reader semantics

| Option | Default | Notes |
|--------|---------|-------|
| `mode` | `PERMISSIVE` | Puts malformed text in `columnNameOfCorruptRecord` and **sets malformed fields to null**. `FAILFAST` throws; `DROPMALFORMED` silently discards |
| `enforceSchema` | `true` | *"the specified or inferred schema will be forcibly applied ... and headers in CSV files will be ignored"* — an explicit schema matches **by position**, not by header name |
| `inferSchema` | `false` | Costs one extra pass over the data |
| `nullable=False` in an explicit schema | **Not enforced** on file reads | SPARK-19950; resolved as *Incomplete*. Enforce required-ness with a downstream test |
| `badRecordsPath` (Databricks only) | — | **Takes precedence over `_corrupt_record`** — rows written there do **not** appear in the resulting DataFrame |

#### Delta Lake

| Behaviour | Notes |
|-----------|-------|
| Time travel on a **named table** | `spark.read.option("versionAsOf", 123).table("cat.sch.tbl")` is documented and valid. Also `table("people10m@v123")` and SQL `VERSION AS OF` / `TIMESTAMP AS OF`. `.format("delta")` before `.table()` is a no-op |
| `VACUUM` retention | 7 days default — bounds how far time travel reaches |
| Partitioning thresholds | Under **1 TB → don't partition**; *"most tables ... with less than 100 TB of data don't need partitioning"*; if partitioning, each partition **≥ 1 GB** |
| Liquid clustering | *"Databricks recommends liquid clustering for all new tables, including streaming tables and materialized views"* |
| `CLUSTER BY` syntax | Cannot stand alone — needs a column definition list, `AS query`, or `LOCATION` |
| Changing clustering keys | Allowed without rewriting; existing data is **not** reorganised until `OPTIMIZE FULL` |

#### DuckDB ↔ table formats

Writing **Iceberg** from DuckDB is supported but requires an attached Iceberg **REST catalog** — too much setup for intern exercises. `dbt-duckdb` **cannot** write Delta or Iceberg at all; that needs `dbt-databricks` / `dbt-spark`.

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

### 🖥️ Backend Track

| Domain | File | Key Topics | Content | Checkpoint |
|--------|------|-----------|---------|-----------|
| B1: Foundations & Dev Setup | `Backend/B1 - Foundations & Dev Setup.md` | Backend intro · Python OOP & exceptions · Package structure · Git & Conventional Commits · Linux CLI · venv/uv · pyproject.toml · `.env` basics · Ruff · pre-commit · Type hints · IDE setup | ✅ Complete | `Backend/Checkpoints/CB1 - Dev Environment Ready.md` |
| B2: Web & API Design | `Backend/B2 - Web & API Fundamentals.md` | async/await · HTTP/REST · FastAPI basics · APIRouter · CORS · Pydantic v2 · field_validator · Error handling · Depends() · yield pattern · gRPC · Route guards · OpenAPI | ✅ Complete | `Backend/Checkpoints/CB2 - API Built & Documented.md` |
| B3: Databases & ORM | `Backend/B3 - Databases & ORM.md` | PostgreSQL · SQLAlchemy async · Alembic · Pydantic Settings · Redis · Repository Pattern · Unit of Work | 🔴 Not started | `Backend/Checkpoints/CB3 - DB & ORM Proficiency.md` |
| B4: Authentication & Security | `Backend/B4 - Authentication & Security.md` | JWT · OAuth 2.0 · Password hashing · HTTPS · Security headers | 🔴 Not started | `Backend/Checkpoints/CB4 - Auth & Security Verified.md` |
| B5: Testing & Code Quality | `Backend/B5 - Testing & Code Quality.md` | pytest · unit/integration/e2e · fixtures · mocking · coverage | 🔴 Not started | `Backend/Checkpoints/CB5 - Test Suite Complete.md` |
| B6: Async, Queues & Background Jobs | `Backend/B6 - Async, Queues & Background Jobs.md` | asyncio deep dive · Celery/ARQ · RabbitMQ/Redis queues · Azure Service Bus | 🔴 Not started | `Backend/Checkpoints/CB6 - Queue & Workers Running.md` |
| B7: Microservices & Containers | `Backend/B7 - Microservices & Containers.md` | Docker · Docker Compose · Microservice patterns · Clean Architecture · IoC containers | 🔴 Not started | `Backend/Checkpoints/CB7 - Service Containerised.md` |
| B8: Capstone Project | `Backend/B8 - Capstone Project.md` | Full backend service integrating B1–B7 | 🔴 Not started | `Backend/Checkpoints/CB8 - Capstone Complete.md` |

### 🗄️ Data Engineering Track

| Domain | File | Key Topics | Content | Checkpoint |
|--------|------|-----------|---------|-----------|
| D1: Foundations & Tooling | `DataEngineering/D1 - Foundations & Tooling.md` | Mindset shift · Python for DE · REST APIs · Git · Linux · DuckDB setup | ✅ Complete | `DataEngineering/Checkpoints/CP1 - Tooling & Environment Ready.md` |
| D2: SQL & Data Modeling | `DataEngineering/D2 - SQL & Data Modeling.md` | Window functions/CTEs · SQL for DE · Query perf · Normalization · Dimensional modeling | ✅ Complete | `DataEngineering/Checkpoints/CP2 - SQL Proficiency.md` |
| D3: Data Storage & Formats | `DataEngineering/D3 - Data Storage & Formats.md` | OLTP/OLAP · Relational vs NoSQL · DWH/Lake/Lakehouse · Object storage · Iceberg/Delta Lake · Catalogs · Formats (CSV/JSON/Parquet/Avro/ORC/Arrow) · Schema evolution · Type mapping & precision loss · Medallion arch · Partitioning & small files · DuckDB local analytics · Vector DBs *(optional)* | ✅ Complete | `DataEngineering/Checkpoints/CP3 - Storage & Modeling.md` |
| D4: Batch Processing & ETL | `DataEngineering/D4 - Batch Processing & ETL.md` | ETL vs ELT · Ingestion (DuckDB file reads, REST) · dbt · Medallion · Delta/Iceberg · Pipeline patterns · Data quality & contracts · Spark · Error handling | ✅ Complete | `DataEngineering/Checkpoints/CP4 - Batch Pipeline.md` |
| D5: Stream Processing | `DataEngineering/D5 - Stream Processing.md` | Batch vs stream (latency-budget test) · Brokers & Kafka *(conceptual)* — partitions, consumer groups, retention/compaction, KRaft · Event Hubs · Delivery guarantees & effectively-once · Event time, windows & watermarks · Stateful processing, state stores & stream joins · Lambda/Kappa & CDC · Structured Streaming hands-on (Auto Loader → Delta) · Streaming ops | ✅ Complete | `DataEngineering/Checkpoints/CP5 - Stream Pipeline.md` |
| D6: Cloud & Orchestration | `DataEngineering/D6 - Cloud & Orchestration.md` | Cloud fundamentals · Docker · ADF orchestration · Governance & cost | 🔴 Not started | `DataEngineering/Checkpoints/CP6 - Cloud Deployment.md` |
| D7: AI-Ready DE *(optional)* | `DataEngineering/D7 - AI-Ready Data Engineering.md` | AI pipelines · Embeddings · Vector DBs · LLM data flows | 🔴 Not started | `DataEngineering/Checkpoints/CP7 - AI Data Engineering.md` |
