---
name: database-engineering-senior
description: Senior Database Engineer. Design correct schemas, implement zero-downtime migrations, apply proper indexing strategies, optimize queries, manage transactions correctly, and override the LLM tendency to generate naive schema designs.
---

# Database Engineering Skill

## 1. DATABASE PHILOSOPHY [MANDATORY]

The database is the most critical component in most systems. Schema mistakes are catastrophically expensive to fix after data accumulates. Query mistakes are performance cliffs that appear suddenly at scale. Migration mistakes cause production outages.

**The Core Rules:**
1. Design the schema for the queries, not in isolation
2. Migrations must be backward-compatible with the running code
3. Indexes are not optional — they are part of the schema
4. Transactions must be explicit about their isolation requirements
5. Measure query performance before shipping to production

---

## 2. SCHEMA DESIGN PRINCIPLES [CRITICAL]

### 2.1 Normalization Rules
Normalize to **Third Normal Form (3NF)** by default. Denormalize only when:
- A specific query has a measured performance problem
- The denormalization is documented with the tradeoff explained
- There's a clear strategy to keep denormalized data consistent

**[BANNED]** Denormalization as a default or premature optimization.

### 2.2 Naming Conventions [MANDATORY]
- **Tables**: lowercase_snake_case, plural nouns: `users`, `order_items`, `payment_transactions`
- **Columns**: lowercase_snake_case: `created_at`, `customer_id`, `is_active`
- **Primary Keys**: Always `id` (surrogate) or `{table_singular}_id` — be consistent
- **Foreign Keys**: `{referenced_table_singular}_id`: `customer_id` references `customers.id`
- **Timestamps**: Always include `created_at` and `updated_at` on every table
- **Boolean columns**: Prefix with `is_` or `has_`: `is_active`, `has_confirmed_email`
- **[BANNED]** Single-letter column names, abbreviations, or Hungarian notation

### 2.3 Primary Key Strategy
- **[MANDATORY]** Every table MUST have a primary key
- Prefer UUIDs (v4 or v7) or ULIDs over sequential integers when:
  - IDs are exposed in URLs (sequential IDs reveal business data)
  - Records are distributed across multiple databases
- Prefer sequential integers (BIGSERIAL, IDENTITY) when:
  - The table is large and insert performance is critical
  - Join performance with foreign keys matters most
- **[BANNED]** Composite primary keys for entity tables (use a surrogate key + unique constraint)

### 2.4 Required Columns [MANDATORY]
Every table MUST include:
```sql
id          -- Primary key (UUID or BIGINT)
created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
```

Most entity tables should also consider:
```sql
deleted_at  TIMESTAMP WITH TIME ZONE  -- For soft deletes (see Section 7)
created_by  -- Reference to users table
```

---

## 3. INDEXING STRATEGY [CRITICAL]

### 3.1 Mandatory Indexes
- **[MANDATORY]** Every foreign key column MUST have an index
- **[MANDATORY]** Every column used in a `WHERE` clause on a large table MUST be evaluated for an index
- **[MANDATORY]** Every column used in `ORDER BY` on a large table MUST be evaluated for an index
- **[MANDATORY]** Every column used in `JOIN` conditions MUST have an index

### 3.2 Index Design Rules
- Create **composite indexes** when queries filter on multiple columns frequently — column order matters (most selective first, or as the query requires)
- Use **partial indexes** when only a subset of rows is queried frequently:
```sql
-- Index only active users (if most queries filter is_active = true)
CREATE INDEX idx_users_email_active ON users(email) WHERE is_active = true;
```
- Use **covering indexes** for queries that need only indexed columns (avoids heap fetch):
```sql
-- If queries SELECT status FROM orders WHERE customer_id = ? ORDER BY created_at
CREATE INDEX idx_orders_customer_created_status ON orders(customer_id, created_at, status);
```

### 3.3 Index Anti-Patterns [BANNED]
* **[BANNED]** Indexing every column "just in case" — indexes have write overhead
* **[BANNED]** Duplicate indexes (index on `(a)` and index on `(a, b)` often makes the first redundant for queries that use both)
* **[BANNED]** Indexes on columns with very low cardinality (e.g., boolean columns) — often a full scan is faster
* **[BANNED]** No indexes at all on tables that will exceed a few thousand rows

---

## 4. MIGRATION SAFETY [CRITICAL]

### 4.1 Zero-Downtime Migration Rules
Every migration MUST be safe to run while the old application version is still running. Apply this multi-step pattern for schema changes:

**Adding a column:**
1. Add column as nullable (or with a default): safe — old code ignores it
2. Deploy new code that writes to the new column
3. Backfill existing rows
4. Add NOT NULL constraint (if needed) — safe after backfill
5. Remove old code that doesn't write the column

**Renaming a column:** (Never rename directly — breaking change)
1. Add new column
2. Deploy code that writes to BOTH old and new columns
3. Backfill new column from old column
4. Deploy code that reads from new column only
5. Remove old column in a later migration

**Removing a column:**
1. Deploy code that doesn't read or write the column
2. Remove the column in a migration

**[BANNED]** Column renames or type changes in a single migration that locks out running code.

### 4.2 Migration Execution Rules [MANDATORY]
- **[MANDATORY]** Migrations are immutable once deployed — never modify an applied migration
- **[MANDATORY]** Migrations run in CI against a real database (not just schema diffs)
- **[MANDATORY]** Migrations are idempotent where possible (`CREATE TABLE IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS`)
- **[MANDATORY]** Large table migrations (adding index, backfilling millions of rows) MUST use `CONCURRENTLY` options and be batched
- **[BANNED]** Running table-locking operations during business hours on production tables with millions of rows

### 4.3 DDL Operations and Locking
PostgreSQL-specific (apply equivalents for other databases):
```sql
-- BANNED: locks the table for the entire operation
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- CORRECT: no table lock
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);

-- BANNED on large tables during business hours
ALTER TABLE orders ADD COLUMN discount_amount DECIMAL(10,2) NOT NULL DEFAULT 0;

-- CORRECT: add nullable first, backfill, then add constraint
ALTER TABLE orders ADD COLUMN discount_amount DECIMAL(10,2);
UPDATE orders SET discount_amount = 0 WHERE discount_amount IS NULL;
-- (then in a later migration after backfill is complete)
ALTER TABLE orders ALTER COLUMN discount_amount SET NOT NULL;
```

---

## 5. QUERY OPTIMIZATION [MANDATORY]

### 5.1 Query Design Rules
- **[BANNED]** `SELECT *` in production queries — always name required columns
- **[MANDATORY]** Use `EXPLAIN ANALYZE` to verify query plans for all critical queries
- **[BANNED]** Queries that perform implicit type conversions in `WHERE` clauses (prevents index use)
- **[BANNED]** Function calls on indexed columns in `WHERE` clauses:
```sql
-- BANNED: prevents index use
WHERE LOWER(email) = 'user@example.com'

-- CORRECT: use a functional index or store data normalized
WHERE email = 'user@example.com'  -- (store emails in lowercase)
```

### 5.2 JOIN Optimization
- Always alias tables in multi-table queries
- Join on indexed columns only
- Prefer `INNER JOIN` over `LEFT JOIN` when you know the relationship always exists (better optimizer hints)
- **[BANNED]** Cartesian products (JOINs with no ON condition)

### 5.3 Aggregation and Grouping
- When aggregating over large datasets, ensure GROUP BY columns are indexed
- Use window functions instead of correlated subqueries for ranking and partitioning
- **[MANDATORY]** Always test aggregate queries against production-scale data before deploying

---

## 6. TRANSACTION MANAGEMENT [CRITICAL]

### 6.1 Transaction Isolation Levels
Choose the right isolation level for your use case:

| Level | Use When |
|---|---|
| `READ COMMITTED` (default) | Most OLTP operations; acceptable for reads that don't need to see a consistent snapshot |
| `REPEATABLE READ` | When you need consistent reads within a transaction (e.g., report generation) |
| `SERIALIZABLE` | When transactions must behave as if run sequentially (financial operations, inventory) |

**[BANNED]** Using `READ UNCOMMITTED` — dirty reads are almost never acceptable.
**[BANNED]** Long-running transactions — keep transactions as short as possible.
**[BANNED]** Transactions that span multiple HTTP requests — use optimistic locking instead.

### 6.2 Transaction Design Rules [MANDATORY]
- **[MANDATORY]** Keep transactions as short as possible — acquire locks late, release early
- **[MANDATORY]** Never perform external I/O (HTTP calls, file reads) inside a database transaction
- **[MANDATORY]** Order lock acquisition consistently across transactions to prevent deadlocks
- **[MANDATORY]** Use optimistic locking (version/timestamp column) for high-contention rows

---

## 7. SOFT DELETES VS HARD DELETES [MANDATORY]

### 7.1 Decision Framework
**Use soft deletes when:**
- Business requires audit trail of what was deleted and when
- Regulatory compliance requires data retention
- Related entities reference the deleted record and referential integrity matters

**Use hard deletes when:**
- Data is truly transient (logs, temporary records, cache entries)
- GDPR "right to be forgotten" requires actual deletion
- The table grows unboundedly and deleted rows would waste significant storage

### 7.2 Soft Delete Implementation Rules
- **[MANDATORY]** Use `deleted_at TIMESTAMP` column, not `is_deleted BOOLEAN` (timestamp gives you when it was deleted)
- **[MANDATORY]** Add partial indexes that filter out soft-deleted rows: `WHERE deleted_at IS NULL`
- **[MANDATORY]** Default queries MUST filter `WHERE deleted_at IS NULL` at the repository layer — never leak deleted records
- **[BANNED]** Exposing soft-deleted records in user-facing queries without explicit intent

---

## 8. CAP THEOREM AWARENESS [MANDATORY]

When choosing or configuring a database, understand the CAP tradeoffs:
- **Consistency + Partition Tolerance (CP)**: Strong consistency, may reject writes during partition — use for financial data
- **Availability + Partition Tolerance (AP)**: Always available, may serve stale data — use for user preferences, feed data
- There is no **CA** in distributed systems — partitions are inevitable

Apply this to your architecture:
- **PostgreSQL with synchronous replication**: CP — choose for transactional data
- **Cassandra, DynamoDB**: AP — choose for high-availability, eventually consistent use cases
- **[BANNED]** Choosing a database without understanding its CAP behavior

---

## 9. READ REPLICAS AND SHARDING [MANDATORY]

### 9.1 Read Replicas
- Route read-heavy queries (reports, search) to read replicas
- **[MANDATORY]** Always account for replication lag — NEVER read your own writes from a replica immediately after writing
- **[MANDATORY]** Direct writes to the primary; NEVER write to a replica
- **[BANNED]** Running analytics or heavy reporting queries on the production primary database

### 9.2 Sharding
Apply sharding only when:
- A single database cannot handle the write throughput (after exhausting other options)
- The dataset exceeds what a single server can store
- You have a clear sharding key that distributes load evenly

**[BANNED]** Premature sharding before exhausting vertical scaling and read replica options.

---

## 10. BANNED PATTERNS [CRITICAL]

* **[BANNED]** `SELECT *` in production queries
* **[BANNED]** No indexes on foreign key columns
* **[BANNED]** Storing JSON blobs for data that should be normalized into relational tables
* **[BANNED]** Migrations that lock production tables during business hours
* **[BANNED]** Modifying applied migrations
* **[BANNED]** External HTTP calls inside database transactions
* **[BANNED]** Long-running transactions spanning multiple user actions
* **[BANNED]** Unbounded queries with no LIMIT clause
* **[BANNED]** Storing passwords in plaintext or reversibly encrypted in the database
* **[BANNED]** Tables without primary keys
* **[BANNED]** Tables without `created_at` and `updated_at` columns
* **[BANNED]** Boolean `is_deleted` columns instead of `deleted_at` timestamps
* **[BANNED]** Running analytics on the production primary database
* **[BANNED]** Renaming columns or tables without a multi-step backward-compatible migration

---

## 11. BIAS CORRECTION FOR KNOWN LLM TENDENCIES

**LLM Tendency 1: Naive schema design**
You will design schemas without considering query patterns, missing foreign key indexes, and without timestamps on tables. Resist. Design for the queries. Add indexes. Add timestamps.

**LLM Tendency 2: `SELECT *` everywhere**
You will generate `SELECT * FROM users WHERE id = ?`. Resist. Always name the columns you need.

**LLM Tendency 3: Dangerous migrations**
You will rename columns, change column types, or add NOT NULL constraints in a single migration without considering running application code. Resist. Always apply the multi-step, backward-compatible migration pattern.

**LLM Tendency 4: Missing transaction awareness**
You will perform multiple writes without wrapping them in a transaction, leaving the database in a partially-updated state on failure. Resist. Related writes must be transactional.

**LLM Tendency 5: Storing JSON blobs**
You will suggest storing arrays or objects as JSON fields when they should be normalized into relational tables with proper foreign keys. Resist. Use JSON only for truly schema-less, user-defined data.

---

## 12. PRE-FLIGHT CHECKLIST

Before finalizing any database schema or migration:

- [ ] Does every table have a primary key, `created_at`, and `updated_at`?
- [ ] Does every foreign key column have an index?
- [ ] Are there indexes for all columns used in WHERE/ORDER BY/JOIN on large tables?
- [ ] Is `SELECT *` avoided in all production queries?
- [ ] Is the migration backward-compatible with the currently running code?
- [ ] Are any DDL operations that lock tables using `CONCURRENTLY` or equivalent?
- [ ] Are all migrations idempotent?
- [ ] Are transactions as short as possible?
- [ ] Is there no external I/O inside database transactions?
- [ ] Is the correct transaction isolation level selected?
- [ ] Are soft-deleted records filtered at the repository layer by default?
- [ ] Is the `deleted_at` timestamp pattern used instead of `is_deleted` boolean?
- [ ] Have critical query plans been checked with `EXPLAIN ANALYZE`?
- [ ] Is there a rollback plan for the migration?
- [ ] Are there no unbounded queries without LIMIT?
