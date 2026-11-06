# NoSQL & Caching

## SQL vs NoSQL

| | Relational (SQL) | NoSQL |
|---|---|---|
| Schema | Fixed, enforced | Flexible / schema-on-read |
| Relationships | Joins, foreign keys | Embedding / denormalization |
| Transactions | Strong ACID | Varies (Mongo has multi-doc ACID; many are BASE) |
| Scaling | Vertical first, replicas, harder to shard | Built for horizontal scaling |
| Query | SQL | Per-database APIs |
| Choose when | Complex relations, integrity, transactions (payments, orders) | Huge scale, flexible/nested data, high write throughput, specific access patterns |

**BASE** = Basically Available, Soft state, Eventually consistent.

## NoSQL families

| Type | Examples | Model | Use cases |
|---|---|---|---|
| **Key-value** | Redis, DynamoDB, Memcached | key → value | Cache, sessions, rate limits, leaderboards |
| **Document** | MongoDB, Couchbase, Firestore | JSON documents | Catalogs, user profiles, CMS |
| **Wide-column** | Cassandra, ScyllaDB, HBase, Bigtable | rows with dynamic columns, partition key | Time series, IoT, messaging at huge write scale |
| **Graph** | Neo4j, Neptune | nodes + edges | Social networks, recommendations, fraud rings |
| **Search** | Elasticsearch, OpenSearch | inverted index | Full-text search, log analytics |
| **Vector** | pgvector, Pinecone, Weaviate, Qdrant | embeddings + ANN index | Semantic search, RAG |

## CAP theorem

During a **network Partition**, a distributed system must choose **Consistency** (every read sees the latest write, or errors) or **Availability** (every request gets a response, possibly stale).

- **CP**: MongoDB (default), HBase, etcd, ZooKeeper.
- **AP**: Cassandra, DynamoDB (configurable), CouchDB.

**PACELC** extends it: if Partition → A or C; **E**lse → **L**atency or **C**onsistency.

**Consistency models:** strong → read-your-writes → monotonic reads → causal → eventual.
Cassandra/Dynamo tune it with quorums: **R + W > N** gives strong consistency.

## MongoDB in practice

```python
from pymongo import MongoClient, ASCENDING
db = MongoClient("mongodb://localhost:27017").shop

db.orders.insert_one({
    "customer": {"id": 7, "name": "Lenin"},          # embedded
    "items": [{"sku": "LAP-1", "qty": 1, "price": 999}],
    "status": "paid",
    "total": 999,
})

db.orders.find({"status": "paid", "total": {"$gte": 500}}, {"_id": 0, "total": 1}).sort("total", -1).limit(5)
db.orders.update_one({"_id": oid}, {"$set": {"status": "shipped"}, "$inc": {"version": 1}})
db.orders.create_index([("customer.id", ASCENDING), ("status", ASCENDING)])

# Aggregation pipeline ≈ SQL GROUP BY
db.orders.aggregate([
    {"$match": {"status": "paid"}},
    {"$unwind": "$items"},
    {"$group": {"_id": "$items.sku", "units": {"$sum": "$items.qty"}}},
    {"$sort": {"units": -1}},
    {"$limit": 3},
])
```

**Embed vs reference:** embed when data is read together and bounded (order → items); reference when many-to-many, unbounded growth, or shared/updated independently (user ↔ orders). 16MB document limit.

## Redis

In-memory data-structure store: microsecond latency, single-threaded command execution (atomic commands), optional persistence (RDB snapshots / AOF log).

| Structure | Commands | Use |
|---|---|---|
| String | `SET k v EX 60 NX`, `GET`, `INCR` | Cache, counters, locks, rate limits |
| Hash | `HSET`, `HGETALL` | Objects / sessions |
| List | `LPUSH`, `BRPOP` | Simple queues |
| Set | `SADD`, `SISMEMBER` | Unique visitors, tags |
| Sorted set | `ZADD`, `ZREVRANGE`, `ZRANGEBYSCORE` | Leaderboards, sliding-window rate limit, delayed jobs |
| Stream | `XADD`, `XREADGROUP` | Event log with consumer groups |
| Pub/Sub | `PUBLISH`, `SUBSCRIBE` | Real-time fan-out (no persistence) |
| HyperLogLog / Bitmap / Geo | `PFADD`, `SETBIT`, `GEOADD` | Approx counts, flags, location |

```python
import redis, json
r = redis.Redis(decode_responses=True)

# Leaderboard
r.zadd("leaderboard", {"lenin": 1500, "asha": 1720})
r.zrevrange("leaderboard", 0, 9, withscores=True)

# Fixed-window rate limit
def allowed(user_id: str, limit=100) -> bool:
    key = f"rl:{user_id}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, 60)
    return count <= limit

# Simple distributed lock (use Redlock / redis-py Lock in production)
token = "unique-uuid"
if r.set("lock:invoice:42", token, nx=True, ex=30):
    try: ...
    finally:
        # release only if we still own it (Lua script for atomicity)
        r.eval("if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) end",
               1, "lock:invoice:42", token)
```

## Caching strategies

| Pattern | How | Pros / cons |
|---|---|---|
| **Cache-aside (lazy loading)** | App reads cache → miss → read DB → write cache | Most common; only caches what's used; first request slow; possible stale data |
| **Read-through** | Cache itself loads from DB on miss | Simpler app code |
| **Write-through** | Write to cache and DB synchronously | Cache always fresh; slower writes |
| **Write-behind (write-back)** | Write to cache, flush to DB async | Fast writes; risk of data loss |
| **Refresh-ahead** | Refresh hot keys before expiry | Avoids misses for hot keys |

```python
def get_product(pid: int) -> dict:
    key = f"product:{pid}"
    if (cached := r.get(key)) is not None:
        return json.loads(cached)                       # hit
    product = db_fetch_product(pid)                     # miss
    r.set(key, json.dumps(product), ex=300 + random.randint(0, 60))   # TTL + jitter
    return product

def update_product(pid: int, data: dict):
    db_update_product(pid, data)
    r.delete(f"product:{pid}")                          # invalidate AFTER DB write
```

### Cache problems interviewers ask about

| Problem | Description | Fix |
|---|---|---|
| **Cache stampede / thundering herd** | Hot key expires, thousands hit DB at once | Lock/single-flight on miss, refresh-ahead, stale-while-revalidate |
| **Cache penetration** | Requests for keys that don't exist bypass cache | Cache negative results briefly, Bloom filter |
| **Cache avalanche** | Many keys expire at the same time | TTL jitter, multi-level cache |
| **Stale data** | Cache and DB disagree | Invalidate on write, short TTL, versioned keys, CDC-driven invalidation |
| **Hot key** | One key overloads one node | Local in-process cache, replicate key |

**Eviction policies:** LRU, LFU, FIFO, TTL. Redis: `maxmemory-policy allkeys-lru`.

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

!!! success "Say this in the interview"
    "My default is a relational database for transactional data. I add Redis as a cache-aside layer with TTL + jitter and invalidate on write, and choose a NoSQL store when a specific access pattern demands it — Cassandra for massive time-series writes, Elasticsearch for search, a vector index for semantic retrieval."
