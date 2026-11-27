# Day 02 — Query Optimization, EXPLAIN Plans, Indexing

Phase 1.1 continued. Day 1 was "write the right query." Day 2 is "know why it's slow and how to prove it" — this is what separates people who can write SQL from people who get hired to fix other people's SQL.

Time budget: ~50 min read, exercises same night or next sitting.

---

## 1. What actually happens when you run a query

```
SQL text
   │
   ▼
Parser        — syntax check, builds a parse tree
   │
   ▼
Optimizer/Planner — considers multiple execution strategies, picks one based on
   │                 table statistics (row counts, data distribution, indexes available)
   ▼
Executor      — runs the chosen plan, pulls rows, returns result
```

The **optimizer** is the part that matters for you. It doesn't run your SQL literally as written — it rewrites/reorders internally and picks a *plan* (join order, scan method, sort strategy) based on stats it has about your tables. Your job as a DE: read what plan it picked and tell if it's a good one.

## 2. EXPLAIN — read the plan, don't guess

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;  -- Postgres: actually runs it, gives real timing
```

- `EXPLAIN` — shows the *planned* strategy and cost estimate, doesn't execute
- `EXPLAIN ANALYZE` (Postgres) — actually executes, shows estimated vs actual rows/time (the gap between these two is often the whole bug — optimizer guessed wrong)
- SQLite equivalent: `EXPLAIN QUERY PLAN SELECT ...` — simpler, more readable output, good for learning

### Key things to spot in a plan
| What you see | What it means |
|---|---|
| `Seq Scan` / `SCAN TABLE` / `TABLE SCAN` | Reading every row, no index used — fine for tiny tables, a red flag on big ones |
| `Index Scan` / `SEARCH TABLE ... USING INDEX` | Using an index to jump straight to matching rows |
| `Index Only Scan` | Best case — query answered entirely from the index, never touches the table |
| `Nested Loop` | For each row in outer table, scan inner table — fine for small inner table, terrible if both sides are huge |
| `Hash Join` | Builds a hash table from one side, probes with the other — good for large unsorted joins |
| `Merge Join` | Both sides already sorted on join key, walks them in parallel — good when data's pre-sorted/indexed that way |
| `Sort` node | Something needed ordering that wasn't already sorted — costs memory/time, watch for this before `GROUP BY`/`ORDER BY`/window functions |
| high **estimated rows** vs low **actual rows** (or vice versa) | Stale statistics — optimizer is flying blind, run `ANALYZE` (Postgres) to refresh stats |

## 3. Indexes — what they actually are and when they help

An index is a separate, sorted data structure pointing back to table rows — trades write cost + storage for faster reads on specific lookup patterns.

### B-tree (the default, used almost everywhere)
- Sorted structure → great for equality (`=`), range (`<`, `>`, `BETWEEN`), and `ORDER BY`/`MIN`/`MAX` on the indexed column
- This is what `CREATE INDEX idx_orders_customer ON orders(customer_id);` gives you

### Hash index
- Only supports equality (`=`), not range — faster than B-tree for pure equality lookups in theory, but most DBs' B-tree is "good enough" that hash indexes are rarely worth the specialization. Know it exists, know its one limitation (no range queries), don't over-invest here.

### Composite (multi-column) index — the one that actually trips people in interviews
```sql
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date);
```
**Leftmost-prefix rule**: this index can be used for queries filtering on `customer_id` alone, or `customer_id AND order_date` — but **not** for a query filtering on `order_date` alone. Column order in the index definition is a real design decision, not arbitrary.

```sql
-- uses the index:
WHERE customer_id = 42
WHERE customer_id = 42 AND order_date > '2024-01-01'

-- does NOT use the index (order_date isn't the leftmost column):
WHERE order_date > '2024-01-01'
```

### Covering index
An index that includes every column the query needs — the engine never has to go back to the actual table (`Index Only Scan`). Common optimization: add a "boring" extra column to an index just so a hot query becomes covering.

## 4. When an index silently doesn't help (common gotchas)

- **Wrapping the indexed column in a function**: `WHERE YEAR(order_date) = 2024` — can't use a plain index on `order_date`, because the engine would have to compute `YEAR()` on every row first. Rewrite as a range: `WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'`.
- **Leading wildcard**: `WHERE name LIKE '%smith'` — can't use a B-tree index (no fixed prefix to jump to). `WHERE name LIKE 'smith%'` can.
- **Low selectivity**: an index on a boolean `is_active` column rarely helps — if 90% of rows match, scanning the whole table is often cheaper than bouncing through an index.
- **Implicit type mismatch**: comparing a TEXT column to a number, or vice versa, can silently disable index use depending on the engine.

## 5. Practical checklist (what to actually do when a query's slow)

1. Run `EXPLAIN` (or `EXPLAIN ANALYZE` on Postgres) — don't guess.
2. Find the most expensive node (biggest estimated/actual cost, or biggest estimated-vs-actual gap).
3. Is it a full scan where an index could help? → check leftmost-prefix rule, check for function-wrapping.
4. Is it a sort you don't need (e.g. already sorted by an index you could use instead)?
5. Is it a nested loop over two large tables? → check if a hash/merge join is possible, check join order.
6. Re-run `EXPLAIN` after the fix, confirm the plan actually changed — don't assume.

---

## Exercises — zero setup, same sandbox as Day 1

Reuse the `trips` table from Day 1 (or re-paste it into sqliteonline.com if you started a fresh session). Add this index for the exercises:

```sql
CREATE INDEX idx_trips_vendor_fare ON trips(vendor_id, fare_amount);
```

1. Run `EXPLAIN QUERY PLAN SELECT * FROM trips WHERE vendor_id = 1;` — confirm it uses `idx_trips_vendor_fare`.
2. Run `EXPLAIN QUERY PLAN SELECT * FROM trips WHERE fare_amount > 20;` — does it use the index? Explain *why not*, in your own words, using the leftmost-prefix rule.
3. Write the query that *would* let this composite index fully narrow the search (filtering on both columns), and confirm via `EXPLAIN QUERY PLAN` that it's used.
4. This one's a thought exercise, no sandbox needed: you have `WHERE LOWER(vendor_name) = 'uber'` on a column with a plain index on `vendor_name`. Will the index be used? What would you change?
5. Another thought exercise: a table has 1 million rows, `is_deleted` boolean, only 50 rows have `is_deleted = true`. Would an index on `is_deleted` help a query filtering `WHERE is_deleted = true`? Would it help `WHERE is_deleted = false`? Why the difference?

## Video (per your preference — watch over reading docs)
- YouTube: **Hussein Nasser** channel — search "how indexes work" or "EXPLAIN ANALYZE" — backend/DB engineering channel, goes deep on exactly this (B-tree internals, query planning), excellent for this exact topic
- YouTube: **pganalyze** channel (if using Postgres day-to-day) — search "EXPLAIN" — short, focused breakdowns of reading real query plans

## Reference (docs only where no good video covers it as well)
- SQLite `EXPLAIN QUERY PLAN`: https://www.sqlite.org/eqp.html (short, worth a skim since the sandbox you're using is SQLite)

---

Attempt the 5 exercises (3 hands-on + 2 thought exercises), come back when done — I'll review, then we move to **Day 03: Partitioning strategies**.
