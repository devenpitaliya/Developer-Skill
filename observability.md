# Observability Reference

## The Three Pillars

| Pillar | What it answers | Tools |
|---|---|---|
| **Logs** | What happened, and when? | Structured JSON logs → ELK, Loki, CloudWatch |
| **Metrics** | How is the system performing? | Prometheus, Datadog, Azure Monitor |
| **Traces** | Where did this request go and why was it slow? | OpenTelemetry, Jaeger, Zipkin, Datadog APM |

All three are required for production systems. Logs alone are insufficient.

---

## Structured Logging

### Always Log JSON in Production

```python
# Python — use structlog or stdlib with JSON formatter
import structlog

logger = structlog.get_logger()

# Good structured log
logger.info(
    "user_registered",
    user_id=user.id,
    email_domain=user.email.split("@")[1],  # never log full email
    registration_source=source,
    duration_ms=elapsed_ms,
)

# Bad — unstructured, unsearchable
logger.info(f"User {user.id} registered successfully in {elapsed_ms}ms")
```

```typescript
// TypeScript — use pino or winston
const logger = pino({ level: 'info' });

logger.info({ userId: user.id, durationMs }, 'user_registered');
```

```go
// Go — use zap or zerolog
logger.Info("user_registered",
    zap.String("user_id", user.ID),
    zap.Int64("duration_ms", elapsedMs),
)
```

### Log Level Guide

| Level | Use For | Volume |
|---|---|---|
| `ERROR` | Unexpected failures requiring attention | Low |
| `WARNING` | Expected failures, degraded behavior | Medium |
| `INFO` | Key business events, state transitions | Low-Medium |
| `DEBUG` | Detailed flow, useful for local dev | High (never in prod) |

### What to Log

```
□ Service startup (config summary, no secrets)
□ Incoming request (method, path, user_id, request_id — never body by default)
□ External call initiated (target, operation)
□ External call result (duration_ms, status, error if any)
□ Business-critical decisions (which path was taken, why)
□ Errors — with full context and stack trace
□ Service shutdown
```

### What NEVER to Log

```
□ Passwords, tokens, API keys, secrets
□ Full PII (email, phone, SSN, DOB)
□ Payment card data
□ Full request/response bodies by default (configurable at DEBUG only)
□ Health check pings (noise)
```

---

## Correlation IDs

Every request gets a unique ID that flows through every log line, every downstream service.

```python
# Python — FastAPI middleware
import uuid
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar('request_id', default='')

@app.middleware("http")
async def add_correlation_id(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    request_id_var.set(request_id)
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response
```

Downstream services: always pass the correlation ID in headers. Log it in every line.

---

## Metrics

### The Four Golden Signals (Google SRE)

| Signal | Description | Alert On |
|---|---|---|
| **Latency** | Time to serve a request | p95, p99 exceeds SLA |
| **Traffic** | Request rate (RPS) | Sudden spikes or drops |
| **Errors** | Error rate (%) | > 1% error rate |
| **Saturation** | How "full" is the service (CPU, memory, queue depth) | > 80% sustained |

These four cover 95% of production incidents.

### Key Metrics to Track

```
HTTP:
  - request_duration_seconds (histogram, by endpoint + status)
  - requests_total (counter, by method + endpoint + status)
  - active_connections (gauge)

Database:
  - query_duration_seconds (histogram, by operation)
  - connection_pool_active (gauge)
  - connection_pool_waiting (gauge)

Business:
  - {entity}_created_total
  - {operation}_success_total / {operation}_failure_total

LLM/AI (if applicable):
  - llm_tokens_input_total
  - llm_tokens_output_total
  - llm_latency_seconds
  - llm_error_total (by model + error_type)
```

---

## Distributed Tracing (OpenTelemetry)

OpenTelemetry is the universal standard. Works with any backend (Jaeger, Zipkin, Datadog, Azure Monitor).

```python
# Python — instrument at the service boundary
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

async def run_analysis(query: str) -> AnalysisResult:
    with tracer.start_as_current_span("run_analysis") as span:
        span.set_attribute("query.length", len(query))
        span.set_attribute("user.id", current_user_id)

        result = await _execute_pipeline(query)

        span.set_attribute("result.status", result.status)
        return result
```

### Span Naming Convention

```
{service}.{operation}
analytics.run_pipeline
user.register
payment.process_charge
```

---

## Alerting Rules

### Alert Criteria

```
□ Alert only on conditions that require human action
□ Alert on symptoms, not causes (high error rate > alert; high CPU > investigate first)
□ Every alert has a runbook link
□ Alerts have a severity: P1 (wake up), P2 (urgent business hours), P3 (track)
□ Alert fatigue kills oncall — prune noisy alerts aggressively
```

### Alert → Runbook Structure

```markdown
## Alert: high_error_rate

**Trigger**: error_rate > 1% for 5 minutes
**Severity**: P1

### Immediate Steps
1. Check recent deployments (last 30 min)
2. Check downstream dependency health
3. Check error logs for pattern

### Escalation
If not resolved in 15 min → escalate to [owner]
```

---

## Health Check Endpoints

Every service exposes at minimum:

```
GET /health/live    → 200 if process is running (liveness)
GET /health/ready   → 200 if all dependencies are healthy (readiness)
```

```python
# Python — FastAPI
@router.get("/health/ready")
async def readiness():
    checks = {
        "database": await check_db(),
        "cache": await check_redis(),
    }
    all_healthy = all(checks.values())
    return JSONResponse(
        status_code=200 if all_healthy else 503,
        content={"status": "ready" if all_healthy else "degraded", "checks": checks}
    )
```

Kubernetes, load balancers, and uptime monitors rely on these.
