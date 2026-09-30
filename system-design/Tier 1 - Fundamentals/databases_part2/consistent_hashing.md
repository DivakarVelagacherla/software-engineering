# Consistent Hashing

## The One-Line Summary

> Consistent hashing ensures that when you add or remove a server, only 1/n keys need to move — instead of almost everything moving with simple modulo hashing.
> 

---

## The Problem With Simple Modulo Hashing

With 4 shards using `user_id % 4`, adding a 5th shard changes the formula to `user_id % 5`. Almost **80% of all keys** need to move to a different shard. With billions of rows, this means:

- Massive data movement across servers
- Heavy database load during migration
- Potential downtime
- Hours or days of migration work

Consistent hashing solves this.

---

## The Hash Ring

Imagine a clock face — a circle with positions 0 to 360 degrees.

1. Place each server at a position on the ring based on a hash of its name/IP
2. Place each user/key at a position based on a hash of their ID
3. To find which server owns a key — go **clockwise** from the key's position until you hit a server

```
        Server A (12)
    /                \
Server D (9)      Server B (3)
    \                /
        Server C (6)
```

- User at 1 o'clock → clockwise → Server B at 3
- User at 4 o'clock → clockwise → Server C at 6
- User at 10 o'clock → clockwise → Server A at 12

---

## Adding a Server — Only Neighbors Are Affected

Add Server E at 2 o'clock:

- Only users between 12 and 2 o'clock are affected — they now hit Server E instead of Server B
- All other users — untouched, still on the same server

**Result:** Only ~1/n keys move when adding a server (where n = number of servers). With 5 servers, ~20% move. Compare to 80% with modulo hashing.

---

## Virtual Nodes — Solving Uneven Distribution

**Problem:** Servers may cluster together on the ring, leaving large gaps. Users in those gaps all pile onto one server — a hot spot.

**Solution:** Place each server multiple times on the ring at different positions. Each placement is a **virtual node**.

- Server A at 12, 4, and 8 o'clock
- Server B at 1, 5, and 9 o'clock
- Server C at 2, 6, and 10 o'clock
- Server D at 3, 7, and 11 o'clock

Now servers are spread evenly. No gaps, no hot spots.

**Tradeoff:** More complexity — the system must maintain a mapping of virtual nodes to real servers. More memory for the ring data structure.

**In practice:** 150–200 virtual nodes per server gives nearly perfect distribution.

---

## When to Use Consistent Hashing vs Simple Modulo

| Scenario | Use |
| --- | --- |
| Cluster size is fixed, no new servers planned | Simple modulo hashing — simpler |
| Cluster will grow or shrink dynamically | Consistent hashing — minimizes data movement |

---

## Real World Context

You rarely implement consistent hashing manually. Databases like **Cassandra, DynamoDB, and Redis Cluster** use consistent hashing internally. When you say “I'll use DynamoDB” in an interview and the interviewer asks “how does it scale?” — consistent hashing is the answer.

---

## Three-Question Ritual

**What problem does consistent hashing solve?**

With simple modulo hashing, adding or removing a server causes ~80% of all keys to move to different shards. Consistent hashing ensures only 1/n keys move, minimizing data migration.

**What breaks without it?**

Adding a server triggers massive data movement — heavy load, potential downtime, hours of migration at scale.

**When NOT to use it?**

When cluster size is fixed and you know you won't be adding or removing servers. Simple modulo hashing is easier to reason about and implement.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Consistent hashing under the hood | Amazon DynamoDB, Amazon ElastiCache (Redis Cluster) |
| Cassandra with consistent hashing | Amazon Keyspaces |