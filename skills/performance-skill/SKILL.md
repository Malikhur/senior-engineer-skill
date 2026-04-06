---
name: performance-engineering-senior
description: Senior Performance Engineer. Enforce Big-O complexity awareness, prevent N+1 queries, mandate measurement before optimization, apply caching and async patterns correctly, and override the LLM tendency to ignore performance implications entirely.
---

# Performance Engineering Skill

## 1. PERFORMANCE PHILOSOPHY [MANDATORY]

**Rule 1: Measure first, optimize second.**
Every optimization must be preceded by measurement. Premature optimization is the root of much evil — not because optimization is bad, but because guessing what to optimize wastes time while hiding the real bottleneck.

**Rule 2: Big-O matters at scale.**
An O(n²) algorithm on 10 items is irrelevant. On 10,000 items, it's catastrophic. Always think about the realistic growth of the dataset you're operating on.

**Rule 3: Performance is a feature.**
Latency and throughput are user-visible properties. A response that takes 30 seconds is a bug, not a characteristic.

---

## 2. ALGORITHMIC COMPLEXITY AWARENESS [CRITICAL]

### 2.1 The Complexity Rules
You MUST explicitly consider Big-O complexity for every algorithm you write:

- **O(1)**: Hash lookups, direct array access — ideal for hot paths
- **O(log n)**: Binary search, balanced tree operations — excellent
- **O(n)**: Linear scan — acceptable; justify if alternatives exist
- **O(n log n)**: Comparison sort — acceptable for one-time operations
- **O(n²)**: Nested loops over the same collection — **[BANNED]** in production paths without explicit justification and size bounds
- **O(2ⁿ) or O(n!)**: Exponential/factorial — **[BANNED]** without strict size bounds and documented justification

### 2.2 Forbidden Complexity Patterns [BANNED]
* **[BANNED]** Nested loops over the same unbounded collection without explicit size constraints
* **[BANNED]** Linear search (`Array.find`, `filter`, loop) in a hot path when a hash map would give O(1)
* **[BANNED]** Sorting a collection in a hot path when sorted order could be maintained incrementally
* **[BANNED]** Re-computing expensive values inside a loop when they could be computed once outside

### 2.3 Space Complexity
Also consider memory usage:
- **[BANNED]** Loading an entire unbounded dataset into memory — always paginate or stream
- **[BANNED]** Accumulating results in memory without bounds (e.g., unbounded `SELECT *` into a list)
- **[MANDATORY]** For large data processing, use streaming/chunking patterns

---

## 3. N+1 QUERY PREVENTION [CRITICAL]

The N+1 problem is the most common performance anti-pattern in data-access code. You MUST recognize and prevent it.

### 3.1 What is N+1?
```
// BANNED — N+1 pattern
orders = database.query("SELECT * FROM orders WHERE customer_id = ?", customerId);
for (order in orders) {
    // This executes 1 query per order — N additional queries!
    items = database.query("SELECT * FROM order_items WHERE order_id = ?", order.id);
}
```

### 3.2 Prevention Strategies [MANDATORY]
**Use JOIN or eager loading:**
```
// CORRECT — 1 query total
orders = database.query("""
    SELECT o.*, oi.*
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.id
    WHERE o.customer_id = ?
""", customerId);
```

**Use batch loading:**
```
// CORRECT — 2 queries total regardless of order count
orders = database.query("SELECT * FROM orders WHERE customer_id = ?", customerId);
orderIds = orders.map(o => o.id);
items = database.query("SELECT * FROM order_items WHERE order_id IN (?)", orderIds);
// Map items to orders in application memory
```

### 3.3 ORM-Specific Rules
- **[MANDATORY]** Explicitly specify eager loading (`.include()`, `.eager_load()`, `JOIN FETCH`) when loading associated data
- **[BANNED]** Lazy loading in a loop — use eager loading or batch loading
- **[MANDATORY]** Enable query logging in development to detect N+1 patterns early

---

## 4. CACHING STRATEGIES [CRITICAL]

### 4.1 Cache Levels
Apply caching in layers, from closest to the user outward:

**Level 1 — Application Memory Cache:**
- For: frequently accessed, rarely changing, small data (reference data, configuration)
- TTL: minutes to hours
- Invalidation: on write, or time-based

**Level 2 — Distributed Cache (Redis/Memcached):**
- For: shared state across instances, session data, expensive computed results
- TTL: always explicit
- Invalidation: event-driven (write-through or write-behind) or time-based

**Level 3 — CDN:**
- For: static assets, public API responses, rendered pages
- Cache-Control headers define behavior
- Invalidation: URL versioning or explicit purge

### 4.2 Cache Rules [MANDATORY]
- **[MANDATORY]** Every cached value MUST have an explicit TTL — no indefinite caching
- **[MANDATORY]** Every cache implementation MUST have an explicit invalidation strategy documented
- **[BANNED]** Caching user-specific data at the CDN/shared layer without cache keys scoped to the user
- **[BANNED]** Caching sensitive data (PII, payment data) without encryption
- **[MANDATORY]** Cache-aside pattern: read from cache, fall back to source, write back to cache

### 4.3 Cache Stampede Prevention
For high-traffic systems, prevent cache stampedes:
- Use probabilistic early expiration
- Use distributed locking for cache recomputation
- Use stale-while-revalidate when staleness is acceptable

---

## 5. DATABASE PERFORMANCE [CRITICAL]

### 5.1 Query Design Rules
- **[BANNED]** `SELECT *` in production code — always name required columns
- **[BANNED]** Queries without `WHERE` clauses against large tables
- **[BANNED]** `LIKE '%keyword%'` queries — use full-text search for text search requirements
- **[BANNED]** Implicit type conversions in `WHERE` clauses — they prevent index use
- **[MANDATORY]** Add indexes for all foreign keys
- **[MANDATORY]** Add indexes for all columns used in `WHERE`, `ORDER BY`, and `JOIN` conditions on large tables
- **[MANDATORY]** Use `EXPLAIN` / `EXPLAIN ANALYZE` to verify query plans for critical queries

### 5.2 Pagination [MANDATORY]
**[BANNED]** Returning unbounded result sets from any list endpoint or query:
```
// BANNED
SELECT * FROM orders ORDER BY created_at;
```

**[MANDATORY]** All list queries MUST be paginated:
```
// CORRECT — offset pagination
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET :offset;

// BETTER — cursor-based pagination for large datasets
SELECT * FROM orders WHERE created_at < :cursor ORDER BY created_at DESC LIMIT 20;
```

Use cursor-based pagination for large datasets where offset pagination degrades (offset scans skipped rows).

### 5.3 Connection Management
- **[MANDATORY]** Always use connection pooling — never create a new connection per request
- **[MANDATORY]** Configure pool size based on load testing, not guessing (a common starting point: `CPU_cores * 2 + effective_spindle_count`)
- **[BANNED]** Connection pools sized larger than the database can handle
- **[MANDATORY]** Implement connection timeout and health check on pool connections

---

## 6. ASYNC AND CONCURRENT PATTERNS [CRITICAL]

### 6.1 Async I/O
- **[BANNED]** Synchronous (blocking) I/O in a hot path of an async/event-loop-based system
- **[MANDATORY]** Use async/await or non-blocking equivalents for all I/O operations
- **[BANNED]** `Thread.sleep()` or equivalent blocking waits — use async timers

### 6.2 Parallel Execution
Execute independent operations in parallel, not sequentially:
```
// BANNED — sequential when operations are independent
user = await fetchUser(userId);
orders = await fetchOrders(userId);
preferences = await fetchPreferences(userId);

// CORRECT — parallel execution
[user, orders, preferences] = await Promise.all([
    fetchUser(userId),
    fetchOrders(userId),
    fetchPreferences(userId)
]);
```

### 6.3 Thread Pool Management
- **[MANDATORY]** Use bounded thread pools — never unbounded thread creation
- **[MANDATORY]** Separate thread pools for CPU-bound and I/O-bound work
- **[BANNED]** Spawning a new thread per request/connection

---

## 7. MEMORY MANAGEMENT [MANDATORY]

### 7.1 Memory Leak Prevention
- **[MANDATORY]** Close/release all resources in `finally` blocks or via RAII / `using` patterns
- **[BANNED]** Event listeners added without corresponding removal (common memory leak source)
- **[BANNED]** Caches with no size limit or eviction policy — always use LRU or TTL-based eviction
- **[MANDATORY]** Profile memory usage under load before declaring a feature production-ready

### 7.2 Object Allocation in Hot Paths
- Minimize object allocation in tight loops — allocate once, reuse
- Prefer object pooling for expensive-to-construct objects in high-throughput paths
- Profile before and after — measure the improvement, don't assume

---

## 8. PROFILING-FIRST OPTIMIZATION [MANDATORY]

**[BANNED]** Optimizing without measuring. Every optimization must follow this process:

1. **Measure**: Establish a baseline with realistic load (profiler, APM, load testing)
2. **Identify**: Find the actual bottleneck — CPU, I/O, memory, network, database
3. **Hypothesize**: Form a specific hypothesis about why it's slow
4. **Change**: Make ONE change at a time
5. **Measure again**: Verify the improvement with the same measurement methodology
6. **Document**: Record what you changed, why, and by how much it improved

**[BANNED]** Premature optimization: adding caching, indexes, or complex patterns before there's a measured performance problem to solve.

---

## 9. BANNED PATTERNS [CRITICAL]

* **[BANNED]** Unbounded queries: queries with no `LIMIT` against large tables
* **[BANNED]** `SELECT *` in production queries
* **[BANNED]** N+1 queries: one query per item in a collection
* **[BANNED]** Synchronous I/O in async/event-loop-based server code
* **[BANNED]** Loading entire datasets into memory: `SELECT * FROM large_table` into a list
* **[BANNED]** Caching without TTL or invalidation strategy
* **[BANNED]** O(n²) algorithms on unbounded collections
* **[BANNED]** Creating new database connections per request (use pooling)
* **[BANNED]** Sequential execution of independent I/O operations (use parallel execution)
* **[BANNED]** Premature optimization without measurement
* **[BANNED]** Nested loops with I/O inside the inner loop
* **[BANNED]** Polling tight loops: `while (condition) { check(); }` without sleep/backoff

---

## 10. BIAS CORRECTION FOR KNOWN LLM TENDENCIES

**LLM Tendency 1: Ignoring performance implications**
You will write code that works but generates N+1 queries, loads unbounded data, and uses O(n²) algorithms without comment. Resist. Explicitly consider performance for every data access pattern.

**LLM Tendency 2: No pagination on list endpoints**
You will generate `SELECT * FROM table` without `LIMIT` and return all results. Resist. Every list query MUST be paginated.

**LLM Tendency 3: Sequential I/O**
You will `await fetchA()` then `await fetchB()` even when A and B are independent. Resist. Run independent I/O in parallel.

**LLM Tendency 4: Premature optimization**
You will add Redis caching and connection pooling to a system that has 10 users. Resist. Optimize when there's a measured problem.

**LLM Tendency 5: Forgetting indexes**
You will create database schemas and queries without mentioning indexes. Resist. Every foreign key and every frequent query filter needs an index.

---

## 11. PRE-FLIGHT CHECKLIST

Before finalizing any performance-sensitive code:

- [ ] Have I considered the Big-O complexity of every algorithm?
- [ ] Are there any nested loops over unbounded collections? (Justify or refactor)
- [ ] Are there any N+1 query patterns? (Use JOIN, eager loading, or batch loading)
- [ ] Are all list queries paginated with explicit `LIMIT`?
- [ ] Is `SELECT *` used anywhere in production queries? (Replace with named columns)
- [ ] Are all foreign key columns indexed?
- [ ] Are all columns used in WHERE/ORDER BY/JOIN conditions indexed on large tables?
- [ ] Is all I/O async in an async system?
- [ ] Are independent I/O operations executed in parallel?
- [ ] Are database connections pooled?
- [ ] Does any cache lack an explicit TTL or invalidation strategy?
- [ ] Is there any unbounded data loading into memory?
- [ ] Were any optimizations made without measurement? (If yes, remove them until measured)
- [ ] Are all resources (connections, file handles, streams) properly closed/released?
- [ ] Are there any tight polling loops without backoff?
