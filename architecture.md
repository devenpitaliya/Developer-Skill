# Architecture & System Design Reference

## Folder & Module Structure

### The Domain-First Rule (Universal)

```
# Python / FastAPI
app/
├── auth/
│   ├── models.py       # domain models
│   ├── service.py      # business logic
│   ├── router.py       # HTTP layer (thin)
│   └── repository.py  # DB queries only
├── payments/
├── notifications/
└── shared/
    ├── config/
    ├── constants/
    ├── enums/
    └── utils/

# TypeScript / Node
src/
├── auth/
│   ├── auth.model.ts
│   ├── auth.service.ts
│   ├── auth.controller.ts
│   └── auth.repository.ts
├── payments/
└── shared/

# Go
internal/
├── auth/
│   ├── handler.go
│   ├── service.go
│   ├── repository.go
│   └── model.go
├── payments/
└── shared/
```

### Layer Responsibilities (Non-Negotiable)

| Layer | Allowed | Forbidden |
|---|---|---|
| Entry (HTTP/CLI/Queue) | Parse input, call service, format response | Business logic, DB queries |
| Service/Domain | Business logic, orchestration | HTTP concerns, raw SQL |
| Repository/Data | DB queries, data mapping | Business logic, HTTP |
| Shared/Infra | Config, constants, utils | Domain logic |

Violations create hidden coupling that breaks at scale.

---

## System Design Principles

### The CAP Theorem in Practice
- **Consistency + Availability**: works when network is perfect (impossible at scale)
- **Consistency + Partition Tolerance**: choose this for financial data, inventory
- **Availability + Partition Tolerance**: choose this for social feeds, analytics, recommendations

Most systems need different strategies for different data domains. Design per domain.

### Design for Failure (Not Just Success)

Every distributed call can fail. For each external dependency, explicitly decide:

| Failure Strategy | When to Use | Example |
|---|---|---|
| Fail fast (throw) | Data integrity critical | Payment processing |
| Return default | UX can degrade gracefully | Personalization |
| Retry with backoff | Transient failures expected | 3rd party APIs |
| Circuit breaker | Cascading failure risk | Downstream microservices |
| Fallback | Must never go down | Auth, core CRUD |

### Scalability Levers (In Order of Reach)

1. **Fix the query** (indexes, N+1s, missing joins) — highest leverage, cheapest
2. **Add caching** (in-memory, distributed) — see `data_and_apis.md`
3. **Async the slow parts** (queues, background workers)
4. **Scale the service** (horizontal replicas)
5. **Partition the data** (sharding, read replicas) — hardest to undo

Never jump to step 4 or 5 before exhausting 1–3.

---

## Common Architectural Patterns

### Repository Pattern (Universal)
Decouples business logic from persistence. Works in any language/DB.

```python
# Python
class UserRepository:
    def __init__(self, db: Database):
        self._db = db

    async def find_by_id(self, user_id: str) -> Optional[User]:
        # ALL db logic here, zero business logic
        ...

    async def save(self, user: User) -> User:
        ...
```

```typescript
// TypeScript
class UserRepository {
  constructor(private db: Database) {}

  async findById(userId: string): Promise<User | null> { ... }
  async save(user: User): Promise<User> { ... }
}
```

```go
// Go
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) (*User, error)
}
```

### Service Layer Pattern (Universal)
Owns business logic. Depends on repository interfaces, not implementations.

```python
class UserService:
    def __init__(self, user_repo: UserRepository, email_svc: EmailService):
        self._user_repo = user_repo
        self._email_svc = email_svc

    async def register(self, request: CreateUserRequest) -> Result[User]:
        if await self._user_repo.find_by_email(request.email):
            return Result.fail("Email already registered")
        user = User.create(request)
        saved = await self._user_repo.save(user)
        await self._email_svc.send_welcome(saved.email)
        return Result.ok(saved)
```

### Result / Either Pattern (Universal)
Avoid returning `null` or raising exceptions for expected failure cases.

```python
# Python
@dataclass
class Result(Generic[T]):
    value: Optional[T]
    error: Optional[str]
    success: bool

    @classmethod
    def ok(cls, value: T) -> 'Result[T]':
        return cls(value=value, error=None, success=True)

    @classmethod
    def fail(cls, error: str) -> 'Result[T]':
        return cls(value=None, error=error, success=False)
```

```typescript
// TypeScript
type Result<T> =
  | { success: true; value: T }
  | { success: false; error: string };
```

```go
// Go — idiomatic (value, error) tuple
func (s *UserService) Register(ctx context.Context, req CreateUserRequest) (*User, error) {
    ...
}
```

---

## Microservices vs Monolith Decision

| Factor | Lean Monolith | Microservices |
|---|---|---|
| Team size | < 10 engineers | > 20 engineers |
| Deployment frequency | Weekly | Multiple times/day per team |
| Domain boundaries | Unclear | Well-defined and stable |
| Scaling needs | Uniform | Per-service |
| Operational maturity | Low | High |

**Default to a well-structured monolith.** Extract services only when you have clear pain that microservices solve. Premature decomposition is one of the most expensive architectural mistakes.

---

## Database Selection Guide

| Use Case | Recommended | Avoid |
|---|---|---|
| Relational data, ACID transactions | PostgreSQL | MongoDB |
| Full-text search | Elasticsearch, PostgreSQL FTS | Rolling your own |
| Vector similarity / RAG | pgvector, Pinecone, Weaviate | Standard RDBMS |
| Caching / session | Redis | PostgreSQL |
| Time series | TimescaleDB, InfluxDB | Standard RDBMS |
| Append-only event log | Kafka, PostgreSQL partitioned | Redis |
| Graph relationships | Neo4j, Amazon Neptune | Relational JOIN chains |

Use the right tool. Don't force a relational DB to do graph traversal.
