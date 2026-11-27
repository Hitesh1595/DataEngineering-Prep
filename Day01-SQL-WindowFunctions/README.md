# Day 01 — SQL Window Functions, CTEs & Complex Subqueries

Phase 1.1 from your roadmap, the single highest-leverage interview skill in DE. You already know SQL joins/aggregates — this day is about the stuff that separates "knows SQL" from "DE-interview-ready SQL."

Time budget: ~50 min read/understand tonight, exercises over 2-3 sittings (fine to spread across tonight + tomorrow).

---

## 1. Window functions — the core idea

A window function computes a value **per row**, looking across a "window" of related rows, **without collapsing them** into one row (that's the difference from `GROUP BY`).

```sql
SELECT
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_total
FROM orders;
```

- `PARTITION BY` — restart the calculation per group (like `GROUP BY`, but rows stay visible)
- `ORDER BY` (inside `OVER`) — defines the row sequence the function walks through
- No `ORDER BY` inside `OVER` → function sees the *whole* partition at once (e.g. a plain `SUM` per partition, not running)

Mental model: `GROUP BY` answers "one row per group." Window functions answer "one row per original row, but aware of its group."

## 2. Ranking functions — know the exact difference (classic interview trap)

| Function | Ties behavior |
|---|---|
| `ROW_NUMBER()` | Always unique, arbitrary tiebreak (1,2,3,4) |
| `RANK()` | Ties share rank, gap after (1,1,3,4) |
| `DENSE_RANK()` | Ties share rank, no gap (1,1,2,3) |
| `NTILE(n)` | Splits rows into `n` roughly-equal buckets |

```sql
SELECT
    product_id,
    revenue,
    RANK()       OVER (ORDER BY revenue DESC) AS rnk,
    DENSE_RANK() OVER (ORDER BY revenue DESC) AS dense_rnk,
    ROW_NUMBER() OVER (ORDER BY revenue DESC) AS rn
FROM product_sales;
```

**Interview trap:** "top 3 products per category" is a `ROW_NUMBER` problem (you want exactly 3), not `RANK` (ties could give you 4+ rows) — pay attention to which one the question actually implies.

## 3. LAG / LEAD — compare a row to a neighboring row

```sql
SELECT
    order_date,
    revenue,
    LAG(revenue, 1)  OVER (ORDER BY order_date) AS prev_day_revenue,
    LEAD(revenue, 1) OVER (ORDER BY order_date) AS next_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY order_date) AS day_over_day_change
FROM daily_revenue;
```

This is the standard pattern for: day-over-day change, detecting a status change (did `status` differ from the previous row for this user?), gap/session detection (is the time since the previous event > 30 min → new session).

## 4. Frame clauses — moving averages / rolling windows

Default frame when you have `ORDER BY` is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (that's why the running-total example above worked). For a rolling N-row window:

```sql
SELECT
    order_date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg
FROM daily_revenue;
```

Know the difference between `ROWS` (physical row count) and `RANGE` (logical value range) — `ROWS` is what you want for "last 7 rows," `RANGE` is what you'd want for "last 7 days" if there are gaps/duplicates in dates.

## 5. CTEs — readability + the recursive case

Non-recursive CTE = named subquery, mainly for readability and reusing a result twice:

```sql
WITH monthly_totals AS (
    SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) AS total
    FROM orders
    GROUP BY 1
)
SELECT month, total,
       total - LAG(total) OVER (ORDER BY month) AS mom_change
FROM monthly_totals;
```

**Recursive CTE** — the one that actually gets asked in interviews (org charts, bill-of-materials, date series generation):

```sql
WITH RECURSIVE org_chart AS (
    -- anchor member
    SELECT employee_id, manager_id, name, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- recursive member
    SELECT e.employee_id, e.manager_id, e.name, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart ORDER BY level;
```

Pattern to memorize: anchor (base case) `UNION ALL` recursive step that joins the CTE back to itself, moving one level at a time. Same shape every time — swap the table/condition.

## 6. Correlated vs non-correlated subqueries (know which is slower and why)

- **Non-correlated**: inner query runs once, independent of outer row. Usually fine.
- **Correlated**: inner query re-runs *per outer row* (references an outer column). Can be slow at scale — this is where `EXPLAIN` and rewriting as a `JOIN` or `WINDOW FUNCTION` often wins (we cover `EXPLAIN` properly on Day 02).

```sql
-- correlated: runs once per customer row
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id AND o.amount > 1000
);
```

Same result, often faster at scale as a window-function rewrite:

```sql
SELECT DISTINCT customer_id, name
FROM (
    SELECT c.customer_id, c.name, o.amount,
           MAX(o.amount) OVER (PARTITION BY c.customer_id) AS max_order
    FROM customers c JOIN orders o USING (customer_id)
) t
WHERE max_order > 1000;
```

This "rewrite a correlated subquery as a window function" move is a genuine senior-level signal in interviews — mention it even if you don't always use it.

---

## Exercises — zero setup needed

No DB, no account, no prior data assumed. Open **https://sqliteonline.com** (or db-fiddle.com) in browser, paste the schema+data block below, then write/run each query against it. Takes 2 min to set up.

```sql
CREATE TABLE trips (
    trip_id INTEGER PRIMARY KEY,
    vendor_id INTEGER,
    pickup_datetime TEXT,
    fare_amount REAL
);

INSERT INTO trips (trip_id, vendor_id, pickup_datetime, fare_amount) VALUES
(1, 1, '2024-01-01 08:00', 25.50),
(2, 1, '2024-01-01 09:30', 40.00),
(3, 1, '2024-01-02 07:15', 15.00),
(4, 2, '2024-01-01 08:45', 60.00),
(5, 2, '2024-01-02 10:00', 18.25),
(6, 2, '2024-01-02 10:40', 22.00),
(7, 1, '2024-01-03 11:00', 33.75),
(8, 2, '2024-01-03 09:00', 45.00);

CREATE TABLE customers (customer_id INTEGER PRIMARY KEY, name TEXT);
CREATE TABLE orders (order_id INTEGER PRIMARY KEY, customer_id INTEGER, amount REAL);

INSERT INTO customers VALUES (1,'Alice'), (2,'Bob'), (3,'Cara');
INSERT INTO orders VALUES
(1,1,100),(2,1,200),(3,2,50),(4,2,60),(5,3,900),(6,3,950);
```

Write the query, don't look up answers first, then ask me to review:

1. For each `vendor_id`, rank trips by `fare_amount` descending and return only the top 3 per vendor (think carefully: `ROW_NUMBER` or `RANK`?).
2. Compute a running total of `fare_amount` per vendor ordered by `pickup_datetime` (the rolling-7-day version needs more days of data than this toy set — understand the pattern here, we'll do a real rolling-window exercise once BigQuery is set up in Phase 3).
3. For each trip, compute the difference in `pickup_datetime` from the *previous* trip by the same `vendor_id` (gap detection pattern — note `pickup_datetime` is stored as TEXT here, so just show the LAG value, don't worry about real date-diff math yet).
4. Write a recursive CTE that generates a date series from `2024-01-01` to `2024-01-10` (common "no calendar table" trick — no source table needed, pure recursive CTE).
5. Rewrite this correlated subquery using a window function instead: find customers whose order `amount` equals their own max order amount.
```sql
SELECT c.customer_id, c.name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id AND o.amount > 500);
```

## Practice resources (yours, from your roadmap doc — not re-listing new ones)
- StrataScratch — https://stratascratch.com
- LeetCode SQL 50 — https://leetcode.com/studyplan/top-sql-50/
- Mode Analytics SQL Tutorial — https://mode.com/sql-tutorial/

## Video (watch instead of/alongside reading official docs)
- YouTube: **techTFQ** channel — search "window functions" — interview-focused, exactly this topic, heavily recommended for DE/analyst interview prep
- YouTube: **Alex The Analyst** channel — search "SQL window functions" — good beginner-friendly walkthrough if techTFQ moves too fast
(Channel names, not pinned video links — channels have multiple relevant videos and playlists get reorganized; search within channel gets you current best one.)

---

When you've attempted the 5 exercises (or want to stop partway), come back and say so — I'll review your queries and we move to **Day 02: Query optimization, EXPLAIN plans, indexing**.


## Solutions

All 5 reviewed and correct.

**1. Top 3 trips per vendor by fare_amount** (`ROW_NUMBER`, not `RANK` — need exactly 3)
```sql
WITH vendor_trips AS (
    SELECT trip_id, vendor_id, fare_amount,
           ROW_NUMBER() OVER (PARTITION BY vendor_id ORDER BY fare_amount DESC) AS rnk
    FROM trips
)
SELECT trip_id, vendor_id, fare_amount
FROM vendor_trips
WHERE rnk <= 3;
```

**2. Running total of fare_amount per vendor**
```sql
SELECT trip_id, fare_amount,
       SUM(fare_amount) OVER (PARTITION BY vendor_id ORDER BY pickup_datetime) AS running_total
FROM trips;
```

**3. Gap detection — diff from previous trip's pickup_datetime per vendor**
```sql
WITH vendor_trips AS (
    SELECT trip_id, vendor_id, pickup_datetime,
           LAG(pickup_datetime) OVER (PARTITION BY vendor_id ORDER BY pickup_datetime ASC) AS prev_pickup_datetime
    FROM trips
)
SELECT trip_id, vendor_id, pickup_datetime, prev_pickup_datetime,
       pickup_datetime - prev_pickup_datetime AS timediff
FROM vendor_trips;
```

**4. Recursive CTE — date series 2024-01-01 to 2024-01-10**
```sql
WITH RECURSIVE calendar AS (
    SELECT DATE('2024-01-01') AS date
    UNION ALL
    SELECT DATE(date, '+1 day')
    FROM calendar
    WHERE date < DATE('2024-01-10')
)
SELECT * FROM calendar;
```

**5. Correlated subquery → window function rewrite** (customers with an order over 500)
```sql
SELECT DISTINCT customer_id, name, max_amount
FROM (
    SELECT c.customer_id, c.name,
           MAX(o.amount) OVER (PARTITION BY o.customer_id) AS max_amount
    FROM customers c
    INNER JOIN orders o ON o.customer_id = c.customer_id
)
WHERE max_amount > 500;
```

