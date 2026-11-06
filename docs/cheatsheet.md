---
hide:
  - navigation
---

# ⚡ Revision Cheatsheet

The night-before page. Each line is a prompt — if you can't explain it in 30 seconds, revisit the linked page.

## Python

| Topic | One-liner |
|---|---|
| Variables | Names bound to objects; assignment never copies |
| Mutability | list/dict/set mutable; int/str/tuple/frozenset immutable; only hashables can be keys |
| Args | Pass-by-object-reference; mutable defaults evaluated once → use `None` |
| `is` vs `==` | identity vs value; `is` only for `None` |
| LEGB | Local → Enclosing → Global → Built-in; `nonlocal`, `global` |
| Closure | Inner function capturing enclosing variables (late binding!) |
| Decorator | `f = deco(f)`; always `@functools.wraps`; 3 layers for args |
| Generator | `yield` → lazy, O(1) memory, single pass; `yield from` delegates |
| Context manager | `__enter__/__exit__` or `@contextmanager`; guaranteed cleanup |
| Descriptor | `__get__/__set__` on class attr; powers `property`, ORM fields |
| Metaclass | Class of a class (`type`); prefer `__init_subclass__` |
| MRO | C3 linearization; `super()` = next in MRO |
| `__new__` vs `__init__` | create vs initialise |
| dataclass / slots | Boilerplate-free data classes / less memory |
| Memory | Refcounting + generational GC for cycles |
| GIL | One thread runs bytecode at a time; threads for I/O, processes for CPU; not a substitute for locks |
| asyncio | Cooperative, single thread, `await` = switch point; never block the loop (`to_thread`) |
| Testing | pytest, fixtures, parametrize, mock where looked up |

## APIs

| Topic | One-liner |
|---|---|
| REST | Resources + HTTP verbs + status codes + stateless |
| Idempotent | GET, PUT, DELETE (+HEAD, OPTIONS); POST via Idempotency-Key |
| PUT vs PATCH | replace vs partial |
| 401 / 403 | not authenticated / not allowed |
| 400 / 422 / 409 / 429 | malformed / invalid / conflict / rate limited |
| Pagination | Cursor (keyset) > offset at scale |
| Versioning | `/v1/`, additive changes only, deprecate with Sunset header |
| ETag | 304 caching, `If-Match` optimistic concurrency → 412 |
| JWT | Signed not encrypted; short-lived access + rotated refresh; pin algorithms |
| OAuth2 | Auth Code + PKCE (users), Client Credentials (services); OIDC adds id_token |
| OWASP #1 | BOLA — check object ownership every request |
| Rate limiting | Token bucket in Redis, 429 + Retry-After |
| Webhooks | HMAC verify, ack fast, process async, dedupe |
| FastAPI | Pydantic in/out schemas, `Depends` DI, async endpoints, `def` → threadpool |

## Databases

| Topic | One-liner |
|---|---|
| Execution order | FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT |
| Joins | INNER, LEFT, anti-join via `LEFT JOIN ... IS NULL` / `NOT EXISTS` |
| Window fns | `ROW_NUMBER / RANK / DENSE_RANK / LAG / LEAD / SUM() OVER (PARTITION BY ... ORDER BY ...)` |
| Nth salary | `DENSE_RANK() OVER (ORDER BY salary DESC) = N` |
| ACID | Atomic, Consistent, Isolated, Durable (WAL) |
| Isolation | RU < RC (PG default) < RR (MySQL default) < Serializable |
| Locks | Atomic UPDATE > optimistic version > `FOR UPDATE`; `SKIP LOCKED` for queues |
| Index | B-tree O(log n); composite leftmost prefix; covering; functions kill index use |
| EXPLAIN | Seq Scan on big table = missing index; estimates vs actuals |
| Normal forms | key (1NF), whole key (2NF), nothing but the key (3NF) |
| Scaling | Index → cache → replicas → partition → shard |
| CAP | Partition ⇒ choose C or A |
| Redis | Cache-aside + TTL jitter; invalidate on write; stampede → single-flight |
| N+1 | `selectinload` / `joinedload`, `select_related` / `prefetch_related` |
| Distributed txn | Saga + outbox; avoid 2PC |

## Design

| Topic | One-liner |
|---|---|
| SOLID | Single resp, Open/closed, Liskov, Interface segregation, Dependency inversion |
| Factory | Registry of types, callers don't know concretes |
| Strategy | Swap algorithm — often just a function |
| Observer | Event bus / pub-sub, decouple side effects |
| Adapter | Wrap third-party SDK behind your interface |
| Decorator vs Proxy | Add behaviour vs control access |
| State | Order lifecycle without if/elif |
| Repository + UoW | Abstract persistence + atomic commit |
| Circuit breaker | Closed → Open (fail fast) → Half-open (probe) |
| Data modeling | Access patterns first; 3NF then denormalize hot paths; money as minor units; UTC |
| System design | Requirements → estimates → API → data → HLD → deep dive → trade-offs |

## DSA

| Signal | Pattern |
|---|---|
| Need O(1) lookup / count | Hash map |
| Sorted array / pairs | Two pointers / binary search |
| Contiguous subarray with constraint | Sliding window |
| Subarray sum = k | Prefix sum + hash map |
| Next greater / smaller | Monotonic stack |
| Top-k / k-th / merge k | Heap |
| Shortest path unweighted | BFS |
| All combinations | Backtracking |
| Dependencies | Topological sort |
| Connected groups | Union-find / DFS |
| Count ways / min cost / overlapping subproblems | DP |
| Prefixes | Trie |

**Complexities:** sort O(n log n) · heap push/pop O(log n) · heapify O(n) · BFS/DFS O(V+E) · Dijkstra O((V+E) log V) · binary search O(log n).

## Behavioural answers (STAR)

Prepare 5 stories in **S**ituation → **T**ask → **A**ction → **R**esult form:

- [ ] A system you designed end-to-end and the trade-offs you made
- [ ] A production incident you led and what changed afterwards
- [ ] A performance problem you diagnosed (with numbers)
- [ ] A disagreement with a stakeholder/tech lead and how it resolved
- [ ] Mentoring / leading a team through a difficult delivery

## Questions to ask the interviewer

- How do you deploy, and how often? What does on-call look like?
- What's the biggest technical challenge the team faces this year?
- How are technical decisions made and documented?
- What would success look like for this role in 6 months?
