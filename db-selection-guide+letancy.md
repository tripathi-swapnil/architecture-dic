# How to Choose a Database in System Design

> A comprehensive guide to database selection — key tradeoffs, decision frameworks, and practical recommendations.

---

## 1. Core Decision Framework

The choice comes down to understanding your **data characteristics**, **access patterns**, and **scale requirements**, then matching those against the strengths and tradeoffs of different database types.

---

## 2. Data Model & Structure

The first question is: **what does your data look like?**

- **Structured with relationships** → Relational databases (PostgreSQL, MySQL) are your default. They give you ACID guarantees, joins, and a mature ecosystem. If your data naturally fits into tables with foreign keys, start here.

- **Semi-structured or flexible schema** → Document stores (MongoDB, DynamoDB) shine when your data shape varies across records or evolves frequently. Think product catalogs where each category has different attributes.

- **Highly connected data** → Graph databases (Neo4j, Amazon Neptune) are purpose-built for traversing relationships — social networks, recommendation engines, fraud detection.

- **Simple key-value lookups** → Redis, DynamoDB, or Memcached when you just need fast get/set by a key.

- **Time-ordered data** → Time-series databases (InfluxDB, TimescaleDB) for metrics, IoT sensor data, or logs.

---

## 3. The Key Tradeoffs

### Consistency vs. Availability (CAP Theorem)

You can't have perfect consistency, availability, and partition tolerance simultaneously. Relational databases lean toward consistency (CP), while many NoSQL databases lean toward availability (AP).

**Ask yourself:** *Can my application tolerate reading slightly stale data?*
- Banking says **no** → strong consistency.
- A social media feed says **yes** → eventual consistency is fine.

### Read-Heavy vs. Write-Heavy Workloads

- **Read-heavy** systems benefit from caching layers (Redis), read replicas, and denormalized schemas.
- **Write-heavy** systems need databases optimized for write throughput — Cassandra and DynamoDB handle massive write volumes well, while a single-node relational DB may bottleneck.

### Normalization vs. Denormalization

- Relational DBs encourage **normalization** (no data duplication, joins at query time).
- NoSQL often requires **denormalization** (duplicate data to avoid joins).
- **Tradeoff:** storage efficiency and write complexity vs. read performance.

### Horizontal vs. Vertical Scaling

- Relational databases traditionally scale **vertically** (bigger machine), though sharding is possible.
- NoSQL databases like Cassandra, DynamoDB, and MongoDB are designed to scale **horizontally** across many nodes.
- If you expect massive scale, this matters a lot.

### Latency vs. Durability

- Writing to disk with full ACID guarantees is slower than writing to memory.
- In-memory stores (Redis) give sub-millisecond reads but risk data loss on crash unless persistence is configured.
- **Decide:** how much durability do you need?

### Query Flexibility vs. Performance

- SQL gives you **ad-hoc query power** — you can slice your data any way you want.
- Key-value and wide-column stores restrict you to **pre-defined access patterns** but are blazingly fast for those patterns.
- If you don't know all your query patterns upfront, SQL is safer.

---

## 4. Practical Decision Checklist

When making the choice, ask yourself:

1. **What are my access patterns?** (point lookups, range queries, full-text search, graph traversals, aggregations)
2. **What's my read/write ratio?**
3. **How important is consistency?** (strong, eventual, or causal)
4. **What's my expected data volume and growth rate?**
5. **Do I need transactions across multiple records?** (if yes, lean relational or use databases with multi-document transactions)
6. **What latency is acceptable?** (p99 under 10ms? under 100ms?)
7. **Does my team already have expertise with a particular database?** (operational cost is real)

---

## 5. Quick Reference by Use Case

| Use Case | Good Fit | Why |
|---|---|---|
| E-commerce orders | PostgreSQL / MySQL | ACID transactions, relational data |
| User sessions / caching | Redis | Sub-ms latency, TTL support |
| Product catalog | MongoDB / DynamoDB | Flexible schema per category |
| Social graph | Neo4j | Relationship traversal |
| Chat messages | Cassandra | Write-heavy, time-ordered |
| Search / autocomplete | Elasticsearch | Full-text indexing |
| Metrics / monitoring | InfluxDB / TimescaleDB | Time-series optimized |
| Event sourcing / logs | Kafka + Cassandra | Append-only, high throughput |

---

## 6. Often Overlooked Tradeoffs

### Operational Complexity

Running Cassandra or Elasticsearch clusters is non-trivial. A managed service (RDS, DynamoDB, Atlas) shifts that burden but costs more. Factor in your team's ability to operate the system.

### Migration Cost

Switching databases later is painful. But don't over-engineer either. Start with what fits your current needs, and design your application layer with enough abstraction that swapping becomes feasible.

### Polyglot Persistence

Most real systems use **multiple databases**. Your user profiles might live in PostgreSQL, your cache in Redis, and your search index in Elasticsearch. That's normal and expected — just be deliberate about which data lives where.

---

## 7. TL;DR

> **Start with your access patterns and consistency requirements**, not with the technology. Once those are clear, the right database often becomes obvious. And when in doubt, **PostgreSQL** is a remarkably capable default that can handle more use cases than people give it credit for.

---

---

## 8. Understanding Access Patterns

An access pattern is simply **how your application reads from and writes to the database** — the shape of your queries, what data you need, how you find it, how often, and in what order.

### Core Access Patterns

**1. Point Lookups (Key-Based Access)**
You know the exact key and want the value. *"Get user with ID 12345."* This is the simplest and fastest pattern. Almost every database handles this well, but key-value stores (Redis, DynamoDB) are optimized specifically for this.

**2. Range Queries**
You want a slice of data based on a range. *"Get all orders placed between Jan 1 and Jan 31"* or *"Get users with age between 25 and 35."* Relational databases and sorted key-value stores (DynamoDB with sort keys, Cassandra with clustering columns) handle this well.

**3. Full Table Scans**
You need to look at every record — *"find all users who haven't logged in for 90 days"* without an index on last login. This is expensive and usually a sign you need an index or a different data model. Avoid this in production hot paths.

**4. Joins / Multi-Entity Access**
You need data from multiple related tables in one query. *"Get the order along with customer name and product details."* This is where relational databases excel. NoSQL databases typically push you to denormalize so you avoid joins entirely.

**5. Graph Traversals**
You need to walk relationships. *"Find all friends of friends who also like cricket."* Each hop is a traversal. Relational databases can do this with recursive joins, but graph databases (Neo4j) are built for it and stay performant at depth.

**6. Aggregations & Analytics**
You need computed results across many rows. *"What's the average order value per city?"* OLTP databases can do light aggregations, but for heavy analytical workloads, columnar stores (ClickHouse, BigQuery, Redshift) are far more efficient.

**7. Full-Text Search**
You need to search within text content. *"Find all articles containing 'system design tradeoffs'."* This requires inverted indexes and relevance scoring. Elasticsearch and Solr are purpose-built for this. PostgreSQL has decent built-in full-text search for simpler needs.

**8. Time-Series Access**
You always write the latest data point and query by time range. *"Get CPU usage for server X over the last 24 hours."* Data is append-heavy, rarely updated, and queried in time windows. Time-series databases (InfluxDB, TimescaleDB) optimize storage and queries for this pattern.

**9. Write-Once, Read-Many (WORM)**
Data is written once and never modified, but read frequently. Think event logs, audit trails, or published content. Append-only stores and immutable data models work well here.

**10. Write-Heavy / Append-Only**
Massive ingestion rates with reads happening later or less frequently. IoT telemetry, clickstream data, messaging systems. Cassandra, Kafka, and DynamoDB handle high write throughput well.

### Read vs. Write Patterns Combined

Most systems have a mix. The key is identifying your **hot path** — the most frequent operation:

| Pattern | Example | Typical Ratio | Good Fit |
|---|---|---|---|
| Read-heavy, point lookup | User profile page | 95% reads | Redis + PostgreSQL |
| Read-heavy, complex queries | Dashboard / reporting | 90% reads | PostgreSQL + ClickHouse |
| Write-heavy, append-only | Logging / metrics | 95% writes | Cassandra / Kafka |
| Balanced read-write | Chat application | ~50/50 | Cassandra / DynamoDB |
| Bursty writes, eventual reads | Social media posts | Burst writes, steady reads | DynamoDB + cache |

### How to Identify Your Access Patterns

Before choosing a database, list out your access patterns explicitly. For each major feature, write down:

1. **What data is being accessed?** (which entities, which fields)
2. **How is it being accessed?** (by primary key, by filter, by range, by text search)
3. **How often?** (requests per second)
4. **What latency is acceptable?** (real-time vs. batch)
5. **Is it a read or a write?**

This exercise alone will often make the database choice obvious.

---

## 9. Latency Percentiles (P50, P90, P95, P99)

### What Are Percentiles?

Percentiles describe the **distribution** of response times, not just the average. If your P99 latency is 500ms, it means **99% of requests complete in ≤ 500ms**.

| Metric | Meaning | Who It Represents |
|--------|---------|-------------------|
| **P50** (median) | 50% of requests are faster than this | The **typical** user |
| **P90** | 90% of requests are faster | Most users |
| **P95** | 95% of requests are faster | Nearly all users |
| **P99** | 99% of requests are faster | All but the worst 1% |
| **P99.9** | 99.9% of requests are faster | Extreme tail |

### Visual Intuition

```
Fast ◄──────────────────────────────────────────► Slow

     ████████████████████████░░░░░░░▒▒▒▓▓▓███
     │                       │       │   │   │
     P0                     P50    P90  P95  P99
     (best)              (median)          (tail)
```

Most requests cluster on the left (fast). The **tail** on the right is where problems hide.

### Why Not Just Use Average?

Average **hides outliers**:

```
9 requests at 100ms, 1 request at 10,000ms

Average = 1,090ms   ← looks bad overall
P50     = 100ms     ← typical user is fine
P99     = 10,000ms  ← worst case is terrible
```

The average (1,090ms) doesn't represent **any** real user experience. Percentiles give a much clearer picture.

### What Is Tail Latency?

Tail latency refers to the **high percentiles** — typically **P99 and P99.9**. It's called "tail" because it represents the **right tail of the distribution curve** — the small percentage of requests that take much longer than the rest.

- **P99 / P99.9** → tail latency (the slowest 1% or 0.1%)
- **P95** → near-tail
- **P50** → median, the "body" of the distribution — never considered tail

When someone says "we need to optimize tail latency," it means: **bring down the worst-case response times that the slowest 1% (or 0.1%) of users experience.**

### Why Tail Latency (P99/P99.9) Matters

**1. High-fan-out amplification**

If a single user request hits **100 backend services** in parallel:

```
P(at least one slow) = 1 - (0.99)^100 = 63%
```

Even with P99 = 500ms, **63% of user-facing requests** will experience that 500ms delay. This is why Amazon, Google, etc. obsess over P99.9.

**2. Your best customers are often the worst hit**

High-percentile users often have **more data** (more orders, more connections, larger accounts) — they're your most valuable customers getting the worst experience.

### Fan-Out Effect

Fan-out is when a **single user request triggers multiple parallel calls** to backend services.

**Example:** A user opens their Amazon homepage. That one request fans out to:
- Product recommendation service
- User profile service
- Cart service
- Ads service
- Inventory service

Say that's **50 parallel calls**, each with P99 = 200ms. The user's response time = **slowest of all 50 calls** (since you wait for all to return).

```
P(all 50 finish under 200ms) = (0.99)^50 = 60.5%

So ~40% of user requests will hit that 200ms tail latency.
```

| Fan-out (parallel calls) | P(at least one hits P99) |
|--------------------------|--------------------------|
| 1 | 1% |
| 10 | 9.6% |
| 50 | 39.5% |
| 100 | 63.4% |
| 500 | 99.3% |

**The more services you fan out to, the more tail latency becomes your typical latency.**

This is why:
- Google and Amazon optimize for **P99.9**, not just P99
- **Hedged requests** (send to 2 replicas, take the faster response) exist specifically for this
- Systems use **timeouts with partial results** — return what you have rather than waiting for the slowest shard

### More Fan-Out Examples

**Example 1: Netflix Home Page**
Opening the Netflix home screen fans out to:
- User profile service (who's watching)
- Continue-watching service
- Recommendations service (ML-powered)
- Trending service
- Personalized rows service (10+ rows, each a separate call)
- Billing / subscription status
- A/B test config service

```
~15 parallel calls, each P99 = 150ms
P(all under 150ms) = (0.99)^15 = 86%
→ 14% of home-page loads hit tail latency
```
**Mitigation:** Render skeleton UI, stream rows as they return, cache heavy recommendations per user.

**Example 2: Uber Ride Request**
Tapping "Request Ride" fans out to:
- User auth + payment validation
- Location / ETA service
- Driver matching (geo-index lookup)
- Surge pricing
- Fraud detection
- Promotions / coupons
- Notification service

```
~7 services, each P99 = 100ms
P(all under 100ms) = (0.99)^7 = 93%
→ 7% of ride requests hit tail
```
**Mitigation:** Parallelize where possible, hard timeouts on non-critical (promotions), degrade gracefully (skip coupon if slow).

**Example 3: E-Commerce Product Page (Amazon-style)**
Loading a single product page:
- Product details (title, price, images)
- Inventory / stock status
- Seller info + ratings
- Reviews (paginated)
- "Customers also bought" recommendations
- Q&A service
- Price history / deals
- Shipping estimate (based on user location)
- Ads / sponsored products

```
~9 parallel calls, each P99 = 200ms
P(all under 200ms) = (0.99)^9 = 91.4%
→ ~9% of page loads exceed 200ms
```
**Mitigation:** Critical path (price + buy button) is synchronous; rest are async/lazy-loaded below the fold.

**Example 4: LinkedIn Feed Load**
One feed refresh triggers:
- Feed ranking service (personalized)
- User graph service (1st/2nd degree connections)
- Post enrichment (per post: likes, comments count, reshare chain)
- Ads injection
- Notification badges
- Trending topics
- "People you may know"

```
If feed has 20 posts × 3 enrichment calls each = 60 calls + 6 top-level
~66 calls, each P99 = 50ms
P(all under 50ms) = (0.99)^66 = 51.4%
→ Nearly HALF of feed loads hit tail latency
```
**Mitigation:** Batch enrichment calls, precompute feed offline, stream posts as they're ready.

**Example 5: Distributed Query (Scatter-Gather)**
A search query across a sharded database with 1000 shards:
- Query sent to all 1000 shards in parallel
- Wait for all to respond
- Merge + rank results

```
Each shard P99 = 100ms
P(all 1000 under 100ms) = (0.99)^1000 ≈ 0.004%
→ 99.996% of queries hit at least one slow shard
```
**This is why scatter-gather is brutal.** At scale, P99 of your slowest shard becomes your P50 user latency.

**Mitigation:**
- **Tail-tolerant design:** return top-K from 95% of shards, skip stragglers
- **Hedged requests:** query each shard on 2 replicas
- **Backup requests:** if shard hasn't responded in P95 time, fire a duplicate

**Example 6: Microservices Auth Chain (Sequential, Not Parallel)**
Sometimes fan-out is sequential — latencies *add* instead of maxing:
```
Client → API Gateway → Auth Service → Authorization → Rate Limiter → App Service → DB

Each hop adds 10ms median, 50ms P99
Total P99 ≈ sum of P99s (not max), since each is sequential
→ 5 hops × 50ms = 250ms just in overhead
```
**Mitigation:** Collapse services (edge auth + rate limit at gateway), use service mesh with sidecar caching, or token validation with public keys (no network call).

**Example 7: GraphQL Resolver Explosion (N+1 Fan-Out)**
A single GraphQL query can silently fan out:
```graphql
query {
  posts(limit: 20) {         # 1 call
    author { name }          # 20 calls (one per post)
    comments { user { name } } # 20 × 5 = 100 calls
  }
}
```
```
Total: 1 + 20 + 100 = 121 calls per user request
If each call P99 = 20ms: P(all fast) = (0.99)^121 = 29.7%
→ 70% of queries hit tail
```
**Mitigation:** DataLoader (batch + dedupe), persisted queries, field-level caching.

### Fan-Out Pattern Comparison

| Pattern | Latency Math | Example |
|---|---|---|
| **Parallel fan-out** | max(call_1, call_2, ..., call_N) — tail dominates | Home page aggregation |
| **Sequential fan-out** | sum(call_1 + call_2 + ... + call_N) — each adds | Auth → authz → app → DB |
| **Scatter-gather** | max across many shards — worst-case tail | Sharded DB query |
| **Pipeline / streaming** | first-result latency, not total | Streaming LLM output |
| **N+1 (accidental fan-out)** | O(N) calls, usually a bug | GraphQL without DataLoader |

### How Latency Compounds Across Services

```
Client → Load Balancer → API Gateway → Service → Database
         +2ms            +5ms          +50ms     +20ms

Each hop ADDS latency, and tail latencies COMPOUND.
```

### Optimizing for P50 vs P99

| | Optimize P50 | Optimize P99 |
|---|---|---|
| Strategy | Caching, CDN, fast path | Eliminate variability, hedged requests |
| Cost | Low | High (need redundancy) |
| Complexity | Simple | Must handle GC pauses, retries, timeouts |

### Techniques to Reduce Tail Latency

| Technique | How It Helps |
|-----------|-------------|
| **Hedged requests** | Send same request to 2 replicas, use first response |
| **Tied requests** | Send to 2 servers, cancel the slower one when first responds |
| **Timeouts + retries** | Cap worst case, retry on fresh server |
| **Tail-tolerant design** | Return partial results if some shards are slow |
| **Request coalescing** | Batch identical in-flight requests |
| **Avoid head-of-line blocking** | Separate queues for fast/slow requests |

### Common Causes of Tail Latency

- **Garbage collection** pauses (JVM, Go)
- **Noisy neighbors** (shared infrastructure / multi-tenant)
- **Cold cache** misses
- **Lock contention** / thread starvation
- **Background tasks** (compaction, replication)
- **Network retransmissions**

### SLA / SLO / SLI — Using Percentiles

```
SLO: P99 latency < 200ms over a 30-day window
SLA: If P99 exceeds 500ms for >0.1% of minutes, customer gets credit
```

- **SLI** (Service Level Indicator): The actual measured P99
- **SLO** (Service Level Objective): The target (internal)
- **SLA** (Service Level Agreement): The contractual promise (external)

### What P99 Number Should You Actually Target?

There's no universal answer — it depends on **what kind of system** you're serving. Here are the industry rules of thumb:

#### P99 Latency Targets by System Type

| System Type | Typical P99 Target | Why |
|---|---|---|
| **In-memory cache (Redis)** | < 1–5 ms | Sub-ms is the whole point |
| **Key-value DB (DynamoDB)** | < 10 ms | Single-digit ms is the SLA promise |
| **OLTP database query** | < 50–100 ms | User-perceived instant |
| **Internal microservice API** | < 100 ms | Leaves budget for fan-out |
| **Public REST API** | < 200–300 ms | Industry standard (Stripe, Twilio) |
| **Web page full load** | < 1–2 s | Google: >3s = 53% bounce on mobile |
| **Search query (Elasticsearch)** | < 200–500 ms | Users tolerate a bit more |
| **Analytics query (OLAP)** | < 5–10 s | Dashboards, not real-time |
| **Batch / reporting** | minutes–hours | Async, no user waiting |

#### The 100ms / 1s / 10s Rule (Jakob Nielsen)

- **< 100 ms** → feels **instant**
- **< 1 s** → user stays in flow, notices delay
- **< 10 s** → user's attention breaks
- **> 10 s** → user leaves

#### Real-World SLO Examples

- **Amazon**: P99 < 100ms for product page backend — every +100ms costs 1% sales
- **Google Search**: P99 < 200ms — every +500ms cuts traffic 20%
- **Stripe API**: P99 < 250ms for charges
- **DynamoDB**: single-digit ms P99 (AWS SLA)
- **Netflix**: P99 < 100ms for playback start

#### Latency Budget Math (What You Actually Plan For)

If a user-facing request has a **500ms P99 budget**, you allocate it across the stack:

```
Network (client → edge)    :  50 ms
Load balancer / gateway    :  10 ms
Auth                       :  20 ms
App logic                  :  50 ms
DB call (or fan-out calls) : 100 ms  ← your DB budget
External API calls         : 150 ms
Rendering / serialization  :  20 ms
Buffer                     : 100 ms
────────────────────────────────────
Total                      : 500 ms
```

This is why "P99 < 50ms for a DB query" isn't arbitrary — it's what's left after every other hop takes its share.

#### Interview Answer Template

> "It depends on the system, but typical targets are: **~10ms** for a cache, **~50ms** for an OLTP DB query, **~100ms** for an internal service, **~200ms** for a public API, and **~1s** for a full web page. For a backend DB call I'd aim for **P99 < 50–100ms**, because we need budget left for network, auth, fan-out, and rendering within the user's overall 200–500ms expectation."

---

### Interview Tip

When discussing latency in system design, always mention:
1. **Which percentile** you're optimizing for (P50 vs P99)
2. **Fan-out effect** — tail latency compounds across services
3. **Specific mitigation** — hedged requests, timeouts, caching

Saying "average latency is 100ms" is a red flag. Saying "P99 is 200ms with hedged requests to handle tail latency" shows depth.

---

## 10. Interview Questions (Likely to Be Asked)

### A. Database Selection & Tradeoffs

1. **"You're designing X. Which database would you pick and why?"** — the classic opener. Always answer with access patterns first, technology second.
2. **When would you pick NoSQL over SQL?** Flexible schema, massive horizontal scale, predictable access patterns, write-heavy workloads. Be ready to defend PostgreSQL as a default.
3. **What is CAP theorem? Give a real-world example of a CP system and an AP system.**
   - CP: PostgreSQL, HBase, MongoDB (with strong consistency config)
   - AP: Cassandra, DynamoDB (eventual), Riak
4. **What is PACELC?** Extension of CAP — "if Partitioned, choose A or C; Else, choose Latency or Consistency." More honest than CAP.
5. **Explain ACID vs. BASE.** When does BASE make sense?
6. **What are the isolation levels in SQL?** Read Uncommitted → Read Committed → Repeatable Read → Serializable. What anomalies does each prevent (dirty reads, non-repeatable reads, phantom reads)?
7. **How do you handle a schema migration on a 1B-row table with zero downtime?** (online DDL, shadow tables, dual writes, backfills)
8. **When would you use a document store vs a relational DB for a product catalog?**
9. **Why not just put everything in one giant PostgreSQL?** (defend polyglot persistence — search, analytics, cache, time-series)

### B. Scaling & Sharding

10. **Vertical vs horizontal scaling — pros and cons of each.**
11. **How does sharding work? What's a good shard key?** (high cardinality, even distribution, aligned with access patterns)
12. **What is a hot partition? How do you avoid it?** (write sharding, composite keys, randomized suffixes)
13. **Consistent hashing — why is it useful in distributed databases?**
14. **What happens when you need to reshard? How do you migrate?**
15. **Read replicas vs sharding — when to use which?**
16. **What's replication lag and how do you handle it?** (read-your-writes, sticky sessions, read from primary for critical reads)

### C. Indexing & Query Performance

17. **How does a B-tree index work? Why is it the default?**
18. **B-tree vs LSM-tree — tradeoffs?** (LSM = write-optimized, used by Cassandra/RocksDB; B-tree = read-optimized, used by PostgreSQL/MySQL)
19. **What is a covering index? When does it help?**
20. **Composite index column order — does it matter? Why?** (leftmost prefix rule)
21. **When should you NOT add an index?** (write-heavy tables, low-cardinality columns, small tables)
22. **Explain query execution plan — what are `EXPLAIN ANALYZE` outputs you look for?** (seq scan vs index scan, nested loop vs hash join, row estimates)
23. **What's an index-only scan vs index scan?**

### D. Access Patterns (Section 8)

24. **Walk me through identifying access patterns for a Twitter timeline.**
25. **Fan-out on write vs fan-out on read — which would you pick for a social feed and why?**
26. **How would you design the data model for a chat app? (1-on-1 + group chat)**
27. **How do you handle range queries at scale?** (sort keys, clustering columns, pagination)
28. **What's the difference between OLTP and OLAP workloads?** When would you use both?

### E. Latency & Tail Latency (Section 9)

29. **What's the difference between P50, P99, and average latency? Why does P99 matter more?**
30. **Explain the fan-out effect on tail latency.** If a request fans out to 100 services each with P99=200ms, what's the user-facing P99?
31. **What is a hedged request? What's the cost?** (2x extra work for faster tails)
32. **What causes tail latency?** GC pauses, noisy neighbors, cold caches, lock contention, compaction.
33. **How would you reduce P99 latency for a read API?** (caching, hedged requests, request coalescing, replicas closer to users)
34. **Difference between SLA, SLO, SLI — give an example of each.**
35. **Your P50 is 20ms but P99 is 2 seconds. How do you debug this?**

### F. Consistency Models

36. **Strong vs eventual vs causal consistency — give a use case for each.**
37. **What is read-after-write consistency? How do you guarantee it in an eventually consistent system?**
38. **How does DynamoDB handle consistency?** (eventual by default, `ConsistentRead=true` for strong — costs 2x RCU)
39. **What's a quorum read/write? (R + W > N)** How does Cassandra use it?
40. **Explain two-phase commit (2PC) and why most NoSQL avoid it.**

### G. Caching

41. **Cache-aside vs write-through vs write-behind — tradeoffs?**
42. **How do you handle cache invalidation?** (the hardest problem in CS)
43. **What is the thundering herd problem? How do you prevent it?** (request coalescing, locks, probabilistic early expiration)
44. **Cache stampede vs thundering herd — are they the same?**
45. **When would you NOT use a cache?** (write-heavy, strongly consistent reads, very small datasets that fit in memory anyway)

### H. Transactions & Concurrency

46. **Optimistic vs pessimistic locking — when to use which?**
47. **What is MVCC?** (Multi-Version Concurrency Control — how PostgreSQL avoids read locks)
48. **How do distributed transactions work?** (Saga pattern, outbox pattern, 2PC tradeoffs)
49. **Explain the outbox pattern.** Why is it useful?
50. **What's idempotency? How do you design idempotent writes?**

### I. Specific DB Deep-Dives

51. **DynamoDB: partition key vs sort key — how do you design for a query that filters by 3 attributes?** (GSI, composite sort keys)
52. **Redis: what data structures does it support and when would you use each?** (strings, hashes, lists, sets, sorted sets, streams, hyperloglog)
53. **PostgreSQL: when would you use JSONB over a normalized schema?**
54. **Cassandra: why is it bad at joins and ad-hoc queries?** (data modeled per-query, wide rows optimized for sequential reads)
55. **Elasticsearch: when is it overkill?** (small datasets, structured queries — use PG full-text search instead)
56. **Kafka: is it a database?** (append-only log; can replace DB for event sourcing but not for random access)

### J. Design Scenarios (Be Ready to Architect)

57. **Design a URL shortener.** (KV store, hash collisions, analytics fan-out)
58. **Design a rate limiter.** (Redis with sliding window, token bucket)
59. **Design a leaderboard.** (Redis sorted sets — ZADD/ZRANGE)
60. **Design Instagram's feed.** (fan-out on write vs read, celebrity problem)
61. **Design a distributed counter.** (sharded counter, eventual consistency, CRDT)
62. **Design a notification system.** (queue + fanout + user preferences)
63. **Design a multi-region active-active database.** (conflict resolution, CRDT, last-write-wins tradeoffs)

### K. Curveballs / Depth Probes

64. **Your database is hitting 95% CPU. What do you check first?** (slow queries, missing indexes, lock contention, connection pool exhaustion)
65. **A query got 10x slower overnight with no code change. What happened?** (stale stats, plan regression, data volume crossed a threshold, index bloat)
66. **You have a 500GB table that's mostly cold. How do you reduce cost?** (partitioning, archival to S3, table compression, tiered storage)
67. **How would you detect and handle a hot key?** (metrics on partition access, request coalescing, key-level caching)
68. **What happens during a failover? How long should it take?** (30s–2min typical; RTO vs RPO distinction)

---

## 11. CHEAT SHEET (One-Page Recall)

### DB Type → Best Fit
| Type | Examples | Best For | Avoid When |
|---|---|---|---|
| **Relational** | PostgreSQL, MySQL | Transactions, joins, ad-hoc queries | Massive horizontal writes |
| **Document** | MongoDB, DynamoDB | Flexible schema, nested data | Complex joins, analytics |
| **Key-Value** | Redis, DynamoDB | Point lookups, caching, sessions | Complex queries |
| **Wide-Column** | Cassandra, HBase | Write-heavy, time-series, known queries | Ad-hoc queries, joins |
| **Graph** | Neo4j, Neptune | Relationship traversal (social, fraud) | Simple CRUD |
| **Time-Series** | InfluxDB, TimescaleDB | Metrics, IoT, logs | Non-temporal data |
| **Search** | Elasticsearch, Solr | Full-text, relevance ranking | Source of truth |
| **Columnar** | ClickHouse, BigQuery, Redshift | OLAP, aggregations | OLTP, single-row ops |

### CAP / PACELC Cheat Line
- **CP**: PostgreSQL, HBase, MongoDB (configured)
- **AP**: Cassandra, DynamoDB (default), Riak
- **PACELC**: "If Partitioned → A or C; Else → Latency or Consistency"

### Consistency Models (Weakest → Strongest)
Eventual → Read-your-writes → Monotonic reads → Causal → Sequential → Linearizable (Strong)

### Latency Benchmarks (Memorize These)
| Op | Latency |
|---|---|
| L1 cache | 1 ns |
| RAM | 100 ns |
| SSD random read | 100 μs |
| Network round-trip (same DC) | 0.5 ms |
| Disk seek (HDD) | 10 ms |
| Network cross-continent | 150 ms |

### P99 Fan-Out Formula
`P(slow) = 1 − (0.99)^N` where N = parallel calls. 100 calls → 63% chance of hitting tail.

### ACID Isolation Levels (Weakest → Strongest)
Read Uncommitted → Read Committed → Repeatable Read → Serializable
- Prevents: dirty / non-repeatable / phantom / all

### Sharding Rules
- Shard key = high cardinality + even distribution + matches access pattern
- Avoid: timestamp as sole shard key (hot partition), low-cardinality keys

### Cache Patterns
- **Cache-aside** (lazy): app checks cache → miss → read DB → populate
- **Write-through**: write cache + DB synchronously
- **Write-behind**: write cache, async to DB (risky)
- **Refresh-ahead**: refresh before expiry

### Tail Latency Toolkit
Hedged requests • Tied requests • Timeouts + retries • Request coalescing • Separate queues • Circuit breakers

### Quick Decision Tree
```
Need transactions + joins?         → PostgreSQL
Massive writes + known queries?    → Cassandra / DynamoDB
Sub-ms lookup, tolerate loss?      → Redis
Full-text search?                  → Elasticsearch
Relationships 3+ hops deep?        → Neo4j
Metrics / time-ranges?             → InfluxDB / Timescale
Analytics across billions of rows? → ClickHouse / BigQuery
Not sure?                          → PostgreSQL (seriously)
```

### Interview Power Phrases
- "Let me start with access patterns before picking a technology."
- "P99 matters more than average because of fan-out amplification."
- "I'd default to PostgreSQL unless I can justify the operational cost of something specialized."
- "Polyglot persistence is normal — the question is where each data type lives."
- "Strong consistency has a latency cost; can we tolerate eventual here?"
- "What's our read/write ratio and hot path?"

### Red Flags to Avoid Saying
- "MongoDB is web scale" (meme, not an answer)
- "Just add a cache" (without invalidation strategy)
- "NoSQL is faster than SQL" (depends on access pattern)
- "We'll shard later" (resharding is painful — design for it upfront)
- "Average latency is X" (always use percentiles)

---

*Generated as a system design reference guide.*
