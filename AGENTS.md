# AGENTS.md — Data Engineering 101

> This file instructs AI agents (Copilot, Claude Code, etc.) on how to work inside this Obsidian vault.
> It supplements `.github/copilot-instructions.md` with project-specific conventions for the Data Engineering 101 onboarding vault.

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
  - DE101
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
  - DE101
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
| **Distributed Processing** | **Databricks Community Edition** (Azure) | Free, browser-based, no cluster setup. PySpark + SparkSQL included. Use for D4 Spark section |
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

### FastAPI / Pydantic

**Docs:** [fastapi.tiangolo.com](https://fastapi.tiangolo.com) · [docs.pydantic.dev](https://docs.pydantic.dev) · **Changelog:** [github.com/tiangolo/fastapi/releases](https://github.com/tiangolo/fastapi/releases)

| Decision | Outcome | Notes |
|----------|---------|-------|
| Where to introduce `pydantic-settings` | **B3** (not B2) | First concrete need is DB URL in B3. Introducing in B2 without a real config value is confusing for interns. |

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
| D3: Data Storage & Formats | `DataEngineering/D3 - Data Storage & Formats.md` | OLTP/OLAP · DWH/Lake/Lakehouse · Formats · Medallion arch · Iceberg/Delta Lake | 🔴 Not started | `DataEngineering/Checkpoints/CP3 - Storage & Modeling.md` |
| D4: Batch Processing & ETL | `DataEngineering/D4 - Batch Processing & ETL.md` | ETL vs ELT · dbt · Pipeline patterns · Data quality · Spark | 🔴 Not started | `DataEngineering/Checkpoints/CP4 - Batch Pipeline.md` |
| D5: Stream Processing | `DataEngineering/D5 - Stream Processing.md` | Batch vs stream · Kafka (conceptual) · Lambda/Kappa arch · Delivery guarantees | 🔴 Not started | `DataEngineering/Checkpoints/CP5 - Stream Pipeline.md` |
| D6: Cloud & Orchestration | `DataEngineering/D6 - Cloud & Orchestration.md` | Cloud fundamentals · Docker · ADF orchestration · Governance & cost | 🔴 Not started | `DataEngineering/Checkpoints/CP6 - Cloud Deployment.md` |
| D7: AI-Ready DE *(optional)* | `DataEngineering/D7 - AI-Ready Data Engineering.md` | AI pipelines · Embeddings · Vector DBs · LLM data flows | 🔴 Not started | `DataEngineering/Checkpoints/CP7 - AI Data Engineering.md` |
