---
tags:
  - DE101
  - checkpoint
  - optional
date: 2026-06-20
status: not-started
domain: "7 of 7"
verified_by: ""
track: data-engineering
---

# CP7 — AI Data Engineering *(Optional)*

**Domain:** [[D7 - AI-Ready Data Engineering]]
**Back to:** [[00 - Onboarding Roadmap]]

> [!CHECKPOINT] Pass Criteria
> Optional milestone — complete after CP6. Demonstrates readiness to support AI/LLM workloads as a data engineer.

---

## ✅ Pass Criteria

- [ ] Explain what a RAG (Retrieval-Augmented Generation) pipeline is and identify the DE's role in it
- [ ] Describe the chunking → embedding → vector store flow with a concrete example
- [ ] Explain the difference between a traditional database and a vector database (storage, query type, use case)
- [ ] Describe at least 2 data quality risks specific to AI pipelines (e.g., duplicate chunks, stale embeddings, bias in training data)
- [ ] Build or describe a simple embedding pipeline: load documents → chunk → embed → store in a vector DB

### Hands-On Criteria

- [ ] Build a chunk table carrying the full metadata contract (`chunk_id`, `doc_id`, `source_uri`, `text`, `content_hash`, `embedding_model`) and show `DESCRIBE` confirming the embedding column is a fixed-size array
- [ ] Create an HNSW index and show `HNSW_INDEX_SCAN` in an `EXPLAIN` plan — then show the query form that silently falls back to `SEQ_SCAN`, and explain why
- [ ] Demonstrate that deleting a source document leaves **zero** orphan chunks, and name the third place a deletion must also reach
- [ ] Compute **recall@5** against a golden set of at least 10 questions, then change one pipeline variable and report the new number with a keep-or-revert verdict

---

**Verified by:** _________________ | **Date:** _________________
