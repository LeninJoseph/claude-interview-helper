# API Q&A Bank

## Fundamentals

??? question "1. What is REST?"
    An architectural style for networked systems: resources identified by URIs, manipulated with standard HTTP methods via representations (JSON), with stateless, cacheable, layered client–server communication and a uniform interface.

??? question "2. PUT vs PATCH vs POST?"
    POST creates (or triggers) — not idempotent. PUT replaces the whole resource at a known URI — idempotent. PATCH partially updates — not inherently idempotent (e.g. `{"op": "increment"}`).

??? question "3. What does idempotent mean? Which methods are?"
    Repeating the request has the same effect on server state as doing it once. GET, HEAD, OPTIONS, PUT, DELETE are idempotent; POST and PATCH are not by default.

??? question "4. How do you make POST idempotent?"
    Client sends a unique `Idempotency-Key`; server atomically stores key → response (Redis `SET NX` or a unique DB constraint) and replays the stored response on retries.

??? question "5. 401 vs 403?"
    401 = not authenticated (missing/invalid credentials). 403 = authenticated but lacks permission.

??? question "6. 400 vs 422?"
    400 for malformed requests (unparseable JSON). 422 for syntactically valid but semantically invalid data (failed validation).

??? question "7. What's statelessness and why does it matter?"
    The server keeps no client session between requests; each request carries its own auth/context. Any instance can handle any request → horizontal scaling behind a load balancer.

??? question "8. Offset vs cursor pagination?"
    Offset is simple but slow at large offsets and inconsistent when data changes. Cursor (keyset) uses an indexed column (`WHERE id > last_id`) — fast and stable, but no random page jumps.

??? question "9. How do you version an API?"
    URL path (`/v1`) is most common. Make additive changes non-breaking; for breaking changes ship a new version, mark the old one deprecated with `Deprecation`/`Sunset` headers, and monitor usage before removal.

??? question "10. What is HATEOAS?"
    Hypermedia As The Engine Of Application State — responses include links to related actions (`"_links": {"cancel": "/orders/1/cancel"}`) so clients discover the API dynamically. Level 3 of Richardson's model; rare in practice.

??? question "11. How do ETags work?"
    Server sends a version hash. Client sends `If-None-Match` on GET → 304 if unchanged (caching), or `If-Match` on PUT → 412 if changed by someone else (optimistic concurrency).

??? question "12. REST vs GraphQL — when would you pick GraphQL?"
    When many clients need different shapes of deeply related data (mobile + web), to avoid over/under-fetching and multiple round trips. Costs: harder HTTP caching, query-complexity limits, N+1 resolver issues (fix with DataLoader).

??? question "13. When gRPC?"
    Internal service-to-service calls needing low latency, strong typed contracts and streaming. Binary Protobuf over HTTP/2.

??? question "14. What is CORS?"
    A browser mechanism letting a server declare which other origins may read its responses. Preflight `OPTIONS` for non-simple requests. It is not server-side security.

??? question "15. How do you handle long-running requests?"
    Return 202 Accepted with a job/status URL, process in a worker queue, let the client poll or receive a webhook/SSE notification.

## Security

??? question "16. How does JWT work? What are its downsides?"
    Signed token with claims (sub, exp, roles) that any service can verify without a DB lookup. Downsides: can't be revoked before expiry (use short TTL + refresh tokens + denylist), payload is readable, size overhead.

??? question "17. Where should a browser store tokens?"
    Prefer httpOnly, Secure, SameSite cookies (not readable by JS) or in-memory access tokens with a refresh cookie. Avoid localStorage (XSS).

??? question "18. Explain OAuth2 Authorization Code with PKCE."
    App redirects user to auth server with a hashed random `code_challenge`; after login, it receives a short-lived code, then exchanges code + original `code_verifier` for tokens. PKCE stops intercepted codes being used.

??? question "19. OAuth2 vs OIDC?"
    OAuth2 is delegated authorization (access tokens for APIs). OIDC adds authentication: an `id_token` describing who the user is.

??? question "20. What's BOLA/IDOR and how do you prevent it?"
    Accessing another user's object by changing an ID. Prevent with object-level authorization on every query (`WHERE id = :id AND owner_id = :user`), non-guessable IDs help but are not a fix.

??? question "21. How do you prevent SQL injection?"
    Parameterized queries / ORM; never build SQL with string formatting; least-privilege DB users.

??? question "22. How do you implement rate limiting in a distributed system?"
    Token bucket or sliding window counters in Redis keyed by client, updated atomically (Lua script), enforced at the API gateway or middleware, returning 429 with `Retry-After`.

??? question "23. How do you store passwords?"
    Argon2id or bcrypt with per-user salt (built in), never reversible encryption or fast hashes.

??? question "24. How do you verify a webhook?"
    Check the HMAC signature of the raw body with the shared secret using constant-time compare, check timestamp to prevent replay, dedupe on event ID.

## Production & performance

??? question "25. How would you improve a slow API endpoint?"
    Measure first (APM, tracing, logs). Typical fixes: fix N+1 queries, add indexes, select only needed columns, cache (Redis/HTTP), paginate, async I/O for upstream calls in parallel, move heavy work to a queue, connection pooling, compression.

??? question "26. What is the N+1 query problem?"
    Fetching N parents with one query, then one query per parent for children. Fix with joins/eager loading (`select_related`, `prefetch_related`, `selectinload`).

??? question "27. How do you handle failures when calling other services?"
    Timeouts on every call, retries with exponential backoff + jitter for idempotent calls only, circuit breakers, fallbacks/degraded responses, bulkheads (separate pools).

??? question "28. What is an API gateway?"
    A single entry point handling routing, auth, rate limiting, TLS termination, request transformation, logging and caching for backend services (Kong, AWS API Gateway, Nginx, Envoy).

??? question "29. How do you document an API?"
    OpenAPI spec (auto-generated in FastAPI), with examples, error schemas and auth described; publish Swagger/Redoc; contract tests.

??? question "30. How do you test an API?"
    Unit tests for services, integration tests with `TestClient`/`httpx` against a real test DB, contract tests (schemathesis/Pact), load tests (Locust, k6).

??? question "31. What's the difference between `BackgroundTasks` and Celery?"
    BackgroundTasks runs in the same process after the response and is lost on crash. Celery uses a broker and workers for durable, retryable, scheduled tasks.

??? question "32. How do you handle file uploads?"
    Stream to object storage (S3) — ideally via pre-signed URLs so the file bypasses your API; validate size and type; scan for malware; store metadata in DB.

??? question "33. How do you secure service-to-service communication?"
    mTLS (service mesh), OAuth2 client credentials with short-lived tokens, network policies, no shared static secrets.

## Design exercises

??? question "34. Design a REST API for a URL shortener"
    ```text
    POST   /api/v1/links            {"url": "...", "custom_alias?": "...", "expires_at?": ...}  -> 201 {"code": "aZ3k9", "short_url": ...}
    GET    /{code}                  -> 301/302 Location: original_url  (cache in Redis)
    GET    /api/v1/links/{code}     -> metadata + click stats
    DELETE /api/v1/links/{code}     -> 204
    ```
    Code generation: base62 of an auto-increment ID or random 7 chars with uniqueness check. 302 if you need analytics on every click, 301 if cacheable. Rate-limit creation, validate URLs (block internal hosts).

??? question "35. Design a payments API"
    ```text
    POST /v1/payment_intents         Idempotency-Key required -> {id, status: requires_confirmation, amount, currency}
    POST /v1/payment_intents/{id}/confirm
    POST /v1/payment_intents/{id}/cancel
    POST /v1/refunds                 {payment_intent_id, amount}
    GET  /v1/payment_intents?cursor=...
    Webhooks: payment.succeeded, payment.failed (HMAC signed)
    ```
    Store money as integer minor units + ISO currency; state machine for status; idempotency keys; double-entry ledger table; audit logs; PCI scope minimised by tokenising cards via the processor.

??? question "36. Your API must call 3 downstream services per request. How do you keep latency low?"
    Call them concurrently with `asyncio.gather` and per-call timeouts; cache stable responses; set an overall deadline; degrade gracefully if a non-critical one fails; circuit breaker for the flaky one.
