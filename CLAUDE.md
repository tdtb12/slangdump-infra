# Project Context

Translation service. Spring Boot exposes the public API; Python does the actual
translation. Frontend is a Next.js BFF on Vercel — **not** a static SPA: it has a
server side (Auth.js sign-in, RS256 token minting, and the server-rendered post
permalink), so it cannot be exported to a static host.

**Current goal: ship.** Deploying to Fly.io. The ECS migration is a separate,
later task — it must not influence current trade-offs, but anything
platform-independent (Dockerfiles, migration flow, service boundaries) should be
done correctly now.

---

## Architecture Decisions

### 1. The async boundary lives inside Spring

Python translation takes **20-60 seconds**. That latency is fully encapsulated
inside Spring. The frontend never observes it.

```
Browser  ──POST──▶  Spring API endpoint ──▶ jobs table (Neon)
   │                (returns 202 + jobId, sub-second)      │
   └──GET poll───▶  Spring API endpoint                    ▼
                                              Spring background worker
                                                  (virtual thread)
                                                           │
                                                           ▼
                                              Python service (synchronous,
                                                blocks 20-60 seconds)
```

**Python stays synchronous and requires no changes.** The wait is absorbed by
Spring's background worker. Do not propose converting Python to a
callback/webhook model or giving it its own queue — that was evaluated and
rejected (two-sided async is not worth the complexity here).

**The frontend talks only to Spring** and has no knowledge that Python exists.
Any design where the browser reaches Python directly violates this.

### 2. Client polling, not SSE

Rationale:
- SSE requires holding a 60s connection to the browser; hostile to mobile
  networks and corporate proxies.
- Polling keeps every frontend request under 1 second, which sidesteps three
  separate platform timeouts: ALB idle (60s), API Gateway (29s), Cloudflare
  proxy read timeout (120s, not adjustable below Enterprise).
- Terminal responses are cacheable; pending responses are cheap primary-key
  lookups.

### 3. The jobs table is the single source of truth

Do not use an in-memory queue or `@Async` with in-memory state — deploys would
lose in-flight work.

```sql
create table translation_jobs (
  id uuid primary key,
  input_hash text not null,
  status text not null,              -- PENDING / RUNNING / DONE / FAILED
  result jsonb,
  error text,
  attempts int not null default 0,
  claimed_at timestamptz,
  created_at timestamptz not null default now()
);
create index on translation_jobs (status, created_at);
create unique index on translation_jobs (input_hash)
  where status in ('PENDING','RUNNING','DONE');
```

The partial unique index gives free deduplication: identical input returns the
existing jobId instead of paying for the translation twice.

---

## Polling Contract

Efficiency is **not** the concern here. Each poll is a sub-millisecond primary
key lookup; even 40 polls per job across 100 concurrent users is ~27 req/s.
The concerns are robustness and letting the server own the timing policy.

### Server drives the interval

The server knows the job's age, so it decides when the client should return.
This keeps the tuning knob server-side — no frontend redeploy needed to change it.

```java
@GetMapping("/api/translations/{id}")
public ResponseEntity<JobView> get(@PathVariable UUID id) {
    var job = repo.findById(id).orElseThrow(JobNotFound::new);

    if (job.isTerminal()) {
        return ResponseEntity.ok()
            // terminal results are immutable — let the browser cache them
            .cacheControl(CacheControl.maxAge(Duration.ofHours(1)).cachePrivate())
            .body(JobView.of(job));
    }

    long age = Duration.between(job.getCreatedAt(), Instant.now()).toSeconds();
    return ResponseEntity.status(HttpStatus.ACCEPTED)
        .header(HttpHeaders.RETRY_AFTER, String.valueOf(nextDelay(age)))
        .cacheControl(CacheControl.noStore())
        .body(JobView.pending(job, age));
}

private long nextDelay(long age) {
    if (age < 12) return 8;   // too early to possibly be done
    if (age < 35) return 2;   // likely-completion window, poll tighter
    if (age < 70) return 4;
    return 8;                 // abnormally long, back off
}
```

### Client requirements (all mandatory)

The client follows `Retry-After` and must handle these five cases. Each one is a
real failure mode, not a hypothetical:

1. **No request stacking.** Use a recursive `setTimeout` / `await` loop, never
   `setInterval`. A slow request must not overlap the next one.
2. **Tolerate individual failures.** Rolling deploys and transient 502s will fail
   single requests. Back off and retry; only give up after ~5 consecutive
   failures. One error must not kill the loop.
3. **Hard give-up condition.** Cap total polling at ~180s. If a job is stuck in
   `RUNNING` and recovery has not kicked in, show "still processing" with a
   manual retry rather than spinning forever.
4. **Abort on unmount.** Use `AbortController` in the effect cleanup, or results
   land on unmounted components.
5. **Handle background-tab throttling.** Chrome throttles background timers to
   ~1/min. Do not fight it — listen for `visibilitychange` and poll immediately
   when the tab becomes visible again.

Also: **put the jobId in the URL** (`/translate/:jobId`) so a page refresh does
not lose in-flight work and the link is shareable.

Add ±15% jitter to every interval to avoid synchronised request spikes.

### Long polling: deliberately not doing this yet

`DeferredResult` with a ~28s hold (safely under ALB's 60s and Cloudflare's 120s)
would cut 60s of polling from ~16 requests to 3. It was evaluated and deferred
because neither benefit applies at current scale:

- Request savings are near zero in absolute cost.
- Latency improvement is 1.5s on an operation the user has already waited 40s
  for — imperceptible.

Costs that would be real: rolling deploys sever held connections (client must
treat a dropped connection as "retry now", not an error); the polling merely
moves server-side as a 500ms DB check per waiter; and `LISTEN/NOTIFY` — the
clean way to avoid that — **does not work on Neon's pooled endpoint** because
PgBouncer runs in transaction mode.

**Revisit when:** hundreds of concurrent waiters become normal, bandwidth cost
becomes visible, or translation latency drops to 3-5s (at which point a 1.5s
polling error is a large fraction of total time).

---

## Implementation Requirements

Each of these addresses a specific known failure. Do not simplify them away.

### Concurrency and recovery

- Workers claim jobs with `SELECT ... FOR UPDATE SKIP LOCKED` — multiple
  instances distribute naturally with no duplicates.
- On startup, scan for `status='RUNNING' AND claimed_at < now() - interval
  '5 minutes'` and reset to `PENDING` where `attempts` is under the limit.
- `server.shutdown=graceful` plus
  `spring.lifecycle.timeout-per-shutdown-phase=110s` so in-flight jobs finish
  after SIGTERM.

### Timeouts must be set explicitly

Defaults will break at 60 seconds.

- Spring to Python: `WebClient` `responseTimeout` = **90s**
- Python uvicorn/gunicorn worker timeout = **120s or more**, otherwise Python
  kills its own request first

### Virtual threads

Java 21 + Spring Boot 3.2+, `spring.threads.virtual.enabled=true`. This is what
makes a 60s blocking worker cheap. Python uses FastAPI + async.

---

## Neon Postgres

**Two connection strings with non-interchangeable roles:**

| Purpose | Connection |
|---|---|
| Flyway migrations | **direct** (no `-pooler`) |
| Application runtime / HikariCP | **pooled** (with `-pooler`) |

Flyway relies on session-level advisory locks to prevent concurrent migrations.
The pooled endpoint is PgBouncer in transaction mode, which breaks that locking.
**No exceptions to this.**

Other notes:
- Region `ap-northeast-1` (Tokyo), aligned with Fly.io `nrt`.
- Scale-to-zero causes cold-start latency on first connect. For production,
  extend the autosuspend window or disable it, or HikariCP will throw
  intermittent connection failures.

---

## Flyway

**Never run migrations at application startup.** Concurrent instance startup
contends on the advisory lock, stretches health checks, and forces use of the
pooled connection.

Run it as a distinct deployment step. On Fly.io that is `release_command` — it
runs before the new version goes live and a failure blocks the whole deploy,
which is the semantics we want.

The ECS equivalent later is a one-off `RunTask` before `update-service`.

---

## Secrets

Fly.io's built-in secrets are used. Values go through the API into a per-app
encrypted vault; the API servers can encrypt but not decrypt. At Machine boot,
the host agent decrypts and injects them as environment variables. No extra
service, no cost.

### Naming convention (load-bearing)

Spring Boot relaxed binding maps env vars onto config properties, so the secret
names are the property names:

| Secret | Value | Consumer |
|---|---|---|
| `SPRING_DATASOURCE_URL` | Neon **pooled** (`-pooler`) | app runtime / HikariCP |
| `SPRING_FLYWAY_URL` | Neon **direct** (no `-pooler`) | `release_command` only |
| `INTERNAL_SERVICE_TOKEN` | shared Spring↔Python auth | both apps |
| `TRANSLATE_API_KEY` | upstream translation provider | Python app |

**This naming is the only mechanical enforcement of the Neon pooler rule.**
Setting `spring.flyway.url` makes Flyway build its own DataSource, so it
physically cannot inherit the pooled connection. Getting this wrong produces no
error — it fails intermittently under multi-instance deploys. Do not "simplify"
by pointing both at one URL.

### Behaviours to account for

- **Values cannot be read back.** `fly secrets list` returns names and digests
  only. Fly is therefore *not* the source of truth — keep the canonical copy in
  1Password (solo) or SOPS + age (if CI or more than one person).
- **Setting a secret triggers a rolling restart.** Batch multiple `KEY=VALUE`
  pairs into one command, or use `--stage` then `fly secrets deploy`.
- **Secrets are per-app.** Spring and Python are separate Fly apps; shared
  values must be set on both.

Non-sensitive config (`SPRING_PROFILES_ACTIVE`, `JAVA_TOOL_OPTIONS`, log levels)
goes in `fly.toml` under `[env]`. `fly.toml` is committed — never put a secret
there.

### On the ECS migration

Source of truth stays platform-independent, so migrating means changing the push
target from `fly secrets import` to `aws ssm put-parameter`, nothing else.
SSM Parameter Store Standard tier is free (10k params; SecureString with the
AWS-managed KMS key costs nothing). Avoid Secrets Manager at $0.40/secret/month
unless automatic rotation becomes a requirement.

Two ECS-specific traps to remember later: the SSM read permission belongs on the
**task execution role**, not the task role; and the task needs a network path to
the SSM endpoint — another consequence of the public-subnet decision.

---

## Secrets and Configuration

### Naming convention (load-bearing)

Spring Boot relaxed binding maps env vars to properties, so secrets are named
after the property they set. Two of them encode the Neon pooler rule:

| Secret | Value | Used by |
|---|---|---|
| `SPRING_DATASOURCE_URL` | Neon **pooled** (`-pooler`) | application runtime |
| `SPRING_FLYWAY_URL` | Neon **direct** (no `-pooler`) | `release_command` only |

Setting `spring.flyway.url` makes Flyway build its own DataSource instead of
reusing the application's, which is what keeps migrations off the pooler.
**This is the only place the Neon rule is actually enforced.** Getting it wrong
does not raise an error — it fails intermittently under multi-instance deploys.

Flyway is gated by profile, not by hoping nobody calls it:
- `application.yml` (default): `spring.flyway.enabled=false`
- `application-migrate.yml`: `spring.flyway.enabled=true`,
  `spring.main.web-application-type=none`
- `release_command` runs with `--spring.profiles.active=migrate`

### Storage

- **Fly.io**: `fly secrets set` / `fly secrets import`. Encrypted vault, injected
  as env vars at Machine boot. Free.
- **Later on ECS**: SSM Parameter Store Standard tier (free up to 10k params;
  SecureString with the AWS-managed KMS key is also free). Not Secrets Manager
  — $0.40/secret/month buys rotation we do not need.

Three Fly behaviours to account for:
1. Setting a secret triggers a rolling restart. Batch multiple `KEY=VALUE` pairs
   into one command, or use `--stage` then `fly secrets deploy`.
2. **Values cannot be read back.** `fly secrets list` returns names and digests
   only. Fly is therefore not the source of truth.
3. Secrets are per-app. The Spring app and Python app need theirs set separately.

### Source of truth lives outside the platform

Because the plan is Fly.io now, ECS later, secrets must not be bound to either.
Use SOPS + age with `secrets/prod.enc.yaml` committed encrypted; age private key
in a password manager and in a CI secret. Deployment decrypts and pushes:

```bash
sops -d secrets/prod.enc.yaml | yq -r 'to_entries|.[]|"\(.key)=\(.value)"' \
  | fly secrets import -a <app> --stage
```

Migrating to ECS replaces only the last line with `aws ssm put-parameter`.

Non-sensitive config (`SPRING_PROFILES_ACTIVE`, `JAVA_TOOL_OPTIONS`,
`PYTHON_SERVICE_URL`) goes in `fly.toml` under `[env]`, which is committed.
Never put a credential there.

When on ECS, the secret-injection permission belongs to the **task execution
role**, not the task role. This is a common mistake.

---

## Fly.io Deployment

- Region: `nrt` (Tokyo), lowest latency from Taiwan.
- Spring: `auto_stop_machines = "suspend"`, `min_machines_running = 0`, two
  machines. The earlier "always on, JVM cold start is too slow" rule was an
  argument against `"stop"` — suspend restores a RAM snapshot, so waking pays no
  JVM boot. Suspend is also the safer mode for this code: translation runs on an
  `@Async` thread (`AIService.processLyricsAsync`) after the HTTP response has
  already been sent, and Fly Proxy's idle decision counts proxied requests only,
  so it cannot see that thread and may idle the machine mid-translation.
  `"stop"` would SIGTERM the JVM, skipping the catch blocks and stranding the
  song in `PENDING` forever — nothing sweeps or retries it, and it burns the
  author's daily post slot. Suspend freezes instead; on resume the dead socket
  throws into the catch block and the song lands on `FAILED`.
  **Keep both machines.** Suspended machines bill no compute, so the second one
  is nearly free, and it is what makes `strategy = "rolling"` actually roll.
  With a single machine a deploy is an in-place restart: dead until readiness
  passes, and up to ~110s of that if the 100s `@Async` drain
  (`spring.task.execution.shutdown.await-termination-period`) fires.
- Python: **auto-stop machines enabled.** It genuinely stops and stops billing
  when no jobs exist. This pairs naturally with the async design and is a
  primary reason Fly.io was chosen.
- Service-to-service over private networking: **`http://<python-app>.flycast`**.
  Flycast goes through Fly Proxy, which serves the `[http_service]` **published**
  ports (80/443) and forwards to `internal_port` (8000) — so target port **80**
  (the default), **not** `:8000`. Hitting the internal port over flycast resets
  the connection ("connection reset by peer"). Python needs no public ingress at all.

  **Use `.flycast`, not `.internal`.** This is critical and easy to get wrong.
  `.internal` resolves straight to Machine IPs over 6PN and bypasses Fly Proxy —
  which means it will *not* wake a stopped Machine, so every request to an
  auto-stopped Python service fails. `.flycast` is a private anycast address
  routed through Fly Proxy, so it triggers auto-start and load balances across
  Machines. Auto-stop is a core reason we chose Fly.io, so this must be
  `.flycast`.

- Because service-to-service uses `.flycast` (above), traffic goes through Fly
  Proxy — and **Fly Proxy dials each VM over a private IPv4 address**. So the
  Python process must bind **`0.0.0.0`** (uvicorn: `--host 0.0.0.0`), not `::`.
  A `::` bind is IPv6-only on Fly's VMs (verified: `127.0.0.1:8000` →
  ECONNREFUSED, `::1` → open), which the proxy cannot reach — every flycast
  request then resets with "connection reset by peer", even though the Machine
  is `started` and uvicorn logs `Uvicorn running`. `::` would be correct only
  for direct `.internal` 6PN (IPv6), which this architecture deliberately does
  not use.

Estimated $10-15/month.

---

## Cloudflare (domain already owned, use free tier aggressively)

- **Pages — not used.** The frontend is a Next.js BFF with a server side, so it
  cannot be a static export. It deploys to Vercel instead (RUNBOOK 7.3).
- **Proxy (orange cloud)** — free DDoS protection, managed WAF rules, TLS.
  **Not in front of the frontend**: `slangdump.com` is DNS-only (grey cloud), and
  RUNBOOK 7.3 records why — an external proxy hides the real client IP from
  Vercel's own bot and WAF logic, and double-CDN caching collapses Next's
  `Vary: RSC` responses onto one cache key. This applies to the Fly-hosted API.
- **Rate Limiting** — free tier allows 1 rule. Apply it to
  `POST /translations` so nobody can run up the translation API bill.
- **Cache Rules** — terminal job responses can be served from cache.
- Proxy read timeout is 120s and not adjustable below Enterprise. Current design
  keeps every frontend request under 1 second, so this is a non-issue —
  **but it is one reason the synchronous design cannot be reinstated.**

---

## Future: ECS Migration (not started)

Listed only to avoid decisions now that would obstruct it later.

- Everything defined in **Terraform**, not the Console.
- Fargate ARM (Graviton, ~20% cheaper than x86).
- **Public subnet with `assignPublicIp`, no NAT Gateway.** NAT starts at
  $32/month; a public IP is $3.6/task. The security boundary is the Security
  Group, not the absence of a public IP.
- ALB for ingress, SG restricted to Cloudflare IP ranges, Authenticated Origin
  Pulls enabled.
- Spring to Python over ECS Service Connect (free, no internal ALB needed).
- Python on Fargate Spot — jobs are retryable, interruption is harmless.
- Flyway becomes a one-off `RunTask`.

Estimated $40-45/month. ALB was chosen over Cloudflare Tunnel deliberately:
target groups, health checks and connection draining have learning value.

**Architecture note — this was previously stated incorrectly.** Fly.io Machines
are **amd64 only**; pushing an arm64 image fails with `image must be amd64
architecture for linux os, found arm64 linux`. This bites on Apple Silicon,
where Docker silently builds arm64 by default.

So: build `linux/amd64` for Fly (use `fly deploy --remote-only`, or
`docker buildx build --platform linux/amd64`). Keep Dockerfiles
architecture-agnostic — no hardcoded arch in base image tags or downloaded
binaries — so the same files produce arm64 images for Graviton at ECS migration
time. Do not add `--platform=arm64` anywhere yet.

---

## Working Agreements

- The decisions above are conclusions from evaluated trade-offs. Before
  proposing an alternative, check whether it was already rejected. Explicitly
  rejected: **making Python async, switching to SSE or WebSockets, in-memory
  job queues, running Flyway at application startup, long polling, and
  Elasticsearch for search.**
- **Search runs entirely inside Neon.** `pg_trgm` for lexical matching,
  `pgvector` for a semantic pass that fires only when the lexical one comes back
  thin. Two things follow that are easy to re-propose by accident:
  - **Elasticsearch was evaluated and rejected**, despite being what the phase
    plan originally named. It is a paid service (or an operated cluster) and the
    corpus is thousands of documents; the cost rule above applies. It stays
    deferred, not disproven — revisit if phrase and proximity queries or real
    CJK analyzers become necessary.
  - **Postgres full-text search is not usable here**, so do not propose a
    `tsvector`. The corpus is mostly Traditional Chinese and Neon's extension
    allowlist has no Chinese tokenizer (`zhparser`, `pg_jieba`, `pgroonga` are
    all absent), so `to_tsvector` collapses a whole sentence to one token.
    Trigrams need no tokenizer. See `docs/adr/0003`.
- **Retry for transient Gemini failures lives in the worker**, not in Spring.
  Only the worker can tell a transient failure from RECITATION, MAX_TOKENS or
  malformed JSON — Spring sees `502` for all of those. There are **two**
  mechanisms in `services/llm_service.py`, because one cannot cover the other:
  - `RETRY_OPTIONS` — HTTP 429/503 only, 3 retries, exponential backoff +
    jitter, ~7-10s worst case. Runs inside the google-genai SDK.
  - `EMPTY_RESPONSE_ATTEMPTS` — Gemini answering **HTTP 200 with a zero-token
    empty candidate** (STOP, no parts, `total_token_count ==
    prompt_token_count`). The SDK's retry gates on HTTP status and so is blind
    to it; `LLMService.process` re-fires it itself, 2 retries at 1s/2s. Gated on
    zero generated tokens, which is what makes the retry free and what
    distinguishes it from the grounding-only empty candidate (real tokens spent
    — not retried, stays `502`). Persisting across all attempts returns `503`,
    not `502`: the input is provably not at fault.

  Spring does **not** retry the worker call; that was not rejected, it simply
  addresses a *different* failure (an unreachable worker) which remains
  unaddressed. Note also that a song which exhausts its retries stays `FAILED`
  forever and is reused as-is by every later post of it (RUNBOOK Phase 9
  item 4) — which is why a transient failure must never be classified as
  permanent.
- Cost-sensitive. Justify why the current approach is insufficient before
  introducing any new paid service (Redis, SQS, extra load balancers).
- Prefer "simple now, easy to move later" over "perfect now".
