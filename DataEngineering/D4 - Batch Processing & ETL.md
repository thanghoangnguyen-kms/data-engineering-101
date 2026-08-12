---
tags:
  - DE101
  - domain-4
  - batch
  - etl
  - dbt
date: 2026-06-20
status: not-started
domain: "4 of 7"
track: data-engineering
---

# D4 — Batch Processing & ETL

**Back to:** [[00 - Onboarding Roadmap]]

> [!NOTE] Domain Overview
> This domain covers how data moves and transforms. The modern standard has shifted to **ELT** (load first, transform with dbt) over classic ETL. `dbt` is now table-stakes in every DE role.

---

## 4.1 — ETL vs ELT

*Content coming soon. Concepts, trade-offs, why ELT won in the cloud era.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 4.2 — dbt (Data Build Tool)

> [!IMPORTANT] Priority Subdomain
> `dbt` is now the standard SQL transformation layer. If you only learn one tool in this domain, it's this one.

*Content coming soon. Models, tests, sources, lineage, dbt core vs cloud.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 4.3 — Medallion Architecture in Practice

*Content coming soon. Implementing Bronze → Silver → Gold with real pipelines. See also [[D3 - Data Storage & Formats#3.5 — Medallion Architecture|D3 §3.5]] for the concept.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 4.4 — Table Formats in Production (Apache Iceberg & Delta Lake)

> [!IMPORTANT] Where Table Formats Live
> You learned *what* Iceberg and Delta Lake are in D3. Here you'll learn *how to write to them* from a pipeline — ACID transactions, schema evolution, time travel in practice.

*Content coming soon. Writing dbt models to Iceberg, partitioning strategies, incremental models.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 4.5 — Batch Pipeline Design Patterns

*Content coming soon.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 4.6 — Data Quality, Contracts & Observability

*Content coming soon. dbt tests, Great Expectations, data contracts, Monte Carlo/Soda patterns.*

> [!IMPORTANT] Cross-System Type Mismatches
> One of the most common data quality failures in pipelines is type drift between systems — e.g., a JSON source sends a number as a string, Spark infers `StringType`, and your downstream SQL `SUM()` silently returns `NULL`. Always validate types at ingestion. See [[D2 - SQL & Data Modeling#2.6 — Data Types & Type Safety|D2 §2.6]] for the SQL-level foundation.

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 4.7 — Apache Spark & Distributed Processing

> [!IMPORTANT] Must-Know for Production DE
> Spark is the industry-standard distributed processing engine. Understanding how it works *under the hood* separates engineers who just run Spark jobs from engineers who can debug, optimize, and design them.

> [!WARNING] Platform Constraint — Read Before Writing This Section
> The hands-on platform is **Databricks Free Edition** (Community Edition is retired). Free Edition is **serverless-only**: custom compute configurations are not supported, so there is **no cluster to create or size**.
>
> Consequences for this section: teach Driver/Executor/Cluster Manager as **concepts**, not as a setup walkthrough — write no "spin up a cluster", "choose a runtime", or `spark.executor.memory` tuning steps. Keep all examples in **Python or SQL** (R and Scala are unsupported). Outbound internet is restricted, so don't call external APIs from a notebook — do ingestion locally as [[D1 - Foundations & Tooling]] §1.3 does. Full constraint table in `AGENTS.md § Apache Spark / Databricks`.

*Content coming soon. This section covers:*

**Architecture**
- Driver vs Executor vs Cluster Manager
- SparkContext and SparkSession
- How a job flows: Job → Stage → Task

**Execution Model**
- Lazy evaluation — transformations vs actions
- DAG (Directed Acyclic Graph) — how Spark builds an execution plan
- Stage boundaries and shuffles (why shuffles are expensive)

**Core Abstractions**
- RDD (Resilient Distributed Dataset) — the low-level API
- DataFrame vs Dataset — the high-level APIs
- Catalyst Optimizer — how Spark rewrites your query for performance

**Must-Know Techniques**
- Partitioning strategies (`repartition` vs `coalesce`)
- Caching and persistence (`cache()`, `persist()`, storage levels)
- Broadcast joins — how to avoid shuffle joins on small tables
- Handling skewed data
- Reading the Spark UI (stages, tasks, spill, shuffle read/write)

**Common PySpark patterns**
- Reading/writing Parquet, Delta, Iceberg
- Window functions in Spark
- User-Defined Functions (UDFs) — and why to avoid them

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content. Known areas: UDF overuse, collecting large DataFrames to the driver, unnecessary shuffles, wrong partitioning count.*

---

## 4.8 — Error Handling & Monitoring

*Content coming soon. Retries, dead-letter patterns, alerting, pipeline observability.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## ✅ Practice Checklist

- [ ] *Tasks to be defined*

---

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://docs.getdbt.com | dbt official documentation — start here |
| https://docs.greatexpectations.io | Great Expectations — data quality framework |
| https://www.datacontract.com | Data Contract specification standard |
| https://docs.getdbt.com/docs/build/tests | dbt tests for data quality |
| https://github.com/andkret/Cookbook/blob/master/sections/03-AdvancedSkills.md#is-etl-still-relevant-for-analytics | ETL vs ELT discussion |

---

## 🃏 Quick-Reference Flash Cards

**Q:** *Questions to be defined*
A: *Answers to be defined*

---

*Checkpoint: [[Checkpoints/CP4 - Batch Pipeline|CP4 - Batch Pipeline]]*

---

*Previous: [[D3 - Data Storage & Formats]] | Next: [[D5 - Stream Processing]]*
