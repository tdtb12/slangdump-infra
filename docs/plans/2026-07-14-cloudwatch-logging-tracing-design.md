# CloudWatch Logging and Cross-Service Trace Correlation

**Date:** 2026-07-14
**Status:** Approved, pending implementation
**Repos affected:** `slangdump-pick-n-roll` (Spring backend), `slangdump-ai-solation` (Python AI worker), `slangdump` (root, docs + compose)

## Context

**There is no log store.** The Spring backend has no logging configuration at all — no
`logback-spring.xml`, no `logging:` block in `application.yml`, no file appender. It runs on
Spring Boot's default Logback console appender, so every log line goes to **stdout and nowhere
else**. Exactly two classes log: `ApiExceptionHandler` (validation warnings and
`log.error("Unhandled exception", e)`) and `AIService` (translation failures). The Python AI
worker has no logging configuration either.

Locally this is survivable: Docker's `json-file` driver captures stdout, so `docker compose logs`
works. **On ECS it is not.** A Fargate task with no log driver configured writes stdout into the
void; when the task stops, the logs are gone. An unhandled 500 in production would leave no stack
trace to read.

Separately, `spring.jpa.show-sql: true` is enabled unconditionally. Hibernate's `show-sql` writes
to `System.out` directly, bypassing SLF4J — so it neither respects log levels nor passes through
any formatter.

## Goals

- Ship backend and AI-worker logs to CloudWatch Logs, queryable by field in Logs Insights.
- Correlate log lines across **both** services, so one request's full story — including the slow
  Gemini translation — can be retrieved by a single trace ID.
- Require no application code change to move between environments.

## Non-goals

- **Provisioning AWS.** No Terraform, ECS task definitions, or IAM policies are created here.
  This spec makes the *applications* ready and records the exact log group names, log driver
  options, and IAM permissions that infrastructure must honour. Same posture as the JWT/Parameter
  Store spec.
- **CloudWatch Metrics and Alarms.** Explicitly deferred. Logging is a separable subsystem.
- **A trace backend** (X-Ray, OTel collector, Jaeger). No spans are exported anywhere. Trace IDs
  exist *only* to correlate log lines. This is a deliberate choice: it delivers the value (follow
  one request across both services) with no collector, no sidecar, and no per-trace cost. An
  exporter can be added later without changing application code.

## Design

### Two findings in the existing code that this design must work around

These are the load-bearing risks. Both would cause tracing to fail *silently* — logs would still
appear, just without correlation, which is the failure you notice last and need most.

1. **`AIService` builds its `RestTemplate` by hand:**
   `RestTemplate(SimpleClientHttpRequestFactory().apply { ... })`. Spring Boot only instruments
   `RestTemplate` instances created through the auto-configured `RestTemplateBuilder`. A
   hand-constructed one carries **no** interceptors, so **no `traceparent` header would ever be
   sent to the AI worker** and the trace would die at the service boundary. `AIService` must take
   an injected `RestTemplateBuilder` and configure the same timeouts through it — the 5s connect
   and 5min read timeouts exist so an unresponsive worker becomes a catchable failure rather than
   blocking the `@Async` thread, and must be preserved exactly.

2. **The AI-worker call runs on an `@Async` thread** (`AIService.processLyricsAsync`). Trace
   context lives in a thread-local and does **not** cross that boundary for free. Without explicit
   context propagation, the longest and most interesting part of the request — the Gemini
   translation — would be precisely the part with no trace ID. Adding `io.micrometer:context-propagation`
   lets Spring Boot's `ContextPropagatingTaskDecorator` carry the context onto the `@Async`
   executor.

### Backend: structured logs

Under the **`prod` profile only**, set `logging.structured.format.console=ecs`. Spring Boot 4
ships built-in structured-logging formatters for Logback (ECS, Logstash, GELF), so this needs
**no new dependency and no `logback-spring.xml`** — it is one property.

Local dev and the test suite keep Spring's normal human-readable console output. The format
therefore differs between local and production. That divergence is accepted deliberately: dense
JSON at a developer's terminal is a daily tax, and the prod profile already carries
environment-specific behavior (`ProductionKeyGuard`).

`spring.jpa.show-sql` is turned **off**. It bypasses SLF4J (writing to `System.out`), so in
production it would emit non-JSON lines into an otherwise structured stream, and it prints every
query — including ones carrying lyrics text and user emails. Developers who want SQL get it
properly through SLF4J with `logging.level.org.hibernate.SQL: DEBUG`.

### Backend: tracing

Add Micrometer Tracing with the OpenTelemetry bridge. It generates trace and span IDs, places
them in the SLF4J MDC automatically (so every structured log line carries `trace_id` and
`span_id`), and injects a W3C `traceparent` header on outbound HTTP from an instrumented client.

Dependencies (versions managed by the Spring Boot BOM; all confirmed to resolve):

- `org.springframework.boot:spring-boot-starter-actuator` — brings the Micrometer observation
  infrastructure the tracer hooks into.
- `io.micrometer:micrometer-tracing-bridge-otel` — the tracer. OpenTelemetry rather than Brave,
  because W3C `traceparent` is what the Python side will parse and OTel is the ecosystem to grow
  into if a collector is ever added.
- `io.micrometer:context-propagation` — carries trace context across the `@Async` boundary.

No exporter dependency is added, because no spans are exported.

Sampling must be set to **always sample** (`management.tracing.sampling.probability: 1.0`). The
default samples only 10% of traces, and an unsampled trace produces log lines with no trace ID —
so 90% of production requests would be uncorrelated, which defeats the entire purpose. There is no
cost argument against full sampling here, since nothing is exported.

### AI worker: JSON logs and trace correlation

`slangdump-ai-solation` has no logging configuration today; this is its first. Two small pieces:

- **A FastAPI middleware** that reads the inbound `traceparent` header, parses the trace ID out of
  it (W3C format: `version-traceid-spanid-flags`), and stores it in a `contextvar`. If the header
  is absent or malformed, it generates a fresh trace ID rather than failing — a request must never
  break because of logging.
- **A JSON log formatter** that stamps `trace_id` onto every line, so the worker's logs join the
  backend's under the same ID.

**No OpenTelemetry SDK on the Python side.** With no trace backend, the worker only needs to
*read* an ID and log it. A ~30-line middleware does that; an OTel dependency tree does not earn
its weight here.

Uvicorn's own access logs are routed through the same formatter so the stream is uniformly JSON.

### CloudWatch delivery: no application code

The ECS task definition sets the `awslogs` log driver, which pipes container stdout to CloudWatch
Logs. The application writes to stdout and knows nothing about AWS — the same posture as the
existing secrets convention, where ECS resolves parameters and the app just reads env vars.

Recorded here for whoever provisions the infrastructure:

| Setting | Backend | AI worker |
|---|---|---|
| `awslogs-group` | `/ecs/slangdump/backend` | `/ecs/slangdump/ai-worker` |
| `awslogs-stream-prefix` | `backend` | `ai-worker` |
| `awslogs-region` | deployment region | deployment region |
| Retention | 30 days | 30 days |

The ECS **task execution role** needs `logs:CreateLogStream` and `logs:PutLogEvents` on those log
groups (and `logs:CreateLogGroup` if the groups are not pre-created by the infrastructure). This
mirrors the execution-role treatment already documented for Parameter Store.

Local `docker-compose` is unaffected: it keeps the default `json-file` driver and the
human-readable log format, since the `local` profile does not enable structured output.

### What must never be logged

Log statements must not emit JWTs, the `Authorization` header, or RSA private keys. This is not
hypothetical: `ApiExceptionHandler` logs request details on failure, and the system now handles
bearer tokens on every authenticated write. A token in CloudWatch is a credential in CloudWatch.

## Testing

- **`traceparent` is actually sent on the outbound AI-worker call.** This is the single assertion
  that catches the hand-built-`RestTemplate` failure, and it is the thing most likely to regress.
- **The trace ID survives the `@Async` hop** — a log line emitted from `processLyricsAsync` carries
  the same trace ID as the request that scheduled it.
- **The `prod` profile emits parseable JSON** containing a `trace_id` field. Parse it as JSON in
  the test; asserting on substrings would pass on malformed output.
- **AI worker:** an inbound `traceparent` reaches the log output; a request *without* one still
  logs cleanly with a generated ID (the middleware must not throw on a missing or malformed
  header).
- The existing backend suite (49 tests) must stay green. `AIService`'s timeouts must be unchanged
  after the `RestTemplateBuilder` switch — assert them.

## Risks

- **Silent trace failure.** Every risk here degrades quietly: logs still flow, they just lose
  correlation. The two tests above (outbound header, `@Async` hop) are the only things standing
  between "tracing works" and "tracing looks like it works."
- **Boot 4 module splits.** Spring Boot 4.0 split auto-configuration into per-technology modules
  (the existing `build.gradle` already carries a comment about this for Flyway). The tracing
  artifacts resolve, but the implementation must confirm the tracer actually auto-configures and
  populates the MDC, rather than assuming it from the dependency being present.
- **Log volume/cost.** Full sampling plus JSON is more bytes than plain text. At MVP scale this is
  negligible, and 30-day retention bounds it. Revisit if log volume ever becomes a cost line.
