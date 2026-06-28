# Data & APIs Reference

## Database Design for Scale

### Schema Design Principles (Universal)

1. **Normalize first, denormalize for performance only with evidence**
2. **Every table gets a surrogate primary key** (UUID or auto-increment int)
3. **Use foreign keys** — enforce referential integrity at the DB level, not just app layer
4. **Timestamps on everything**: `created_at`, `updated_at` — non-negotiable
5. **Soft deletes with `deleted_at`** for anything that needs audit trails
6. **Never store computed values** unless caching is explicitly intentional

```sql
-- Universal table template
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) NOT NULL UNIQUE,
    status      VARCHAR(50)  NOT NULL DEFAULT 'active',
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ  -- NULL = active, non-NULL = soft-deleted
);

-- Always index foreign keys and filter columns
CREATE INDEX idx_users_email   ON users(email);
CREATE INDEX idx_users_status  ON users(status) WHERE deleted_at IS NULL;
```

### Indexing Strategy

| Situation | Index Type |
|---|---|
| Equality filter (`WHERE id = ?`) | B-tree (default) |
| Range filter (`WHERE created_at > ?`) | B-tree |
| Full-text search | GIN + tsvector (PostgreSQL) |
| Vector similarity | IVFFlat / HNSW (pgvector) |
| JSON key lookup | GIN (PostgreSQL jsonb) |
| Composite filter (col_a AND col_b) | Composite index — order matters |

**Index the column that reduces rows the most first in composites.**

### N+1 Query Prevention

```python
# BAD — N+1: 1 query for orders + N queries for users
orders = await order_repo.get_all()
for order in orders:
    order.user = await user_repo.find_by_id(order.user_id)  # N queries!

# GOOD — 2 queries total
orders = await order_repo.get_all()
user_ids = [o.user_id for o in orders]
users = await user_repo.find_by_ids(user_ids)
user_map = {u.id: u for u in users}
for order in orders:
    order.user = user_map.get(order.user_id)
```

### Migrations Best Practices

- **Always forward-only migrations** in production
- Never rename a column — add new, migrate data, remove old
- Never drop a column immediately — mark deprecated, remove in next release
- Large table migrations: add column with default first, backfill in batches, then apply NOT NULL
- Numbering: `001_create_users.sql`, `002_add_user_index.sql`

---

## API Design

### REST Principles (Non-Negotiable)

```
# Resource naming — nouns, not verbs
GET    /users              → list users
POST   /users              → create user
GET    /users/{id}         → get single user
PUT    /users/{id}         → replace user
PATCH  /users/{id}         → partial update
DELETE /users/{id}         → delete user

# Nested resources — max 2 levels deep
GET    /users/{id}/orders       ✅
GET    /users/{id}/orders/{oid} ✅
GET    /users/{id}/orders/{oid}/items/{iid}  ❌ — flatten this
```

### HTTP Status Codes (Use Correctly)

| Code | Meaning | Use For |
|---|---|---|
| 200 | OK | Successful GET, PATCH, PUT |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation errors, malformed input |
| 401 | Unauthorized | Missing or invalid auth token |
| 403 | Forbidden | Authenticated but not permitted |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate, state conflict |
| 422 | Unprocessable | Semantically invalid (passed schema, failed logic) |
| 429 | Rate Limited | Too many requests |
| 500 | Server Error | Unexpected internal failure |

### Consistent Error Response Shape

```json
{
  "error": {
    "code": "USER_EMAIL_TAKEN",
    "message": "A user with this email already exists.",
    "details": { "field": "email", "value": "dev@example.com" },
    "request_id": "req_abc123",
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

Rule: **always include a machine-readable `code` and a human-readable `message`.**

### API Versioning

- **URL versioning** (`/v1/`, `/v2/`) — explicit, easy to route, recommended for most APIs
- **Header versioning** (`API-Version: 2`) — cleaner URLs, harder to test in browser
- Never break an existing version — add `/v2/` instead
- Deprecate with `Sunset` header and documentation

### Pagination

```json
// Cursor-based (preferred for large/live datasets)
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6IjEyMyJ9",
    "has_more": true
  }
}

// Offset-based (simpler, fine for small stable datasets)
{
  "data": [...],
  "pagination": {
    "page": 2,
    "per_page": 20,
    "total": 438
  }
}
```

---

## Caching Strategy

### The Cache Decision Tree

```
Is the data expensive to compute or fetch?
  └─ NO  → Don't cache. Not worth the complexity.
  └─ YES → Is it read frequently?
      └─ NO  → Cache only if computation is very expensive
      └─ YES → How stale can it be?
          └─ Real-time required → Cache write-through + short TTL
          └─ Seconds OK         → Cache-aside pattern, TTL 30–60s
          └─ Minutes OK         → Cache-aside, TTL 5–30 min
          └─ Hours OK           → Pre-computed / background refresh
```

### Cache-Aside Pattern (Universal)

```python
async def get_user(user_id: str) -> User:
    # 1. Check cache
    cached = await cache.get(f"user:{user_id}")
    if cached:
        return User.parse(cached)

    # 2. Cache miss — fetch from source
    user = await user_repo.find_by_id(user_id)
    if not user:
        raise NotFoundError(f"User {user_id} not found")

    # 3. Populate cache
    await cache.set(f"user:{user_id}", user.serialize(), ttl=300)
    return user
```

### Cache Invalidation Rules

- **Invalidate on write**, not on read
- Use **consistent key naming**: `{entity}:{id}` or `{entity}:{scope}:{filter}`
- Never cache PII or secrets without encryption
- Set a **max TTL** on everything — no infinite cache entries
- Use **cache stampede protection** (lock or probabilistic early expiry) for hot keys

### What NOT to Cache

- Session tokens (use dedicated session store)
- Audit logs
- Financial transaction state
- Anything requiring real-time accuracy (inventory, seat availability)
