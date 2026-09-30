# Database Indexing

## The One-Line Summary

> An index is a sorted data structure that lets the database find rows instantly instead of scanning every row. Reads get faster, writes get slower.
> 

---

## The Problem Without Indexes

You have a `users` table with 50 million rows. A query comes in:

```sql
SELECT * FROM users WHERE email = 'divakar@gmail.com'
```

Without an index, the database does a **full table scan** — checks every single row until it finds a match. 50 million comparisons for one row. Slow, expensive, gets worse as the table grows.

---

## What an Index Is

An index is a separate data structure — usually a **B-tree** — that stores indexed column values in sorted order, with a pointer back to the full row.

**How it works:**

1. Query hits the index instead of the table
2. Database uses binary search on the sorted B-tree — finds the value in a few dozen steps instead of 50 million
3. Follows the pointer directly to the full row

> Primary keys are indexed automatically by default in PostgreSQL and most SQL databases. Any other column you want fast lookups on, you create the index manually.
> 

---

## The Core Tradeoff

|  | Without Index | With Index |
| --- | --- | --- |
| **Read speed** | Slow — full table scan | Fast — B-tree lookup |
| **Write speed** | Fast — just insert the row | Slower — must update the index too |
| **Storage** | Just the table | Table + index data structure |

**Why writes slow down:** Every insert, update, or delete must keep the B-tree sorted and balanced. The new value has to be placed in the right position. The more indexes on a table, the more work on every write.

---

## When NOT to Add an Index

- **Small tables** — full table scan on 1,000 rows is fast enough. Index adds overhead for no benefit.
- **Write-heavy tables** — logging tables, event streams, millions of inserts per second. Write overhead outweighs read benefit.
- **Low cardinality columns** — cardinality = number of unique values. A `gender` column with 2-3 unique values across 50 million rows is useless to index — the database still returns millions of rows per lookup. Index `email`, `user_id`, `order_id` — high cardinality columns where each value is unique or nearly unique.
- **Non-critical, infrequently queried columns** — if slow reads are acceptable, don't pay the write cost.

---

## Composite Indexes

An index on multiple columns together. Example: index on `(last_name, first_name)`.

**The left prefix rule:** A composite index only helps queries that use the leftmost columns in the index definition.

| Query | Uses index? |
| --- | --- |
| `WHERE last_name = 'X'` | ✅ Yes |
| `WHERE last_name = 'X' AND first_name = 'Y'` | ✅ Yes |
| `WHERE first_name = 'Y'` | ❌ No — first_name is not the leftmost column |

> Column order in a composite index matters. Put the most frequently filtered column first.
> 

---

## Three-Question Ritual

**What problem do indexes solve?**

Full table scans are slow on large tables. Indexes let the database find rows instantly using a sorted B-tree structure instead of scanning every row.

**What breaks if you add too many indexes?**

Write performance degrades. Every insert, update, and delete must update all indexes on that table. Storage also bloats — each index is a separate data structure.

**When NOT to add an index?**

Small tables, write-heavy tables, low cardinality columns, and columns that are rarely queried.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| SQL with indexes | Amazon RDS (PostgreSQL/MySQL) — indexes configured at the table level |
| Managed index tuning | Amazon DevOps Guru for RDS — detects missing or redundant indexes |