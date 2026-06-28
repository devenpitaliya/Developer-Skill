# Code Quality Reference

## Function Design Laws (Universal)

### Law 1: One Thing. Named Exactly.

```python
# BAD
def validate_and_save_user(user): ...

# GOOD
def validate_user(user: RawUser) -> ValidationResult: ...
def save_user(user: ValidatedUser) -> User: ...
```

```typescript
// BAD
function processAndSendEmail(user) { ... }

// GOOD
function formatUserEmail(user: User): EmailPayload { ... }
async function sendEmail(payload: EmailPayload): Promise<void> { ... }
```

### Law 2: Keep Functions Short
- Aim for **under 20 lines**
- Max 40 lines before it's a code smell
- If it's longer, it's doing too much — extract

### Law 3: Arguments Tell a Story
- Max **3–4 arguments**
- More? Use a **config/request object**

```python
# BAD
def create_user(name, email, role, dept, manager_id, is_active, send_email): ...

# GOOD
@dataclass
class CreateUserRequest:
    name: str
    email: str
    role: UserRole
    department: str
    manager_id: Optional[str] = None
    is_active: bool = True
    send_welcome_email: bool = True

def create_user(request: CreateUserRequest) -> User: ...
```

```typescript
// BAD
function createUser(name: string, email: string, role: string, dept: string) {}

// GOOD
interface CreateUserRequest {
  name: string;
  email: string;
  role: UserRole;
  department: string;
  isActive?: boolean;
}
function createUser(request: CreateUserRequest): User {}
```

```go
// Go — use a struct
type CreateUserRequest struct {
    Name       string
    Email      string
    Role       UserRole
    Department string
    IsActive   bool
}
func (s *UserService) CreateUser(ctx context.Context, req CreateUserRequest) (*User, error) {}
```

### Law 4: Consistent Return Types
Never return `null | value` from the same code path.
Use Result/Option types or raise explicit exceptions — never both.

### Law 5: Pure Functions Where Possible
A function that takes inputs and returns outputs (no side effects) is:
- Trivially testable
- Safe to call in parallel
- Easy to reason about

---

## Naming Conventions

### Universal Principles
- Names describe **what**, not **how**
- Be as specific as the context demands
- Avoid: `data`, `result`, `temp`, `value`, `obj`, `x`, `flag`

### By Construct

| Construct | Convention | Example |
|---|---|---|
| Function / method | verb + noun | `fetchUserById`, `validateEmail` |
| Boolean | is/has/can/should | `isActive`, `hasPermission`, `canEdit` |
| Collection | plural noun | `users`, `pendingOrders` |
| Single item | singular noun | `user`, `currentOrder` |
| Constants | SCREAMING_SNAKE (Python/JS/Go) | `MAX_RETRY_COUNT` |
| Interface (Go/TS) | noun, no prefix | `UserRepository`, not `IUserRepository` |
| Class | PascalCase noun | `UserService`, `PaymentProcessor` |
| Private fields | language convention | `_repo` (Python), `#repo` (JS), unexported (Go) |

### Language-Specific

```python
# Python
snake_case for variables/functions/files
PascalCase for classes
SCREAMING_SNAKE for module-level constants
```

```typescript
camelCase for variables, functions, files
PascalCase for classes, interfaces, types, enums
SCREAMING_SNAKE for true constants
kebab-case for file names (optional, team-standard)
```

```go
camelCase for unexported (private)
PascalCase for exported (public)
Short but clear receiver names: u *User, s *Service
Avoid stuttering: user.UserID → user.ID
```

---

## Abstraction & DRY

### The Rule of Three
Don't abstract on the first duplication. Wait for the third occurrence.
Premature abstraction creates wrong abstractions — harder to undo than duplication.

### When to Abstract
- Same logic in 3+ places
- Clear, stable boundary (not "might change")
- The abstraction has a single, nameable purpose

### When NOT to Abstract
- Two things that look similar but serve different domains
- "Just in case" abstractions for futures that may never arrive
- When the abstraction makes the code harder to read than the duplication

---

## Comments & Documentation

### Comments Explain WHY, Not WHAT

```python
# BAD — describes what the code already says
# increment retry count
retry_count += 1

# GOOD — explains why
# Azure AD tokens expire silently after 55min; proactively refresh
# before the 60min hard limit to prevent mid-request failures
if token_age_seconds > 3300:
    await refresh_token()
```

### When to Write a Comment
- Non-obvious business rule: why, not what
- Workaround for external library bug — with link to issue
- Performance trade-off explanation
- Intentional "hack" with TODO and owner

### When NOT to Write a Comment
- When renaming the function/variable would make it obvious
- Restating the code in English
- Marking closing braces (`} // end if`)

---

## Code Review Standards

### Review Priority Order

1. **Correctness** — Does it solve the real problem? Edge cases handled?
2. **Security** — Injection, exposed secrets, missing auth, untrusted input?
3. **Performance** — N+1 queries, blocking I/O in async, missing indexes?
4. **Readability** — Can a new team member understand this in 5 minutes?
5. **Maintainability** — Easy to change without breaking other things?

### Comment Framework

```
BLOCKING (must fix before merge):
🔴 [BUG] This will panic when result is nil
🔴 [SECURITY] SQL injection — use parameterized query
🔴 [PERF] N queries in a loop — batch this

NON-BLOCKING (improve if easy, otherwise track):
🟡 [SUGGESTION] Extract to a helper — used 3 times
🟡 [CLARITY] `x` → `session_context`

DISCUSSION:
💬 [QUESTION] Why this approach over X?
💬 [IDEA] We might want to cache this later
```

### Green Flags vs Red Flags

| ✅ Green | 🔴 Red |
|---|---|
| Functions < 20 lines | 100-line functions |
| Constants for all literals | Magic strings / numbers |
| Explicit types everywhere | `any` typed everything |
| Single responsibility | "And" in function names |
| Errors handled or propagated | Empty catch / except blocks |
| Parameterized queries | String-formatted SQL |
| Tests for edge cases | No tests at all |
| One concern per PR | 5 features + 3 refactors in one PR |
