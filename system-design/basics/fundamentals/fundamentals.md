---

## Vertical vs Horizontal Scaling

### What problem does it solve?

A single server can only handle so many requests. When traffic grows, you need a strategy to handle more load.

### The concept

- **Vertical Scaling** — make the one server bigger. More CPU, more RAM, more storage. Simple, but has a hard ceiling and requires downtime to upgrade.
- **Horizontal Scaling** — add more servers. No ceiling, just add more. Used by Amazon, Google, Netflix.

### Drawbacks of Vertical Scaling

1. **Hard ceiling** — there's only so big one machine can get
2. **Downtime** — upgrading requires taking the server offline
3. **No redundancy** — if the one server fails, everything fails (no failover)

### New problems Horizontal Scaling introduces

- **Routing** — something needs to decide which server handles each request (→ Load Balancer, Session 2)
- **State management** — servers don't share memory, so session/user state must live somewhere shared (→ Stateless vs Stateful, Session 3)

### When NOT to use Horizontal Scaling

- When load is small enough that vertical scaling is sufficient (2x–3x growth)
- At early stage — two servers are harder to manage than one; don't add complexity before you need it

### Plain-English explanation

Vertical scaling is making your one kitchen bigger. Horizontal scaling is opening more branches. Branches scale forever but require coordination — one kitchen is simpler until it can't keep up.

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | The hard limits of vertical scaling — no ceiling, no single point of failure |
| What breaks without it? | At scale, one server maxes out, goes down for upgrades, and has no backup |
| When NOT to use it? | When load is manageable with one server, or system is too early-stage to justify the complexity |

---

## Load Balancing

### What problem does it solve?

When you have multiple servers, something needs to decide which server handles each incoming request — and stop sending traffic to servers that are down.

### The concept

A **load balancer** sits in front of your servers and:

- Routes incoming requests to healthy servers
- Performs **health checks** — constantly monitoring which servers are alive
- Stops sending traffic to servers that fail health checks

**Real-world analogy:** A host at a busy restaurant — customers don't pick their own table. The host knows which tables are free, which waiters are available, and directs each customer accordingly.

### Routing Strategies

| Strategy | How it works | Problem |
| --- | --- | --- |
| **Round Robin** | Rotates through servers in order | Ignores actual load — heavy requests can pile up on one server |
| **Least Connections** | Routes to server with fewest active connections | Smarter, handles uneven request weights better |

### Load Balancer Redundancy

The load balancer itself is a single point of failure — so you need multiple load balancers:

- **Active-Passive** — one floating IP, primary handles traffic, backup on standby. Heartbeat signal between them — if primary goes silent, backup takes over the floating IP.
- **Active-Active** — DNS round robin across both load balancers. Both handle traffic simultaneously. If one fails, the other absorbs all traffic.

### Layer 4 vs Layer 7

|  | Layer 4 (NLB) | Layer 7 (ALB) |
| --- | --- | --- |
| **Operates on** | IP addresses + ports | HTTP — URLs, headers, cookies |
| **Routing intelligence** | None — just forwards packets | Content-based routing (e.g. `/api/videos` → powerful servers) |
| **Speed** | Faster, lower overhead | Slightly more overhead |
| **Use when** | Ultra-low latency needed (gaming, trading) | Most modern web applications |

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | Routes traffic across multiple servers, removes dead servers from rotation, eliminates single point of failure |
| What breaks without it? | DNS round robin has no health checks — dead servers keep getting traffic, requests fail |
| When NOT to use Layer 7? | When raw speed matters more than smart routing — use Layer 4 (NLB) instead |

---

## Stateless vs Stateful Servers

### What problem does it solve?

When multiple servers handle requests, user state (login, cart, preferences) can get trapped on one server — breaking horizontal scaling and redundancy.

### The concept

- **Stateful server** — holds user state locally in memory or on disk. Requests must return to the same server or the user loses their session.
- **Stateless server** — holds nothing locally. All state lives in an external shared store (e.g. Redis). Any server can handle any request.

### Why stateless is required for horizontal scaling

- With stateful servers, you can't freely add or remove servers — users are stuck to specific ones
- If a stateful server goes down, all users on it lose their sessions
- With stateless servers, any server can be spun up or shut down at any time — users never notice

### Naive solutions and why they fail

| Approach | Problem |
| --- | --- |
| **Sticky sessions** | Load balancer always routes user to same server. Breaks when that server goes down. Creates uneven load. |
| **Servers sharing state directly** | Complex, inefficient, doesn't scale |

### The real solution

Store session data in a **shared external store** — Redis is the standard choice.

- Any server reads/writes session data from Redis
- Servers become identical and interchangeable
- Scale up or down freely without affecting users

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | User state trapped on one server breaks horizontal scaling and redundancy |
| What breaks without it? | Servers go down and users lose sessions; can't freely scale horizontally |
| When NOT to use external session store? | Small scale where sticky sessions is acceptable; truly stateless APIs with no user sessions at all |

---

## DNS (Domain Name System)

### What problem does it solve?

Humans remember names, not IP addresses. DNS translates human-readable domain names (www.google.com) into IP addresses that computers use to route requests.

### The lookup chain

When you type `www.google.com`:

1. **DNS Resolver** (run by your ISP or a service like Cloudflare 1.1.1.1 / Google 8.8.8.8) checks its cache
    - Cache hit → returns IP immediately
    - Cache miss → starts the full lookup
2. Resolver asks **Root DNS servers** → returns address of the `.com` TLD server
3. Resolver asks **`.com` TLD server** → returns address of Google's authoritative server
4. Resolver asks **Google's Authoritative DNS server** → returns the actual IP address
5. Resolver **caches the result** with the TTL specified by Google
6. Resolver returns the IP to your browser
7. Browser connects to that IP

### TTL — Time To Live

Every DNS record has a TTL — how long resolvers should cache it before fetching a fresh copy.

| TTL | Tradeoff |
| --- | --- |
| **Long TTL** (24hrs+) | Faster lookups, less DNS traffic. Stale records persist longer after IP changes. |
| **Short TTL** (5 mins) | Records update quickly after changes. More DNS lookups, more load on servers. |

**Rule of thumb:**

- Normal operations → long TTL (1–24 hours)
- 1–2 days before a planned migration → shorten to 5 minutes so the world picks up the new IP quickly
- After migration is stable → set TTL back to long

### Who sets the TTL?

The **domain owner** sets TTL on their authoritative DNS server — not Cloudflare or the resolver. Resolvers just read and obey whatever TTL the authoritative server specifies.

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | Translates human-readable domain names to IP addresses so humans don't need to remember IPs |
| What breaks without caching/TTL? | Authoritative servers get overwhelmed with every lookup; stale cached records can't be corrected after IP changes |
| When to use short TTL? | 1–2 days before a planned server migration so DNS propagates the new IP quickly |

---

## CDN (Content Delivery Network)

### What problem does it solve?

Serving static content from a single origin server to users worldwide creates high latency and overwhelms the origin. A CDN caches static content on edge servers distributed globally, serving users from a location physically close to them.

### The concept

A CDN is a network of **edge servers (Points of Presence / PoPs)** spread around the world. Static content is cached at these edge servers so users get fast local delivery instead of traveling to the origin.

**Analogy:** Publisher in New York = your origin server. Local bookstores worldwide = CDN edge servers. Same content, local delivery.

### What belongs on a CDN

| CDN (static) | Origin server (dynamic) |
| --- | --- |
| Images, videos, fonts | User profiles |
| CSS, JavaScript files | Real-time prices |
| HTML templates | Personalized feeds |

### Pull vs Push

|  | Pull | Push |
| --- | --- | --- |
| **How it works** | Edge fetches from origin on first request | You proactively upload to all edge servers |
| **First request** | Slow — cache miss | Fast — already cached |
| **Storage** | Only caches what's requested | Stores everything, even unused content |
| **Best for** | Frequently requested content | Large files guaranteed to be needed (e.g. Netflix top 10) |

**In practice:** Hybrid — push popular/guaranteed content, pull long-tail content on demand.

### TTL and Cache Invalidation

- CDN content has a **TTL** — after expiry, edge server fetches fresh copy from origin
- **Cache invalidation** — forcing CDN to forget a file before TTL expires
    - **Purge** — manually delete file from all edge servers immediately
    - **Version filenames** — `style.v2.css` instead of `style.css` — new name = automatic cache miss = fresh fetch

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | High latency and origin server overload when serving static content to global users |
| What breaks without it? | Users far from origin get slow load times; origin server gets overwhelmed serving the same static files repeatedly |
| When NOT to use it? | Mostly dynamic content with no static assets to cache; small scale with local users where CDN cost isn't justified |

---

## Caching Fundamentals

### What problem does it solve?

Repeated identical database queries are slow and expensive. A cache stores frequently accessed data in fast memory so the database isn't hit every time.

### The concept

A **cache** is a fast, temporary storage layer that sits between your application and your database, holding frequently accessed data in RAM.

**Why it's faster than a database:**

- Traditional database → stores data on **disk** → slow, milliseconds
- Cache (Redis) → stores data in **RAM** → fast, microseconds → ~100x faster

The cache is not a replacement for the database — it's a fast layer in front of it. Full dataset lives on disk; hot data lives in RAM.

### Cache Hit vs Cache Miss

|  | What it means | What happens |
| --- | --- | --- |
| **Cache hit** | Data IS in the cache | Returned immediately, no database call |
| **Cache miss** | Data is NOT in the cache | Fetch from database, store in cache, return to user |

### Why not cache everything locally per server?

Local in-memory cache on each server = stateful servers. Each server has its own copy, they go out of sync. Same problem as stateful servers from Session 3.

**Solution:** Centralized shared cache (Redis) — same pattern as shared session storage. All servers read from and write to one place.

### The tradeoff

RAM is expensive and limited. You can't store everything in cache. Only store hot, frequently accessed data. Everything else stays in the database.

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | Reduces repeated expensive database calls for the same data — speeds up response times and reduces database load |
| What breaks without it? | Database gets overwhelmed with repeated identical queries, becomes the bottleneck under heavy traffic |
| When NOT to use it? | Queries are random with no repetition (low hit rate, cache adds cost with no benefit); data changes so frequently that cached values are always stale |

---

## Cache Strategies

### What problem does it solve?

Different systems have different needs — some prioritize read speed, some write speed, some consistency, some durability. Cache strategies define how your app reads from and writes to the cache and database to balance these tradeoffs.

### The Four Strategies

#### 1. Cache-Aside (Lazy Loading)

Application code manages the cache manually.

- **Read:** Check cache → hit: return data. Miss: fetch from DB → store in cache → return data
- **Write:** Write to DB → invalidate cache entry (delete stale data, let next read repopulate)
- **Best for:** Read-heavy workloads, general purpose
- **Tradeoff:** Cache is empty on cold start; developer must remember to manage cache correctly

#### 2. Write-Through

Every write goes to cache AND database simultaneously.

- **Read:** Same as cache-aside
- **Write:** Write to cache AND database at the same time
- **Best for:** Systems where read speed matters and data must always be fresh
- **Tradeoff:** Wasted writes — data cached that may never be read again; slower writes

#### 3. Write-Behind (Write-Back)

Write to cache first, return success to user immediately. Database updated asynchronously in background.

- **Write:** Write to cache → return success → background process flushes to DB
- **Best for:** Write-heavy systems where data loss is acceptable (like counts, view counts, analytics)
- **Tradeoff:** If cache crashes before flush, data is permanently lost. Cache and DB temporarily inconsistent.

#### 4. Read-Through

Cache sits in front of database and manages itself. App only talks to cache.

- **Read:** App requests from cache → hit: return data. Miss: cache fetches from DB, stores it, returns it
- **Best for:** Simplifying application code, teams where cache management shouldn't be the app's concern
- **Tradeoff:** Less flexible than cache-aside; cache controls what gets stored

### Quick Comparison

| Strategy | Who manages cache? | Write speed | Risk |
| --- | --- | --- | --- |
| Cache-aside | Application | Normal | Stale data on writes |
| Write-through | Application | Slower (double write) | Wasted cache memory |
| Write-behind | Cache (async) | Fastest | Data loss on crash |
| Read-through | Cache (auto) | Normal | Less control |

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | Defines how to balance speed, consistency, and durability between cache and database for different workloads |
| What breaks with wrong strategy? | Data loss (write-behind in critical systems), wasted memory (write-through everywhere), stale data (no invalidation on writes) |
| When to use write-behind? | When data loss is acceptable and write speed is critical — like counts, view counts, analytics, shopping cart |

---

## Cache Eviction Policies

### What problem does it solve?

RAM is limited and expensive. When the cache is full and a new item needs to be stored, something has to be removed. Eviction policies decide which item gets evicted to make room.

### The Three Policies

#### LRU — Least Recently Used

Evict the item that hasn't been accessed for the longest time.

- **Assumption:** Recent access = likely to be accessed again soon
- **Best for:** Most web applications — recent usage is the best predictor of future usage
- **Tradeoff:** Every read must update the access timestamp — overhead at high throughput. Redis uses a randomized approximation rather than true LRU for efficiency.

#### LFU — Least Frequently Used

Evict the item that has been accessed the fewest times overall.

- **Assumption:** Consistently popular items should stay in cache regardless of recent access
- **Best for:** Consistently popular data over long periods — homepages, top products, reference data
- **Tradeoff:** Can get "stuck" on historically popular data that's no longer relevant (e.g. a viral news article from last week)

#### FIFO — First In First Out

Evict whatever has been in the cache the longest, regardless of access patterns.

- **Simple to implement but blind to usage** — most popular item gets evicted just because it was cached first
- **Rarely the right choice** for production systems

### LRU vs LFU — When to pick which

| Scenario | Better choice |
| --- | --- |
| General web app, social feeds, user sessions | LRU |
| Consistently popular data (homepage, top 10 products) | LFU |
| Burst traffic (viral content, breaking news) | LRU — LFU gets stuck on stale burst data |
| Long-term stable popularity | LFU |

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | Cache is full — decides which item to remove to make room for new data |
| What breaks without it? | Cache runs out of memory, crashes or rejects new entries, hit rate drops, database gets overwhelmed |
| When to use LFU over LRU? | When data has consistently high popularity over long periods, not just recent or burst traffic |

---

## Redis Deep Dive

### What problem does it solve?

Redis goes beyond simple caching — it provides rich data structures, persistence, and replication, moving complexity out of application code and into the data layer.

### Redis Data Types (what matters for system design)

| Data Type | What it stores | Use case |
| --- | --- | --- |
| **String** | Any single value — text, number, serialized JSON | Simple caching, counters |
| **Hash** | Multiple field-value pairs under one key | User profiles — update individual fields without rewriting the whole object |
| **List** | Ordered sequence, push/pop from either end | Activity feeds, queues |
| **Sorted Set** | Unique members each with a score, auto-sorted | Leaderboards, priority queues, rate limiting |
| **Set** | Unique unordered values | Unique visitor counts, tags |

**String vs Hash — key difference:**

- String = sealed envelope. Fetch whole thing, update in app code, write back. 4 steps.
- Hash = folder with separate documents. Update one field with one command. 1 step.

### TTL in Redis

Same concept as DNS/CDN — keys automatically expire and get deleted after a set time. TTL alone doesn't solve staleness when data changes unpredictably → see Cache Invalidation (Session 10).

### Redis Persistence

RAM is volatile — Redis crashes = data gone. Two mechanisms to persist to disk:

|  | RDB (Snapshot) | AOF (Append Only File) |
| --- | --- | --- |
| **How** | Periodic full snapshot to disk | Every write logged to disk immediately |
| **Data loss risk** | Everything since last snapshot | Milliseconds at most |
| **Speed** | Faster | Slower (every write hits disk) |
| **Best for** | Cache data — loss acceptable | Session data — loss unacceptable |

**Production:** Often use both — AOF for durability, RDB for fast restores.

### Redis vs Memcached

|  | Redis | Memcached |
| --- | --- | --- |
| **Data types** | String, Hash, List, Set, Sorted Set | String only |
| **Persistence** | RDB + AOF | None |
| **Replication** | Yes | No |
| **Use when** | Almost always | Pure simple caching, legacy systems |

**Bottom line:** Redis covers everything Memcached does, plus more. Default to Redis. Memcached is largely legacy.

### AWS equivalent

**ElastiCache** — managed Redis or Memcached. AWS handles scaling, patching, and failover.

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve beyond caching? | Rich data structures move complexity out of app code — Hashes for objects, Sorted Sets for leaderboards, Lists for queues |
| What breaks if Redis crashes with no persistence? | All data lost — cache misses flood the database, sessions lost, leaderboards gone |
| When to choose Memcached? | Pure simple string caching only — but Redis covers this too, so rarely a compelling reason |

---

## Cache Invalidation

### What problem does it solve?

When source data changes in the database, the cache can serve stale data to users. Cache invalidation ensures the cache stays consistent with the database when data changes.

> *"There are only two hard things in computer science: cache invalidation and naming things."*
> 

### Why TTL alone isn't enough

TTL is too blunt — you can't always predict when data will change. A 24-hour TTL on a product price means users see the wrong price for up to 24 hours after a change.

### The two main strategies

#### 1. TTL-Based Expiration

Let the cache expire naturally. Stale data exists until TTL runs out, then fresh data is fetched from the database.

- **Simple** — no extra infrastructure
- **Risk** — stale window can be long
- **Use when** — low stakes, slowly changing data (profile pictures, article descriptions, usernames)

#### 2. Event-Driven Invalidation

When data changes, immediately tell the cache.

**Option A — Application level (most common):**

When app writes to database → immediately delete or update cache entry.

- Simpler, fewer moving parts
- Application owns the write, so it owns the invalidation

**Option B — Database trigger:**

Database detects change → fires event → listener invalidates cache.

- More reliable for complex systems
- Adds a third component to manage

**Production pattern — queue as fallback:**

If Redis is down when invalidation is attempted → put invalidation event in a message queue → consumer processes when Redis recovers. Decoupled and failure-tolerant.

### Delete vs Update on write

|  | Delete cache entry | Update cache entry |
| --- | --- | --- |
| **Next read** | Cache miss → fetches fresh from DB | Cache hit → served immediately |
| **Race condition risk** | None — delete is safe | Yes — concurrent writes can leave cache out of sync with DB |
| **Correctness** | Always correct | Can get stale if writes overlap |

**Bottom line:** Delete is safer than update. One extra DB read on next request is worth guaranteed correctness.

### When to use which

| Scenario | Strategy |
| --- | --- |
| Product prices, inventory | Event-driven + delete on write |
| Authentication tokens | Event-driven + delete on write |
| User profile pictures | TTL-based |
| Article view counts | TTL-based |
| Username, bio | TTL-based |

### Database first, cache second

Always update the database first — it's the source of truth. Cache is secondary. If Redis is down, stale cache is acceptable. Stale database is not.

### Three-Question Ritual

| Question | Answer |
| --- | --- |
| What problem does it solve? | Ensures cache stays consistent with database when source data changes |
| What breaks without it? | Users see stale data — wrong prices, outdated inventory, old profile info. Cache lies to the user. |
| When to accept TTL-based? | Low stakes, slowly changing data where brief staleness causes no real harm |

---

## Terms Worth Knowing

Use these naturally in sentences during interviews. Don't recite definitions — use them in context.

### Reliability & Failure

| Term | Plain English |
| --- | --- |
| **Fault tolerant** | System keeps working even when parts fail |
| **Resilient** | System recovers quickly from failures |
| **Redundancy** | Duplicate components so one failure doesn't stop everything |
| **Failover** | Automatically switching to a backup when primary fails |
| **Single point of failure** | One component whose failure brings down the whole system |
| **Graceful degradation** | System works at reduced capacity rather than failing completely |
| **Race condition** | Two operations happening simultaneously producing unpredictable results |
| **Deadlock** | Two processes each waiting for the other, neither can proceed |

### Performance

| Term | Plain English |
| --- | --- |
| **Latency** | Time taken for one request to complete |
| **Throughput** | Number of requests handled per second |
| **Bottleneck** | The slowest part of the system that limits overall performance |
| **Hot spot** | One server or resource getting disproportionately more traffic than others |
| **Cache hit rate** | Percentage of requests served from cache vs database |

### Data

| Term | Plain English |
| --- | --- |
| **Consistency** | All users see the same data at the same time |
| **Durability** | Data survives crashes and restarts |
| **Availability** | System is up and responding to requests |
| **Persistence** | Data survives beyond the process that created it |
| **Idempotency** | Performing the same operation multiple times produces the same result |
| **Atomicity** | Operation either completes fully or not at all, never halfway |
| **Stale data** | Cached data that no longer matches the source of truth |
| **Source of truth** | The authoritative, definitive copy of data |

### Scaling

| Term | Plain English |
| --- | --- |
| **Horizontal scaling** | Adding more machines |
| **Vertical scaling** | Making one machine bigger |
| **Stateless** | Server holds no user data between requests |
| **Stateful** | Server remembers user data between requests |
| **Sharding** | Splitting data across multiple databases |
| **Replication** | Copying data across multiple databases |
| **Partitioning** | Dividing data into smaller chunks |

### System Design

| Term | Plain English |
| --- | --- |
| **Tradeoff** | Gaining one thing at the cost of another |
| **Overhead** | Extra cost (time, memory, compute) of doing something |
| **Coupling** | How dependent two components are on each other |
| **Decoupled** | Components that operate independently of each other |
| **Asynchronous** | Operations that don't wait for each other to complete |
| **Synchronous** | Operations that wait for each other to complete |
| **Eventual consistency** | Data will be consistent across all nodes, but not immediately |
| **Strong consistency** | Data is immediately consistent across all nodes |

### Communication

| Term | Plain English |
| --- | --- |
| **Payload** | The actual data being sent in a request |
| **Polling** | Client repeatedly asking server for updates |
| **Event driven** | Components react to events rather than being directly called |

---

> **How to use these:** Instead of "if the server breaks, the other server takes over" — say "we need failover so that if the primary goes down, the replica takes over automatically — otherwise we have a single point of failure." Same idea, completely different signal to the interviewer.
>