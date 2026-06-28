# Engineering 101

An Obsidian vault for onboarding intern engineers through two structured learning tracks — **Backend Engineering** (B1–B8) and **Data Engineering** (D1–D7) — from foundations to production-ready skills.

## 🛣️ Tracks

### 🖥️ Backend Engineering Track

| # | Domain | Status |
|---|--------|--------|
| B1 | Foundations & Dev Setup | ✅ Complete |
| B2 | Web & API Fundamentals | ✅ Complete |
| B3 | Databases & ORM | ✅ Complete |
| B4 | Authentication & Security | ✅ Complete |
| B5 | Testing & Code Quality | ✅ Complete |
| B6 | Async, Queues & Background Jobs | ✅ Complete |
| B7 | Microservices & Containers | ✅ Complete |
| B8 | Capstone Project | ✅ Complete |

**Estimated duration:** ~3 weeks at 24–30 hrs/week
- Week 1: B1 → B2 → B3
- Week 2: B4 → B5 → B6
- Week 3: B7 → B8 (capstone)

**Tech stack:** Python · FastAPI · PostgreSQL · SQLAlchemy (async) · Alembic · Redis · Azure Service Bus · Docker · uv · Ruff · mypy · pytest

### 🗄️ Data Engineering Track

| # | Domain | Status |
|---|--------|--------|
| D1 | Foundations & Tooling | ✅ Complete |
| D2 | SQL & Data Modeling | ✅ Complete |
| D3 | Data Storage & Formats | 🔴 Not started |
| D4 | Batch Processing & ETL | 🔴 Not started |
| D5 | Stream Processing | 🔴 Not started |
| D6 | Cloud & Orchestration | 🔴 Not started |
| D7 | AI-Ready Data Engineering *(optional)* | 🔵 Optional |

**Estimated duration:** ~3 months at 5–10 hrs/week

**Tech stack:** Python · DuckDB · dbt Core · Databricks Community Edition (Azure) · Azure Blob / ADLS Gen2 · Azure Data Factory · Docker · Git · Kafka *(conceptual only)*

## 📖 How to Use

This vault is designed for [Obsidian](https://obsidian.md). Open the folder as a vault to get full support for wikilinks, callouts, and frontmatter.

1. Start at `00 - Onboarding Roadmap.md`
2. Read `00.1 - How to Use This Vault.md` for navigation and pacing guidance
3. Work through domains in order — complete each checkpoint before moving to the next

## 📁 Structure

```
├── 00 - Onboarding Roadmap.md        ← Start here
├── 00.1 - How to Use This Vault.md   ← Navigation & pacing guide
├── Backend/
│   ├── 00 - Backend Track Roadmap.md
│   ├── B1–B8 domain notes
│   └── Checkpoints/                  ← CB1–CB8 milestone checklists
├── DataEngineering/
│   ├── 00 - DE Track Roadmap.md
│   ├── D1–D7 domain notes
│   └── Checkpoints/                  ← CP1–CP7 milestone checklists
├── docs/                             ← Supplementary reference guides
│   ├── clean-architecture.md
│   ├── event-driven.md
│   ├── caching.md
│   └── command-queue.md
└── AGENTS.md                         ← AI agent conventions & verified behaviors
```
