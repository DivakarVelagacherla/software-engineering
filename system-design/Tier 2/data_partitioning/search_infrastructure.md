# Search Infrastructure

## The One-Line Summary

> Search infrastructure solves what SQL can't — full-text matching, fuzzy search, relevance ranking, and fast search across millions of records using an inverted index.
> 

---

## Why SQL LIKE Fails for Search

```sql
SELECT * FROM products WHERE description LIKE '%wireless headphones%'
```

- **Full table scan** — can't use indexes, O(n) on millions of rows
- **No partial matching** — "headphon" won't find "headphones"
- **No synonyms** — "wireless" won't find "Bluetooth" even though they're related
- **No relevance ranking** — returns everything that matches, in random order
- **No natural language** — can't parse "under $50" as a price filter

---

## The Inverted Index

The core data structure behind search engines.

**Regular index:** row → words

```
Product 1 → "wireless bluetooth headphones sony"
Product 2 → "wireless earbuds apple"
```

**Inverted index:** word → rows

```
"wireless"    → [Product 1, Product 2]
"bluetooth"   → [Product 1]
"headphones"  → [Product 1]
"earbuds"     → [Product 2]
```

**Search "wireless headphones":**

1. Look up "wireless" → [Product 1, Product 2]
2. Look up "headphones" → [Product 1]
3. Intersection → Product 1

O(1) lookup instead of O(n) full table scan.

---

## Relevance Ranking

Two signals combined:

### 1. Text Relevance (BM25 — Elasticsearch default)

- **Term Frequency (TF)** — how many times does the search term appear in the document?
- **Inverse Document Frequency (IDF)** — how rare is the term? Common words ("wireless") get lower weight. Rare words ("XM5") get higher weight.

### 2. Behavioral Signals

- Number of sales / purchase rate
- Click-through rate
- Reviews and ratings
- Recency of purchases

Batch job computes popularity score → stored as a field → Elasticsearch combines text relevance + popularity for final ranking.

---

## Elasticsearch

Most common search infrastructure. Built on Apache Lucene.

**Features:**

- Inverted index with BM25 relevance scoring
- Full-text search — partial matches, fuzzy matching ("headphon" → "headphones")
- Filters — price < $50, category = electronics
- Horizontal scaling — shards index across multiple nodes, parallel search
- Near real-time — new documents searchable within ~1 second

**Elasticsearch is not your source of truth.** Postgres is. Elasticsearch mirrors your data optimized for search. If it goes down, rebuild the index from Postgres.

---

## Syncing Data to Elasticsearch

### Option 1 — Dual Write

```
App → Postgres (source of truth)
App → Elasticsearch (search index)
```

Simple but risky — if Elasticsearch write fails, data is out of sync.

### Option 2 — CDC (Change Data Capture)

```
Postgres WAL → Debezium → Kafka/Kinesis → Elasticsearch
```

Eventually consistent, decoupled, reliable. Debezium reads Postgres WAL and streams every change as an event.

### Option 3 — Batch Sync

```
CloudWatch → Lambda → query Postgres → update Elasticsearch
```

Simple, not real-time. Good for low-frequency changes.

---

## Debezium — What It Is

Open source CDC tool. Java application that reads the database WAL and publishes every insert, update, delete as an event to Kafka.

**Why WAL over triggers or polling:**

- Triggers add overhead to every write
- Polling misses deletes and rapid changes
- WAL captures everything with zero write overhead

**Deployment options:**

- Kafka Connect plugin (most common)
- ECS container
- **AWS DMS** (managed equivalent — no infrastructure to manage)

**For AWS stacks:** AWS DMS + Kinesis replaces Debezium. Configure source (RDS Postgres), target (Kinesis/S3), enable CDC — AWS manages everything.

---

## Three-Question Ritual

**What problem does search infrastructure solve?**

Three problems SQL can't handle: full-text/fuzzy matching, relevance ranking (most relevant results first), and performance (inverted index vs full table scan on millions of records).

**What breaks using SQL LIKE on a million product catalog?**

Full table scan kills performance, no fuzzy/partial matching, no relevance ranking. Users get slow results in random order.

**When NOT to use Elasticsearch?**

Small datasets where SQL is fast enough. Simple exact-match search where relevance ranking isn't needed. Don't add Elasticsearch complexity for a 10,000 product catalog.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Search infrastructure | Amazon OpenSearch Service (managed Elasticsearch) |
| CDC to OpenSearch | AWS DMS + Kinesis → Lambda → OpenSearch |
| Simple search | DynamoDB with GSI for basic lookups |
| AI-powered search | Amazon Kendra (NLP-based enterprise search) |