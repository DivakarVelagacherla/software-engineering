# Distributed Locking

## The One-Line Summary

> Distributed locking prevents race conditions when multiple servers simultaneously try to access and modify the same shared resource. One server holds the lock, others wait.
> 

---

## The Problem — Race Condition

Flash sale. 1 iPhone in stock. 3 inventory service instances check simultaneously:

```
Server 1: CHECK → 1 iPhone available ✅
Server 2: CHECK → 1 iPhone available ✅
Server 3: CHECK → 1 iPhone available ✅

All 3 create orders → oversold
```

Without a lock, all servers pass the check and write simultaneously. One iPhone sold three times.

---

## Solution 1 — Database Locking (SELECT FOR UPDATE)

True enforcement — database engine physically prevents other transactions from reading the locked row.

```sql
BEGIN;
SELECT stock FROM inventory WHERE product_id = 1 FOR UPDATE;
-- row is now locked, Server 2 is BLOCKED waiting
-- application code checks stock, creates order
UPDATE inventory SET stock = stock - 1 WHERE product_id = 1;
COMMIT;
-- lock released, Server 2 now proceeds, reads stock=0, rejects
```

**How it works in Python:**

```python
conn.autocommit = False  # manual transaction control
cursor.execute("SELECT stock ... FOR UPDATE")  # lock acquired
stock = cursor.fetchone()[0]
if stock > 0:
    cursor.execute("INSERT INTO orders ...")
    cursor.execute("UPDATE inventory SET stock = stock - 1 ...")
    conn.commit()   # lock released
else:
    conn.rollback() # lock released, no changes
```

**Key insight:** Between BEGIN and COMMIT, your application code runs on the server but the database row stays locked. Keep transactions short — long locks block other requests.

**Use when:** Entire operation fits in one database transaction. Simpler, enforced by database engine, can't be bypassed.

---

## Solution 2 — Redis Distributed Lock

Honor system — all servers agree to check Redis before proceeding. Nothing technically prevents bypassing it.

```
SET lock:iphone "server1" NX EX 10
```

- `lock:iphone` — key name (naming convention, colon just namespaces the key)
- `"server1"` — value identifying who holds the lock
- `NX` — only set if key does Not eXist (atomic — prevents two servers setting simultaneously)
- `EX 10` — expire after 10 seconds (safety net if server crashes holding lock)

**Full flow:**

```
Server 1: SET lock:iphone "server1" NX EX 10 → Success — holds lock
Server 2: SET lock:iphone "server2" NX EX 10 → Fails — waits
Server 3: SET lock:iphone "server3" NX EX 10 → Fails — waits

Server 1: checks inventory → 1 available
Server 1: creates order
Server 1: DEL lock:iphone → releases lock

Server 2: acquires lock → reads stock=0 → rejects
Server 3: acquires lock → reads stock=0 → rejects
```

**Safe release — only lock holder can release:**

```python
if redis.get("lock:iphone") == "server1":
    redis.delete("lock:iphone")
```

Prevents Server 2 from accidentally releasing Server 1's lock.

**Why expiry matters:** If Server 1 crashes while holding the lock, expiry auto-releases it. Without expiry — lock held forever, system deadlocked.

**Use when:** Operation spans multiple services or steps that can't fit in one database transaction.

---

## Database Lock vs Redis Lock

|  | Database Lock (SELECT FOR UPDATE) | Redis Distributed Lock |
| --- | --- | --- |
| **Enforcement** | Hard — database engine blocks others | Soft — honor system, servers must cooperate |
| **Scope** | Single database transaction | Any operation, any duration |
| **Complexity** | Simple | More complex |
| **Use when** | Operation fits in one DB transaction | Multi-service, multi-step operations |

---

## Three-Question Ritual

**What problem does distributed locking solve?**

Race conditions when multiple servers simultaneously check and modify the same shared resource. Without a lock, all servers pass the check and write simultaneously — overselling, double booking, data corruption.

**What breaks without it in a flash sale?**

Overselling — multiple servers all see 1 item available, all create orders. Customers get confirmation emails, one shows up with no product.

**When to use database locking instead of Redis?**

When the entire operation fits in one database transaction — simpler, enforced by the engine, can't be bypassed. Single server or low traffic where Redis overhead isn't justified.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Redis distributed lock | ElastiCache (Redis) with SET NX EX |
| Database row locking | RDS PostgreSQL SELECT FOR UPDATE |
| Managed distributed locking | AWS DynamoDB conditional writes (optimistic locking) |