# SQL

Reach for this for joins, indexing decisions, window functions, and query tuning.
Examples are PostgreSQL-flavored.

## Joins

```sql
-- INNER: only matching rows
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id;

-- LEFT: all orders, customer null if none
SELECT o.id, c.name
FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id;

-- Find orphans (anti-join)
SELECT o.id
FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id
WHERE c.id IS NULL;
```

| Join | Returns |
|---|---|
| `INNER` | rows matching in both |
| `LEFT` | all left + matched right (nulls otherwise) |
| `RIGHT` | all right + matched left |
| `FULL` | everything, nulls where no match |
| `CROSS` | cartesian product |

## Aggregation - GROUP BY / HAVING

```sql
SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total
FROM orders
WHERE status = 'PAID'          -- filters rows BEFORE grouping
GROUP BY customer_id
HAVING SUM(amount) > 1000      -- filters groups AFTER aggregating
ORDER BY total DESC;
```

`WHERE` → before grouping (can't use aggregates). `HAVING` → after (can use aggregates).

## Window functions

```sql
-- Top order per customer (rank within a partition)
SELECT * FROM (
    SELECT id, customer_id, amount,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rn
    FROM orders
) t
WHERE rn = 1;

-- Running total
SELECT id, amount,
       SUM(amount) OVER (ORDER BY created_at) AS running_total
FROM orders;

-- Compare to previous row
SELECT id, amount,
       LAG(amount) OVER (ORDER BY created_at) AS prev,
       amount - LAG(amount) OVER (ORDER BY created_at) AS delta
FROM orders;
```

| Function | Use |
|---|---|
| `ROW_NUMBER()` | unique sequential rank (no ties) |
| `RANK()` / `DENSE_RANK()` | ranking with ties (gaps / no gaps) |
| `LAG()` / `LEAD()` | previous / next row's value |
| `SUM()/AVG() OVER` | running / windowed aggregate |
| `NTILE(n)` | bucket rows into n groups |

Window functions keep every row (unlike `GROUP BY` which collapses).

## CTEs (`WITH`) over nested subqueries

```sql
WITH paid_orders AS (
    SELECT * FROM orders WHERE status = 'PAID'
),
totals AS (
    SELECT customer_id, SUM(amount) AS total FROM paid_orders GROUP BY customer_id
)
SELECT c.name, t.total
FROM totals t JOIN customers c ON c.id = t.customer_id
WHERE t.total > 1000;
```

Readable, composable. Recursive variant with `WITH RECURSIVE` for trees/hierarchies.

## Indexing

```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_orders_cust_status ON orders (customer_id, status);   -- composite
CREATE UNIQUE INDEX idx_orders_ref ON orders (reference);
CREATE INDEX idx_orders_created ON orders (created_at) WHERE status = 'PAID';  -- partial
```

- **B-tree** (default) - equality + range + sort + prefix.
- **Composite `(a, b)`** helps `WHERE a`, `WHERE a AND b`, and `ORDER BY a, b` - **not** `WHERE b` alone (leftmost-prefix rule).
- **Covering index** - includes all columns a query needs → index-only scan, no table hit.
- Index write cost: every index slows inserts/updates. Index for reads you actually run.

## `EXPLAIN` / `EXPLAIN ANALYZE`

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;
```

Read it: `Seq Scan` on a big table = missing index. `Index Scan` / `Index Only Scan` = good.
Watch actual vs estimated **rows** (bad estimates → stale stats, run `ANALYZE`) and the top-level cost.

## Transactions & isolation

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- or ROLLBACK;
```

| Level | Prevents |
|---|---|
| Read Committed (PG default) | dirty reads |
| Repeatable Read | + non-repeatable reads |
| Serializable | + phantoms (full isolation, may abort) |

## Upsert

```sql
INSERT INTO inventory (sku, qty) VALUES ('ABC', 10)
ON CONFLICT (sku) DO UPDATE SET qty = inventory.qty + EXCLUDED.qty;

INSERT INTO users (email, name) VALUES ('a@b.com', 'A')
ON CONFLICT (email) DO NOTHING;
```

## Common tuning wins

- **N+1**: one query per row in a loop → replace with a single `JOIN` / `IN`. (JPA: `JOIN FETCH` / `@EntityGraph`.)
- **`SELECT *`**: select only needed columns → enables covering indexes, less IO.
- **Missing index** on a `WHERE`/`JOIN`/`ORDER BY` column → `Seq Scan`.
- **`OR` across columns** kills index use → rewrite as `UNION` or use separate indexes.
- **Functions on indexed columns** (`WHERE lower(email) = ...`) skip the index → use an expression index.
- **Leading wildcard** `LIKE '%foo'` can't use a B-tree → trigram/GIN index or restructure.

## Gotchas / things I always forget

- `WHERE` filters before grouping; `HAVING` after. Don't put aggregates in `WHERE`.
- `LEFT JOIN` + a `WHERE right.col = x` silently becomes an inner join (nulls fail the predicate) - put the condition in `ON` instead.
- Composite index `(a, b)` does nothing for `WHERE b` alone - column order matters (leftmost prefix).
- `NULL` is never `=` anything, not even `NULL`. Use `IS NULL` / `IS DISTINCT FROM`.
- `COUNT(*)` counts rows; `COUNT(col)` skips NULLs - different answers.
- `ROW_NUMBER` needs a deterministic `ORDER BY` or results are non-repeatable.
- More indexes = slower writes and more storage; don't index everything.

## Quick reference

| Task | SQL |
|---|---|
| Top N per group | `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` |
| Running total | `SUM(x) OVER (ORDER BY ...)` |
| Prev/next value | `LAG()` / `LEAD()` |
| Orphan rows | `LEFT JOIN ... WHERE r.id IS NULL` |
| Upsert | `INSERT ... ON CONFLICT DO UPDATE` |
| See the plan | `EXPLAIN ANALYZE ...` |
| Composite index rule | leftmost prefix |
| Reusable subquery | `WITH cte AS (...)` |
