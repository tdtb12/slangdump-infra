# Observability: Logs and Trace Correlation

## How it works

Both services write **newline-delimited JSON to stdout**. Nothing in either application talks
to AWS. In production the ECS `awslogs` log driver pipes container stdout to CloudWatch Logs —
the same posture as secrets, where ECS resolves the parameter and the app just reads an env var.

Every log line carries a **`traceId`**. The Spring backend generates it (or adopts an inbound
W3C `traceparent`), sends `traceparent` on its call to the AI worker, and the worker stamps the
same id on its own lines. `traceId` is **byte-identical** on both services — that identity is
what makes cross-service correlation possible. One CloudWatch query therefore returns a
request's full story across both services:

```
fields @timestamp, `log.level`, `log.logger`, `service.name`, message
| filter traceId = "4bf92f3577b34da6a3ce929d0e0e4736"
| sort @timestamp asc
```

This query works against both services' logs even though, as explained below, their JSON is
differently *shaped* — CloudWatch Logs Insights flattens nested JSON to dot notation, so
`` `log.level` `` matches the backend's nested `"log":{"level":...}` AND the worker's already-flat
`"log.level"` key.

No traces are exported anywhere. There is no X-Ray, no collector, no sidecar. Trace ids exist
**only** to correlate log lines. An exporter can be added later without touching app code.

## Log format

Both services emit the same **field names** — that identity of names is load-bearing, since a
divergence would break any query trying to reference a field on both services. But they do
**not** emit the same **shape**, and the doc previously (incorrectly) implied they did:

- **Backend** (Spring Boot's built-in ECS structured-logging formatter): genuinely **nested**
  JSON objects — `"log":{"level":...,"logger":...}`, `"ecs":{"version":...}`,
  `"error":{"stack_trace":...}`, plus `"process":{...}` and
  `"service":{"name":"slangdump-pick-n-roll"}`.
- **Worker** (`core/observability.py`'s `JsonFormatter`): **flat**, dotted-key JSON —
  `"log.level"`, `"log.logger"`, `"ecs.version"`, `"error.stack_trace"`,
  `"service.name":"slangdump-ai-solation"`.

This is **not** a functional break: CloudWatch Logs Insights flattens nested JSON to dot
notation for field references, so a query written against dotted names (as above) matches both
shapes. It only breaks if these logs were ever forwarded to Elasticsearch/OpenSearch instead of
CloudWatch — the worker's flat dotted keys are **not valid ECS** there (real ECS requires nested
objects), even though the field *names* mirror ECS conventions. Don't read this doc as claiming
ECS parity for the worker; it doesn't have it. `traceId` and `spanId` are the exception on both
services: genuinely flat, top-level, camelCase, no dot — never backtick-quote them or give them
a dotted name, or a query matches nothing, silently.

| Field | Meaning | Backend shape | Worker shape |
|---|---|---|---|
| `@timestamp` | ISO-8601, UTC | top-level | top-level |
| `log.level` | `INFO`, `WARN`, `ERROR`, … | nested: `log.level` | flat key: `"log.level"` |
| `log.logger` | logger name | nested: `log.logger` | flat key: `"log.logger"` |
| `message` | the log message | top-level | top-level |
| `traceId` | correlates lines within and across services | flat, top-level | flat, top-level |
| `spanId` | backend only | flat, top-level | — |
| `ecs.version` | ECS schema version | nested: `ecs.version` | flat key: `"ecs.version"` |
| `service.name` | which service emitted the line | nested: `service.name` | flat key: `"service.name"` |
| `error.stack_trace` | present on exceptions | nested: `error.stack_trace` | flat key: `"error.stack_trace"` |

**Backend JSON is emitted only under the `prod` profile.** Local dev and tests keep the readable
console format, and local `docker-compose` runs the `local` profile — so `docker compose logs`
stays human-readable and the default `json-file` driver is unchanged.

## ECS task definitions

Each container needs a `logConfiguration` block. **Without it, stdout is discarded and the logs
are gone when the task stops** — there is no other copy.

```json
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-group": "/ecs/slangdump/backend",
    "awslogs-region": "<deployment-region>",
    "awslogs-stream-prefix": "backend"
  }
}
```

| Service | Log group | Stream prefix |
|---|---|---|
| Spring backend | `/ecs/slangdump/backend` | `backend` |
| Python AI worker | `/ecs/slangdump/ai-worker` | `ai-worker` |

Set **30-day retention** on both log groups. CloudWatch log groups default to *never expire*,
which quietly accrues cost forever.

Remember the backend container must also set `SPRING_PROFILES_ACTIVE=prod`. This is doubly
load-bearing: it is required for `ProductionKeyGuard` (see
`slangdump-ai-solation/docs/secrets-management.md`), **and** it is what turns on JSON logging.
Without it you get plain text, and CloudWatch Logs Insights cannot filter by field at all.

## IAM

The ECS **task execution role** (not the task role — the agent configures the log driver before
the container starts) needs, on both log groups:

- `logs:CreateLogStream`
- `logs:PutLogEvents`
- `logs:CreateLogGroup` — only if the groups are not pre-created by the infrastructure

## What must never be logged

JWTs, the `Authorization` header, and RSA private keys. The system handles a bearer token on
every authenticated write, and `ApiExceptionHandler` logs request details on failure. **A token
in CloudWatch is a credential in CloudWatch.**

**User content and lyrics text.** Once logs are centralized, they're retained 30 days in
CloudWatch — this cuts against the project's lyrics/copyright posture (source lyrics are never
persisted or reconstructed server-side beyond what's needed; see the frontend copyright notes),
so lyrics text must not land there either. The concrete risk: when the AI worker rejects a
request (FastAPI 422), `RestTemplate` folds up to 512 chars of the worker's response body into
`RestClientResponseException`'s message — and a FastAPI validation error echoes back the
rejected `input`, i.e. the submitted lyrics. `AIService` handles this exception type separately
from other failures, logging only the status code and response-body **length**, never the body
itself; other exception types (which carry no user content) still get full logging, stack trace
included.

## Not built (deliberately)

- **CloudWatch Metrics / Alarms.** Logging is separable; metrics are a distinct subsystem.
- **A trace backend.** See above — ids correlate logs, nothing more.
