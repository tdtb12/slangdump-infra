# Observability: Logs and Trace Correlation

## How it works

Both services write **newline-delimited JSON to stdout**. Nothing in either application talks
to AWS. In production the ECS `awslogs` log driver pipes container stdout to CloudWatch Logs —
the same posture as secrets, where ECS resolves the parameter and the app just reads an env var.

Every log line carries a **`traceId`**. The Spring backend generates it (or adopts an inbound
W3C `traceparent`), sends `traceparent` on its call to the AI worker, and the worker stamps the
same id on its own lines. One CloudWatch query therefore returns a request's full story across
both services:

```
fields @timestamp, `log.level`, `log.logger`, message
| filter traceId = "4bf92f3577b34da6a3ce929d0e0e4736"
| sort @timestamp asc
```

No traces are exported anywhere. There is no X-Ray, no collector, no sidecar. Trace ids exist
**only** to correlate log lines. An exporter can be added later without touching app code.

## Log format

Elastic Common Schema, emitted by Spring Boot's built-in structured-logging formatter on the
backend and by `core/observability.py` on the worker. Both use the same field names — this is
load-bearing: if they diverged, no query could join the two services.

| Field | Meaning |
|---|---|
| `@timestamp` | ISO-8601, UTC |
| `log.level` | `INFO`, `WARN`, `ERROR`, … |
| `log.logger` | logger name |
| `message` | the log message |
| `traceId` | correlates lines within and across services |
| `spanId` | backend only |
| `error.stack_trace` | present on exceptions |

Note the naming split: `@timestamp`, `log.level`, `log.logger`, `ecs.version`, and
`error.stack_trace` are genuinely dotted/nested ECS-style keys — a CloudWatch Logs Insights
query referencing them needs backtick-quoting, e.g. `` `log.level` ``. `traceId` and `spanId`
are flat camelCase top-level keys with no dot, and must **not** be backtick-quoted or given a
dotted name — a query that does either matches nothing, silently.

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

## Not built (deliberately)

- **CloudWatch Metrics / Alarms.** Logging is separable; metrics are a distinct subsystem.
- **A trace backend.** See above — ids correlate logs, nothing more.
