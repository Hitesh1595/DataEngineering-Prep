# Day 03 — Partitioning Strategies

Phase 1.1 wrap-up. Day 2 was indexing (finding rows faster within a table). Day 3 is partitioning — splitting the table itself so queries don't even have to look at most of it. Different lever, same goal: don't touch data you don't need.

Time budget: ~40 min read — lighter day, mostly conceptual, since the hands-on version of this comes back hard in Phase 3 with BigQuery.

---

## 1. What partitioning actually is

A partitioned table is **one logical table, split into multiple physical pieces** ("partitions"), each holding a subset of rows based on a partition key. The application still queries it as one table — the engine decides internally which physical pieces to read.

```
orders (logical table)
├── orders_2024_01   (partition: Jan 2024 rows)
├── orders_2024_02   (partition: Feb 2024 rows)
├── orders_2024_03   (partition: Mar 2024 rows)
└── ...
```

```sql
SELECT * FROM orders WHERE order_date >= '2024-02-01' AND order_date < '2024-03-01';
```
If `orders` is range-partitioned by month on `order_date`, the engine knows this query only needs `orders_2024_02` — it skips every other partition entirely. That skip is called **partition pruning**, and it's the entire point.

## 2. Partitioning vs indexing — different lever, not a replacement

- **Indexing** (Day 2): within a scanned set of data, jump straight to matching rows via a tree structure.
- **Partitioning**: before scanning even starts, throw away whole chunks of the table that can't possibly match.

They stack together — a well-designed table is partitioned (coarse pruning) *and* indexed within each partition (fine-grained lookup). Partitioning also helps independently of query speed: easier maintenance (drop/archive an old partition instantly instead of a slow `DELETE`), parallel loading, smaller working sets for backups.

## 3. The three classic types

### Range partitioning — by far the most common in DE
Partition boundaries are ranges of a value — almost always a date/timestamp in data pipelines.
```sql
-- Postgres syntax, conceptual (SQLite can't actually run this — see note below)
CREATE TABLE orders (
    order_id INT,
    order_date DATE,
    amount NUMERIC
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```
This is exactly what BigQuery's "partition by date column" does under the hood (Phase 3, Day 21-22) — same idea, managed for you.

### List partitioning — by discrete known values
Partition by a column with a small, known set of values — region, country, tenant, status.
```sql
CREATE TABLE orders_by_region (
    order_id INT, region TEXT, amount NUMERIC
) PARTITION BY LIST (region);

CREATE TABLE orders_apac PARTITION OF orders_by_region FOR VALUES IN ('IN', 'SG', 'JP');
CREATE TABLE orders_emea PARTITION OF orders_by_region FOR VALUES IN ('UK', 'DE', 'FR');
```
Good when most queries naturally filter by that discrete dimension (e.g. a region-scoped dashboard, a per-tenant report).

### Hash partitioning — when there's no natural range/list key
Partition key is hashed, rows spread evenly across a fixed number of partitions — used purely for **even distribution** (avoiding one huge partition / hotspot), not because the query filters on it.
```sql
CREATE TABLE events (event_id INT, user_id INT, payload TEXT) PARTITION BY HASH (user_id);
CREATE TABLE events_p0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- ... p1, p2, p3
```
Doesn't give you pruning for arbitrary range queries — it gives you balanced writes/storage when the data itself has no natural date/category to split on.

## 4. Choosing a partition key — the real design skill

The partition key is only useful if it matches how the table is **actually queried**. Common mistakes:

- **Partition key doesn't match the filter column** — table partitioned by `order_date`, but most queries filter by `customer_id` → zero pruning benefit, you're paying partition overhead for nothing.
- **Too many tiny partitions** — partitioning a low-volume table by day creates thousands of mostly-empty partitions. Partition metadata itself has overhead (the planner has to consider every partition), and this is the same root problem as the "small files problem" you'll hit again with Parquet files in cloud storage (Phase 2/3) — same lesson, different layer.
- **Partition key with skewed values** — list/range partitioning where one partition ends up holding 90% of the data defeats the purpose; that's when hash partitioning (or a different key) is the right call.

Rule of thumb: pick the partition key that matches your most common, highest-value `WHERE` clause — usually a date column for anything event/transaction-like.

## 5. A note on SQLite (your sandbox) vs real partitioning

SQLite has **no native table partitioning** — the `CREATE TABLE ... PARTITION BY` syntax above is Postgres/BigQuery-style, shown for you to read and understand, not to run in sqliteonline.com. The exercises below are reasoning exercises for this reason — real hands-on partitioning returns in Phase 3 (BigQuery `PARTITION BY` + `CLUSTER BY`, Day 21-22), where you'll actually create and query partitioned tables.

---

## Exercises — reasoning, no sandbox needed this time

1. An `orders` table is queried almost exclusively with `WHERE order_date BETWEEN ... AND ...` for the last 30/90 days. Design a partitioning scheme: which type, what granularity (daily/monthly/yearly), and why that granularity specifically (think about the "too many tiny partitions" gotcha).
2. A multi-tenant SaaS table `events` is always queried `WHERE tenant_id = X` (never cross-tenant). Which partition type fits, and what's the pruning benefit for a single-tenant query?
3. A table is partitioned by `order_date` (monthly), but the dashboard's slowest query actually filters `WHERE customer_id = 123` with no date filter at all. Will partitioning help this query? What would you need to add to actually speed it up?
4. You hash-partition a `user_events` table by `user_id` into 8 partitions to spread write load evenly. A query comes in: `WHERE user_id = 42`. Does this query benefit from partition pruning the same way a range-partitioned date query does? Why or why not (think about what hash partitioning is actually optimizing for).
5. A table gets partitioned by day for a system that only ingests ~200 rows/day. After a year, how many partitions exist, and what's likely to go wrong (connect this to the "small files problem" mentioned above)?

## Solutions

**1.** Range partitioning, **monthly** granularity. Range partitioning prunes out every month outside the queried window. Monthly (not daily) avoids the over-partitioning gotcha — a 30/90-day query window still only touches 1-3 monthly partitions, while daily granularity would mean hundreds of tiny partitions for no extra pruning benefit.

**2.** List partitioning by `tenant_id`. A single-tenant query only scans that one tenant's partition — every other tenant's data is pruned out entirely.

**3.** No, partitioning doesn't help — there's no date filter, so every monthly partition still has to be scanned for `customer_id = 123`. Partitioning and indexing are separate levers: the real fix here is a (non-clustered) **index on `customer_id`**, which works independently of how the table is partitioned.

**4.** Yes, for this exact query — hashing `user_id = 42` lands on exactly one partition, same pruning benefit as a range match. The gap shows up on a *range* query instead (`WHERE user_id BETWEEN 10 AND 50`): hashing destroys ordering, so those ids scatter across all 8 partitions and nothing gets pruned. Hash partitioning optimizes for equality lookups + even write distribution, not range queries.

**5.** ~365 partitions after a year, each holding only ~200 rows — way too many near-empty partitions. The planner has to consider every partition's metadata even for a simple query, so overhead grows while each partition does almost no useful pruning work — the same "small files problem" you'll see again with Parquet files in cloud storage.

## Video (verified titles — search these exact phrases)
- YouTube, **Hussein Nasser** channel — exact video title: **"They Enabled Postgres Partitioning and their Backend Fell Apart"** — real-world case study of partitioning done wrong, directly relevant to exercise 3/5 above
- Same channel, course playlist **"Fundamentals of Database Engineering"** — module **"06. Database Partitioning"** — the structured version of this same material

## Reference (docs only where no video covers it as well)
- Postgres table partitioning: https://www.postgresql.org/docs/current/ddl-partitioning.html (skim only — syntax reference if you want to see more examples)

---

Attempt the 5 reasoning exercises, come back when done — then **Day 04: Python for Data Engineering (Pandas/NumPy, file formats)**.


## NOTE(PERSONAL)
- Sharding:method of distribing of data across multiple machines(db server)
- partition:Splitting of data within same instance or can be split across diff shards(machine have db server)

