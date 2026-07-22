# One Drop Per Day + Unified Error Contract — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enforce one post per user per Asia/Taipei calendar day (backend-authoritative, frontend-mirrored with a live countdown), and migrate the whole API to a unified `{ code, message }` numeric error contract.

**Architecture:** The Spring backend gains a `TaipeiClock`, a repository existence check, a guard in `PostService.create()`, a `GET /api/v1/posts/eligibility` endpoint, and a rewritten `ApiExceptionHandler` emitting numeric codes. The Next.js frontend gains an `errorCodes` module, an eligibility proxy route, and a `DailyDropEditor` that locks with a live `HH:MM:SS` countdown replacing the hardcoded `Reset in 14:22:05`.

**Tech Stack:** Kotlin / Spring Boot 4 / JUnit 5 / Mockito / Testcontainers (backend); Next.js App Router / TypeScript / Vitest / React Testing Library (frontend).

**Design doc:** `docs/plans/2026-07-16-daily-post-limit-design.md`

## Global Constraints

- One post per authenticated user per **calendar day in Asia/Taipei**; reset at 00:00 Taipei. A post counts even if its translation later becomes `FAILED`.
- Every error body is `{ "code": <int>, "message": "<display text>" }`; codes follow `HTTP status × 100 + sequence` (catalog below). Frontend branches on `code`, never on `message`.
- New time fields (`resetAt`, `nextResetAt`) are **epoch milliseconds UTC** (`Long`/`number`). Existing DTO fields (e.g. `createdAt`) keep their current serialization.
- Daily-limit rejection is **429** with a `Retry-After` header in seconds.
- Every JUnit test method gets a `@DisplayName` (user preference).
- Backend tests need Docker running (Testcontainers Postgres).
- Repos are **separate git repositories**. Work on branch `feat/daily-post-limit` in each: `slangdump-pick-n-roll` (Tasks 1–4) and `slangdump-poster` (Tasks 5–10). Both branch off `master`.

### Error-code catalog (single source of truth)

| Code  | Constant           | Meaning                               | HTTP | Extra fields |
|-------|--------------------|---------------------------------------|------|--------------|
| 40001 | VALIDATION_ERROR   | Validation error (detailed message)   | 400  | — |
| 40002 | MALFORMED_BODY     | Malformed request body                | 400  | — |
| 40003 | MISSING_PARAMETER  | Missing required parameter            | 400  | — |
| 40004 | INVALID_PARAMETER  | Invalid parameter value               | 400  | — |
| 40100 | UNAUTHORIZED       | No session (emitted by Next.js proxy) | 401  | — |
| 40400 | NOT_FOUND          | Not found                             | 404  | — |
| 42901 | DAILY_POST_LIMIT   | Daily post limit reached              | 429  | `resetAt` (epoch ms) |
| 50000 | INTERNAL_ERROR     | Internal error                        | 500  | — |

### Noted deviations from the design doc (intentional)

- The design doc names an `ApiErrorResponse` data class with `@JsonInclude(NON_NULL)`. The handler instead builds `Map<String, Any>` bodies (the file's existing idiom); the wire shape is identical and extra fields are simply only added when present.
- The design doc suggests an `Intl.DateTimeFormat` fallback for the client-side Taipei midnight. Taipei is UTC+8 year-round (no DST), so the fallback uses a fixed offset — exact, simpler, unit-testable.
- The design doc lists a dedicated `PostRepository` boundary test. The derived query is exercised end-to-end against real Postgres by the 429/eligibility integration tests (both true and false outcomes); the midnight *boundary* math is covered by `TaipeiClockTest` and `PostServiceTest` with fixed clocks. No separate repository test class.

---

## Task 1: Unified `{ code, message }` error contract (backend)

**Files:**
- Create: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/web/ApiError.kt`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/web/ApiExceptionHandler.kt` (full rewrite of body shape)
- Test: `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/controller/PostControllerIntegrationTest.kt`

**Interfaces:**
- Consumes: existing `ValidationException`, existing handler structure.
- Produces: `enum class ApiError(val code: Int, val status: HttpStatus, val defaultMessage: String)` with constants `VALIDATION_ERROR(40001)`, `MALFORMED_BODY(40002)`, `MISSING_PARAMETER(40003)`, `INVALID_PARAMETER(40004)`, `NOT_FOUND(40400)`, `DAILY_POST_LIMIT(42901)`, `INTERNAL_ERROR(50000)`; every error body is `{ "code": Int, "message": String }`. Later tasks add handlers to this file.

- [ ] **Step 0: Create the branch**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
git checkout master && git checkout -b feat/daily-post-limit
```

- [ ] **Step 1: Update the integration-test assertions to the new shape (failing tests)**

In `PostControllerIntegrationTest.kt`, change the five error-shape assertions:

```kotlin
    @Test
    @DisplayName("POST rejects blank songName with 400, code 40001, and a message naming the field")
    fun `POST rejects blank songName with 400`() {
        mockMvc.perform(
            postJson("""{ "artist": "Drake", "songName": "", "rawLyrics": "lyrics" }""")
        ).andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.code").value(40001))
            .andExpect(jsonPath("$.message").value(org.hamcrest.Matchers.containsString("songName")))
    }

    @Test
    @DisplayName("POST rejects a missing required field with 400 and code 40001 naming the field")
    fun `POST rejects missing required field with 400`() {
        // artist omitted entirely — nullable + @NotBlank surfaces it as a validation error.
        mockMvc.perform(
            postJson("""{ "songName": "One Dance", "rawLyrics": "lyrics" }""")
        ).andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.code").value(40001))
            .andExpect(jsonPath("$.message").value(org.hamcrest.Matchers.containsString("artist")))
    }

    @Test
    @DisplayName("POST rejects a malformed JSON body with 400 and code 40002")
    fun `POST rejects malformed JSON body with 400`() {
        mockMvc.perform(postJson("""{ "artist": "Drake", """))
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.code").value(40002))
            .andExpect(jsonPath("$.message").value("Malformed request body"))
    }

    @Test
    @DisplayName("GET detail of a missing post returns 404 with code 40400 and a generic message")
    fun `GET detail of missing post returns 404 generic`() {
        mockMvc.perform(get("/api/v1/posts/99999999"))
            .andExpect(status().isNotFound)
            .andExpect(jsonPath("$.code").value(40400))
            // Generic reason phrase only — the frontend should not learn which exception fired.
            .andExpect(jsonPath("$.message").value("Not Found"))
    }

    @Test
    @DisplayName("GET list rejects malformed cursor with 400, code 40001, and a detailed message")
    fun `GET list rejects malformed cursor with 400`() {
        mockMvc.perform(get("/api/v1/posts").param("cursor", "not-a-cursor!!"))
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.code").value(40001))
            .andExpect(jsonPath("$.message").value("Malformed cursor"))
    }
```

- [ ] **Step 2: Run them to verify they fail**

Run: `./gradlew test --tests 'com.example.demo.controller.PostControllerIntegrationTest'`
Expected: the five updated tests FAIL (body still has `error`, no `code`/`message`); the rest pass.

- [ ] **Step 3: Create the `ApiError` catalog**

Create `src/main/kotlin/com/example/demo/web/ApiError.kt`:

```kotlin
package com.example.demo.web

import org.springframework.http.HttpStatus

/**
 * Catalog of API error codes. Scheme: `HTTP status × 100 + sequence`, so
 * `code / 100` recovers the HTTP status and the last two digits identify the
 * specific error. The frontend mirrors these constants (`src/lib/errorCodes.ts`
 * in slangdump-poster) and branches on `code`, never on `message`.
 *
 * 40100 (unauthorized) is emitted by the Next.js proxy, not listed here — backend
 * 401s come from Spring Security before any controller runs.
 */
enum class ApiError(val code: Int, val status: HttpStatus, val defaultMessage: String) {
    VALIDATION_ERROR(40001, HttpStatus.BAD_REQUEST, "Validation failed"),
    MALFORMED_BODY(40002, HttpStatus.BAD_REQUEST, "Malformed request body"),
    MISSING_PARAMETER(40003, HttpStatus.BAD_REQUEST, "Missing required parameter"),
    INVALID_PARAMETER(40004, HttpStatus.BAD_REQUEST, "Invalid parameter value"),
    NOT_FOUND(40400, HttpStatus.NOT_FOUND, "Not Found"),
    DAILY_POST_LIMIT(42901, HttpStatus.TOO_MANY_REQUESTS, "You've already dropped today. Come back after reset."),
    INTERNAL_ERROR(50000, HttpStatus.INTERNAL_SERVER_ERROR, "Internal Server Error"),
}
```

- [ ] **Step 4: Rewrite `ApiExceptionHandler` to emit `{ code, message }`**

Replace the entire body of `src/main/kotlin/com/example/demo/web/ApiExceptionHandler.kt` with:

```kotlin
package com.example.demo.web

import org.slf4j.LoggerFactory
import org.springframework.http.HttpStatus
import org.springframework.http.HttpStatusCode
import org.springframework.http.ResponseEntity
import org.springframework.http.converter.HttpMessageNotReadableException
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.MissingServletRequestParameterException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import org.springframework.web.method.annotation.HandlerMethodValidationException
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException
import org.springframework.web.server.ResponseStatusException

/**
 * Centralizes the error-body contract for the API. Every error responds with
 * `{ "code": <int>, "message": "<display text>" }` — see [ApiError] for the catalog.
 * Specific errors may add fields (the daily post limit adds `resetAt`, epoch ms).
 * The frontend branches on `code`, never on `message`.
 *
 * Two buckets, as before:
 *  - **Validation errors** (bad client input): detailed `message` naming exactly which
 *    parameter is wrong, so the caller can fix the request.
 *  - **Everything else**: the generic HTTP reason phrase only. Internal reasons (e.g.
 *    "post 42 not found", stack messages) are logged server-side, never leaked.
 */
@RestControllerAdvice
class ApiExceptionHandler {

    private val log = LoggerFactory.getLogger(ApiExceptionHandler::class.java)

    // --- Validation bucket: detailed, names the offending parameter. ---

    @ExceptionHandler(ValidationException::class)
    fun handleValidation(e: ValidationException): ResponseEntity<Map<String, Any>> {
        val detail = e.message ?: "Validation failed"
        log.warn("Validation failed: {}", detail)
        return error(ApiError.VALIDATION_ERROR, detail)
    }

    // Raised by Bean Validation (@Validated controller + @NotBlank etc. on parameters).
    @ExceptionHandler(HandlerMethodValidationException::class)
    fun handleMethodValidation(e: HandlerMethodValidationException): ResponseEntity<Map<String, Any>> {
        val detail = e.parameterValidationResults.joinToString("; ") { result ->
            val name = result.methodParameter.parameterName ?: "parameter"
            val message = result.resolvableErrors.mapNotNull { it.defaultMessage }.joinToString(", ")
            "$name $message".trim()
        }.ifBlank { "Validation failed" }
        log.warn("Method validation failed: {}", detail)
        return error(ApiError.VALIDATION_ERROR, detail)
    }

    // Raised by @Valid on a @RequestBody DTO (e.g. @NotBlank fields on CreatePostRequest).
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleBodyValidation(e: MethodArgumentNotValidException): ResponseEntity<Map<String, Any>> {
        val detail = e.bindingResult.fieldErrors.joinToString("; ") { fe ->
            "${fe.field} ${fe.defaultMessage ?: "is invalid"}".trim()
        }.ifBlank { "Validation failed" }
        log.warn("Body validation failed: {}", detail)
        return error(ApiError.VALIDATION_ERROR, detail)
    }

    // Body that can't be parsed at all (bad JSON syntax, wrong types). Keep it opaque —
    // don't echo Jackson's internal parse message back to the client.
    @ExceptionHandler(HttpMessageNotReadableException::class)
    fun handleUnreadableBody(e: HttpMessageNotReadableException): ResponseEntity<Map<String, Any>> {
        log.warn("Unreadable request body: {}", e.mostSpecificCause.message)
        return error(ApiError.MALFORMED_BODY)
    }

    @ExceptionHandler(MissingServletRequestParameterException::class)
    fun handleMissingParam(e: MissingServletRequestParameterException): ResponseEntity<Map<String, Any>> {
        log.warn("Missing required parameter: {}", e.parameterName)
        return error(ApiError.MISSING_PARAMETER, "Missing required parameter: ${e.parameterName}")
    }

    @ExceptionHandler(MethodArgumentTypeMismatchException::class)
    fun handleTypeMismatch(e: MethodArgumentTypeMismatchException): ResponseEntity<Map<String, Any>> {
        log.warn("Invalid value for parameter '{}': {}", e.name, e.value)
        return error(ApiError.INVALID_PARAMETER, "Invalid value for parameter: ${e.name}")
    }

    // --- Everything else: opaque. Keep the status, return only its reason phrase. ---

    @ExceptionHandler(ResponseStatusException::class)
    fun handleStatus(e: ResponseStatusException): ResponseEntity<Map<String, Any>> {
        // Log the internal reason server-side; the client only gets the generic phrase.
        log.warn("Request failed with {}: {}", e.statusCode, e.reason ?: "(no reason)")
        // `status × 100` is the catalog fallback; 404 lands on ApiError.NOT_FOUND (40400)
        // by construction of the numbering scheme.
        return ResponseEntity.status(e.statusCode)
            .body(mapOf<String, Any>("code" to e.statusCode.value() * 100, "message" to reasonPhrase(e.statusCode)))
    }

    @ExceptionHandler(Exception::class)
    fun handleGeneric(e: Exception): ResponseEntity<Map<String, Any>> {
        log.error("Unhandled exception", e)
        return error(ApiError.INTERNAL_ERROR)
    }

    private fun error(err: ApiError, message: String = err.defaultMessage): ResponseEntity<Map<String, Any>> =
        ResponseEntity.status(err.status).body(mapOf<String, Any>("code" to err.code, "message" to message))

    private fun reasonPhrase(status: HttpStatusCode): String =
        (HttpStatus.resolve(status.value())?.reasonPhrase) ?: "Error"
}
```

- [ ] **Step 5: Run the integration tests to verify they pass**

Run: `./gradlew test --tests 'com.example.demo.controller.PostControllerIntegrationTest'`
Expected: PASS (all tests).

- [ ] **Step 6: Run the full backend suite (nothing else asserted on `$.error` — verified by grep)**

Run: `./gradlew test`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/main/kotlin/com/example/demo/web/ApiError.kt \
        src/main/kotlin/com/example/demo/web/ApiExceptionHandler.kt \
        src/test/kotlin/com/example/demo/controller/PostControllerIntegrationTest.kt
git commit -m "feat: unified { code, message } error contract with numeric codes"
```

---

## Task 2: `TaipeiClock` (backend)

**Files:**
- Create: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/util/TaipeiClock.kt`
- Create: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/config/ClockConfig.kt`
- Test: `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/util/TaipeiClockTest.kt`

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: `@Component class TaipeiClock(clock: Clock)` with `fun startOfToday(): Instant` (Taipei midnight that began the current Taipei day) and `fun nextReset(): Instant` (the coming Taipei midnight); a `Clock` bean (`Clock.systemUTC()`).

- [ ] **Step 1: Write the failing tests**

Create `src/test/kotlin/com/example/demo/util/TaipeiClockTest.kt`:

```kotlin
package com.example.demo.util

import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import java.time.Clock
import java.time.Instant
import java.time.ZoneOffset
import kotlin.test.assertEquals

class TaipeiClockTest {

    private fun clockAt(instant: String) = Clock.fixed(Instant.parse(instant), ZoneOffset.UTC)

    @Test
    @DisplayName("startOfToday is Taipei midnight expressed as an instant (16:00Z the previous UTC day)")
    fun `startOfToday is taipei midnight`() {
        // 2026-07-16T02:00:00Z == 10:00 in Taipei (UTC+8)
        val tc = TaipeiClock(clockAt("2026-07-16T02:00:00Z"))
        assertEquals(Instant.parse("2026-07-15T16:00:00Z"), tc.startOfToday())
    }

    @Test
    @DisplayName("nextReset is the coming Taipei midnight")
    fun `nextReset is next taipei midnight`() {
        val tc = TaipeiClock(clockAt("2026-07-16T02:00:00Z"))
        assertEquals(Instant.parse("2026-07-16T16:00:00Z"), tc.nextReset())
    }

    @Test
    @DisplayName("around Taipei midnight: just before it resets in minutes, just after a full day")
    fun `boundary behavior around taipei midnight`() {
        val before = TaipeiClock(clockAt("2026-07-16T15:59:00Z")) // 23:59 Taipei
        assertEquals(Instant.parse("2026-07-16T16:00:00Z"), before.nextReset())

        val after = TaipeiClock(clockAt("2026-07-16T16:01:00Z")) // 00:01 Taipei, next day
        assertEquals(Instant.parse("2026-07-16T16:00:00Z"), after.startOfToday())
        assertEquals(Instant.parse("2026-07-17T16:00:00Z"), after.nextReset())
    }
}
```

- [ ] **Step 2: Run to verify they fail**

Run: `./gradlew test --tests 'com.example.demo.util.TaipeiClockTest'`
Expected: COMPILATION FAILURE — `TaipeiClock` does not exist.

- [ ] **Step 3: Implement `TaipeiClock` and the `Clock` bean**

Create `src/main/kotlin/com/example/demo/util/TaipeiClock.kt`:

```kotlin
package com.example.demo.util

import org.springframework.stereotype.Component
import java.time.Clock
import java.time.Instant
import java.time.LocalDate
import java.time.ZoneId

/**
 * The posting-day boundary: calendar days in Asia/Taipei. Every "has this user posted
 * today" check and every reset countdown keys off this one component so the backend
 * and the eligibility endpoint can never disagree. The [clock] is injected so tests
 * can pin the current instant right at the midnight boundary.
 */
@Component
class TaipeiClock(private val clock: Clock) {

    private val zone = ZoneId.of("Asia/Taipei")

    /** Start of the current Taipei calendar day, as an instant (16:00Z the previous UTC day). */
    fun startOfToday(): Instant =
        LocalDate.now(clock.withZone(zone)).atStartOfDay(zone).toInstant()

    /** The coming Taipei midnight — when posting resets. */
    fun nextReset(): Instant =
        LocalDate.now(clock.withZone(zone)).plusDays(1).atStartOfDay(zone).toInstant()
}
```

Create `src/main/kotlin/com/example/demo/config/ClockConfig.kt`:

```kotlin
package com.example.demo.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.time.Clock

/** System clock bean so time-dependent components (e.g. TaipeiClock) can be pinned in tests. */
@Configuration
class ClockConfig {
    @Bean
    fun clock(): Clock = Clock.systemUTC()
}
```

- [ ] **Step 4: Run to verify they pass**

Run: `./gradlew test --tests 'com.example.demo.util.TaipeiClockTest'`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/main/kotlin/com/example/demo/util/TaipeiClock.kt \
        src/main/kotlin/com/example/demo/config/ClockConfig.kt \
        src/test/kotlin/com/example/demo/util/TaipeiClockTest.kt
git commit -m "feat: TaipeiClock — Asia/Taipei day boundary with injectable clock"
```

---

## Task 3: Daily-limit guard in `PostService` + 429 handler (backend)

**Files:**
- Create: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/web/DailyPostLimitException.kt`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/repository/PostRepository.kt`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/service/PostService.kt`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/web/ApiExceptionHandler.kt`
- Test: `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/service/PostServiceTest.kt`
- Test: `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/controller/PostControllerIntegrationTest.kt`

**Interfaces:**
- Consumes: `ApiError.DAILY_POST_LIMIT` (Task 1), `TaipeiClock.startOfToday()` / `nextReset()` (Task 2).
- Produces: `class DailyPostLimitException(val resetAt: Instant)`; `PostRepository.existsByAuthorAndCreatedAtGreaterThanEqual(author: User, since: Instant): Boolean`; `PostService` constructor gains a `taipeiClock: TaipeiClock` parameter (after `userService`); wire shape `429 { code: 42901, message, resetAt: <epoch ms> }` + `Retry-After` header.

- [ ] **Step 1: Write the failing unit tests**

In `PostServiceTest.kt`:

(a) Add imports:

```kotlin
import com.example.demo.util.TaipeiClock
import com.example.demo.web.DailyPostLimitException
import org.junit.jupiter.api.assertThrows
import java.time.Clock
import java.time.Instant
import java.time.ZoneOffset
import kotlin.test.assertEquals
```

(b) Add an `eqv` matcher helper next to the existing `any()` helper (Mockito's `eq` returns null, which Kotlin non-null params reject):

```kotlin
    private fun <T> eqv(value: T): T = org.mockito.ArgumentMatchers.eq(value) ?: value
```

(c) Add a fixed clock and pass it to the service constructor (replace the existing `private val service = ...` block):

```kotlin
    // 2026-07-16T02:00:00Z == 10:00 Taipei: today started 2026-07-15T16:00:00Z,
    // resets 2026-07-16T16:00:00Z.
    private val taipeiClock = TaipeiClock(
        Clock.fixed(Instant.parse("2026-07-16T02:00:00Z"), ZoneOffset.UTC)
    )

    private val service = PostService(
        postRepository, songRepository, llmModelService, aiService, userService, taipeiClock,
        modelProvider = "google", modelName = "gemini-2.5-flash",
    )
```

(d) In `stubCommon()`, add the default no-post-today stub:

```kotlin
        `when`(
            postRepository.existsByAuthorAndCreatedAtGreaterThanEqual(any(), any())
        ).thenReturn(false)
```

(e) Add the two new tests:

```kotlin
    @Test
    @DisplayName("second post on the same Taipei day is rejected before any song or worker activity")
    fun `second post same day is rejected`() {
        stubCommon()
        `when`(postRepository.existsByAuthorAndCreatedAtGreaterThanEqual(any(), any())).thenReturn(true)

        val ex = assertThrows<DailyPostLimitException> { service.create(request, claims) }

        assertEquals(Instant.parse("2026-07-16T16:00:00Z"), ex.resetAt)
        verify(songRepository, never()).saveAndFlush(any<Song>())
        verify(postRepository, never()).save(any<Post>())
        verify(aiService, never()).processLyricsAsync(anyLong(), anyString())
    }

    @Test
    @DisplayName("the daily-limit check keys on the start of the current Taipei day")
    fun `limit check uses start of taipei day`() {
        stubCommon()
        `when`(
            songRepository.findByArtistNormAndSongNameNormAndAlbumNormAndLlmModelId(
                anyString(), anyString(), anyString(), anyLong(),
            )
        ).thenReturn(Optional.of(song(20L, PostStatus.COMPLETED)))

        service.create(request, claims)

        verify(postRepository).existsByAuthorAndCreatedAtGreaterThanEqual(
            any(), eqv(Instant.parse("2026-07-15T16:00:00Z")),
        )
    }
```

- [ ] **Step 2: Run to verify they fail**

Run: `./gradlew test --tests 'com.example.demo.service.PostServiceTest'`
Expected: COMPILATION FAILURE — `DailyPostLimitException` and the repository method don't exist, `PostService` has no `taipeiClock` parameter.

- [ ] **Step 3: Implement exception, repository method, service guard, and 429 handler**

Create `src/main/kotlin/com/example/demo/web/DailyPostLimitException.kt`:

```kotlin
package com.example.demo.web

import java.time.Instant

/**
 * The author has already posted in the current Asia/Taipei calendar day.
 * [resetAt] is the coming Taipei midnight, when posting reopens.
 */
class DailyPostLimitException(val resetAt: Instant) : RuntimeException("daily post limit reached")
```

In `PostRepository.kt`, add inside the interface:

```kotlin
    // "Has this author posted since the start of the current Taipei day?" — the
    // one-drop-per-day guard. `since` comes from TaipeiClock.startOfToday().
    fun existsByAuthorAndCreatedAtGreaterThanEqual(author: User, since: Instant): Boolean
```

and add the import:

```kotlin
import com.example.demo.model.User
```

In `PostService.kt`:

(a) Add imports:

```kotlin
import com.example.demo.util.TaipeiClock
import com.example.demo.web.DailyPostLimitException
```

(b) Add the constructor parameter after `userService`:

```kotlin
    private val userService: UserService,
    private val taipeiClock: TaipeiClock,
```

(c) In `create()`, right after `val author = userService.syncFromClaims(claims)`:

```kotlin
        // One drop per Taipei day. Reject before any song lookup/create or worker
        // scheduling — a blocked post must have zero side effects.
        if (postRepository.existsByAuthorAndCreatedAtGreaterThanEqual(author, taipeiClock.startOfToday())) {
            throw DailyPostLimitException(taipeiClock.nextReset())
        }
```

In `ApiExceptionHandler.kt`, add after `handleTypeMismatch` (before the "Everything else" section):

```kotlin
    // Daily post limit: client-actionable, so the body carries the machine-readable
    // reset time (epoch ms) alongside the display message.
    @ExceptionHandler(DailyPostLimitException::class)
    fun handleDailyLimit(e: DailyPostLimitException): ResponseEntity<Map<String, Any>> {
        log.info("Daily post limit hit; resets at {}", e.resetAt)
        // Retry-After is seconds, rounded up so a retry at exactly that time succeeds.
        val retryAfterSeconds = ((java.time.Duration.between(java.time.Instant.now(), e.resetAt)
            .toMillis() + 999) / 1000).coerceAtLeast(0)
        return ResponseEntity.status(ApiError.DAILY_POST_LIMIT.status)
            .header("Retry-After", retryAfterSeconds.toString())
            .body(
                mapOf<String, Any>(
                    "code" to ApiError.DAILY_POST_LIMIT.code,
                    "message" to ApiError.DAILY_POST_LIMIT.defaultMessage,
                    "resetAt" to e.resetAt.toEpochMilli(),
                )
            )
    }
```

- [ ] **Step 4: Run the unit tests to verify they pass**

Run: `./gradlew test --tests 'com.example.demo.service.PostServiceTest'`
Expected: PASS (5 tests).

- [ ] **Step 5: Fix the multi-post integration tests (same-author posts now collide) and add the 429 test**

The daily limit breaks existing tests that create several posts with the same default author. In `PostControllerIntegrationTest.kt`:

(a) In the pagination test, give every post a unique author (replace the `repeat(25)` block):

```kotlin
        // Submit 25 posts — each from a distinct author, since an author may only post
        // once per Taipei day.
        repeat(25) { i ->
            mockMvc.perform(
                postJson(
                    """{ "artist": "Artist $i", "songName": "Song $i", "rawLyrics": "x" }""",
                    token = TestJwt.token(sub = "google-pager-$i", email = "pager$i@example.com"),
                )
            ).andExpect(status().isOk)
        }
```

(b) Add the 429 test:

```kotlin
    @Test
    @DisplayName("second post by the same author on the same Taipei day returns 429 with code 42901 and resetAt")
    fun `second post same day returns 429`() {
        val token = TestJwt.token(sub = "google-limit", email = "limit@example.com", name = "Limit")

        mockMvc.perform(postJson(validBody, token = token)).andExpect(status().isOk)

        mockMvc.perform(postJson(validBody, token = token))
            .andExpect(status().isTooManyRequests)
            .andExpect(header().exists("Retry-After"))
            .andExpect(jsonPath("$.code").value(42901))
            .andExpect(jsonPath("$.message").value("You've already dropped today. Come back after reset."))
            .andExpect(jsonPath("$.resetAt").isNumber)
    }
```

(Note: `header()` comes from the existing wildcard import `MockMvcResultMatchers.*`. The other single-post tests — the PENDING-post test with the default author and the stamps-author test with `google-alice` — each create only one post per author and stay as they are.)

- [ ] **Step 6: Run the integration tests**

Run: `./gradlew test --tests 'com.example.demo.controller.PostControllerIntegrationTest'`
Expected: PASS (including the new 429 test).

- [ ] **Step 7: Run the full backend suite**

Run: `./gradlew test`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add -A src/main/kotlin src/test/kotlin
git commit -m "feat: enforce one post per user per Asia/Taipei day with 429 + resetAt"
```

---

## Task 4: Eligibility endpoint (backend)

**Files:**
- Create: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/dto/PostEligibilityDto.kt`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/service/PostService.kt`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/controller/PostController.kt`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/config/SecurityConfig.kt`
- Test: `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/service/PostServiceTest.kt`
- Test: `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/controller/PostControllerIntegrationTest.kt`

**Interfaces:**
- Consumes: `TaipeiClock` (Task 2), `existsByAuthorAndCreatedAtGreaterThanEqual` + guard state (Task 3).
- Produces: `GET /api/v1/posts/eligibility` (authenticated) → `200 { "canPostToday": Boolean, "nextResetAt": Long /* epoch ms */ }`; `PostService.eligibility(claims: ProviderClaims): PostEligibilityDto`.

- [ ] **Step 1: Write the failing tests**

(a) In `PostServiceTest.kt`, add:

```kotlin
    @Test
    @DisplayName("eligibility reports the next Taipei reset and flips canPostToday when a post exists today")
    fun `eligibility maps repository existence`() {
        stubCommon()
        `when`(postRepository.existsByAuthorAndCreatedAtGreaterThanEqual(any(), any())).thenReturn(true)

        val dto = service.eligibility(claims)

        assertEquals(false, dto.canPostToday)
        assertEquals(Instant.parse("2026-07-16T16:00:00Z").toEpochMilli(), dto.nextResetAt)
    }
```

(b) In `PostControllerIntegrationTest.kt`, add:

```kotlin
    @Test
    @DisplayName("GET eligibility without a token returns 401")
    fun `eligibility requires auth`() {
        mockMvc.perform(get("/api/v1/posts/eligibility"))
            .andExpect(status().isUnauthorized)
    }

    @Test
    @DisplayName("eligibility is true before the day's post and false after it")
    fun `eligibility flips after posting`() {
        val token = TestJwt.token(sub = "google-daily", email = "daily@example.com", name = "Daily")

        mockMvc.perform(get("/api/v1/posts/eligibility").header("Authorization", "Bearer $token"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.canPostToday").value(true))
            .andExpect(jsonPath("$.nextResetAt").isNumber)

        mockMvc.perform(postJson(validBody, token = token)).andExpect(status().isOk)

        mockMvc.perform(get("/api/v1/posts/eligibility").header("Authorization", "Bearer $token"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.canPostToday").value(false))
    }
```

- [ ] **Step 2: Run to verify they fail**

Run: `./gradlew test --tests 'com.example.demo.service.PostServiceTest' --tests 'com.example.demo.controller.PostControllerIntegrationTest'`
Expected: `PostServiceTest` COMPILATION FAILURE (no `eligibility` method). (Without the service change compiling, the integration tests can't run yet either.)

- [ ] **Step 3: Implement DTO, service method, controller endpoint, security rule**

Create `src/main/kotlin/com/example/demo/dto/PostEligibilityDto.kt`:

```kotlin
package com.example.demo.dto

/**
 * Whether the authenticated user may still post in the current Asia/Taipei calendar day.
 * [nextResetAt] is the coming Taipei midnight in epoch milliseconds (UTC) — the frontend
 * counts down to it with plain `nextResetAt - Date.now()`.
 */
data class PostEligibilityDto(
    val canPostToday: Boolean,
    val nextResetAt: Long,
)
```

In `PostService.kt`, add the import `com.example.demo.dto.PostEligibilityDto` and this method after `create()`:

```kotlin
    /** Same check as the create-guard, exposed read-only so the editor can lock proactively. */
    fun eligibility(claims: ProviderClaims): PostEligibilityDto {
        val author = userService.syncFromClaims(claims)
        val postedToday = postRepository.existsByAuthorAndCreatedAtGreaterThanEqual(
            author, taipeiClock.startOfToday(),
        )
        return PostEligibilityDto(
            canPostToday = !postedToday,
            nextResetAt = taipeiClock.nextReset().toEpochMilli(),
        )
    }
```

In `PostController.kt`, add this endpoint after `create()` (no new import needed — the file already has the wildcard `com.example.demo.dto.*`):

```kotlin
    // Exact-path mapping: Spring MVC prefers "/eligibility" over the "/{id}" variable
    // pattern, so this never shadows detail lookups.
    @GetMapping("/eligibility")
    fun eligibility(@AuthenticationPrincipal jwt: Jwt): PostEligibilityDto =
        postService.eligibility(jwt.toProviderClaims())
```

In `SecurityConfig.kt`, insert the authenticated matcher **before** the GET permitAll line (first match wins):

```kotlin
                it.requestMatchers(HttpMethod.GET, "/api/v1/posts/eligibility").authenticated()
                it.requestMatchers(HttpMethod.GET, "/api/v1/posts", "/api/v1/posts/**").permitAll()
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `./gradlew test --tests 'com.example.demo.service.PostServiceTest' --tests 'com.example.demo.controller.PostControllerIntegrationTest'`
Expected: PASS.

- [ ] **Step 5: Run the full backend suite**

Run: `./gradlew test`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add -A src/main/kotlin src/test/kotlin
git commit -m "feat: GET /api/v1/posts/eligibility — proactive daily-limit check"
```

---

## Task 5: `errorCodes` module (frontend)

**Files:**
- Create: `slangdump-poster/src/lib/errorCodes.ts`
- Test: `slangdump-poster/src/lib/__tests__/errorCodes.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: numeric constants `VALIDATION_ERROR`, `MALFORMED_BODY`, `MISSING_PARAMETER`, `INVALID_PARAMETER`, `UNAUTHORIZED`, `NOT_FOUND`, `DAILY_POST_LIMIT`, `INTERNAL_ERROR`; `interface ApiErrorBody { code: number; message: string; resetAt?: number }`; `parseApiError(res: Response): Promise<ApiErrorBody>`.

- [ ] **Step 0: Create the branch**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-poster
git checkout master && git checkout -b feat/daily-post-limit
```

- [ ] **Step 1: Write the failing tests**

Create `src/lib/__tests__/errorCodes.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { parseApiError, INTERNAL_ERROR, DAILY_POST_LIMIT } from '../errorCodes';

describe('parseApiError', () => {
  it('parses a conforming error body, keeping extra fields', async () => {
    const res = new Response(
      JSON.stringify({ code: DAILY_POST_LIMIT, message: 'limit', resetAt: 123 }),
      { status: 429 },
    );
    expect(await parseApiError(res)).toEqual({ code: DAILY_POST_LIMIT, message: 'limit', resetAt: 123 });
  });

  it('synthesizes INTERNAL_ERROR for a non-JSON body, preserving the text', async () => {
    const res = new Response('<html>boom</html>', { status: 502 });
    const err = await parseApiError(res);
    expect(err.code).toBe(INTERNAL_ERROR);
    expect(err.message).toBe('<html>boom</html>');
  });

  it('synthesizes a status-based message for an empty body', async () => {
    const res = new Response('', { status: 500 });
    expect(await parseApiError(res)).toEqual({ code: INTERNAL_ERROR, message: 'Request failed (500)' });
  });

  it('synthesizes INTERNAL_ERROR for JSON missing code or message', async () => {
    const res = new Response(JSON.stringify({ error: 'old shape' }), { status: 400 });
    const err = await parseApiError(res);
    expect(err.code).toBe(INTERNAL_ERROR);
  });
});
```

- [ ] **Step 2: Run to verify they fail**

Run: `npx vitest run src/lib/__tests__/errorCodes.test.ts`
Expected: FAIL — cannot resolve `../errorCodes`.

- [ ] **Step 3: Implement the module**

Create `src/lib/errorCodes.ts`:

```ts
// Numeric API error codes, mirroring the backend catalog (ApiError.kt in
// slangdump-pick-n-roll). Scheme: HTTP status × 100 + sequence — so
// Math.floor(code / 100) recovers the HTTP status. Branch on `code`, never
// on `message` (message is display text only).
export const VALIDATION_ERROR = 40001;
export const MALFORMED_BODY = 40002;
export const MISSING_PARAMETER = 40003;
export const INVALID_PARAMETER = 40004;
export const UNAUTHORIZED = 40100;
export const NOT_FOUND = 40400;
export const DAILY_POST_LIMIT = 42901;
export const INTERNAL_ERROR = 50000;

export interface ApiErrorBody {
  code: number;
  message: string;
  /** Epoch ms (UTC) when the daily post limit resets. Present only with DAILY_POST_LIMIT. */
  resetAt?: number;
}

// Parse an error Response into the unified shape. Non-JSON, empty, or
// non-conforming bodies are synthesized into INTERNAL_ERROR so callers can
// always branch on `code`.
export async function parseApiError(res: Response): Promise<ApiErrorBody> {
  const text = await res.text();
  try {
    const body = JSON.parse(text);
    if (typeof body?.code === 'number' && typeof body?.message === 'string') {
      return body as ApiErrorBody;
    }
  } catch {
    // fall through to the synthesized error
  }
  return { code: INTERNAL_ERROR, message: text || `Request failed (${res.status})` };
}
```

- [ ] **Step 4: Run to verify they pass**

Run: `npx vitest run src/lib/__tests__/errorCodes.test.ts`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add src/lib/errorCodes.ts src/lib/__tests__/errorCodes.test.ts
git commit -m "feat: numeric API error codes + parseApiError"
```

---

## Task 6: Proxy routes emit the unified contract (frontend)

**Files:**
- Modify: `slangdump-poster/src/app/api/posts/route.ts`
- Modify: `slangdump-poster/src/app/api/posts/[id]/route.ts`
- Test: `slangdump-poster/src/app/api/posts/__tests__/route.test.ts`
- Test: `slangdump-poster/src/app/api/posts/[id]/__tests__/route.test.ts` (new)

**Interfaces:**
- Consumes: `UNAUTHORIZED`, `INTERNAL_ERROR` from `@/lib/errorCodes` (Task 5).
- Produces: proxy-originated errors in the unified shape — 401 → `{ code: 40100, message: 'Sign in to post' }`, catch-all → `{ code: 50000, message }`; `[id]` route passes backend bodies (success and error) through verbatim.

- [ ] **Step 1: Extend the tests (failing)**

(a) In `src/app/api/posts/__tests__/route.test.ts`, extend the existing 401 test with a body assertion:

```ts
  it('returns 401 without a session and never calls the backend', async () => {
    auth.mockResolvedValue(null);
    const fetchMock = vi.fn();
    vi.stubGlobal('fetch', fetchMock);

    const res = await POST(makeReq());

    expect(res.status).toBe(401);
    expect(await res.json()).toEqual({ code: 40100, message: 'Sign in to post' });
    expect(fetchMock).not.toHaveBeenCalled();
    expect(mintBffToken).not.toHaveBeenCalled();
  });
```

(b) Create `src/app/api/posts/[id]/__tests__/route.test.ts`:

```ts
import { describe, it, expect, vi, afterEach } from 'vitest';
import { GET } from '../route';

const call = () =>
  GET(new Request('http://localhost/api/posts/9'), { params: Promise.resolve({ id: '9' }) });

describe('GET /api/posts/[id] proxy', () => {
  afterEach(() => vi.unstubAllGlobals());

  it('passes backend error bodies through unchanged (unified contract)', async () => {
    vi.stubGlobal('fetch', vi.fn(async () =>
      new Response(JSON.stringify({ code: 40400, message: 'Not Found' }), {
        status: 404,
        headers: { 'Content-Type': 'application/json' },
      }),
    ));

    const res = await call();

    expect(res.status).toBe(404);
    expect(await res.json()).toEqual({ code: 40400, message: 'Not Found' });
  });

  it('passes success bodies through unchanged', async () => {
    vi.stubGlobal('fetch', vi.fn(async () =>
      new Response(JSON.stringify({ id: 9, status: 'COMPLETED' }), {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
      }),
    ));

    const res = await call();

    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ id: 9, status: 'COMPLETED' });
  });

  it('returns code 50000 when the backend is unreachable', async () => {
    vi.stubGlobal('fetch', vi.fn(async () => { throw new Error('ECONNREFUSED'); }));

    const res = await call();

    expect(res.status).toBe(500);
    expect((await res.json()).code).toBe(50000);
  });
});
```

- [ ] **Step 2: Run to verify the new assertions fail**

Run: `npx vitest run src/app/api/posts`
Expected: the 401-body assertion FAILS (body is `{ error: 'Sign in to post' }`) and the `[id]` pass-through tests FAIL (route synthesizes `{ error: 'Post not found' }`).

- [ ] **Step 3: Update the routes**

(a) In `src/app/api/posts/route.ts`: add the import

```ts
import { UNAUTHORIZED, INTERNAL_ERROR } from '@/lib/errorCodes';
```

replace the 401 body in `POST`:

```ts
      return NextResponse.json({ code: UNAUTHORIZED, message: 'Sign in to post' }, { status: 401 });
```

and replace the catch-block bodies in **both** `POST` and `GET`:

```ts
    return NextResponse.json({ code: INTERNAL_ERROR, message: msg }, { status: 500 });
```

(b) Replace `src/app/api/posts/[id]/route.ts` entirely with:

```ts
import { NextResponse } from 'next/server';
import { INTERNAL_ERROR } from '@/lib/errorCodes';

export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params;
    const backendUrl = process.env.SPRING_BACKEND_URL || 'http://localhost:8080';
    const response = await fetch(`${backendUrl}/api/v1/posts/${id}`, {
      cache: 'no-store',
    });

    // Success and error bodies both pass through verbatim — the backend already
    // speaks the unified { code, message } contract, so synthesizing a body here
    // would only hide the real code from the client.
    const text = await response.text();
    return new NextResponse(text, {
      status: response.status,
      headers: { 'Content-Type': response.headers.get('Content-Type') ?? 'application/json' },
    });
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : 'Unknown error';
    return NextResponse.json({ code: INTERNAL_ERROR, message: msg }, { status: 500 });
  }
}
```

- [ ] **Step 4: Run to verify they pass**

Run: `npx vitest run src/app/api/posts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/app/api/posts
git commit -m "feat: proxy routes speak the unified { code, message } error contract"
```

---

## Task 7: Eligibility proxy route (frontend)

**Files:**
- Create: `slangdump-poster/src/app/api/posts/eligibility/route.ts`
- Test: `slangdump-poster/src/app/api/posts/eligibility/__tests__/route.test.ts`

**Interfaces:**
- Consumes: `auth()` session (fields `user`, `provider`, `providerSub`), `mintBffToken(claims)`, `UNAUTHORIZED` / `INTERNAL_ERROR` (Task 5).
- Produces: `GET /api/posts/eligibility` → forwards to `GET {BACKEND}/api/v1/posts/eligibility` with a minted bearer token; passes the backend JSON through; own errors in unified shape.

- [ ] **Step 1: Write the failing tests**

Create `src/app/api/posts/eligibility/__tests__/route.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

const auth = vi.fn();
vi.mock('@/auth', () => ({ auth: () => auth() }));
vi.mock('@/lib/bffToken', () => ({ mintBffToken: vi.fn(async () => 'signed-token') }));

import { GET } from '../route';
import { mintBffToken } from '@/lib/bffToken';

describe('GET /api/posts/eligibility proxy', () => {
  beforeEach(() => {
    auth.mockReset();
    vi.mocked(mintBffToken).mockClear();
    vi.unstubAllGlobals();
  });

  it('returns 401 with code 40100 without a session and never calls the backend', async () => {
    auth.mockResolvedValue(null);
    const fetchMock = vi.fn();
    vi.stubGlobal('fetch', fetchMock);

    const res = await GET();

    expect(res.status).toBe(401);
    expect(await res.json()).toEqual({ code: 40100, message: 'Sign in to post' });
    expect(fetchMock).not.toHaveBeenCalled();
    expect(mintBffToken).not.toHaveBeenCalled();
  });

  it('forwards a bearer token and passes the backend body through', async () => {
    auth.mockResolvedValue({
      user: { email: 'a@x.com', name: 'A', image: null },
      provider: 'google',
      providerSub: 'sub1',
    });
    const fetchMock = vi.fn(async () =>
      new Response(JSON.stringify({ canPostToday: false, nextResetAt: 123 }), {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
      }),
    );
    vi.stubGlobal('fetch', fetchMock);

    const res = await GET();

    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ canPostToday: false, nextResetAt: 123 });
    const opts = fetchMock.mock.calls[0][1] as RequestInit;
    expect((opts.headers as Record<string, string>).Authorization).toBe('Bearer signed-token');
  });
});
```

- [ ] **Step 2: Run to verify they fail**

Run: `npx vitest run src/app/api/posts/eligibility`
Expected: FAIL — cannot resolve `../route`.

- [ ] **Step 3: Implement the route**

Create `src/app/api/posts/eligibility/route.ts`:

```ts
import { NextResponse } from 'next/server';
import { auth } from '@/auth';
import { mintBffToken } from '@/lib/bffToken';
import { UNAUTHORIZED, INTERNAL_ERROR } from '@/lib/errorCodes';

const BACKEND = process.env.SPRING_BACKEND_URL ?? 'http://localhost:8080';

// Daily-limit eligibility for the signed-in user. Mirrors the POST /api/posts auth
// flow: identity comes from the server session, the backend re-verifies the token.
export async function GET() {
  try {
    const session = await auth();
    if (!session?.user || !session.providerSub) {
      return NextResponse.json({ code: UNAUTHORIZED, message: 'Sign in to post' }, { status: 401 });
    }

    const token = await mintBffToken({
      provider: session.provider ?? 'google',
      sub: session.providerSub,
      email: session.user.email,
      name: session.user.name,
      picture: session.user.image,
    });

    const response = await fetch(`${BACKEND}/api/v1/posts/eligibility`, {
      headers: { Authorization: `Bearer ${token}` },
      cache: 'no-store',
    });

    const text = await response.text();
    return new NextResponse(text, {
      status: response.status,
      headers: { 'Content-Type': response.headers.get('Content-Type') ?? 'application/json' },
    });
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : 'Unknown error';
    return NextResponse.json({ code: INTERNAL_ERROR, message: msg }, { status: 500 });
  }
}
```

- [ ] **Step 4: Run to verify they pass**

Run: `npx vitest run src/app/api/posts/eligibility`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/app/api/posts/eligibility
git commit -m "feat: eligibility proxy route with BFF token"
```

---

## Task 8: `dailyReset` countdown helpers (frontend)

**Files:**
- Create: `slangdump-poster/src/lib/dailyReset.ts`
- Test: `slangdump-poster/src/lib/__tests__/dailyReset.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `nextTaipeiResetMs(nowMs?: number): number` (epoch ms of the next 00:00 Asia/Taipei) and `formatCountdown(msRemaining: number): string` (`HH:MM:SS`, clamped at `00:00:00`).

- [ ] **Step 1: Write the failing tests**

Create `src/lib/__tests__/dailyReset.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { nextTaipeiResetMs, formatCountdown } from '../dailyReset';

describe('nextTaipeiResetMs', () => {
  it('returns the coming Taipei midnight (16:00 UTC)', () => {
    // 2026-07-16T02:00:00Z == 10:00 Taipei → resets 2026-07-16T16:00:00Z
    expect(nextTaipeiResetMs(Date.parse('2026-07-16T02:00:00Z')))
      .toBe(Date.parse('2026-07-16T16:00:00Z'));
  });

  it('rolls a full day when called exactly at a reset instant', () => {
    expect(nextTaipeiResetMs(Date.parse('2026-07-16T16:00:00Z')))
      .toBe(Date.parse('2026-07-17T16:00:00Z'));
  });

  it('is seconds away just before Taipei midnight', () => {
    expect(nextTaipeiResetMs(Date.parse('2026-07-16T15:59:59Z')))
      .toBe(Date.parse('2026-07-16T16:00:00Z'));
  });
});

describe('formatCountdown', () => {
  it('formats HH:MM:SS with zero padding', () => {
    expect(formatCountdown(((14 * 3600) + (22 * 60) + 5) * 1000)).toBe('14:22:05');
    expect(formatCountdown(61_000)).toBe('00:01:01');
  });

  it('clamps negative durations to 00:00:00', () => {
    expect(formatCountdown(-5000)).toBe('00:00:00');
  });
});
```

- [ ] **Step 2: Run to verify they fail**

Run: `npx vitest run src/lib/__tests__/dailyReset.test.ts`
Expected: FAIL — cannot resolve `../dailyReset`.

- [ ] **Step 3: Implement the helpers**

Create `src/lib/dailyReset.ts`:

```ts
// Client-side fallback for the daily-drop reset countdown. Taipei is UTC+8
// year-round (no DST), so the next Taipei midnight is pure fixed-offset math —
// no timezone database needed. The authoritative value is the backend's
// nextResetAt; this only covers the gap before eligibility loads.
const TAIPEI_OFFSET_MS = 8 * 60 * 60 * 1000;
const DAY_MS = 24 * 60 * 60 * 1000;

/** Epoch ms (UTC) of the next 00:00 in Asia/Taipei strictly after `nowMs`. */
export function nextTaipeiResetMs(nowMs: number = Date.now()): number {
  return (Math.floor((nowMs + TAIPEI_OFFSET_MS) / DAY_MS) + 1) * DAY_MS - TAIPEI_OFFSET_MS;
}

/** Format a remaining duration as HH:MM:SS, clamped at zero. */
export function formatCountdown(msRemaining: number): string {
  const total = Math.max(0, Math.floor(msRemaining / 1000));
  const h = Math.floor(total / 3600);
  const m = Math.floor((total % 3600) / 60);
  const s = total % 60;
  return [h, m, s].map((n) => String(n).padStart(2, '0')).join(':');
}
```

- [ ] **Step 4: Run to verify they pass**

Run: `npx vitest run src/lib/__tests__/dailyReset.test.ts`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add src/lib/dailyReset.ts src/lib/__tests__/dailyReset.test.ts
git commit -m "feat: Taipei reset countdown helpers"
```

---

## Task 9: `DailyDropEditor` — live countdown, lock, 429 handling (frontend)

**Files:**
- Modify: `slangdump-poster/src/components/DailyDropEditor.tsx`
- Test: `slangdump-poster/src/components/__tests__/DailyDropEditor.test.tsx` (new)

**Interfaces:**
- Consumes: `GET /api/posts/eligibility` → `{ canPostToday: boolean; nextResetAt: number }` (Task 7); `parseApiError`, `DAILY_POST_LIMIT`, `UNAUTHORIZED` (Task 5); `nextTaipeiResetMs`, `formatCountdown` (Task 8).
- Produces: the editor UI. No other component consumes its internals.

- [ ] **Step 1: Write the failing tests**

Create `src/components/__tests__/DailyDropEditor.test.tsx`:

```tsx
import { describe, it, expect, vi, afterEach } from 'vitest';
import { render, screen, fireEvent, act } from '@testing-library/react';
import { DailyDropEditor } from '../DailyDropEditor';

// Pinned only in the fake-timer test: 02:00Z == 10:00 Taipei → 14:00:00 to reset.
const NOW = Date.parse('2026-07-16T02:00:00Z');

const jsonResponse = (body: unknown, status = 200) =>
  new Response(JSON.stringify(body), {
    status,
    headers: { 'Content-Type': 'application/json' },
  });

// Always in the future relative to real time, so real-timer tests never see the
// countdown expire (which would trigger an eligibility refetch and unlock).
const resetAtFuture = () => Date.now() + 14 * 3600 * 1000;

const eligibility = (canPostToday: boolean) =>
  jsonResponse({ canPostToday, nextResetAt: resetAtFuture() });

afterEach(() => {
  vi.unstubAllGlobals();
  vi.useRealTimers();
});

describe('DailyDropEditor countdown', () => {
  it('replaces the hardcoded timer with a live countdown that ticks', async () => {
    vi.useFakeTimers();
    vi.setSystemTime(NOW);
    // Eligibility never resolves: the countdown must run from the client-side fallback.
    vi.stubGlobal('fetch', vi.fn(() => new Promise<Response>(() => {})));

    render(<DailyDropEditor onPostCreated={vi.fn()} />);

    expect(screen.getByText('Reset in 14:00:00')).toBeInTheDocument();
    await act(async () => { await vi.advanceTimersByTimeAsync(1000); });
    expect(screen.getByText('Reset in 13:59:59')).toBeInTheDocument();
  });
});

describe('DailyDropEditor daily lock', () => {
  it('locks the editor when eligibility says the user already posted today', async () => {
    vi.stubGlobal('fetch', vi.fn(async () => eligibility(false)));

    render(<DailyDropEditor onPostCreated={vi.fn()} />);

    expect(await screen.findByText(/you've dropped today/i)).toBeInTheDocument();
    expect(screen.queryByRole('button', { name: /drop it/i })).not.toBeInTheDocument();
  });

  it('keeps the form when eligibility allows posting', async () => {
    vi.stubGlobal('fetch', vi.fn(async () => eligibility(true)));

    render(<DailyDropEditor onPostCreated={vi.fn()} />);

    expect(await screen.findByRole('button', { name: /drop it/i })).toBeInTheDocument();
    expect(screen.queryByText(/you've dropped today/i)).not.toBeInTheDocument();
  });

  const fillAndSubmit = () => {
    fireEvent.change(screen.getByPlaceholderText('SONG TITLE...'), { target: { value: 'One Dance' } });
    fireEvent.change(screen.getByPlaceholderText('ARTIST NAME...'), { target: { value: 'Drake' } });
    fireEvent.change(screen.getByPlaceholderText('ENTER RAW LYRICS HERE...'), { target: { value: 'baby I like your style' } });
    fireEvent.click(screen.getByRole('button', { name: /drop it/i }));
  };

  it('locks and adopts resetAt when the POST returns the daily-limit error', async () => {
    vi.stubGlobal('fetch', vi.fn(async (url: RequestInfo | URL, init?: RequestInit) => {
      if (String(url).endsWith('/api/posts/eligibility')) return eligibility(true);
      if (init?.method === 'POST') {
        return jsonResponse(
          { code: 42901, message: "You've already dropped today. Come back after reset.", resetAt: resetAtFuture() },
          429,
        );
      }
      throw new Error(`unexpected fetch ${url}`);
    }));

    render(<DailyDropEditor onPostCreated={vi.fn()} />);
    await screen.findByRole('button', { name: /drop it/i });

    fillAndSubmit();

    expect(await screen.findByText(/you've dropped today/i)).toBeInTheDocument();
  });

  it('locks after a successful drop completes and still reports the post', async () => {
    const onPostCreated = vi.fn();
    vi.stubGlobal('fetch', vi.fn(async (url: RequestInfo | URL, init?: RequestInit) => {
      if (String(url).endsWith('/api/posts/eligibility')) return eligibility(true);
      if (init?.method === 'POST') return jsonResponse({ id: 7, status: 'PENDING' });
      if (String(url).endsWith('/api/posts/7')) return jsonResponse({ id: 7, status: 'COMPLETED', lines: [] });
      throw new Error(`unexpected fetch ${url}`);
    }));

    render(<DailyDropEditor onPostCreated={onPostCreated} />);
    await screen.findByRole('button', { name: /drop it/i });

    fillAndSubmit();

    // Status polling runs on a real 1.5 s interval.
    expect(await screen.findByText(/you've dropped today/i, undefined, { timeout: 4000 })).toBeInTheDocument();
    expect(onPostCreated).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 2: Run to verify they fail**

Run: `npx vitest run src/components/__tests__/DailyDropEditor.test.tsx`
Expected: FAIL — no countdown text, no locked panel (the hardcoded `Reset in 14:22:05` is still rendered).

- [ ] **Step 3: Implement the editor changes**

All edits in `src/components/DailyDropEditor.tsx`:

(a) Replace the react import and add the new imports:

```tsx
import React, { useCallback, useEffect, useState } from 'react';
import { Loader2, AlertCircle } from 'lucide-react';
import { parseApiError, DAILY_POST_LIMIT, UNAUTHORIZED } from '@/lib/errorCodes';
import { nextTaipeiResetMs, formatCountdown } from '@/lib/dailyReset';
import type { PostDetailDto } from '@/types/api';
```

(b) After the existing "Execution states" block (`errorMessage` state), add:

```tsx
  // Daily-limit state. Defaults are optimistic (can post, fallback countdown);
  // the backend's 429 is the authoritative gate either way.
  const [canPostToday, setCanPostToday] = useState(true);
  const [nextResetAt, setNextResetAt] = useState<number | null>(null);
  const [countdown, setCountdown] = useState(() => formatCountdown(nextTaipeiResetMs() - Date.now()));

  const fetchEligibility = useCallback(async () => {
    try {
      const res = await fetch('/api/posts/eligibility');
      if (!res.ok) return; // keep optimistic defaults — POST is still gated server-side
      const data = await res.json() as { canPostToday: boolean; nextResetAt: number };
      setCanPostToday(data.canPostToday);
      setNextResetAt(data.nextResetAt);
    } catch {
      // network error: keep defaults
    }
  }, []);

  useEffect(() => { fetchEligibility(); }, [fetchEligibility]);

  // Live HH:MM:SS countdown to the next Taipei reset. When it hits zero the day
  // rolled over: drop the stale target and re-check eligibility.
  useEffect(() => {
    const tick = () => {
      const target = nextResetAt ?? nextTaipeiResetMs();
      const remaining = target - Date.now();
      setCountdown(formatCountdown(remaining));
      if (remaining <= 0) {
        setNextResetAt(null);
        fetchEligibility();
      }
    };
    tick();
    const id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, [nextResetAt, fetchEligibility]);
```

(c) In `handleDropIt`, replace this block:

```tsx
      if (res.status === 401) {
        throw new Error('YOUR SESSION EXPIRED. PLEASE SIGN IN AGAIN.');
      }
      if (!res.ok) {
        throw new Error(await res.text());
      }

      const postData = await res.json();
      const postId = postData.id;
```

with:

```tsx
      if (!res.ok) {
        const apiError = await parseApiError(res);
        if (apiError.code === DAILY_POST_LIMIT) {
          // Backend says today's drop is already used — mirror it and show the lock.
          if (apiError.resetAt) setNextResetAt(apiError.resetAt);
          setCanPostToday(false);
          setIsSubmitting(false);
          return;
        }
        if (apiError.code === UNAUTHORIZED) {
          throw new Error('YOUR SESSION EXPIRED. PLEASE SIGN IN AGAIN.');
        }
        throw new Error(apiError.message);
      }

      const postData = await res.json();
      // The post row exists from here on — today's drop is used even if the
      // translation later fails.
      setCanPostToday(false);
      const postId = postData.id;
```

(d) Replace the hardcoded header timer:

```tsx
            <div className="flex items-center gap-2 text-orange-600">
              <span className="material-symbols-outlined text-sm">timer</span>
              <span className="font-inter font-black italic text-sm uppercase">Reset in 14:22:05</span>
            </div>
```

with:

```tsx
            <div className="flex items-center gap-2 text-orange-600">
              <span className="material-symbols-outlined text-sm">timer</span>
              <span className="font-inter font-black italic text-sm uppercase">Reset in {countdown}</span>
            </div>
```

(e) Add the locked branch to the main JSX ternary. Change:

```tsx
      {isSubmitting ? (
        /* Loading Skeleton with Rich Aesthetics */
```

(no change to the skeleton itself), and change the `) : (` that follows the skeleton's closing `</div>` to:

```tsx
      ) : !canPostToday ? (
        /* Locked: today's drop is used — countdown to the next Taipei midnight */
        <div className="py-16 flex flex-col items-center justify-center space-y-6 border-2 border-dashed border-zinc-800 bg-zinc-950">
          <div className="w-24 h-24 bg-zinc-900 flex items-center justify-center border-2 border-orange-600/50">
            <span className="material-symbols-outlined text-4xl text-orange-500">lock_clock</span>
          </div>
          <div className="text-center space-y-3">
            <h3 className="text-white font-black italic text-2xl tracking-tighter uppercase">
              YOU&apos;VE DROPPED TODAY
            </h3>
            <p className="text-zinc-500 text-[10px] font-black tracking-widest uppercase">
              ONE DROP PER DAY. COME BACK AFTER THE RESET.
            </p>
            <div className="flex items-center justify-center gap-2 text-orange-600">
              <span className="material-symbols-outlined text-sm">timer</span>
              <span className="font-inter font-black italic text-lg uppercase">Reset in {countdown}</span>
            </div>
          </div>
        </div>
      ) : (
```

(the form branch and everything inside it stay unchanged).

- [ ] **Step 4: Run to verify they pass**

Run: `npx vitest run src/components/__tests__/DailyDropEditor.test.tsx`
Expected: PASS (5 tests; the success-flow test takes ~2 s from the real polling interval).

- [ ] **Step 5: Commit**

```bash
git add src/components/DailyDropEditor.tsx src/components/__tests__/DailyDropEditor.test.tsx
git commit -m "feat: daily-drop lock with live Taipei reset countdown"
```

---

## Task 10: Migrate remaining error consumers (frontend)

**Files:**
- Modify: `slangdump-poster/src/hooks/useAppState.ts`
- Modify: `slangdump-poster/src/components/FeedItem.tsx`

**Interfaces:**
- Consumes: `parseApiError` (Task 5).
- Produces: nothing new — all user-visible error text now comes from `ApiErrorBody.message`.

- [ ] **Step 1: Update `useAppState.ts`**

Add the import:

```ts
import { parseApiError } from '@/lib/errorCodes';
```

Replace the initial-load effect body:

```ts
  useEffect(() => {
    fetch('/api/posts?limit=10')
      .then(async res => {
        if (!res.ok) throw new Error((await parseApiError(res)).message);
        const page = await res.json() as CursorPage<PostSummaryDto>;
        if (!Array.isArray(page.items)) throw new Error(`Failed to load posts (${res.status})`);
        setPosts(page.items);
        setNextCursor(page.nextCursor ?? null);
        setError(null);
      })
      .catch(err => {
        console.error('failed to load posts', err);
        setError(err instanceof Error ? err.message : 'Failed to load posts');
      });
  }, []);
```

And the corresponding block inside `loadMore` (the `try` body up to `setError(null);`):

```ts
      const res = await fetch(`/api/posts?cursor=${encodeURIComponent(nextCursor)}&limit=10`);
      if (!res.ok) throw new Error((await parseApiError(res)).message);
      const page = await res.json() as CursorPage<PostSummaryDto>;
      if (!Array.isArray(page.items)) throw new Error(`Failed to load more posts (${res.status})`);
      setPosts(prev => [...prev, ...page.items]);
      setNextCursor(page.nextCursor ?? null);
      setError(null);
```

- [ ] **Step 2: Update `FeedItem.tsx`**

Add the import:

```ts
import { parseApiError } from '@/lib/errorCodes';
```

In `handleTranslate`, replace:

```ts
      const res = await fetch(`/api/posts/${post.id}`);
      if (!res.ok) throw new Error('Failed to load translation');
```

with:

```ts
      const res = await fetch(`/api/posts/${post.id}`);
      if (!res.ok) throw new Error((await parseApiError(res)).message);
```

- [ ] **Step 3: Run the full frontend suite**

Run: `npm test`
Expected: PASS — all suites, including the ones from Tasks 5–9.

- [ ] **Step 4: Build check**

Run: `npx tsc --noEmit`
Expected: no type errors.

- [ ] **Step 5: Commit**

```bash
git add src/hooks/useAppState.ts src/components/FeedItem.tsx
git commit -m "refactor: feed error consumers read unified error messages"
```

---

## Final verification (both repos)

- [ ] Backend: `cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll && ./gradlew test` → PASS (Docker running).
- [ ] Frontend: `cd /Users/jimhsu/Documents/slangdump/slangdump-poster && npm test && npx tsc --noEmit` → PASS.
- [ ] Manual smoke (optional but recommended): run backend + frontend locally, sign in, post once → editor locks with live countdown; reload → still locked (eligibility); second POST via curl with the same token → 429 `{ code: 42901, resetAt }`.
