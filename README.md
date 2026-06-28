# 🛠️ Elite Software Development Skill

> A language-agnostic engineering playbook for building production-grade software.  
> Distilled from real-world systems, FAANG engineering practices, and first-principles thinking.

---

## What Is This?

This is an open-source **Claude Skill** — a structured knowledge file that tells Claude how to think and write code like a senior engineer.

But even without Claude, the markdown files are a **standalone reference** you can read, bookmark, and share with your team.

Works for any language. Any stack. Any paradigm.

---

## What's Inside

```
elite-software-dev/
├── SKILL.md                    ← Master file: core laws, checklists, decision framework
└── references/
    ├── architecture.md         ← Folder structure, system design, DB selection
    ├── code_quality.md         ← Functions, naming, abstractions, code review
    ├── data_and_apis.md        ← DB schema, API design, caching strategy
    ├── reliability.md          ← Error handling, async, security, testing
    ├── observability.md        ← Logging, metrics, tracing, alerting
    └── ai_llm.md               ← LLM/agent patterns, RAG, prompt management
```

---

## Topics Covered

| Module | Topics |
|---|---|
| **Architecture** | Domain-first folder structure, layered architecture, system design patterns, microservices vs monolith, DB selection guide |
| **Code Quality** | Function design laws, naming conventions, abstraction rules, comment philosophy, code review framework |
| **Data & APIs** | Database schema design, indexing strategy, REST principles, HTTP status codes, pagination, caching patterns |
| **Reliability** | Error handling strategies, retry with backoff, async/concurrency patterns, security fundamentals, testing pyramid |
| **Observability** | Structured logging, correlation IDs, the four golden signals, distributed tracing with OpenTelemetry, alerting rules |
| **AI / LLM** | Prompt versioning, structured output extraction, RAG with hybrid search, agentic state design, LLM observability |

---

## Language Coverage

Every pattern includes examples in:

- 🐍 **Python**
- 🟦 **TypeScript / JavaScript**
- 🐹 **Go**

The principles apply equally to Java, Rust, C#, and any other language.

---

## How to Use

### Option 1 — Read the Markdown (No Setup)

Just browse the files directly on GitHub. Use them as a reference while coding.

Start here → [`SKILL.md`](./elite-software-dev/SKILL.md)

### Option 2 — Install as a Claude Skill

If you use [Claude](https://claude.ai) or [Claude Code](https://claude.ai/code):

1. Download [`elite-software-dev.skill`](./elite-software-dev.skill)
2. Install it in your Claude skills directory
3. Claude will now follow these engineering standards when writing or reviewing code for you

### Option 3 — Clone and Customize

```bash
git clone https://github.com/YOUR_USERNAME/elite-software-dev.git
```

Edit the reference files to match your team's conventions, then install as a personal skill.

---

## Core Principles (Preview)

```
1. Make it work. Make it right. Make it fast. (In that order.)
2. The best code is no code.
3. Names are documentation.
4. Explicit beats implicit.
5. Coupling is debt.
6. Design the failure mode first.
7. Move fast on reversible decisions. Slow down on irreversible ones.
8. Test behavior, not implementation.
9. Leave the code better than you found it.
10. Observability is a feature.
```

---

## Pre-PR Checklist (Preview)

```
□ No business logic in HTTP handlers
□ No DB queries outside the data layer
□ No magic strings or numbers — use constants/enums
□ Each function does one thing
□ All exceptions caught or propagated intentionally
□ External calls have timeouts
□ User input validated at the boundary
□ No secrets in logs or error messages
□ Business logic has unit tests
□ Key decisions are logged with context
```

---

## Contributing

Found a gap? Have a pattern worth adding?

1. Fork the repo
2. Add or improve a reference module
3. Open a PR with a clear description of what you added and why

All contributions should be language-agnostic and backed by real-world usage.

---

## License

MIT — free to use, modify, and distribute.

---

## Author

Built by **Dev** — Senior GenAI Engineer building production AI systems.

Follow on LinkedIn for more AI engineering content → www.linkedin.com/in/devendra-pitaliya

---

⭐ Star this repo if it helped you. It helps others find it.
