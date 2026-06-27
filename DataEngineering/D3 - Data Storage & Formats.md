---
tags:
  - DE101
  - domain-3
  - storage
  - data-formats
date: 2026-06-20
status: not-started
domain: "3 of 7"
track: data-engineering
---

# D3 — Data Storage & Formats

**Back to:** [[00 - Onboarding Roadmap]]

> [!NOTE] Domain Overview
> This domain covers **where data lives and how it's structured on disk**. You'll learn the fundamental OLTP vs OLAP split, common storage paradigms, and the file formats every DE works with daily.

---

## 3.1 — OLTP vs OLAP

> [!IMPORTANT] Core Distinction
> This is the foundational split in data storage: **OLTP** (transactional, row-oriented, fast writes) vs **OLAP** (analytical, column-oriented, fast reads on large datasets). Every storage decision flows from this.

*Content coming soon. Relational databases as OLTP, data warehouses as OLAP, when each fits.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 3.2 — Relational vs NoSQL Databases

*Content coming soon.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 3.3 — Data Warehouse vs Data Lake vs Lakehouse

*Content coming soon. Trade-offs, use cases, and why Lakehouse is the modern default.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

> [!NOTE] Platform Landscape (2026)
> Cloud DWH platforms you'll encounter: **Snowflake** (most common), **Google BigQuery**, **Databricks**, **Azure Synapse / Microsoft Fabric**. Know what they are — deep expertise comes later on the job.

---

## 3.4 — File Formats

*Content coming soon. CSV, JSON, Parquet, Avro — when to use each and why Parquet wins for analytics.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 3.5 — Medallion Architecture

*Content coming soon. Bronze / Silver / Gold layers — the standard pattern for organizing data in a Lakehouse. You'll implement this in [[D4 - Batch Processing & ETL]].*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 3.6 — Partitioning, Indexing & Query Optimization

*Content coming soon.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 3.7 — DuckDB for Local Analytics

> [!TIP] Modern DE Essential
> DuckDB is an embedded, serverless analytical database that runs SQL directly on Parquet, CSV, and JSON files. No server to set up — just `pip install duckdb`. By 2026 it's a standard tool in every DE toolkit.

*Content coming soon. Installing DuckDB, querying files locally, DuckDB vs pandas for data exploration.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## 3.8 — Vector Databases *(Optional — AI workloads)*

> [!TIP] Optional Section
> Only relevant if you'll be supporting AI/LLM workloads. See also [[D7 - AI-Ready Data Engineering]].

*Content coming soon. Embedding storage, similarity search, Pinecone/pgvector overview.*

> [!WARNING] Common Anti-Patterns
> *To be defined when writing content.*

---

## ✅ Practice Checklist

- [ ] *Tasks to be defined*

---

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://www.databricks.com/glossary/medallion-architecture | Medallion Architecture overview |
| https://duckdb.org/docs/ | DuckDB documentation |
| https://parquet.apache.org/docs/ | Apache Parquet format spec |
| https://github.com/andkret/Cookbook/blob/master/sections/03-AdvancedSkills.md#store | Storage section from the Cookbook |

---

## 🃏 Quick-Reference Flash Cards

**Q:** *Questions to be defined*
A: *Answers to be defined*

---

*Checkpoint: [[Checkpoints/CP3 - Storage & Modeling|CP3 - Storage & Modeling]]*

---

*Previous: [[D2 - SQL & Data Modeling]] | Next: [[D4 - Batch Processing & ETL]]*
