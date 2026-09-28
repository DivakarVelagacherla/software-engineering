# Time Series Databases

## The One-Line Summary

> Time series databases are optimized for high-frequency timestamped data — write once, read by time range, never update. Built-in compression, retention, and downsampling that regular databases lack.
> 

---

## What Makes Time Series Data Different

|  | Regular Data (Orders) | Time Series Data (Sensor readings) |
| --- | --- | --- |
| **Write pattern** | Random inserts, updates | Append-only, always with timestamp |
| **Read pattern** | By ID, status, user | Always by time range |
| **Updates** | Frequent | Never — readings don't change |
| **Retention** | Keep forever | Downsample and delete old data |
| **Query frequency** | Occasionally | Constantly — real-time analysis |

---

## Why PostgreSQL Struggles

1,000 IoT sensors reporting every second = 1,000 writes/sec = 86 million rows/day.

- **Write throughput** — PostgreSQL struggles with sustained high-frequency inserts
- **Index maintenance** — every insert updates the index, slows writes further
- **Storage bloat** — ~50GB/day for sensor data vs ~500MB in InfluxDB
- **No built-in retention** — you must build downsampling and deletion yourself
- **Query performance** — time-range queries on billions of rows are slow

---

## How Time Series Databases Compress Data

Columnar storage + delta compression:

**Timestamps — delta encoding:**

```
Raw:        1704067201, 1704067202, 1704067203
Compressed: base=1704067201, deltas=[+1, +1]
```

**Temperatures — delta encoding:**

```
Raw:        72.3, 72.4, 72.5, 72.4
Compressed: base=72.3, deltas=[+0.1, +0.1, -0.1]
```

**Sensor IDs — run-length encoding:**

```
Raw:        42, 42, 42, 42, 42, 42
Compressed: (42, ×6)
```

Result: **10-50x compression** over PostgreSQL for typical IoT workloads.

---

## Data Retention + Downsampling

You don't need second-by-second readings from 3 years ago — just the trend.

```
Raw (last 7 days):        every 1 second
Hourly rollup (90 days):  one point per hour
Daily rollup (2 years):   one point per day
Older:                    deleted
```

Time series databases handle this automatically via **retention policies** and **continuous queries** (background aggregation jobs). Your application just writes raw data.

**AWS Timestream example:**

```
Memory store: 1 hour (fast, expensive)
Magnetic store: 1 year (slower, cheap)
After 1 year: auto-deleted
```

---

## Query Languages

| Database | Language | Notes |
| --- | --- | --- |
| **TimescaleDB** | SQL | PostgreSQL extension — easiest migration path |
| **InfluxDB** | Flux / InfluxQL | SQL-like with time functions |
| **Amazon Timestream** | SQL-like | AWS native, serverless |
| **Prometheus** | PromQL | Unique language, metrics-focused |
| **Cassandra** | CQL | SQL-like, extreme write throughput |

**TimescaleDB example:**

```sql
SELECT time_bucket('5 minutes', time) AS bucket,
       AVG(temperature)
FROM sensor_data
WHERE time > NOW() - INTERVAL '1 hour'
AND sensor_id = 42
GROUP BY bucket;
```

---

## Database Comparison

|  | Cassandra | InfluxDB | TimescaleDB | Timestream |
| --- | --- | --- | --- | --- |
| **Write throughput** | Extreme | High | High | High |
| **Compression** | Good | Best | Good | Good |
| **Retention policies** | Manual | Built-in | Manual | Built-in |
| **Query language** | CQL | Flux | SQL | SQL-like |
| **Use when** | Massive scale, already on Cassandra | Purpose-built time series | Already on PostgreSQL | AWS-native serverless |

---

## Three-Question Ritual

**What problem does a time series database solve?**

Storing and querying high-frequency timestamped data where PostgreSQL struggles — sustained write throughput, efficient compression, built-in retention policies and downsampling, and fast time-range queries.

**What breaks using PostgreSQL for IoT at scale?**

Write throughput collapses, index maintenance slows every insert, storage bloats (50GB/day vs 500MB), no built-in downsampling, time-range queries on billions of rows are slow.

**When NOT to use a time series database?**

When time isn't the primary query dimension. When write frequency is low — a few hundred inserts per day doesn't justify the operational complexity.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Time series database | Amazon Timestream |
| Metrics + Grafana | Amazon Managed Grafana + Timestream |
| IoT ingestion | AWS IoT Core → Timestream |
| Cassandra at scale | Amazon Keyspaces |