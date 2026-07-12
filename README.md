# slangdump

AI lyrics-translation platform: paste rap/hip-hop lyrics, get a Traditional-Chinese
translation with slang/idiom annotations and cultural context.

## Architecture

Three services plus PostgreSQL (multi-repo). The frontend never talks to the AI worker
directly — Spring is the orchestrator and the **only** service that touches the database.

```
Browser ──► slangdump-poster ──► slangdump-pick-n-roll ──► slangdump-ai-solation ──► Gemini
 :3000      (Next.js)        :8080 (Spring/Kotlin)     :8000 (FastAPI, stateless)
                                   │
                                   ▼
                              PostgreSQL :5432
```

| Component | Repo | Tech | Port | Stores data? |
| --- | --- | --- | --- | --- |
| Frontend | `slangdump-poster` | Next.js, Tailwind | 3000 | No (proxies `/api/posts*` → Spring) |
| Backend | `slangdump-pick-n-roll` | Spring Boot 4, Kotlin, Java 21 | 8080 | **Yes — owns Postgres** (Flyway + JPA) |
| AI worker | `slangdump-ai-solation` | Python FastAPI, google-genai | 8000 | No — stateless, calls Gemini only |
| Database | — | PostgreSQL 16 | 5432 | — |

Original lyrics are never persisted or returned: the user pastes them per request, the AI
worker translates them in memory, and only translation artefacts (translated lines,
alignments, annotations) are stored by Spring.

## Run the full stack

`docker-compose.yml` defines all three backend services — `postgres`, `spring`, and
`ai-worker`. There are two ways to bring the stack up; pick based on whether you're
actively editing the backend.

The `ai-worker` container reads `slangdump-ai-solation/.env`, so that file must exist with
a `GEMINI_API_KEY` before either path.

### Dev loop (recommended while editing the backend)

Run only the **dependencies** in Docker, and run the services you're editing — Spring and
the frontend — on the host so they hot-reload. `spring` is intentionally **left out** of
the compose command here: you run it with `./gradlew bootRun` instead.

```sh
# from this directory (repo root)
docker-compose up -d postgres ai-worker
cd slangdump-pick-n-roll && ./gradlew bootRun &   # Spring on :8080 (host, hot reload)
cd ../slangdump-poster   && npm install && npm run dev &
```

Host-run Spring reaches the containers via published ports (`postgres` on :5432,
`ai-worker` on :8000) using its `localhost` defaults — no extra config needed.

### Full container stack

Build and run all three backend services in Docker, then run only the frontend on the
host:

```sh
# from this directory (repo root)
docker-compose up -d                              # postgres + spring + ai-worker
cd slangdump-poster && npm install && npm run dev &
```

Here the `spring` container talks to the others over the compose network (`postgres:5432`,
`ai-worker:8000`, wired via env in `docker-compose.yml`).

---

Either way, open http://localhost:3000. The frontend proxies `/api/posts*` to Spring on
:8080, which orchestrates the Python AI worker on :8000 and persists to Postgres on :5432.

To reset the database during development, see
[`slangdump-pick-n-roll/README.md`](slangdump-pick-n-roll/README.md#dev-db-reset).

## Repos

- [`slangdump-poster`](slangdump-poster/README.md) — Next.js frontend.
- [`slangdump-pick-n-roll`](slangdump-pick-n-roll/README.md) — Spring Boot orchestrator + Postgres.
- [`slangdump-ai-solation`](slangdump-ai-solation/README.md) — Python AI translation worker.
