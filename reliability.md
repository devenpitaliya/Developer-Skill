# Reliability Reference

## Error Handling

### The Four Strategies (Universal)

| Strategy | When to Use | Example |
|---|---|---|
| **Fail fast** (throw/panic/return error) | Integrity critical, caller must know | DB write failed, invalid token |
| **Return default** | UX can degrade gracefully | Missing preference → default value |
| **Retry with backoff** | Transient failures expected | 3rd-party API timeouts |
| **Circuit breaker** | Cascading failure risk | Downstream service unavailable |

Decide the strategy **before** writing the success path.

### Error Hierarchy Design

Define a consistent error hierarchy per project:

```python
# Python
class AppError(Exception):
    def __init__(self, message: str, code: str, status_code: int = 500):
        self.message = message
        self.code = code
        self.status_code = status_code

class NotFoundError(AppError):
    def __init__(self, resource: str, id: str):
        super().__init__(f"{resource} '{id}' not found", "NOT_FOUND", 404)

class ValidationError(AppError):
    def __init__(self, field: str, reason: str):
        super().__init__(f"Validation failed: {field} — {reason}", "VALIDATION_FAILED", 400)

class ConflictError(AppError):
    def __init__(self, message: str):
        super().__init__(message, "CONFLICT", 409)
```

```typescript
// TypeScript
class AppError extends Error {
  constructor(
    public message: string,
    public code: string,
    public statusCode: number = 500
  ) { super(message); }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} '${id}' not found`, 'NOT_FOUND', 404);
  }
}
```

```go
// Go — idiomatic errors with sentinel values
type AppError struct {
    Code       string
    Message    string
    StatusCode int
    Cause      error
}

func (e *AppError) Error() string { return e.Message }
func (e *AppError) Unwrap() error { return e.Cause }

var ErrNotFound = &AppError{Code: "NOT_FOUND", StatusCode: 404}
```

### Never Swallow Errors

```python
# NEVER do this
try:
    result = await risky_call()
except Exception:
    pass  # silently fails — debugging nightmare

# ALWAYS log and decide
try:
    result = await risky_call()
except SpecificError as e:
    logger.warning("risky_call failed, using fallback", error=str(e))
    result = fallback_value
except Exception as e:
    logger.error("Unexpected error in risky_call", exc_info=True)
    raise
```

### Retry with Exponential Backoff

```python
import asyncio
from typing import TypeVar, Callable

T = TypeVar('T')

async def retry(
    fn: Callable,
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    retryable_exceptions: tuple = (TimeoutError, ConnectionError),
) -> T:
    for attempt in range(max_attempts):
        try:
            return await fn()
        except retryable_exceptions as e:
            if attempt == max_attempts - 1:
                raise
            delay = min(base_delay * (2 ** attempt), max_delay)
            logger.warning(f"Attempt {attempt+1} failed, retrying in {delay}s", error=str(e))
            await asyncio.sleep(delay)
```

---

## Async & Concurrency

### Golden Rules

1. **Never block an async thread** — no `time.sleep()`, no sync I/O in async context
2. **Run independent I/O in parallel** — use `gather` / `Promise.all` / goroutines
3. **Every external call gets a timeout** — no exceptions
4. **Don't share mutable state between concurrent tasks** — use queues or immutable data

### Parallel I/O (Universal Pattern)

```python
# Python — asyncio.gather
user, orders, prefs = await asyncio.gather(
    user_repo.find_by_id(user_id),
    order_repo.find_by_user(user_id),
    preference_repo.find_by_user(user_id),
)
```

```typescript
// TypeScript — Promise.all
const [user, orders, prefs] = await Promise.all([
  userRepo.findById(userId),
  orderRepo.findByUser(userId),
  prefRepo.findByUser(userId),
]);
```

```go
// Go — goroutines + errgroup
g, ctx := errgroup.WithContext(ctx)
var user *User
var orders []*Order

g.Go(func() error {
    var err error
    user, err = userRepo.FindByID(ctx, userID)
    return err
})
g.Go(func() error {
    var err error
    orders, err = orderRepo.FindByUser(ctx, userID)
    return err
})
if err := g.Wait(); err != nil { return err }
```

### Timeout Everywhere

```python
# Python
async with asyncio.timeout(5.0):  # Python 3.11+
    result = await external_call()

# Or with httpx
async with httpx.AsyncClient(timeout=5.0) as client:
    response = await client.get(url)
```

```typescript
const result = await Promise.race([
  externalCall(),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), 5000)
  ),
]);
```

---

## Security Fundamentals

### Input Validation (Always at the Boundary)

- Validate at the **entry point** — HTTP handler, queue consumer, CLI args
- Use schema validation libraries: Pydantic (Python), Zod (TS), go-playground/validator (Go)
- **Whitelist** allowed values, don't blacklist bad values
- Validate type, length, format, range — not just presence

### Parameterized Queries (No Exceptions)

```python
# NEVER
query = f"SELECT * FROM users WHERE email = '{email}'"

# ALWAYS
query = "SELECT * FROM users WHERE email = $1"
result = await db.fetch(query, email)
```

```typescript
// NEVER
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ALWAYS
const result = await db.query('SELECT * FROM users WHERE email = $1', [email]);
```

### Secrets Management

- Secrets **never** in source code, ever
- Secrets from environment variables or a vault (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault)
- Secrets **never** in logs or error messages
- Rotate secrets regularly — build rotation support from day one

### Auth Checklist

```
□ Authentication: who are you? (JWT, session, API key)
□ Authorization: what can you do? (RBAC, ABAC, scopes)
□ Verify auth on EVERY protected endpoint — never assume middleware ran
□ Tokens expire and rotate
□ Rate limiting on auth endpoints
□ Audit log on sensitive actions (login, role change, delete)
```

---

## Testing Philosophy

### The Testing Pyramid

```
         /\
        /E2E\        — 10% — critical user journeys only, slow
       /------\
      / Intgr  \     — 30% — service + real DB, medium speed
     /----------\
    / Unit Tests \   — 60% — pure business logic, fast
   /--------------\
```

### What Deserves Tests

| Priority | What to Test |
|---|---|
| Critical | Business logic with conditions (if/else in service) |
| Critical | Data transformations (parsing, calculation, mapping) |
| High | Repository queries (against a real test DB) |
| High | API contract (request/response schema) |
| Medium | Error paths and edge cases |
| Low | Simple getters/setters |
| Skip | Framework code, third-party library internals |

### Unit Test Pattern (Universal)

```python
# Python
class TestUserService:
    @pytest.fixture
    def service(self):
        user_repo = AsyncMock(spec=UserRepository)
        email_svc = AsyncMock(spec=EmailService)
        return UserService(user_repo=user_repo, email_svc=email_svc)

    async def test_register_success(self, service):
        service._user_repo.find_by_email.return_value = None
        service._user_repo.save.return_value = User(id="u1", email="dev@example.com")

        result = await service.register(CreateUserRequest(name="Dev", email="dev@example.com"))

        assert result.success is True
        assert result.value.email == "dev@example.com"
        service._email_svc.send_welcome.assert_called_once()

    async def test_register_duplicate_email(self, service):
        service._user_repo.find_by_email.return_value = User(id="u0", email="dev@example.com")

        result = await service.register(CreateUserRequest(name="Dev", email="dev@example.com"))

        assert result.success is False
        assert "already registered" in result.error
```

```typescript
// TypeScript
describe('UserService', () => {
  let service: UserService;
  let mockUserRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockUserRepo = { findByEmail: jest.fn(), save: jest.fn() } as any;
    service = new UserService(mockUserRepo);
  });

  it('registers new user successfully', async () => {
    mockUserRepo.findByEmail.mockResolvedValue(null);
    mockUserRepo.save.mockResolvedValue({ id: 'u1', email: 'dev@x.com' });

    const result = await service.register({ name: 'Dev', email: 'dev@x.com' });

    expect(result.success).toBe(true);
    expect(mockUserRepo.save).toHaveBeenCalledTimes(1);
  });
});
```

### Test Naming Convention

```
test_{what}_when_{condition}_should_{expected_outcome}
test_register_when_email_exists_should_return_conflict_error
```

### Integration Test Rules

- Use a **real test DB** — not mocks
- **Truncate tables** between tests — never share state
- Run against the same DB engine as production (not SQLite if prod is PostgreSQL)
- Mark slow tests and exclude from pre-commit: `@pytest.mark.integration`
