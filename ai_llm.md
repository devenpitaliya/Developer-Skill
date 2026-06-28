# AI / LLM Application Patterns Reference

## Core Principles for LLM Systems

1. **LLMs are non-deterministic external APIs** — treat them like any other external service (timeouts, retries, fallbacks, logging)
2. **Prompt is code** — version it, review it, test it
3. **Every LLM call must be observable** — tokens in/out, latency, model, prompt version, success/failure
4. **Validate LLM output** — never trust raw LLM output as structured data without parsing + validation
5. **Design for partial failure** — LLM may timeout, hallucinate, or return malformed output

---

## Prompt Management

### Never Hardcode Prompts Inline

```python
# BAD — prompt buried in logic, not versioned
async def analyze(query: str) -> str:
    response = await llm.invoke(f"Analyze this query and return JSON: {query}")
    ...

# GOOD — prompt as a versioned artifact
# prompts/analysis_prompt.py
ANALYSIS_PROMPT_V2 = """
You are an expert data analyst. Analyze the user query and return a JSON object.

## Query
{query}

## Instructions
- Identify the core intent
- Extract relevant entities
- Return ONLY valid JSON — no markdown, no preamble

## Output Format
{
  "intent": "...",
  "entities": [...],
  "confidence": 0.0-1.0
}
""".strip()
```

### Prompt Versioning Strategy

```
prompts/
├── analysis/
│   ├── v1.py     # deprecated
│   ├── v2.py     # current
│   └── README.md # changelog
├── sql_gen/
│   └── v3.py
└── router/
    └── v1.py
```

Track version in logs: `prompt_version="analysis_v2"`

---

## Structured Output Extraction (Universal)

Never trust raw LLM text as structured data. Always parse + validate.

```python
# Python — Pydantic-based structured output
from pydantic import BaseModel, ValidationError
import json

class AnalysisResult(BaseModel):
    intent: str
    entities: list[str]
    confidence: float

async def get_analysis(query: str) -> AnalysisResult:
    raw = await llm.invoke(ANALYSIS_PROMPT_V2.format(query=query))

    # Clean common LLM output artifacts
    text = raw.strip()
    if text.startswith("```"):
        text = text.split("```")[1]
        if text.startswith("json"):
            text = text[4:]

    try:
        data = json.loads(text)
        return AnalysisResult(**data)
    except (json.JSONDecodeError, ValidationError) as e:
        logger.error("LLM output parsing failed", raw_output=raw, error=str(e))
        raise LLMOutputParseError(f"Failed to parse LLM response: {e}")
```

```typescript
// TypeScript — Zod-based structured output
import { z } from 'zod';

const AnalysisSchema = z.object({
  intent: z.string(),
  entities: z.array(z.string()),
  confidence: z.number().min(0).max(1),
});

async function getAnalysis(query: string): Promise<z.infer<typeof AnalysisSchema>> {
  const raw = await llm.invoke(ANALYSIS_PROMPT.replace('{query}', query));

  const parsed = JSON.parse(cleanLLMOutput(raw));
  return AnalysisSchema.parse(parsed);
}
```

---

## LLM Call Wrapper (Observable, Retriable)

```python
import time
import asyncio
from typing import Optional

async def invoke_llm(
    prompt: str,
    model: str,
    prompt_version: str,
    timeout_seconds: float = 30.0,
    max_retries: int = 2,
    temperature: float = 0.0,
) -> str:
    start = time.monotonic()
    last_error = None

    for attempt in range(max_retries + 1):
        try:
            async with asyncio.timeout(timeout_seconds):
                response = await llm_client.invoke(
                    prompt=prompt,
                    model=model,
                    temperature=temperature,
                )

            elapsed_ms = int((time.monotonic() - start) * 1000)
            logger.info(
                "llm_call_success",
                model=model,
                prompt_version=prompt_version,
                input_tokens=response.usage.input_tokens,
                output_tokens=response.usage.output_tokens,
                latency_ms=elapsed_ms,
                attempt=attempt + 1,
            )
            return response.content

        except asyncio.TimeoutError as e:
            last_error = e
            logger.warning("llm_call_timeout", model=model, attempt=attempt + 1)
        except RateLimitError as e:
            last_error = e
            await asyncio.sleep(2 ** attempt)
        except Exception as e:
            logger.error("llm_call_error", model=model, error=str(e), exc_info=True)
            raise

    raise LLMCallError(f"LLM call failed after {max_retries+1} attempts") from last_error
```

---

## RAG (Retrieval Augmented Generation) Patterns

### Hybrid Search (Semantic + Keyword)

Pure vector search misses exact matches. Pure keyword search misses semantic similarity. Hybrid wins.

```python
async def hybrid_search(query: str, top_k: int = 10) -> list[Document]:
    # Run both searches in parallel
    semantic_results, keyword_results = await asyncio.gather(
        vector_store.similarity_search(query, k=top_k),
        bm25_index.search(query, k=top_k),
    )

    # Reciprocal Rank Fusion — language-agnostic reranking
    scores: dict[str, float] = {}
    k = 60  # RRF constant

    for rank, doc in enumerate(semantic_results):
        scores[doc.id] = scores.get(doc.id, 0) + 1 / (k + rank + 1)

    for rank, doc in enumerate(keyword_results):
        scores[doc.id] = scores.get(doc.id, 0) + 1 / (k + rank + 1)

    all_docs = {d.id: d for d in semantic_results + keyword_results}
    ranked = sorted(scores.keys(), key=lambda id: scores[id], reverse=True)
    return [all_docs[id] for id in ranked[:top_k]]
```

### Context Window Management

```python
def build_context(docs: list[Document], max_tokens: int = 3000) -> str:
    """
    Fit retrieved documents into available context window.
    Most relevant docs first; stop before exceeding budget.
    """
    context_parts = []
    token_count = 0

    for doc in docs:
        doc_tokens = estimate_tokens(doc.content)  # ~4 chars/token rule of thumb
        if token_count + doc_tokens > max_tokens:
            break
        context_parts.append(f"[Source: {doc.source}]\n{doc.content}")
        token_count += doc_tokens

    return "\n\n---\n\n".join(context_parts)
```

---

## Agentic Systems Patterns

### Tool Design Principles

```python
# Tools should be:
# 1. Single-purpose
# 2. Idempotent where possible
# 3. Observable (log every invocation)
# 4. Validated at input AND output

async def execute_sql(query: str) -> dict:
    """
    Execute a validated SELECT query against the analytics DB.
    ONLY SELECT statements allowed — mutation is forbidden.
    """
    if not is_safe_select(query):
        raise ToolError("Only SELECT statements are permitted")

    logger.info("tool_execute_sql", query_length=len(query))

    async with asyncio.timeout(30.0):
        result = await db.fetch(query)

    logger.info("tool_execute_sql_success", row_count=len(result))
    return {"rows": result, "count": len(result)}
```

### State Schema Design (LangGraph / Agentic)

```python
@dataclass
class AgentState:
    # Identity
    session_id: str
    user_id: str

    # Input
    user_query: str

    # Processing state
    current_step: str = "init"
    iteration_count: int = 0

    # Intermediate results
    retrieved_context: list[Document] = field(default_factory=list)
    generated_sql: Optional[str] = None
    sql_result: Optional[dict] = None

    # Output
    final_answer: Optional[str] = None
    error_message: Optional[str] = None

    # Guardrails
    MAX_ITERATIONS: ClassVar[int] = 10

    def is_stuck(self) -> bool:
        return self.iteration_count >= self.MAX_ITERATIONS
```

### LLM Fallback Chain

```python
async def invoke_with_fallback(prompt: str) -> str:
    """
    Try fast/cheap model first. Fall back to capable model on failure.
    """
    try:
        return await invoke_llm(
            prompt, model="gpt-4o-mini", timeout_seconds=5.0
        )
    except (LLMCallError, asyncio.TimeoutError):
        logger.warning("Fast model failed, falling back to main model")
        return await invoke_llm(
            prompt, model="gpt-4o", timeout_seconds=30.0
        )
```

---

## LLM Observability Checklist

Instrument every LLM call with:
```
□ model used
□ prompt version
□ input token count
□ output token count
□ latency (ms)
□ success / failure
□ error type (if failed)
□ correlation/request ID
□ retry attempt number
```

Tools: Langfuse, LangSmith, Azure Application Insights + custom spans, OpenTelemetry.

---

## Common LLM Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Inline prompts | Not versioned, not reviewable | Centralize in `prompts/` |
| Trust raw LLM output | Crashes on malformed JSON | Always parse + validate |
| No timeout | Hangs indefinitely | Always set timeout |
| Sync LLM calls in async code | Blocks thread pool | Use async client |
| One massive prompt | Hard to debug, expensive | Split by concern |
| No token tracking | Surprise cost bills | Log tokens every call |
| No fallback | Single point of failure | Fallback chain |
| LLM for everything | Slow + expensive for simple ops | Use LLM only where needed |
