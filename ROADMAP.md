# Data Engineering Transition Roadmap — Day-by-Day Execution

Source of truth: your own `DE_Roadmap.docx` (Phase 1-4, 7-9 month plan). This file breaks that into day-sized study sessions matching your real schedule (45-60 min weeknights after 10pm, 2-3hr weekend blocks), and is where I'll drop deep tutor content one topic at a time as you go. Don't jump ahead and read future days — ask and I'll generate the next one when you're ready.

Pace note: ~5 nights/week + weekend blocks ≈ 3-4 "days" of content per week → whole roadmap ≈ 9-10 weeks of calendar time for Phases 1-3, plus however long the capstone + interview loop takes.

Convention: every day's content includes a **Video** section (YouTube channel + what to search) alongside written notes — official docs get linked only when there's no good video equivalent, since video is the preferred learning mode here.

## How the capstone project ties everything together (the "end-to-end" map you asked for)

One pipeline, built incrementally as you learn each piece — not a separate project bolted on at the end:

```
[Public API / NYC Taxi data]
        │  (ingestion script — Python)
        ▼
[Docker container running ingestion]
        │
        ▼
[Airflow DAG]  ──schedules & retries──▶  orchestrates every step below
        │
        ▼
[Google Cloud Storage]  (data lake — raw landing zone)
        │
        ▼
[BigQuery]  (data warehouse — partitioned + clustered raw tables)
        │
        ▼
[dbt models]  (staging → intermediate → marts, + dbt tests)
        │
        ▼
[Delta Lake / lakehouse layer]  (if you add the streaming project — Bronze/Silver/Gold)
        │
        ▼
[Looker Studio dashboard]  or  [reverse ETL back to an app]
```

None of this exists yet — it's the target picture, built one box at a time starting Phase 2 (Docker/Airflow). Days before that (SQL, Python, modeling) need zero setup — just a browser-based SQL sandbox, no account/install/data required.

Every phase below bolts one more box onto this diagram. By the time you finish Phase 3, you've built Project 1 from your roadmap end-to-end for real, not just in notes. Databricks gets folded into the Lakehouse step (3.3) — it's the managed platform for exactly that Bronze/Silver/Gold + Delta Lake pattern, so it's additive to what your roadmap already planned, not a separate track.

## Reference books you already own (used where relevant, not re-explained from scratch)
- *Fundamentals of Data Engineering* (Reis & Housley) — lifecycle framework, already covered together in this chat
- *Designing Data-Intensive Applications* (Kleppmann) — distributed systems depth for Phase 3 system design rounds
- *System Design Interview* (Alex Xu) — general system design muscle, applies to DE design questions too

---

## Phase 1 — Foundations (SQL, Python, DSA-lite)

| Day | Topic | Status |
|---|---|---|
| 01 | SQL — Window functions, CTEs, complex subqueries | ✅ content ready (`Day01-SQL-WindowFunctions/`) |
| 02 | SQL — Query optimization, EXPLAIN plans, indexing (B-tree/hash/composite) | next |
| 03 | SQL — Partitioning strategies (range/list/hash) + practice set | pending |
| 04 | Python for DE — Pandas/NumPy, file formats (Parquet/Avro/ORC/JSON/CSV) | pending |
| 05 | Python for DE — OOP patterns, clean/testable pipeline code | pending |
| 06 | Data Modeling — Star vs Snowflake, Fact/Dim tables, OLAP vs OLTP | pending |
| 07 | Data Modeling — Slowly Changing Dimensions (SCD 1/2/3) | pending |
| — | **Mock/Revision checkpoint 1** — SQL + modeling, timed | pending |

DSA (parallel, not day-numbered): your own NeetCode Blind75 track — https://neetcode.io/practice/practice/blind75 — keep doing 2-3/week alongside the above; I won't rebuild a separate problem set, just ask if you want a specific pattern explained.

## Phase 2 — Core DE Tools

| Day | Topic |
|---|---|
| 08-09 | Docker — containerizing the ingestion script (ties into capstone diagram above) |
| 10-12 | Airflow (deep) — DAGs, operators, scheduling/dependencies, XComs, retries/alerting |
| 13-15 | PySpark — DataFrames/Spark SQL, partitioning & shuffle, caching, RDD fundamentals, tuning |
| 16-18 | dbt — models, tests, sources/refs, Jinja macros, lineage |
| 19-20 | Kafka — topics/partitions/offsets, producer/consumer, Spark Streaming integration |
| — | **Mock/Revision checkpoint 2** — explain the capstone pipeline end-to-end out loud |

## Phase 3 — Cloud + DE System Design

| Day | Topic |
|---|---|
| 21-22 | GCP core — BigQuery deep (partitioning/clustering/cost), GCS, Pub/Sub, Cloud Composer |
| 23 | AWS equivalents (Redshift, Glue, S3, EMR) — map, don't deep-dive both clouds |
| 24-26 | Lakehouse — Delta Lake, Apache Iceberg, Medallion architecture, **Databricks** (workspace, Delta Lake, Unity Catalog, job clusters, DLT) |
| 27-28 | DE system design practice — ride-sharing pipeline, e-commerce warehouse, 1TB/day ETL, CDC system design (using DDIA + Alex Xu for depth) |
| 29 | Databases/file formats map — when to use what (warehouse vs OLTP vs NoSQL vs cache vs file format) |
| — | **Mock/Revision checkpoint 3** — full system design mock interview |

## Phase 4 — Supporting Skills + Apply

| Day | Topic |
|---|---|
| 30 | DevOps for DE — Docker/Git/CI-CD for pipelines, Linux/cron/bash |
| 31 | Data quality & observability — Great Expectations, dbt tests, data contracts |
| 32-35 | **Capstone finishing** — wire every piece into one polished repo + README architecture diagram + Looker Studio dashboard |
| 36+ | Resume + "how to present your 6yr SWE → DE transition" story, mock interviews, revision rotation |

---

## Revision cadence (so earlier topics don't fade)
- End of each phase: one checkpoint day (marked `—` above) — no new content, just explain-out-loud + redo one practice problem from each prior topic.
- Every 2 weeks regardless of phase: 15 min flipping back through Day-N folders you've already done, not rereading in full — just headings + exercises.

Next up: **Day 01 — SQL Window Functions & CTEs**, in `Day01-SQL-WindowFunctions/`.
