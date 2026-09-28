# Geospatial Data

## The One-Line Summary

> Geospatial data handling solves finding nearby locations efficiently. Geohashing converts 2D coordinates into an indexed string so proximity queries become fast prefix lookups instead of expensive O(n) distance calculations.
> 

---

## The Problem

500,000 active Uber drivers globally. Rider requests nearest driver.

**Naive approach:**

```sql
SELECT driver_id FROM drivers
WHERE distance(lat, lng, rider_lat, rider_lng) < 10
ORDER BY distance ASC LIMIT 10;
```

Calculates distance to every driver — O(n). With 500,000 drivers and millions of simultaneous rider requests, this collapses.

---

## Geohashing

Converts 2D location (latitude, longitude) into a single string representing a geographic area. Longer string = smaller area = more precise.

```
"q"      → entire Northeast USA
"dr"     → Philadelphia region
"dr4"    → smaller Philadelphia area
"dr4b"   → neighborhood level
"dr4b2c" → building level
```

**Key insight: nearby locations share the same geohash prefix.**

Rider at `dr4b2c` → find drivers where geohash starts with `dr4b` → all within the same neighborhood.

```sql
SELECT driver_id FROM drivers
WHERE geohash LIKE 'dr4b%'
AND status = 'available';
```

String prefix index — O(log n) instead of O(n) distance calculations.

---

## Uber's Architecture

**The challenge:** 500,000 drivers updating location every second = 500,000 writes/second. No SQL database handles this.

### Driver location updates

```
Driver App → WebSocket → Location Service (ECS/Go)
                              ↓
                         Redis GEOADD (current location)
                              ↓
                         Kafka → Cassandra (location history)
```

**Why WebSocket?** One persistent connection per driver. Location update = small message on open connection. vs HTTP: 500,000 connections opened/closed every second — too expensive.

**Why Go for Location Service?** 500,000 open WebSocket connections simultaneously. Go goroutines handle this cheaply (~2KB each). Java threads (~1MB each) would require ~500GB RAM.

### Rider matching

```
Rider request → Matching Service → Redis GEORADIUS → nearest drivers
```

### Storage split

- **Redis** — current driver locations (in-memory, fast reads/writes, native geospatial commands)
- **Cassandra** — location history (persistent, write-heavy, time series)

---

## Redis Geospatial Commands

Redis has built-in geospatial support using geohashing internally:

```bash
# Add driver location
GEOADD drivers:available 75.1652 39.9526 "driver_123"

# Find drivers within 10 miles, sorted by distance
GEORADIUS drivers:available 75.1652 39.9526 10 mi ASC COUNT 10
```

Redis handles geohashing internally — you get the benefits without managing it yourself.

---

## Three-Question Ritual

**What problem does geospatial data handling solve?**

Efficiently finding nearby locations. Without geohashing, proximity queries require O(n) distance calculations across all records. Geohashing converts coordinates to an indexed string — proximity becomes a fast prefix lookup.

**What breaks without geohashing at Uber's scale?**

O(n) distance calculations on every rider request across 500,000 drivers simultaneously. Response time goes from milliseconds to seconds. Matching breaks down entirely.

**When NOT to use geohashing?**

No location-based queries. Small dataset where simple distance calculations are fast enough — 50 drivers, no need for prefix indexing.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Geospatial queries | Amazon Location Service |
| Redis geospatial | ElastiCache (Redis) with GEOADD/GEORADIUS |
| Location history storage | Amazon Keyspaces (Cassandra) or DynamoDB |
| Real-time location streaming | Amazon Kinesis Data Streams |
| WebSocket connections | API Gateway WebSocket API |