# One Drop Per Day + Unified Error Contract — Design

**Date:** 2026-07-16
**Projects:** `slangdump-pick-n-roll` (Kotlin/Spring backend), `slangdump-poster` (Next.js frontend)
**Status:** Approved design, pre-implementation

## Overview

An authenticated user may create **one post per calendar day in Asia/Taipei**. The backend
is the authoritative gate; the frontend mirrors the state so the Daily Drop editor locks
after the day's post and shows a live countdown to the next reset (00:00 Asia/Taipei),
replacing today's hardcoded `Reset in 14:22:05` string.

Alongside this, the API's error body is upgraded from `{ "error": "<message>" }` to a
**unified `{ code, message }` contract with numeric codes**, so the frontend can branch on a
stable code rather than matching on message text.

## Goals

- Enforce **one post per user per Taipei day** on the backend, before any song/worker work.
- A **proactive** eligibility signal so the editor renders locked (not just an error on submit).
- A **live `HH:MM:SS` countdown** in the editor header to the next Taipei midnight.
- A **unified error contract** (`{ code, message, ... }`) with **numeric codes** applied
  API-wide (backend handlers + Next.js proxy routes), consumed by code on the frontend.
- New time fields expressed as **epoch milliseconds (UTC)**.

## Non-Goals

- Rate limits other than the one-per-day post rule (no per-hour, per-IP, etc.).
- Admin/exempt accounts or per-user overrides.
- A hard DB-level uniqueness guarantee against the check-then-insert race (see Accepted Risks).
- Changing existing DTO fields (e.g. `createdAt`) to epoch millis — only the two *new* time
  fields use millis; existing fields keep their current serialization to preserve the feed contract.

## Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Day boundary | Calendar day in `Asia/Taipei`, reset 00:00 | Single fixed zone; reset is the same wall-clock for everyone. |
| What counts as a post | Any `Post` row created since the last Taipei midnight, even if the shared translation later `FAILED` | The post row is created regardless of translation outcome; the limit is on posting, not on successful translation. |
| Frontend awareness | Proactive `GET /eligibility` endpoint | Editor locks on load; POST 429 remains the real gate (defense in depth). |
| Block status code | **429 Too Many Requests** + `Retry-After` | Semantically "you may retry later"; fits a per-day cap better than 409. |
| Error contract | Unified `{ code, message, ... }`, **API-wide** | One error shape everywhere so the frontend never falls back to message matching. |
| Code format | **Numeric**, `HTTP-status × 100 + sequence` | Self-locating: `floor(code/100)` is the HTTP status; last two digits identify the specific error. |
| Time serialization (new fields) | **Epoch milliseconds (UTC)** | Countdown math is `resetAt - Date.now()` with no parse step; `new Date(ms)` also just works. |

## Error contract

Every error response — from the Spring backend **and** the Next.js proxy routes — uses:

```json
{ "code": 42901, "message": "You've already dropped today. Come back after reset.", "resetAt": 1752681600000 }
```

- Base shape is `{ code, message }`. Specific errors may add fields (`resetAt` for the daily
  limit), omitted when null.
- The frontend branches on `code`, never on `message`. `message` is display text only.

### Code catalog

| Code | Meaning | HTTP | Extra fields |
|------|---------|------|--------------|
| 40001 | Validation error (invalid field values) | 400 | — |
| 40002 | Malformed request body | 400 | — |
| 40003 | Missing required parameter | 400 | — |
| 40004 | Invalid parameter value | 400 | — |
| 40100 | Unauthorized (sign in to post) — emitted by the Next.js proxy | 401 | — |
| 40400 | Not found | 404 | — |
| 42901 | Daily post limit reached | 429 | `resetAt` (epoch ms) |
| 50000 | Internal error | 500 | — |

The scheme is `HTTP-status × 100 + sequence`; the generic `ResponseStatusException` path maps
404 → 40400 and falls back to `status × 100` for any other status.

## Architecture — Backend (`slangdump-pick-n-roll`)

- **`TaipeiClock`** (new small component) — wraps an injectable `java.time.Clock` and a fixed
  `ZoneId.of("Asia/Taipei")`:
  - `startOfToday(): Instant` = `LocalDate.now(clock.withZone(zone)).atStartOfDay(zone).toInstant()`.
  - `nextReset(): Instant` = next Taipei midnight.
  - Injectable clock makes the midnight boundary deterministically testable.
- **`ApiError` enum** — single source of truth for the catalog: `(code: Int, status: HttpStatus,
  defaultMessage: String)`. One constant per row in the catalog above.
- **`ApiErrorResponse`** — `data class (code: Int, message: String, resetAt: Long? = null)` with
  Jackson `@JsonInclude(NON_NULL)` so `resetAt` is omitted unless set.
- **`DailyPostLimitException(resetAt: Instant)`** — new exception carrying the reset instant.
- **`ApiExceptionHandler`** — every handler now returns `ApiErrorResponse` instead of
  `{ error }`:
  - Validation buckets → 40001/40002/40003/40004 (messages unchanged, still detailed).
  - `ResponseStatusException` → 404 maps to 40400; otherwise `status × 100`, generic reason phrase.
  - Generic `Exception` → 50000.
  - New `DailyPostLimitException` → 429, code 42901, message + `resetAt` (`Instant.toEpochMilli()`),
    plus a `Retry-After` header in **seconds** (`(resetAt - now)` rounded up).
- **`PostRepository`** — add `fun existsByAuthorAndCreatedAtGreaterThanEqual(author: User,
  since: Instant): Boolean`.
- **`PostService.create()`** — immediately after `val author = userService.syncFromClaims(claims)`,
  if `postRepository.existsByAuthorAndCreatedAtGreaterThanEqual(author, taipeiClock.startOfToday())`
  then throw `DailyPostLimitException(taipeiClock.nextReset())` — before any song lookup/create
  or worker scheduling.
- **`PostController`** — add `GET /api/v1/posts/eligibility` (authenticated):
  resolves the author from the JWT, returns
  `PostEligibilityDto(canPostToday: Boolean, nextResetAt: Long /* epoch ms */)` computed from the
  same repository check + `TaipeiClock`.

## Architecture — Frontend (`slangdump-poster`)

- **`src/lib/errorCodes.ts`** (new) — exports the numeric code constants (e.g.
  `DAILY_POST_LIMIT = 42901`, `UNAUTHORIZED = 40100`, …), the `ApiErrorBody` type
  (`{ code: number; message: string; resetAt?: number }`), and
  `parseApiError(res: Response): Promise<ApiErrorBody>` (parses JSON, tolerates a non-JSON body
  by synthesizing `{ code: 50000, message: <text> }`).
- **Proxy routes** (`src/app/api/posts/route.ts`, `[id]/route.ts`, and the new eligibility route)
  — emit the unified shape for errors they originate: the POST route's own 401 → `{ code: 40100,
  message: 'Sign in to post' }`; the catch-all 500 → `{ code: 50000, message }`. Backend error
  bodies already conform and pass through unchanged.
- **`src/app/api/posts/eligibility/route.ts`** (new) — `GET` mirrors the POST route's auth:
  requires a session, mints a BFF token, forwards to `GET {BACKEND}/api/v1/posts/eligibility`,
  passes the JSON through.
- **`DailyDropEditor.tsx`**:
  - On mount (authenticated), `GET /api/posts/eligibility`; store `canPostToday` + `nextResetAt`.
  - **Countdown**: replace `Reset in 14:22:05` with a live `HH:MM:SS` counter driven by
    `nextResetAt - Date.now()`, updated every second via `setInterval` (cleared on unmount).
    Fallback if eligibility hasn't loaded: compute the next Taipei midnight client-side via
    `Intl.DateTimeFormat` with `timeZone: 'Asia/Taipei'`.
  - **Locked state** (`!canPostToday`): render a "you've dropped today — come back after reset"
    panel with the prominent countdown; inputs and `DROP IT` disabled.
  - After a successful post (`onPostCreated`), set `canPostToday = false` locally so it locks
    immediately without a refetch.
  - On a POST error, `parseApiError`: if `code === DAILY_POST_LIMIT` → lock and set `nextResetAt`
    from `resetAt`; otherwise show `message`. Existing 401 handling maps to code 40100.
  - When the countdown reaches zero, re-fetch eligibility.
- **Other consumers** that currently surface `error` text switch to `parseApiError(...).message`:
  `FeedItem` (translate fetch) and `page.tsx` (feed list load).

## Data flow

1. Editor mounts → `GET /api/posts/eligibility` → proxy (BFF token) → backend computes
   `canPostToday` / `nextResetAt` from `TaipeiClock` + repository → editor renders enabled or locked,
   countdown ticking.
2. User submits → `POST /api/posts` → proxy → `PostService.create()`:
   - Already posted today → `DailyPostLimitException` → 429 `{ code: 42901, message, resetAt }`
     → editor locks and syncs the countdown to `resetAt`.
   - Otherwise → post created; frontend sets `canPostToday = false` and locks.

## Testing

**Backend**
- `PostService`: second post the same Taipei day throws `DailyPostLimitException`; a post whose
  `createdAt` is just before Taipei midnight does **not** block the next day — asserted with a
  fixed injected `Clock` around the boundary.
- `PostRepository`: `existsByAuthorAndCreatedAtGreaterThanEqual` returns true/false across the boundary.
- `ApiExceptionHandler`: each bucket emits the correct `code` + HTTP status + body; the daily-limit
  case includes `resetAt` (epoch ms) and a `Retry-After` header.
- `PostController`: eligibility endpoint returns correct `canPostToday` / `nextResetAt`; create
  returns 429 with code 42901 when already posted.

**Frontend**
- `parseApiError`: JSON body parsed; non-JSON body synthesized to `{ code: 50000, message }`.
- `DailyDropEditor`: locked when `canPostToday=false`; countdown renders and ticks (fake timers);
  `DROP IT` disabled when locked; a 42901 POST response locks the editor and sets the countdown;
  editor locks after a successful post.
- Eligibility proxy route: unauthenticated → 40100; authenticated → forwards and passes JSON through.

## Accepted risks

- **Check-then-insert race.** Two near-simultaneous submits from the same user could both pass the
  existence check before either commits, allowing two posts in one day. For a one-drop-per-day
  social feature this window is negligible and accepted. Optional future hardening (out of scope): a
  partial unique index on `(user_id, <taipei_date>)` via a generated date column, or a
  `SELECT … FOR UPDATE` guard.
