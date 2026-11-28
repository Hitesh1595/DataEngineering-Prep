# Day 04 — Python for Data Engineering: Pandas/NumPy + File Formats

Phase 1.2 from your roadmap. You already know Python as a language — this day is about the DE-specific layer on top: how pandas is actually used in pipelines (not notebooks-for-analysis), and the file format decisions that show up in almost every DE interview.

Time budget: ~50 min read, exercises need a Python environment (see zero-setup option below).

---

## 1. Where pandas actually sits in a DE pipeline

Pandas is your **prototyping and small/medium-data layer** — exploring a new source, writing a quick transform, validating a sample before you commit to a Spark job that'll run the same logic at scale. It is not what you run on 500GB of data (that's Spark/Databricks, Phase 3). Knowing *when* pandas is the right tool (fits in memory, one-machine job, ad hoc/exploratory) vs when it isn't (anything that doesn't fit in RAM, needs distributed compute) is itself an interview-relevant judgment call.

## 2. NumPy underneath — why pandas is fast (when used right)

Pandas is built on NumPy arrays. The speed comes from **vectorization** — operating on a whole array at once in compiled C code, instead of looping row-by-row in Python.

```python
import pandas as pd

df = pd.DataFrame({'fare': [25.5, 40.0, 15.0, 60.0]})

# slow: Python-level loop, defeats the whole point of pandas
df['fare_with_tax'] = df['fare'].apply(lambda x: x * 1.1)

# fast: vectorized, runs in compiled code across the whole column at once
df['fare_with_tax'] = df['fare'] * 1.1
```
**Interview-relevant rule**: `.apply()` with a row-wise lambda is a code smell in pandas — it's barely faster than a plain Python loop. If you reach for `.apply()`, first ask whether a vectorized expression, `.map()` on a Series, or a built-in method (`.str.*`, `.dt.*`) can do it instead.

## 3. Core DE-relevant pandas patterns

### Reading/writing multiple formats (same API shape every time)
```python
df = pd.read_csv('orders.csv')
df = pd.read_parquet('orders.parquet')
df = pd.read_json('orders.json')

df.to_parquet('orders_out.parquet', index=False)
df.to_csv('orders_out.csv', index=False)
```

### GroupBy + aggregation
```python
df.groupby('vendor_id').agg(
    total_fare=('fare_amount', 'sum'),
    trip_count=('trip_id', 'count'),
    avg_fare=('fare_amount', 'mean')
).reset_index()
```

### Merge/join — same concepts as SQL joins, different syntax
```python
pd.merge(trips, vendors, on='vendor_id', how='left')
```

### dtypes matter at scale (memory, not just correctness)
```python
df.info(memory_usage='deep')          # see actual memory per column
df['vendor_id'] = df['vendor_id'].astype('category')   # low-cardinality string/int -> huge memory win
df['fare_amount'] = df['fare_amount'].astype('float32')  # float64 -> float32 halves memory if precision allows
```
A DE who doesn't think about dtypes is the person whose "it works on my laptop" pipeline falls over the first time it hits a real dataset.

### Missing data — be explicit, don't let pandas guess silently
```python
df.isna().sum()                 # see what's actually missing, column by column, before doing anything
df['amount'].fillna(0)          # or dropna(), or a flag column — but a *decision*, not a default
```

## 4. File formats — the part that gets asked in almost every DE interview

### Row-oriented vs columnar — the one concept that explains everything else
- **Row-oriented** (CSV, JSON, Avro): one row's data stored together, contiguous on disk. Good for writing/appending records one at a time, and for reading whole rows.
- **Columnar** (Parquet, ORC): one column's data stored together across all rows. Good for analytical queries that only touch a few columns out of many (`SELECT customer_id, amount FROM ...` on a 50-column table reads 2 columns' worth of bytes, not 50).

| Format | Orientation | Schema | Compression | Splittable | Typical use |
|---|---|---|---|---|---|
| CSV | Row | None (just text) | Poor (unless gzipped externally) | Yes (by line) | Interchange, human-readable, legacy sources |
| JSON | Row | Self-describing, nested | Poor | Depends (newline-delimited JSON yes, single JSON array no) | APIs, semi-structured/nested data |
| Avro | Row | Schema embedded, strong **schema evolution** support | Good | Yes | Streaming/Kafka payloads — schema can change between producer/consumer versions safely |
| Parquet | Columnar | Schema embedded | Excellent (per-column, type-aware) | Yes | **The DE default for analytics** — data lakes, warehouses, Spark/BigQuery native format |
| ORC | Columnar | Schema embedded | Excellent | Yes | Same niche as Parquet, historically Hive-ecosystem default |

### Why Parquet specifically won as "the" DE format
1. **Column pruning** — query touches 3 of 50 columns, engine reads only those 3 columns' bytes from disk/storage.
2. **Predicate pushdown** — Parquet stores min/max stats per column chunk, so a filter like `WHERE amount > 1000` can skip whole chunks without even decompressing them (same spirit as partition pruning from Day 3, one level more granular).
3. **Type-aware compression** — compressing a column of all-integers or all-timestamps compresses far better than compressing mixed-type rows (CSV/JSON).
4. **Splittable + parallelizable** — a single large Parquet file can be read in parallel chunks by Spark/BigQuery, unlike a single JSON array.

### When you'd still pick something else
- **CSV**: a one-off export someone needs to open in Excel, or a tiny interchange file where human-readability beats performance.
- **JSON**: the data is genuinely nested/semi-structured (not naturally tabular) and you're closer to the API/application layer than the analytics layer.
- **Avro**: a Kafka topic where producers and consumers evolve their schema independently over time and you need safe forward/backward compatibility — this is Avro's actual killer feature, not just "it's row-oriented."

---

## Exercises — zero local setup option: Google Colab (colab.research.google.com), free, pandas pre-installed, no pip install needed

If you prefer local: `pip install pandas pyarrow` (pyarrow is the engine pandas uses for Parquet).

```python
import pandas as pd

data = {
    'trip_id': range(1, 1001),
    'vendor_id': [1, 2] * 500,
    'fare_amount': [25.5, 40.0, 15.0, 60.0, 18.25] * 200,
    'pickup_datetime': pd.date_range('2024-01-01', periods=1000, freq='h'),
}
df = pd.DataFrame(data)
```

1. Write `df` to both `trips.csv` and `trips.parquet`. Compare file sizes with `import os; os.path.getsize(...)` — by how much smaller is the Parquet file, and why (tie back to section 4).
2. Do a `groupby('vendor_id')` aggregation computing total and average `fare_amount` per vendor — same logic as Day 1's SQL window function exercise, now in pandas. Notice how much more verbose this is than the equivalent SQL — that's normal, it's why DE pipelines often push aggregation back into SQL/the warehouse rather than doing it in pandas.
3. Convert `vendor_id` to `category` dtype and compare `df.info(memory_usage='deep')` before/after.
4. Rewrite this row-wise `.apply()` as a vectorized expression, and explain in one sentence why the vectorized version is faster:
   ```python
   df['fare_with_tax'] = df['fare_amount'].apply(lambda x: x * 1.1)
   ```
5. Thought exercise, no code needed: you're ingesting a Kafka topic where the producer team plans to add a new optional field to events every few months, and old consumers must keep working without breaking. Which file format from the table above fits this requirement best, and why specifically (not just "because it's good")?

## Video (verified titles)
- YouTube, **Corey Schafer** channel — playlist **"Pandas Tutorials"** (confirmed real, ~11 videos, well-known Python/pandas series) — good for the pandas mechanics themselves if any of section 2-3 felt unfamiliar
- YouTube — talk title **"The Apache Spark File Format Ecosystem"** (Databricks/Spark Summit conference talk, ~29 min) — covers Parquet/ORC/Avro origins and tradeoffs in more depth than section 4 above

## Reference (docs only where no video covers it as well)
- Apache Parquet overview: https://parquet.apache.org/docs/overview/ (short, worth a skim for the "why" behind columnar storage)

---

Attempt the 5 exercises, come back when done — then **Day 05: Python OOP patterns for production-grade pipelines**.
