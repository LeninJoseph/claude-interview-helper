# Database Q&A Bank

??? question "1. What is a primary key vs unique key vs foreign key?"
    PK uniquely identifies a row, is NOT NULL, one per table (often the clustered index). Unique key enforces uniqueness, allows NULL (one or many depending on DB), many per table. FK references another table's key to enforce referential integrity.

??? question "2. Natural key vs surrogate key?"
    Natural key comes from the data (email, ISBN) and can change. Surrogate key is generated (serial, UUID) and stable. Prefer surrogate PK + unique constraint on natural key.

??? question "3. Auto-increment vs UUID primary key?"
    Integers: small, fast, ordered inserts, but guessable and need central generation. UUIDv4: globally unique, merge-friendly, not guessable, but larger and random inserts fragment B-trees. UUIDv7 / ULID are time-ordered — best of both.

??? question "4. WHERE vs HAVING?"
    WHERE filters rows before grouping; HAVING filters groups after aggregation.

??? question "5. INNER vs LEFT vs FULL JOIN?"
    INNER returns matches only; LEFT returns all left rows plus matches (NULLs otherwise); FULL returns all rows from both sides.

??? question "6. UNION vs UNION ALL?"
    UNION removes duplicates (sort/hash cost); UNION ALL keeps all rows and is faster.

??? question "7. DELETE vs TRUNCATE vs DROP?"
    DELETE removes selected rows (logged, triggers, rollback); TRUNCATE removes all rows quickly; DROP removes the table definition and data.

??? question "8. What are window functions? ROW_NUMBER vs RANK vs DENSE_RANK?"
    Functions computed over a window of related rows without collapsing them. For ties at 2nd: ROW_NUMBER 1,2,3,4; RANK 1,2,2,4; DENSE_RANK 1,2,2,3.

??? question "9. What's a CTE? When use a recursive CTE?"
    A named temporary result (`WITH`) improving readability and reuse. Recursive CTEs walk hierarchies: org charts, category trees, graph paths.

??? question "10. Explain ACID."
    Atomicity (all-or-nothing), Consistency (constraints preserved), Isolation (concurrent transactions don't interfere), Durability (committed data survives crashes via WAL).

??? question "11. Explain isolation levels and anomalies."
    Read Uncommitted (dirty reads), Read Committed (no dirty reads; PG default), Repeatable Read (no non-repeatable reads; MySQL default), Serializable (no phantoms or write skew; may abort transactions that must be retried).

??? question "12. How does an index work and what does it cost?"
    A sorted B-tree (usually) mapping key values to row locations → O(log n) lookups instead of full scans. Costs: storage, slower INSERT/UPDATE/DELETE, maintenance.

??? question "13. What's a composite index and the leftmost prefix rule?"
    An index on multiple columns. It can be used for queries filtering on a leftmost prefix of its columns: (a,b,c) serves a, a+b, a+b+c but not b alone.

??? question "14. Clustered vs non-clustered index?"
    Clustered defines the physical row order (one per table, InnoDB PK). Non-clustered is a separate structure with pointers to rows.

??? question "15. What's a covering index?"
    An index containing all columns a query needs, allowing an index-only scan without touching the table.

??? question "16. Why might the DB not use my index?"
    Functions/casts on the column, leading wildcard LIKE, low selectivity, wrong column order, stale statistics, or the planner estimates a seq scan is cheaper.

??? question "17. What is normalization? Explain 1NF, 2NF, 3NF."
    Organizing data to reduce redundancy. 1NF atomic values; 2NF no partial dependency on part of a composite key; 3NF no transitive dependencies between non-key columns.

??? question "18. When would you denormalize?"
    Read-heavy paths where joins are too costly: reporting, counters, read models (CQRS), search documents. Accept the cost of keeping copies in sync (triggers, events).

??? question "19. Optimistic vs pessimistic locking?"
    Optimistic checks a version on write and retries on conflict — good when conflicts are rare. Pessimistic locks rows (`SELECT FOR UPDATE`) up front — good for hot contended rows.

??? question "20. What's a deadlock and how to avoid it?"
    Two transactions waiting on each other's locks. Avoid by consistent lock ordering, short transactions, proper indexes (fewer locked rows), and retry on deadlock errors.

??? question "21. What is MVCC?"
    Multi-version concurrency control: writers create new row versions, readers see a consistent snapshot — readers and writers don't block each other.

??? question "22. What is the N+1 query problem?"
    One query to load N parents plus N queries for their children. Fix with JOINs or batch eager loading.

??? question "23. How do you scale reads? Writes?"
    Reads: indexes, caching, read replicas, materialized views. Writes: batching, partitioning, queues to smooth spikes, sharding, or a write-optimized store.

??? question "24. Replication lag — what problems and fixes?"
    Users may not see their own recent writes. Fix: read-your-writes routing to primary for a short time after a write, session consistency tokens, synchronous replicas for critical data.

??? question "25. Partitioning vs sharding?"
    Partitioning splits a table within one database instance. Sharding distributes data across multiple database servers by a shard key.

??? question "26. How do you choose a shard key?"
    High cardinality, even distribution, aligned with most queries (so they hit one shard), avoids hotspots (not a monotonically increasing timestamp alone).

??? question "27. What is the CAP theorem?"
    Under a network partition, a distributed system must choose between consistency and availability. PACELC adds the latency vs consistency trade-off when there's no partition.

??? question "28. When choose MongoDB over PostgreSQL?"
    Flexible/nested documents read as a unit, rapidly evolving schemas, horizontal scale needs. PostgreSQL (with JSONB) is usually a better default when relationships and transactions matter.

??? question "29. What is Redis used for?"
    Caching, sessions, rate limiting, distributed locks, leaderboards (sorted sets), queues/streams, pub/sub.

??? question "30. Cache-aside vs write-through?"
    Cache-aside: app loads into cache on miss and invalidates on write. Write-through: every write goes to cache and DB synchronously, keeping the cache fresh at the cost of write latency.

??? question "31. How do you prevent cache stampede?"
    Single-flight locking on misses, stale-while-revalidate, refresh-ahead for hot keys, TTL jitter.

??? question "32. What is a distributed transaction? Saga vs 2PC?"
    A transaction spanning services/DBs. 2PC coordinates prepare/commit — consistent but blocking. Saga chains local transactions with compensating actions — available and scalable, eventually consistent.

??? question "33. What is the outbox pattern?"
    Persist domain changes and the event to publish in one local transaction (outbox table), then a relay/CDC publishes to the broker — guarantees no lost or phantom events.

??? question "34. How do you store money?"
    `NUMERIC/DECIMAL` with fixed scale or integer minor units, plus a currency code. Never float.

??? question "35. How do you store time?"
    UTC with timezone (`TIMESTAMPTZ`), convert at the edges.

??? question "36. How would you do a zero-downtime schema change?"
    Expand and contract: add new nullable column/table, dual-write, backfill in batches, switch reads, then drop the old column in a later release; build indexes concurrently.

??? question "37. Write SQL: second highest salary per department"
    ```sql
    SELECT dept_id, salary FROM (
      SELECT dept_id, salary, DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) r
      FROM employees) t
    WHERE r = 2;
    ```

??? question "38. Write SQL: customers with orders in every month of 2026"
    ```sql
    SELECT customer_id FROM orders
    WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
    GROUP BY customer_id
    HAVING COUNT(DISTINCT DATE_TRUNC('month', created_at)) = 12;
    ```

??? question "39. Write SQL: percentage of revenue by category"
    ```sql
    SELECT p.category,
           SUM(oi.quantity * oi.unit_price) AS revenue,
           ROUND(100.0 * SUM(oi.quantity * oi.unit_price)
                 / SUM(SUM(oi.quantity * oi.unit_price)) OVER (), 2) AS pct
    FROM order_items oi JOIN products p ON p.id = oi.product_id
    GROUP BY p.category;
    ```

??? question "40. Write SQL: first order of each customer"
    ```sql
    SELECT DISTINCT ON (customer_id) customer_id, id, created_at
    FROM orders ORDER BY customer_id, created_at;      -- PostgreSQL
    -- portable: ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) = 1
    ```
