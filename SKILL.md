---
name: elite-software-dev
description: >
  Elite, language-agnostic software engineering playbook. Use this skill whenever
  the user asks you to write, review, design, or reason about code or software systems —
  regardless of language (Python, TypeScript, Go, Java, Rust, etc.), paradigm, or stack.
  Triggers on: writing functions or classes, designing APIs or databases, reviewing code,
  architecting systems, debugging production issues, refactoring, building AI/LLM apps,
  async/concurrency problems, security reviews, testing strategies, observability setup,
  or any task where producing high-quality, maintainable, production-grade code matters.
  Always use this skill before generating non-trivial code (>20 lines).
---

# Elite Software Development Skill

> Language-agnostic. Stack-agnostic. Principle-first.
> Apply these principles before writing a single line of code.

---

## How to Use This Skill

This skill has a **two-layer structure**:

1. **This file** — Core principles and decision framework. Always read this first.
2. **Reference modules** — Deep patterns for specific concerns. Load only what's needed.

| Reference File | Load When... |
|---|---|
| `references/architecture.md` | Designing structure, folder layout, domain modeling, system design |
| `references/code_quality.md` | Writing functions, naming, abstractions, code review |
| `references/data_and_apis.md` | Database design, API contracts, caching, querying |
| `references/reliability.md` | Error handling, async/concurrency, security, testing |
| `references/observability.md` | Logging, tracing, metrics, alerting |
| `references/ai_llm.md` | LLM/agent applications, prompt management, RAG, agentic systems |

---

## The Foundation: How Elite Engineers Think

### Three Questions Before Any Code

1. **What problem am I actually solving?** (Not the ticket — the real underlying need)
2. **What is the simplest correct solution?** (Not the most impressive one)
3. **What breaks this at 10x scale?** (Think in growth, not just today)

### The Complexity Budget

Every system has a finite complexity budget. Spend it wisely:

- Solve today's needs + 10x growth = **correct**
- Design for 1000x before you have 100 users = **waste**
- Solve a problem you don't have yet = **tech debt disguised as engineering**

Ruthlessly favor simplicity. The best code is often the code you didn't write.

### Think in Reversibility

| Decision | Reversibility | Time to Spend |
|---|---|---|
| Variable name | Minutes | Seconds |
| Function signature | Hours | 2 min |
| Module boundary | Days | 15 min |
| Database schema | Weeks | 1 hour |
| API contract | Months | Half a day |
| Core architecture | Never | Get it right |

**Invest thinking time proportional to reversal cost.** Obsess over the bottom. Don't bikeshed the top.

---

## Universal Code Quality Laws

These apply in every language, every paradigm:

### 1. Single Responsibility
Every unit — function, class, module, service — does **one thing**.
If you need "and" to describe what it does, split it.

### 2. Explicit Over Implicit
If something important happens, make it visible in code.
Not hidden in a default, a global, or a magic import side-effect.

### 3. Name Things What They Are
`user_session_token` beats `token`.
`sql_generation_timeout_ms` beats `timeout`.
Names are the first documentation a reader sees.

### 4. Coupling Is Debt
Every time module A knows module B's internals, you've paid for it later.
Depend on interfaces and contracts, not implementations.

### 5. Design the Failure Mode First
Every external call **will** fail eventually.
Decide: loud (throw/panic), silent (return default), or recovered (retry)?
Decide this before you code the success path.

### 6. Test Behavior, Not Implementation
Tests that break when you refactor internals are worse than no tests.
Test what the code **does**, not how it does it.

### 7. Observability Is a Feature
If you can't see what the system is doing in production, you can't fix it.
Log, trace, and instrument from day one — not as an afterthought.

### 8. Make It Work. Make It Right. Make It Fast.
In that order. Premature optimization is the root of all evil.

---

## Stack-Agnostic Project Structure Principles

Regardless of language or framework, these structural rules hold:

**Group by domain/feature, not by type.**
```
# WRONG — type-first (breaks at scale)
models/ controllers/ services/ utils/

# RIGHT — domain-first (survives 200+ files)
auth/ payments/ analytics/ notifications/ shared/
```

**Layer separation is non-negotiable:**
- **Entry layer** (HTTP, CLI, queue consumer) — translation only, zero business logic
- **Service/domain layer** — all business logic lives here
- **Data layer** — all queries/persistence live here, nowhere else
- **Shared/infra layer** — config, constants, utilities, cross-cutting concerns

**Config and secrets:**
- One config file per external system
- Secrets always from environment / vault — never hardcoded
- Always commit `.env.example`, never `.env`

---

## The Pre-Code Checklist

Before writing any non-trivial piece of code, confirm:

```
□ Is the problem clearly defined? (not just the symptom)
□ Is this the simplest correct solution?
□ Are the failure modes designed first?
□ Is the domain boundary right? (what owns this logic?)
□ What's the test strategy?
□ Will this be observable in production?
□ Is there a simpler existing solution (library, config, pattern)?
```

---

## The Pre-PR Checklist

```
STRUCTURE
□ Code is in the right layer (no business logic in HTTP handlers)
□ No persistence queries outside the data layer
□ No magic strings/numbers — use constants/enums

FUNCTIONS
□ Each function does one thing
□ Names exactly describe behavior
□ Return types are explicit and consistent

RELIABILITY
□ All exceptions caught or propagated intentionally
□ No empty catch/except blocks
□ External calls have timeouts and error handling

SECURITY
□ User input is validated at the boundary
□ Queries use parameterized inputs (no string interpolation)
□ No secrets in logs, responses, or error messages

ASYNC
□ Independent I/O operations run concurrently where possible
□ No blocking calls inside async/concurrent contexts
□ All external calls have timeouts

TESTS
□ Business logic has unit tests
□ Edge cases covered (null, empty, zero, boundary values)
□ Integration tests for data layer

OBSERVABILITY
□ Key decisions and state transitions are logged
□ No PII or secrets in log output
□ Errors logged with full context (stack trace, correlation ID)
```

---

## The 10 Laws Every Senior Engineer Follows

1. **Make it work first. Make it right second. Make it fast third.**
2. **The best code is no code.** Ask: can I solve this with config, a library, a simpler approach?
3. **Names are documentation.** A well-named function needs no comment.
4. **Explicit beats implicit.** Visible is debuggable.
5. **Coupling is debt.** Depend on contracts, not internals.
6. **Design failure modes first.** Every external call will fail.
7. **Move fast on reversible decisions. Slow down on irreversible ones.**
8. **Test behavior, not implementation.**
9. **Leave the code better than you found it.** (Boy Scout Rule)
10. **Observability is a feature.** If you can't see it, you can't fix it.

---

*This is the foundation. Load reference modules for domain-specific depth.*
