# CloudWatch Logging & Trace Correlation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Emit structured JSON logs from the Spring backend and the Python AI worker, correlated by a shared W3C trace ID, so ECS can ship them to CloudWatch Logs and one request's full story is retrievable across both services.

**Architecture:** The backend gains Micrometer Tracing (OpenTelemetry bridge), which generates trace IDs, puts them in the MDC, and injects a `traceparent` header on outbound HTTP. Under the `prod` profile only, Spring Boot 4's built-in ECS structured-logging formatter turns console output into JSON — no new logging dependency, no `logback-spring.xml`. The Python worker reads the inbound `traceparent` and stamps the same trace ID onto its own JSON log lines. Neither service talks to AWS: ECS's `awslogs` log driver pipes container stdout to CloudWatch.

**Tech Stack:** Kotlin 2.1 / Java 21, Spring Boot 4.0.6, Micrometer Tracing + OpenTelemetry bridge, Gradle, JUnit 5 + `kotlin.test`, Testcontainers; Python 3.10 / FastAPI / uvicorn, pytest.

**Design doc:** `docs/plans/2026-07-14-cloudwatch-logging-tracing-design.md`

## Global Constraints

- **Repo paths.** Root: `/Users/jimhsu/Documents/slangdump`. Backend: `slangdump-pick-n-roll/`. AI worker: `slangdump-ai-solation/`. **These are separate git repos.** Every `git commit` runs inside the correct repo; no commit spans repos.
- **BRANCH FROM `feat/jwt-key-management`, NOT `master`.** This work depends on that branch: the `prod` Spring profile and `ProductionKeyGuard` only exist there, and this plan configures logging *under that profile*. Both repos are currently on `feat/jwt-key-management` and it is unmerged. Create `feat/observability-cloudwatch` from it in each repo, or commit onto `feat/jwt-key-management` directly — but do not branch from `master`, or the prod profile will not exist and Task 1 cannot be verified.
- **Booting the backend with the `prod` profile requires a non-dev public key.** `ProductionKeyGuard` refuses to start under `prod` if the resolved JWT public key is the committed dev key. Any prod-profile run must set `AUTH_JWT_PUBLIC_KEY` — use `$(cat ../jwt-keys/jwt-public.pem)`. **Never commit that file or its contents.**
- **Sampling must be 1.0.** The framework default samples 10% of traces, and an unsampled trace yields log lines with **no trace ID at all** — 90% of production requests would be uncorrelated, defeating the purpose. Nothing is exported anywhere, so there is no cost argument against full sampling.
- **No trace backend.** Do not add an OpenTelemetry exporter, X-Ray, a collector, or a sidecar. Trace IDs exist only to correlate log lines.
- **Never log JWTs, the `Authorization` header, or private keys.** A token in CloudWatch is a credential in CloudWatch.
- **Every JUnit test gets a `@DisplayName`** plus a backticked function name — match `PostControllerIntegrationTest.kt`.
- **Backend tests need Docker** (Testcontainers starts a real `postgres:16`).
- Local `docker-compose` behavior must not change: it runs the `local` profile with plain-text logs and the default `json-file` driver.

---

### Task 1: Structured JSON logs under the prod profile

Config-only. The backend currently has *no* logging configuration at all, so this establishes it.

**Files:**
- Create: `slangdump-pick-n-roll/src/main/resources/application-prod.yml`
- Modify: `slangdump-pick-n-roll/src/main/resources/application.yml`

**Interfaces:**
- Consumes: the `prod` profile, which exists on this branch (`ProductionKeyGuard` is `@Profile("prod")`).
- Produces: under `prod`, stdout is newline-delimited JSON in Elastic Common Schema. **Task 3 must reuse the exact field names this task's Step 4 captures** — the Python worker has to emit the same field names or cross-service CloudWatch queries won't join.

- [ ] **Step 1: Create the prod profile config**

There is no `application-prod.yml` yet. Create `slangdump-pick-n-roll/src/main/resources/application-prod.yml`:

```yaml
# Production-only logging. Spring Boot 4 ships structured-logging formatters for Logback
# (ECS / Logstash / GELF), so JSON output needs no dependency and no logback-spring.xml.
# Local dev and tests deliberately keep the human-readable console format.
logging:
  structured:
    format:
      console: ecs
```

- [ ] **Step 2: Turn off `show-sql` and route SQL logging through SLF4J**

In `slangdump-pick-n-roll/src/main/resources/application.yml`, find:

```yaml
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
```

and change it to:

```yaml
  jpa:
    hibernate:
      ddl-auto: validate
    # Hibernate's show-sql writes straight to System.out, bypassing SLF4J — so it ignores log
    # levels and would spray non-JSON lines into the structured stream in production. It also
    # prints every query, including ones carrying lyrics text and user emails. Developers who
    # want SQL set the logger below to DEBUG instead.
    show-sql: false
```

Then add a `logging` block at the top level of the same file (sibling of `spring:`, `auth:`, `ai:`, `server:`):

```yaml
logging:
  level:
    # Raise to DEBUG locally to see SQL — properly, through SLF4J.
    org.hibernate.SQL: INFO
```

- [ ] **Step 3: Confirm the existing suite still passes**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test
```

Expected: `BUILD SUCCESSFUL`, 49 tests. (Tests do not run under the `prod` profile, so they are unaffected — this run proves you did not break the base config.)

- [ ] **Step 4: Verify prod really emits valid JSON — and CAPTURE THE FIELD NAMES**

Boot under the `prod` profile and confirm the startup lines parse as JSON. The `prod` profile activates `ProductionKeyGuard`, so a non-dev public key must be supplied or the app will refuse to start (that is the guard working as designed, not a bug).

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
AUTH_JWT_PUBLIC_KEY="$(cat ../jwt-keys/jwt-public.pem)" \
SPRING_PROFILES_ACTIVE=prod \
./gradlew bootRun 2>/dev/null | head -20 | tee /tmp/prod-log-sample.txt

# Every line must be valid JSON. If jq errors, the format is not active.
head -5 /tmp/prod-log-sample.txt | jq -c .
# Print the exact field names, which Task 3 must match:
head -1 /tmp/prod-log-sample.txt | jq -r 'keys[]'
```

Expected: `jq` parses each line without error. The keys should be Elastic Common Schema names — `@timestamp`, `log.level`, `log.logger`, `message`, `ecs.version`, and (once Task 2 lands) `trace.id` and `span.id`.

**Record the exact key list in your report.** Task 3 depends on it: if the Python worker emits `trace_id` while the backend emits `trace.id`, a CloudWatch query cannot join the two services, and the whole point of this work is lost. Stop the app when done.

- [ ] **Step 5: Commit**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
git add src/main/resources/application.yml src/main/resources/application-prod.yml
git commit -m "feat(logging): emit structured JSON logs under the prod profile

The backend had no logging config at all — stdout only, which on ECS means
logs vanish when a task stops. Under prod, use Spring Boot's built-in ECS
structured formatter so CloudWatch Logs Insights can query by field. Local
dev keeps the readable console format.

Also disable Hibernate show-sql: it writes to System.out, bypassing SLF4J, so
it would emit non-JSON lines into the structured stream — and it logs every
query, including ones carrying lyrics and user emails."
```

---

### Task 2: Trace IDs, propagated to the AI worker and across the @Async hop

The core task. Two existing-code hazards make this fail *silently* if handled wrong — logs would still flow, just uncorrelated, which is the failure you notice last and need most.

**Files:**
- Modify: `slangdump-pick-n-roll/build.gradle`
- Modify: `slangdump-pick-n-roll/src/main/resources/application.yml`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/service/AIService.kt` (constructor + `restTemplate` field only)
- Test (create): `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/observability/TracePropagationIntegrationTest.kt`

**Interfaces:**
- Consumes: the structured-log config from Task 1 (trace IDs appear as fields in that JSON).
- Produces: every log line carries `trace.id`/`span.id`; outbound calls to the AI worker carry a W3C `traceparent` header whose trace ID equals the inbound request's. **Task 3 consumes that header.**

- [ ] **Step 1: Write the failing test**

This single test proves the whole chain at once: an inbound `traceparent` → the backend's trace context → across the `@Async` thread boundary → out to the AI worker with the *same* trace ID. If either hazard is unhandled, it fails.

Create `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/observability/TracePropagationIntegrationTest.kt`:

```kotlin
package com.example.demo.observability

import com.example.demo.support.TestJwt
import com.sun.net.httpserver.HttpServer
import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.testcontainers.service.connection.ServiceConnection
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc
import org.springframework.http.MediaType
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import org.testcontainers.containers.PostgreSQLContainer
import java.net.InetSocketAddress
import java.util.concurrent.CountDownLatch
import java.util.concurrent.TimeUnit
import java.util.concurrent.atomic.AtomicReference
import kotlin.test.assertEquals
import kotlin.test.assertNotNull
import kotlin.test.assertTrue

/**
 * The trace must survive two hops that are easy to break and fail silently:
 *   1. AIService's RestTemplate must be instrumented, or no `traceparent` is ever sent.
 *   2. The worker call runs on an @Async thread, and trace context is thread-local.
 * Asserting the AI worker receives the SAME trace id the request arrived with covers both.
 */
@SpringBootTest
@AutoConfigureMockMvc
class TracePropagationIntegrationTest {

    companion object {
        @JvmStatic
        @ServiceConnection
        val postgres: PostgreSQLContainer<*> = PostgreSQLContainer("postgres:16").apply { start() }

        val receivedTraceparent = AtomicReference<String?>(null)
        val workerCalled = CountDownLatch(1)

        // Stands in for the Python AI worker. A JDK HttpServer keeps this dependency-free.
        @JvmStatic
        val stubWorker: HttpServer = HttpServer.create(InetSocketAddress("127.0.0.1", 0), 0).apply {
            createContext("/api/v1/process") { exchange ->
                receivedTraceparent.set(exchange.requestHeaders.getFirst("traceparent"))
                val body = """
                    {"metaInfo":{"detectedSourceLanguage":"English","artistBackground":"a",
                    "albumContext":"b","songMeaning":"c"},"lines":[],
                    "modelInfo":{"provider":"google","model":"gemini-2.5-flash"}}
                """.trimIndent().replace("\n", "")
                val bytes = body.toByteArray()
                exchange.responseHeaders.add("Content-Type", "application/json")
                exchange.sendResponseHeaders(200, bytes.size.toLong())
                exchange.responseBody.use { it.write(bytes) }
                workerCalled.countDown()
            }
            start()
        }

        @JvmStatic
        @DynamicPropertySource
        fun workerUrl(registry: DynamicPropertyRegistry) {
            registry.add("ai.worker.url") { "http://127.0.0.1:${stubWorker.address.port}" }
        }
    }

    @Autowired lateinit var mockMvc: MockMvc

    @Test
    @DisplayName("inbound trace id reaches the AI worker unchanged, across the @Async hop")
    fun `inbound trace id reaches the ai worker unchanged`() {
        val traceId = "4bf92f3577b34da6a3ce929d0e0e4736"

        mockMvc.perform(
            post("/api/v1/posts")
                .header("Authorization", "Bearer ${TestJwt.token()}")
                .header("traceparent", "00-$traceId-00f067aa0ba902b7-01")
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    """
                    {
                      "artist": "Drake",
                      "songName": "One Dance",
                      "album": "Views",
                      "lyricUrl": "https://genius.com/...",
                      "rawLyrics": "baby I like your style",
                      "description": "trace propagation test"
                    }
                    """.trimIndent()
                )
        ).andExpect(status().isOk)

        assertTrue(
            workerCalled.await(30, TimeUnit.SECONDS),
            "the AI worker was never called — the @Async translation never ran",
        )

        val sent = receivedTraceparent.get()
        assertNotNull(
            sent,
            "no traceparent header reached the AI worker: AIService's RestTemplate is not instrumented",
        )
        assertEquals(
            traceId,
            sent.split("-")[1],
            "trace id changed between the inbound request and the outbound worker call — " +
                "context did not survive the @Async thread hop",
        )
    }
}
```

- [ ] **Step 2: Run the test and watch it fail**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test --tests 'com.example.demo.observability.TracePropagationIntegrationTest'
```

Expected: FAIL. Most likely `no traceparent header reached the AI worker` — because `AIService` hand-builds its `RestTemplate` and no tracer exists yet. **Read the actual failure message and record it**; it tells you which hazard bites first.

- [ ] **Step 3: Add the tracing dependencies**

In `slangdump-pick-n-roll/build.gradle`, add to the `dependencies` block, immediately after the existing `spring-boot-starter-oauth2-resource-server` line:

```groovy
    // Observability: Micrometer Tracing generates trace/span ids, puts them in the SLF4J MDC
    // (so they appear as fields in the structured logs), and injects a W3C `traceparent`
    // header on outbound HTTP from instrumented clients. The OTel bridge is used rather than
    // Brave because traceparent is what the Python worker parses. NO exporter is added — we
    // export no spans; trace ids exist only to correlate log lines.
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
    // Carries trace context across the @Async thread boundary in AIService. Without this the
    // Gemini translation — the slowest, most interesting part of a request — logs with no trace id.
    implementation 'io.micrometer:context-propagation'
```

Versions are managed by the Spring Boot BOM; do not pin them.

- [ ] **Step 4: Force full sampling**

In `slangdump-pick-n-roll/src/main/resources/application.yml`, add a top-level `management` block (sibling of `spring:`, `auth:`, `ai:`, `server:`, `logging:`):

```yaml
management:
  tracing:
    sampling:
      # Sample every trace. The 10% default would leave 90% of requests with NO trace id in
      # their log lines, which defeats the point. Nothing is exported, so full sampling is free.
      probability: 1.0
```

- [ ] **Step 5: Instrument AIService's RestTemplate**

Spring Boot only instruments `RestTemplate`s built through the auto-configured `RestTemplateBuilder`. The hand-constructed one carries no interceptors, so it sends no `traceparent`.

In `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/service/AIService.kt`, replace the constructor and the `restTemplate` field. Change the imports: remove `org.springframework.http.client.SimpleClientHttpRequestFactory`, add `org.springframework.boot.web.client.RestTemplateBuilder`.

Replace:

```kotlin
@Service
class AIService(
    private val songRepository: SongRepository,
    @Value("\${ai.worker.url:http://localhost:8000}") private val workerUrl: String,
) {
    private val log = LoggerFactory.getLogger(AIService::class.java)

    // Bounded timeouts so an unresponsive worker becomes a catchable failure
    // (-> status FAILED) instead of blocking the @Async thread indefinitely and
    // leaving the song stuck in PENDING. Read timeout is generous because Gemini
    // calls can take a while.
    private val restTemplate = RestTemplate(
        SimpleClientHttpRequestFactory().apply {
            setConnectTimeout(Duration.ofSeconds(5))
            setReadTimeout(Duration.ofMinutes(5))
        }
    )
```

with:

```kotlin
@Service
class AIService(
    private val songRepository: SongRepository,
    restTemplateBuilder: RestTemplateBuilder,
    @Value("\${ai.worker.url:http://localhost:8000}") private val workerUrl: String,
) {
    private val log = LoggerFactory.getLogger(AIService::class.java)

    // Built via RestTemplateBuilder so Spring Boot's observability instrumentation is applied:
    // a hand-constructed RestTemplate carries no interceptors, so no `traceparent` header would
    // reach the worker and the trace would die at the service boundary.
    //
    // Timeouts unchanged: bounded so an unresponsive worker becomes a catchable failure
    // (-> status FAILED) instead of blocking the @Async thread indefinitely and leaving the
    // song stuck in PENDING. Read timeout is generous because Gemini calls can take a while.
    private val restTemplate: RestTemplate = restTemplateBuilder
        .connectTimeout(Duration.ofSeconds(5))
        .readTimeout(Duration.ofMinutes(5))
        .build()
```

If the compiler reports `connectTimeout`/`readTimeout` as unresolved on this Boot version, use `setConnectTimeout`/`setReadTimeout` instead — same `Duration` arguments, same meaning. Do not change the timeout values.

**On testing the timeouts.** The design doc asks for an assertion that the 5s/5min timeouts survive this switch. There is no public accessor for them on a built `RestTemplate` — reading them back requires reflecting into the request factory's private fields, which is brittle and could easily pass while asserting nothing meaningful. A test that cannot meaningfully fail is worse than no test, so this is a **review checkpoint instead**: the reviewer must confirm from the diff that `Duration.ofSeconds(5)` and `Duration.ofMinutes(5)` are carried over exactly. They exist so an unresponsive worker becomes a catchable failure rather than pinning an `@Async` thread and stranding the song in `PENDING` — losing them would be a real regression.

- [ ] **Step 6: Run the test again**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test --tests 'com.example.demo.observability.TracePropagationIntegrationTest'
```

Expected: PASS.

**If it fails with `trace id changed between the inbound request and the outbound worker call`,** the `@Async` boundary is dropping context — Spring Boot did not auto-apply a context-propagating task decorator. Fix it explicitly by creating `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/config/AsyncTracingConfig.kt`:

```kotlin
package com.example.demo.config

import io.micrometer.context.ContextSnapshotFactory
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.task.TaskDecorator

/**
 * Trace context is thread-local, so it does not cross the @Async boundary in AIService by
 * itself. Without this, the Gemini translation — the slowest part of a request — would log
 * with no trace id, which is exactly the correlation we need most.
 */
@Configuration
class AsyncTracingConfig {

    @Bean
    fun contextPropagatingTaskDecorator(): TaskDecorator {
        val factory = ContextSnapshotFactory.builder().build()
        return TaskDecorator { runnable -> factory.captureAll().wrap(runnable) }
    }
}
```

Then re-run. Record in your report whether this was needed — that tells us whether Boot's auto-configuration covers it.

- [ ] **Step 7: Run the full suite**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test
```

Expected: `BUILD SUCCESSFUL`, 50 tests (49 existing + 1 new). The `AIService` constructor changed, so this also proves nothing else depended on its old shape.

- [ ] **Step 8: Confirm trace ids now appear in the prod JSON logs**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
AUTH_JWT_PUBLIC_KEY="$(cat ../jwt-keys/jwt-public.pem)" \
SPRING_PROFILES_ACTIVE=prod \
./gradlew bootRun 2>/dev/null | head -30 | tee /tmp/prod-trace-sample.txt
# In another terminal, hit a public endpoint so a request is traced:
#   curl -s -H 'traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01' \
#        http://localhost:8080/api/v1/posts > /dev/null
grep -o '"trace\.id":"[^"]*"' /tmp/prod-trace-sample.txt | head -3
```

Expected: at least one line carrying a `trace.id` field. Record the exact field name (`trace.id` vs `traceId`) in your report — **Task 3 must match it**. Stop the app when done.

- [ ] **Step 9: Commit**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
git add build.gradle src/main/resources/application.yml \
        src/main/kotlin/com/example/demo/service/AIService.kt \
        src/test/kotlin/com/example/demo/observability/TracePropagationIntegrationTest.kt
# add AsyncTracingConfig.kt too, if Step 6 required it
git commit -m "feat(observability): correlate logs with W3C trace ids across services

AIService hand-built its RestTemplate, which Spring does not instrument — so no
traceparent header would ever have reached the AI worker and the trace would die
at the service boundary. Build it via RestTemplateBuilder instead, timeouts intact.

The worker call also runs on an @Async thread and trace context is thread-local,
so context propagation is required or the slowest part of the request logs with
no trace id. The new test asserts the inbound trace id reaches the worker unchanged,
which covers both hazards at once.

Sampling is forced to 1.0: the 10% default would leave most requests with no trace
id at all, and nothing is exported, so full sampling costs nothing."
```

---

### Task 3: Trace-correlated JSON logs in the Python AI worker

`slangdump-ai-solation` has no logging configuration and no tests. This adds both.

**Files:**
- Create: `slangdump-ai-solation/core/observability.py`
- Create: `slangdump-ai-solation/tests/test_observability.py`
- Modify: `slangdump-ai-solation/main.py`
- Modify: `slangdump-ai-solation/pyproject.toml` (add a dev dependency group)

**Interfaces:**
- Consumes: the `traceparent` header Task 2 makes the backend send, and **the exact JSON field names Task 1 Step 4 / Task 2 Step 8 recorded from the backend's real output.**
- Produces: JSON log lines carrying the same trace-id field name as the backend, so one CloudWatch query joins both services.

- [ ] **Step 1: Match the backend's field names**

Read the field names recorded in the Task 1 and Task 2 reports. The backend emits Elastic Common Schema, so the trace field is expected to be **`trace.id`** (not `trace_id`). **Use whatever the backend actually emits.** If the Python worker writes `trace_id` while the backend writes `trace.id`, no CloudWatch query can join the two services and this entire feature is pointless.

The expected ECS keys are `@timestamp`, `log.level`, `log.logger`, `message`, `trace.id`, `ecs.version`, and `error.stack_trace` on exceptions. The code below uses those; adjust only if the recorded output differs.

- [ ] **Step 2: Add the dev dependency group**

In `slangdump-ai-solation/pyproject.toml`, append (there is no test framework today):

```toml
[dependency-groups]
dev = [
    "pytest>=8.0",
    "httpx>=0.27",
]
```

`httpx` is required by FastAPI's `TestClient`.

- [ ] **Step 3: Write the failing tests**

Create `slangdump-ai-solation/tests/test_observability.py`:

```python
import json
import logging

from fastapi import FastAPI
from fastapi.testclient import TestClient

from core.observability import (
    JsonFormatter,
    TraceIdMiddleware,
    current_trace_id,
    parse_traceparent,
)

VALID_TRACE_ID = "4bf92f3577b34da6a3ce929d0e0e4736"
VALID_TRACEPARENT = f"00-{VALID_TRACE_ID}-00f067aa0ba902b7-01"


def _app() -> FastAPI:
    app = FastAPI()
    app.add_middleware(TraceIdMiddleware)

    @app.get("/echo-trace")
    def echo_trace():
        return {"trace_id": current_trace_id()}

    return app


def test_parses_a_valid_w3c_traceparent():
    assert parse_traceparent(VALID_TRACEPARENT) == VALID_TRACE_ID


def test_rejects_malformed_traceparent():
    assert parse_traceparent(None) is None
    assert parse_traceparent("") is None
    assert parse_traceparent("garbage") is None
    assert parse_traceparent("00-tooshort-00f067aa0ba902b7-01") is None
    # An all-zero trace id is invalid per the W3C spec.
    assert parse_traceparent(f"00-{'0' * 32}-00f067aa0ba902b7-01") is None


def test_inbound_trace_id_is_used_for_the_request():
    client = TestClient(_app())
    resp = client.get("/echo-trace", headers={"traceparent": VALID_TRACEPARENT})
    assert resp.json()["trace_id"] == VALID_TRACE_ID


def test_request_without_traceparent_still_gets_a_trace_id():
    # A missing or broken header must never break the request — logging is not worth a 500.
    client = TestClient(_app())
    resp = client.get("/echo-trace")
    trace_id = resp.json()["trace_id"]
    assert len(trace_id) == 32
    int(trace_id, 16)  # raises if not hex


def test_malformed_traceparent_does_not_raise():
    client = TestClient(_app())
    resp = client.get("/echo-trace", headers={"traceparent": "not-a-traceparent"})
    assert resp.status_code == 200
    assert len(resp.json()["trace_id"]) == 32


def test_formatter_emits_json_with_the_backend_field_names():
    record = logging.LogRecord(
        name="svc", level=logging.INFO, pathname=__file__, lineno=1,
        msg="hello %s", args=("world",), exc_info=None,
    )
    line = JsonFormatter().format(record)

    payload = json.loads(line)  # must parse as JSON, not merely look like it
    assert payload["message"] == "hello world"
    assert payload["log.level"] == "INFO"
    assert payload["log.logger"] == "svc"
    assert "@timestamp" in payload
    assert "trace.id" in payload


def test_formatter_includes_the_stack_trace_on_exceptions():
    try:
        raise ValueError("boom")
    except ValueError:
        import sys
        record = logging.LogRecord(
            name="svc", level=logging.ERROR, pathname=__file__, lineno=1,
            msg="failed", args=(), exc_info=sys.exc_info(),
        )
    payload = json.loads(JsonFormatter().format(record))
    assert "ValueError: boom" in payload["error.stack_trace"]
```

- [ ] **Step 4: Run the tests and watch them fail**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-ai-solation
uv run pytest tests/ -v
```

Expected: FAIL — `ModuleNotFoundError: No module named 'core.observability'`.

- [ ] **Step 5: Implement the module**

Create `slangdump-ai-solation/core/observability.py`:

```python
"""Trace correlation and JSON logging.

The Spring backend sends a W3C `traceparent` header on every call. We parse the trace id out
of it and stamp it onto every log line, using the same field names the backend's Elastic
Common Schema formatter emits — so a single CloudWatch Logs Insights query on `trace.id`
returns one request's full story across BOTH services.

No OpenTelemetry SDK: with no trace backend, the worker only needs to READ an id and log it.
"""

import json
import logging
import secrets
import sys
from contextvars import ContextVar
from datetime import datetime, timezone

from starlette.middleware.base import BaseHTTPMiddleware

_trace_id: ContextVar[str] = ContextVar("trace_id", default="")

_HEX = set("0123456789abcdef")
_ZERO_TRACE_ID = "0" * 32


def current_trace_id() -> str:
    """The trace id for the request being handled on this task, or "" outside a request."""
    return _trace_id.get()


def new_trace_id() -> str:
    """A fresh 128-bit trace id, for requests that arrive without a usable traceparent."""
    return secrets.token_hex(16)


def parse_traceparent(header: str | None) -> str | None:
    """Extract the trace id from a W3C traceparent (`version-traceid-spanid-flags`).

    Returns None if the header is absent or malformed — the caller generates an id instead.
    """
    if not header:
        return None
    parts = header.split("-")
    if len(parts) < 4:
        return None
    trace_id = parts[1].lower()
    if len(trace_id) != 32 or not set(trace_id) <= _HEX or trace_id == _ZERO_TRACE_ID:
        return None
    return trace_id


class TraceIdMiddleware(BaseHTTPMiddleware):
    """Bind the inbound trace id (or a fresh one) for the duration of the request."""

    async def dispatch(self, request, call_next):
        trace_id = parse_traceparent(request.headers.get("traceparent")) or new_trace_id()
        token = _trace_id.set(trace_id)
        try:
            return await call_next(request)
        finally:
            _trace_id.reset(token)


class JsonFormatter(logging.Formatter):
    """Render log records as ECS-shaped JSON, matching the backend's field names."""

    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "@timestamp": datetime.fromtimestamp(
                record.created, tz=timezone.utc
            ).isoformat(),
            "log.level": record.levelname,
            "log.logger": record.name,
            "message": record.getMessage(),
            "trace.id": current_trace_id(),
            "ecs.version": "8.11.0",
        }
        if record.exc_info:
            payload["error.stack_trace"] = self.formatException(record.exc_info)
        return json.dumps(payload)


def configure_logging() -> None:
    """Route every logger — including uvicorn's — through the JSON formatter on stdout.

    ECS's awslogs driver ships stdout to CloudWatch, so stdout is the only destination needed.
    """
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())

    root = logging.getLogger()
    root.handlers = [handler]
    root.setLevel(logging.INFO)

    # uvicorn installs its own handlers; clear them so its lines are JSON too, rather than a
    # mix of formats in one stream.
    for name in ("uvicorn", "uvicorn.error", "uvicorn.access"):
        uvicorn_logger = logging.getLogger(name)
        uvicorn_logger.handlers = []
        uvicorn_logger.propagate = True
```

- [ ] **Step 6: Run the tests and see them pass**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-ai-solation
uv run pytest tests/ -v
```

Expected: PASS, 7 tests.

- [ ] **Step 7: Wire it into the app**

Replace `slangdump-ai-solation/main.py` with:

```python
import logging

from fastapi import FastAPI

from api.routes import router
from core.observability import TraceIdMiddleware, configure_logging

configure_logging()

app = FastAPI(title="Slangdump AI Worker")
app.add_middleware(TraceIdMiddleware)
app.include_router(router, prefix="/api/v1")

log = logging.getLogger(__name__)


@app.get("/health")
def health_check():
    return {"status": "ok"}
```

- [ ] **Step 8: Verify the running app emits correlated JSON**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-ai-solation
uv run uvicorn main:app --port 8000 > /tmp/worker-log.txt 2>&1 &
sleep 3
curl -s -H 'traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01' \
     http://localhost:8000/health > /dev/null
sleep 1
kill %1
# Every line must be valid JSON, and the access-log line must carry the trace id we sent.
jq -c . < /tmp/worker-log.txt | head -5
grep -c '4bf92f3577b34da6a3ce929d0e0e4736' /tmp/worker-log.txt
```

Expected: `jq` parses every line without error, and the grep count is at least 1 — the trace id we sent appears in the worker's own logs. That is the cross-service correlation working.

- [ ] **Step 9: Commit**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-ai-solation
git add core/observability.py main.py pyproject.toml tests/test_observability.py
git commit -m "feat(observability): JSON logs correlated with the backend's trace id

The worker had no logging config at all. Emit ECS-shaped JSON on stdout using the
same field names the Spring backend emits, and stamp the inbound W3C traceparent's
trace id on every line — so one CloudWatch query on trace.id returns a request's
full story across both services.

No OpenTelemetry SDK: with no trace backend, the worker only needs to read an id
and log it. A missing or malformed header generates a fresh id rather than failing
the request — logging is never worth a 500."
```

---

### Task 4: Document CloudWatch delivery for whoever provisions AWS

No application code ships logs to AWS: the ECS `awslogs` log driver pipes container stdout to CloudWatch Logs. That is a task-definition setting, and if it is omitted the logs go nowhere — so the requirement has to be written down where infrastructure work will find it.

**Files:**
- Create: `/Users/jimhsu/Documents/slangdump/docs/observability.md`

**Interfaces:**
- Consumes: the log format and trace-id field name established in Tasks 1–3.
- Produces: the authoritative log group names, driver options, and IAM permissions AWS provisioning must honour. No code.

- [ ] **Step 1: Write the doc**

Create `/Users/jimhsu/Documents/slangdump/docs/observability.md`:

````markdown
# Observability: Logs and Trace Correlation

## How it works

Both services write **newline-delimited JSON to stdout**. Nothing in either application talks
to AWS. In production the ECS `awslogs` log driver pipes container stdout to CloudWatch Logs —
the same posture as secrets, where ECS resolves the parameter and the app just reads an env var.

Every log line carries a **`trace.id`**. The Spring backend generates it (or adopts an inbound
W3C `traceparent`), sends `traceparent` on its call to the AI worker, and the worker stamps the
same id on its own lines. One CloudWatch query therefore returns a request's full story across
both services:

```
fields @timestamp, `log.level`, `log.logger`, message
| filter `trace.id` = "4bf92f3577b34da6a3ce929d0e0e4736"
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
| `trace.id` | correlates lines within and across services |
| `span.id` | backend only |
| `error.stack_trace` | present on exceptions |

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

Remember the backend container must also set `SPRING_PROFILES_ACTIVE=prod` — it is required for
`ProductionKeyGuard` (see `slangdump-ai-solation/docs/secrets-management.md`) **and** it is what
turns on JSON logging. Without it you get plain text, and CloudWatch Logs Insights cannot filter
by field.

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
````

- [ ] **Step 2: Verify the doc matches reality**

The doc claims a field name. Confirm it against the actual captured output rather than trusting the plan:

```bash
grep -o '"trace\.id"' /tmp/prod-trace-sample.txt | head -1   # backend (from Task 2 Step 8)
grep -o '"trace\.id"' /tmp/worker-log.txt | head -1          # worker  (from Task 3 Step 8)
```

Expected: both print `"trace.id"`. If they differ from each other, the services cannot be joined — go back and fix the mismatch before committing this doc, because the doc would otherwise be documenting a query that returns nothing.

- [ ] **Step 3: Commit**

```bash
cd /Users/jimhsu/Documents/slangdump
git add docs/observability.md
git commit -m "docs: how logs reach CloudWatch and how traces correlate them

Records the awslogs driver config, log group names, retention, and the IAM the
execution role needs — none of which live in application code. Also records that
SPRING_PROFILES_ACTIVE=prod is what turns on JSON logging, not just the key guard."
```

---

## Done when

- Backend under `prod` emits valid JSON (`jq` parses every line) carrying `trace.id`; under `local` it stays human-readable.
- An inbound `traceparent` reaches the AI worker with the **same** trace id, proven by `TracePropagationIntegrationTest` — covering both the `RestTemplate` instrumentation and the `@Async` hop.
- The AI worker emits JSON carrying the same `trace.id` field name as the backend.
- `./gradlew test` green (50 tests); `uv run pytest` green (7 tests).
- `spring.jpa.show-sql` is off, so no `System.out` lines pollute the JSON stream.
- `docs/observability.md` records the log groups, driver options, retention, and IAM.

## Deferred to AWS provisioning day

Add the `logConfiguration` block to both ECS task definitions, create both log groups with 30-day
retention, and grant the execution role `logs:CreateLogStream` / `logs:PutLogEvents`. Commands and
exact values are in `docs/observability.md`. Without the log driver, **the applications will run
correctly and their logs will be silently discarded.**
