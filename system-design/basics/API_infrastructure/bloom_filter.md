# Bloom Filter

## The One-Line Summary

> A Bloom Filter is a memory-efficient probabilistic data structure that answers "has this item been seen before?" It eliminates expensive database lookups for the "definitely not exists" case.
> 

---

## The Problem

URL shortener with 10 billion existing URLs. Need to check if a new ID is unique before storing it.

- **Database lookup** — too slow at scale, database becomes bottleneck
- **Cache all IDs in Redis** — ~480GB of memory, too expensive

Need something fast, memory-efficient, that answers "does this exist?"

---

## How a Bloom Filter Works

A fixed-size **bit array** (all zeros to start) + multiple hash functions.

### Adding an item

1. Run item through 3 hash functions
2. Each returns a position in the bit array
3. Flip those positions to 1

### Checking if item exists

1. Run item through same 3 hash functions
2. Check those positions
3. **Any position is 0** — item definitely does NOT exist (100% certain)
4. **All positions are 1** — item probably exists (verify with database)

### Concrete example — 10-bit array

```
Start:        0 0 0 0 0 0 0 0 0 0

Add "abc123" → hash positions 2, 5, 8:
              0 0 1 0 0 1 0 0 1 0

Add "xyz789" → hash positions 1, 5, 7:
              0 1 1 0 0 1 0 1 1 0

Check "def456" → hash positions 3, 6, 9:
  Position 3 = 0 → DEFINITELY NOT EXISTS ✔ No database lookup needed

Check "ghi789" → hash positions 1, 5, 7:
  All = 1 → PROBABLY EXISTS → verify with database
  (False positive — ghi789 was never added, but positions were set by other IDs)
```

---

## False Positives vs False Negatives

|  | Bloom Filter says "exists" | Bloom Filter says "doesn't exist" |
| --- | --- | --- |
| **Item actually exists** | True positive ✔ | Never happens ✔ |
| **Item doesn't exist** | False positive ✘ (possible) | True negative ✔ |
- **False positives:** Possible — hash positions happened to be ON from other items
- **False negatives:** Impossible — if item was added, its positions are always ON

**The guarantee:** "Definitely not exists" is always correct. "Probably exists" needs database verification.

---

## Sizing — Bits vs False Positive Rate

More bits per item = lower false positive rate = more memory.

| Bits per item | False positive rate |
| --- | --- |
| 7 bits | ~5% |
| 10 bits | ~1% |
| 20 bits | ~0.1% |

**For 10 billion IDs at 1% false positive rate:**

- Bloom Filter: 10B × 10 bits = ~12GB
- Storing actual IDs: ~480GB
- **40x more memory efficient**

**You cannot easily resize a Bloom Filter.** Solutions for growth:

- **Overprovision upfront** — size for maximum expected IDs. Most common approach.
- **Scalable Bloom Filters** — chain multiple filters, add new one when first gets full
- **Rebuild periodically** — create larger filter from scratch, switch over

---

## Implementation — It's a Library, Not a Service

Bloom Filter is a data structure in your application code, not a separate deployed component.

**Options:**

- **In-memory library** — Python `bloom-filter`, Java Guava `BloomFilter`
- **Redis module (RedisBloom)** — most common in production, shared across all service instances

```bash
# Redis commands
BF.ADD myfilter "abc123"         # add item
BF.EXISTS myfilter "abc123"      # 1 = probably exists, 0 = definitely not
```

---

## Real World Use Cases

| Company/System | How they use it |
| --- | --- |
| **Google Chrome** | Check if URL is malicious before hitting safe browsing database |
| **Cassandra** | Check if key exists in SSTable before disk read |
| **Medium** | Track which articles user has read to avoid repeat recommendations |
| **Spam filters** | Check if email is in known spam list |
| **URL shorteners** | Check if generated ID is unique |

**The pattern:** Eliminate expensive lookups for the "definitely not" case, which is the majority.

---

## When NOT to Use a Bloom Filter

- **Small datasets** — fits in Redis or database index, no need for complexity
- **Zero false positives required** — checking username availability, financial records. False positive means telling user "username taken" when it's actually free.

---

## Three-Question Ritual

**What problem does a Bloom Filter solve?**

Expensive database lookups for uniqueness checks at massive scale. Eliminates lookups for the "definitely not exists" case using a fraction of the memory required to cache actual data.

**What breaks without it at 10 billion rows?**

Every uniqueness check hits the database. Thousands of checks per second become a bottleneck. Latency spikes, database overwhelmed, costs explode.

**When NOT to use it?**

Small datasets where a database index or Redis cache suffices. Systems where false positives are unacceptable — any check where a wrong "exists" answer causes real harm.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Redis Bloom Filter | ElastiCache with RedisBloom module |
| Approximate membership checking | Amazon DynamoDB (no native Bloom Filter, implement in app layer) |