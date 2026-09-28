# Data Lakes Vs Warehouses

## The One-Line Summary

> Data Lakes store raw unstructured data cheaply with no schema required. Data Warehouses store structured, cleaned data optimized for fast analytical queries. They complement each other, not compete.
> 

---

## The Core Distinction

|  | OLTP (Postgres) | Data Warehouse (OLAP) | Data Lake |
| --- | --- | --- | --- |
| **Purpose** | Transactions | Fast analytics | Raw storage |
| **Data** | Current state, structured | Cleaned, structured | Everything raw |
| **Schema** | Fixed upfront | Fixed upfront | Schema on read |
| **Query speed** | Fast for single records | Fast for aggregations | Slow, flexible |
| **Examples** | PostgreSQL, MySQL | Redshift, BigQuery, Snowflake | S3, HDFS, Azure Data Lake |
| **Use for** | User places order | CEO quarterly report | Raw log exploration |

---

## Schema on Write vs Schema on Read

**Data Warehouse — Schema on Write:**

Define columns before storing. Data must conform. Fast queries because structure is known.

**Data Lake — Schema on Read:**

Store raw in any format (CSV, JSON, logs, images, video). Apply structure when querying. Flexible but slower.

---

## Data Lake

Store everything raw — clickstream events, server logs, customer reviews, images, sensor readings — in original format. Figure out structure later when you know what questions to ask.

**When to use:**

- Unstructured or semi-structured data
- Exploratory analysis — you don’t know the questions yet
- Raw data from multiple sources with different schemas
- Long-term cheap storage of everything

**When NOT to use:**

- Structured data with known schema — warehouse is faster and cheaper to query
- When fast analytical queries are required — lake adds transformation overhead

---

## Data Warehouse

Cleaned, structured, pre-organized data optimized for analytical queries. Columnar storage, massively parallel execution.

**When to use:**

- Known schema, structured data
- Fast analytical queries — CEO dashboard, quarterly reports
- BI tools and reporting

---

## Getting Data from Postgres to Warehouse/Lake

Three approaches, all eventually consistent — user doesn’t wait for warehouse write:

### ETL/ELT — AWS Glue

Managed service. Connects to Postgres, extracts on schedule, transforms, loads into Redshift or S3.

```
Postgres → AWS Glue (scheduled) → Redshift / S3
```

### CDC — Change Data Capture (near real-time)

Captures every Postgres change by reading the WAL (write-ahead log). Streams changes as events.

```
Postgres WAL → Debezium → Kinesis → Lambda → S3 / Redshift
```

### Scheduled Lambda (simple)

Run hourly, query Postgres for new records since last run, write to S3 or Redshift.

```
CloudWatch Events → Lambda → query Postgres → write to S3
```

**For AWS-native stacks:** CDC via Kinesis into S3 (data lake), then query with Athena. Natural fit for Lambda + Kinesis architectures.

---

## Modern Pattern — Lakehouse

Many teams now use a **Lakehouse** — combines both. Store raw data in S3 (lake), use a query layer like AWS Athena or Delta Lake that applies structure on read and caches results for speed. Best of both worlds.

---

## Three-Question Ritual

**What problem does a data lake solve?**

Storing massive raw unstructured data in its original format with no upfront schema. Store everything, apply structure only when you know what questions to ask.

**What breaks using a warehouse for raw unstructured data?**

Warehouse requires schema on write. Raw logs, clickstream, images don’t fit predefined columns. You’d lose information or spend enormous time transforming before knowing what questions you’ll ask.

**When NOT to use a data lake?**

When data is structured with a known schema — warehouse queries it faster with no transformation overhead. When fast analytical response time is required.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Data Lake | Amazon S3 |
| Data Warehouse | Amazon Redshift |
| ETL | AWS Glue |
| Query data lake with SQL | Amazon Athena |
| CDC streaming | Amazon DMS, Debezium + Kinesis |
| Lakehouse | AWS Lake Formation + Athena |