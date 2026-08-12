---
tags:
  - DE101
  - checkpoint
date: 2026-06-20
status: not-started
domain: "3 of 7"
verified_by: ""
track: data-engineering
---

# CP3 — Storage & Modeling

**Domain:** [[D3 - Data Storage & Formats]]
**Back to:** [[00 - Onboarding Roadmap]]

> [!CHECKPOINT] Pass Criteria
> Complete all items below. Have your mentor verify and sign off before moving to D4.

---

## ✅ Pass Criteria

- [ ] Explain the difference between OLTP and OLAP with a real-world example of each
- [ ] Describe when you'd choose a Data Warehouse vs a Data Lake vs a Lakehouse
- [ ] Name 3 columnar file formats and explain why Parquet is preferred for analytics
- [ ] Explain the Bronze / Silver / Gold medallion layers and what transformations happen at each
- [ ] Use DuckDB to query a local Parquet file and return aggregated results
- [ ] Describe one trade-off between Iceberg and Delta Lake (schema evolution, time travel, or vendor support)
- [ ] Write the same table both partitioned by `(year, month)` and as a single file, then use `EXPLAIN ANALYZE` to show the `Scanning Files: n/N` pruning evidence — and state whether partitioning actually made the query faster
- [ ] Demonstrate the small file problem: split a table into 1,000+ small Parquet files, time an aggregate over them, compact to one file, time it again, and report both the total-bytes and query-time ratio
- [ ] Sum 1,000 rows of `9.99::DECIMAL(10,2)` after round-tripping through Parquet and through JSON, report both sums with `typeof()`, and explain why one lost precision
- [ ] Explain schema-on-write vs schema-on-read, naming one NoSQL family that uses each

---

**Verified by:** _________________ | **Date:** _________________
