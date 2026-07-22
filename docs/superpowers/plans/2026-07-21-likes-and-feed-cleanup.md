# Likes API and Feed Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship real post likes, remove LLM identity from the API, clear the hard-coded placeholders out of the feed card, and make the left nav a mobile overlay drawer.

**Architecture:** Likes are a new `post_likes` table with `UNIQUE (post_id, user_id)` — counts are always derived, never denormalized. A new `PostLikeService` answers counts and the viewer's liked set with two batched queries per page, leaving the subtle cursor-paging queries in `PostRepository` untouched. Write endpoints are auth-gated with explicit Spring Security rules, because the existing rules would otherwise let them fall through to `permitAll()`.

**Tech Stack:** Spring Boot 4 / Kotlin / Java 21 / JPA / Flyway / Testcontainers (`slangdump-pick-n-roll`); Next.js 16 / React 19 / next-auth v5 / Tailwind 4 / Vitest + Testing Library (`slangdump-poster`).

**Spec:** `docs/superpowers/specs/2026-07-21-likes-and-feed-cleanup-design.md`

## Global Constraints

- Multi-repo. `slangdump-pick-n-roll` and `slangdump-poster` are **separate git repos**. Commit inside the repo you changed; never from the `slangdump` root.
- Flyway migrations never run at application startup. Adding `V5__post_likes.sql` is a file-only change; deployment runs it as a distinct release step against the **direct** (non-pooled) Neon connection.
- The `LlmModel` entity, `llm_models` table, and `songs.llm_model_id` column all stay. Only the API surface loses LLM fields. No migration for that change.
- The polling contract is untouched: `GET /api/v1/posts/{id}` keeps returning 202 + `Retry-After` while `PENDING` and cacheable 200 when terminal, and the BFF keeps forwarding both headers.
- Comments are **not** implemented. The comment button ships disabled.
- Exact AI-notice copy, used verbatim: `AI-generated translations may contain errors, omissions, or debatable readings. Verify anything you rely on.`
- Exact comment-tooltip copy, used verbatim: `Comments arrive in the next phase.`
- Out of scope, do not touch: `timeLoc` and `tagsList` placeholders in `FeedItem`, `LeftBar.tsx`, `TopBar.tsx`, `TopNavBar` bell/settings buttons.
- Backend tests need Docker running (Testcontainers).

## File Structure

**`slangdump-pick-n-roll`**

| File | Responsibility |
|---|---|
| `src/main/resources/db/migration/V5__post_likes.sql` | create (new) — the `post_likes` table |
| `src/main/kotlin/com/example/demo/model/PostLike.kt` | create (new) — entity |
| `src/main/kotlin/com/example/demo/repository/PostLikeRepository.kt` | create (new) — batched + scalar like queries |
| `src/main/kotlin/com/example/demo/dto/LikeStateDto.kt` | create (new) — like/unlike response |
| `src/main/kotlin/com/example/demo/service/PostLikeService.kt` | create (new) — owns all like reads and writes |
| `src/main/kotlin/com/example/demo/controller/PostController.kt` | modify — like endpoints, viewer resolution, DTO wiring, LLM removal |
| `src/main/kotlin/com/example/demo/config/SecurityConfig.kt` | modify — auth rules for the like endpoints |
| `src/main/kotlin/com/example/demo/dto/PostSummaryDto.kt` | modify — `likeCount`, `likedByMe` |
| `src/main/kotlin/com/example/demo/dto/PostDetailDto.kt` | modify — `likeCount`, `likedByMe`; drop LLM fields |

**`slangdump-poster`**

| File | Responsibility |
|---|---|
| `src/types/api.ts` | modify — mirror the DTO changes |
| `src/app/api/posts/[id]/like/route.ts` | create (new) — BFF POST/DELETE like proxy |
| `src/app/api/posts/route.ts` | modify — optional token forwarding on the list GET |
| `src/components/PostLikeButton.tsx` | create (new) — owns optimistic like state and network |
| `src/components/FeedItem.tsx` | modify — placeholders out, AI notice in, footer rebuilt |
| `src/components/SideNavBar.tsx` | modify — drawer classes, `onClose`, POST LYRIC button removed |
| `src/app/page.tsx` | modify — FAB removed, backdrop added, responsive margin |
| `src/hooks/useAppState.ts` | modify — breakpoint-aware sidebar default |

`PostLikeButton` is a separate component on purpose: `FeedItem.tsx` is already 393 lines, and isolating the network + optimistic state makes it testable without rendering a whole feed card.

---

## Task 1: `post_likes` table, entity, and repository

**Repo:** `slangdump-pick-n-roll`

**Files:**
- Create: `src/main/resources/db/migration/V5__post_likes.sql`
- Create: `src/main/kotlin/com/example/demo/model/PostLike.kt`
- Create: `src/main/kotlin/com/example/demo/repository/PostLikeRepository.kt`
- Test: `src/test/kotlin/com/example/demo/repository/PostLikeRepositoryIntegrationTest.kt`

**Interfaces:**
- Consumes: existing `Post`, `User` entities; the Flyway-seeded `llm_models` row (`google` / `gemini-2.5-flash`) used to build test songs.
- Produces:
  - `class PostLike(id: Long?, post: Post, user: User, createdAt: Instant)`
  - `PostLikeRepository.countsByPostIds(postIds: Collection<Long>): List<Array<Any>>` — rows of `[postId, count]`, posts with zero likes absent
  - `PostLikeRepository.likedPostIds(userId: Long, postIds: Collection<Long>): List<Long>`
  - `PostLikeRepository.countByPostId(postId: Long): Long`
  - `PostLikeRepository.existsByPostIdAndUserId(postId: Long, userId: Long): Boolean`
  - `PostLikeRepository.deleteByPostIdAndUserId(postId: Long, userId: Long): Long`

- [ ] **Step 1: Write the migration**

Create `src/main/resources/db/migration/V5__post_likes.sql`:

```sql
-- One like per (post, user). The unique constraint IS the dedup rule and also serves
-- count-by-post lookups, so no separate index on post_id is needed. There is no
-- denormalized counter column: counts are always derived, so they cannot drift.
CREATE TABLE post_likes (
  id         BIGSERIAL PRIMARY KEY,
  post_id    BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id    BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  UNIQUE (post_id, user_id)
);

-- Supports "which of these posts has this user liked", the viewer-side lookup.
CREATE INDEX idx_post_likes_user ON post_likes(user_id);
```

- [ ] **Step 2: Write the entity**

Create `src/main/kotlin/com/example/demo/model/PostLike.kt`:

```kotlin
package com.example.demo.model

import jakarta.persistence.*
import java.time.Instant

/**
 * One user's like of one [Post]. Uniqueness of (post, user) is enforced by the database,
 * not by application checks — a concurrent double-insert surfaces as a constraint
 * violation that callers treat as success, since the user's intent is already satisfied.
 */
@Entity
@Table(name = "post_likes")
class PostLike(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    var post: Post,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    var user: User,

    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now(),
)
```

- [ ] **Step 3: Write the repository**

Create `src/main/kotlin/com/example/demo/repository/PostLikeRepository.kt`:

```kotlin
package com.example.demo.repository

import com.example.demo.model.PostLike
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import org.springframework.transaction.annotation.Transactional

interface PostLikeRepository : JpaRepository<PostLike, Long> {

    // Counts for a whole page in ONE query, regardless of page size. Reading `l.post.id`
    // uses the FK column directly, so this never joins the posts table. Posts with no
    // likes are simply absent from the result — callers default them to 0.
    @Query(
        """
        SELECT l.post.id, count(l) FROM PostLike l
        WHERE l.post.id IN :postIds
        GROUP BY l.post.id
        """
    )
    fun countsByPostIds(@Param("postIds") postIds: Collection<Long>): List<Array<Any>>

    // Which of these posts the viewer has liked. Skipped entirely when signed out.
    @Query(
        """
        SELECT l.post.id FROM PostLike l
        WHERE l.user.id = :userId AND l.post.id IN :postIds
        """
    )
    fun likedPostIds(
        @Param("userId") userId: Long,
        @Param("postIds") postIds: Collection<Long>,
    ): List<Long>

    // Scalar forms for the single-post detail and write paths.
    fun countByPostId(postId: Long): Long

    fun existsByPostIdAndUserId(postId: Long, userId: Long): Boolean

    // Derived deletes need their own transaction; @Modifying is not used here because it
    // only applies to @Query methods.
    @Transactional
    fun deleteByPostIdAndUserId(postId: Long, userId: Long): Long
}
```

- [ ] **Step 4: Write the failing test**

Create `src/test/kotlin/com/example/demo/repository/PostLikeRepositoryIntegrationTest.kt`:

```kotlin
package com.example.demo.repository

import com.example.demo.model.Post
import com.example.demo.model.PostLike
import com.example.demo.model.PostStatus
import com.example.demo.model.Song
import com.example.demo.model.User
import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest
import org.springframework.boot.jdbc.test.autoconfigure.AutoConfigureTestDatabase
import org.springframework.boot.testcontainers.service.connection.ServiceConnection
import org.testcontainers.containers.PostgreSQLContainer
import kotlin.test.assertEquals
import kotlin.test.assertFalse
import kotlin.test.assertTrue

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class PostLikeRepositoryIntegrationTest {

    companion object {
        @JvmStatic
        @ServiceConnection
        val postgres: PostgreSQLContainer<*> = PostgreSQLContainer("postgres:16").apply { start() }
    }

    @Autowired lateinit var likes: PostLikeRepository
    @Autowired lateinit var posts: PostRepository
    @Autowired lateinit var users: UserRepository
    @Autowired lateinit var songs: SongRepository
    @Autowired lateinit var models: LlmModelRepository

    private var seq = 0

    private fun newUser(): User {
        seq++
        return users.saveAndFlush(User(email = "liker$seq-${System.nanoTime()}@example.com"))
    }

    private fun newPost(author: User): Post {
        seq++
        val n = "$seq-${System.nanoTime()}"
        val model = models.findByProviderAndModel("google", "gemini-2.5-flash").orElseThrow()
        val song = songs.saveAndFlush(
            Song(
                artist = "Artist $n", songName = "Song $n", album = null,
                artistNorm = "artist $n", songNameNorm = "song $n", albumNorm = "",
                llmModel = model, status = PostStatus.PENDING,
            )
        )
        return posts.saveAndFlush(Post(author = author, song = song))
    }

    @Test
    @DisplayName("countsByPostIds returns one row per liked post and omits unliked posts")
    fun `countsByPostIds omits unliked posts`() {
        val author = newUser()
        val liked = newPost(author)
        val unliked = newPost(author)
        likes.saveAndFlush(PostLike(post = liked, user = newUser()))
        likes.saveAndFlush(PostLike(post = liked, user = newUser()))

        val rows = likes.countsByPostIds(listOf(liked.id!!, unliked.id!!))
            .associate { (it[0] as Number).toLong() to (it[1] as Number).toLong() }

        assertEquals(2L, rows[liked.id])
        assertEquals(null, rows[unliked.id], "posts with no likes must be absent, not zero")
    }

    @Test
    @DisplayName("likedPostIds returns only the posts this viewer liked")
    fun `likedPostIds is viewer scoped`() {
        val author = newUser()
        val mine = newPost(author)
        val theirs = newPost(author)
        val me = newUser()
        likes.saveAndFlush(PostLike(post = mine, user = me))
        likes.saveAndFlush(PostLike(post = theirs, user = newUser()))

        val ids = likes.likedPostIds(me.id!!, listOf(mine.id!!, theirs.id!!))

        assertEquals(listOf(mine.id), ids)
    }

    @Test
    @DisplayName("unique constraint rejects the same user liking the same post twice")
    fun `unique constraint rejects duplicate like`() {
        val user = newUser()
        val post = newPost(newUser())
        likes.saveAndFlush(PostLike(post = post, user = user))

        assertThrows<Exception> {
            likes.saveAndFlush(PostLike(post = post, user = user))
        }
    }

    @Test
    @DisplayName("exists and delete by (post, user) round-trip")
    fun `exists and delete round trip`() {
        val user = newUser()
        val post = newPost(newUser())
        likes.saveAndFlush(PostLike(post = post, user = user))

        assertTrue(likes.existsByPostIdAndUserId(post.id!!, user.id!!))
        assertEquals(1L, likes.countByPostId(post.id!!))

        likes.deleteByPostIdAndUserId(post.id!!, user.id!!)

        assertFalse(likes.existsByPostIdAndUserId(post.id!!, user.id!!))
        assertEquals(0L, likes.countByPostId(post.id!!))
    }
}
```

- [ ] **Step 5: Run the test**

```bash
cd slangdump-pick-n-roll && ./gradlew test --tests "com.example.demo.repository.PostLikeRepositoryIntegrationTest"
```

Expected: PASS, 4 tests. If Flyway reports a checksum or ordering error, confirm `V5__post_likes.sql` is the highest version present and no `V5__` file already exists.

- [ ] **Step 6: Commit**

```bash
cd slangdump-pick-n-roll
git add src/main/resources/db/migration/V5__post_likes.sql \
        src/main/kotlin/com/example/demo/model/PostLike.kt \
        src/main/kotlin/com/example/demo/repository/PostLikeRepository.kt \
        src/test/kotlin/com/example/demo/repository/PostLikeRepositoryIntegrationTest.kt
git commit -m "feat: add post_likes table, entity, and repository"
```

---

## Task 2: `PostLikeService`

**Repo:** `slangdump-pick-n-roll`

**Files:**
- Create: `src/main/kotlin/com/example/demo/dto/LikeStateDto.kt`
- Create: `src/main/kotlin/com/example/demo/service/PostLikeService.kt`
- Test: `src/test/kotlin/com/example/demo/service/PostLikeServiceIntegrationTest.kt`

**Interfaces:**
- Consumes: `PostLikeRepository` (Task 1), existing `PostRepository`, `ProviderAccountRepository`, `UserService`, `ProviderClaims`.
- Produces:
  - `data class LikeStateDto(likeCount: Long, likedByMe: Boolean)`
  - `PostLikeService.findExistingUserId(claims: ProviderClaims): Long?`
  - `PostLikeService.countsFor(postIds: Collection<Long>): Map<Long, Long>`
  - `PostLikeService.likedBy(userId: Long?, postIds: Collection<Long>): Set<Long>`
  - `PostLikeService.countFor(postId: Long): Long`
  - `PostLikeService.likedByViewer(userId: Long?, postId: Long): Boolean`
  - `PostLikeService.like(postId: Long, claims: ProviderClaims): LikeStateDto`
  - `PostLikeService.unlike(postId: Long, claims: ProviderClaims): LikeStateDto`

- [ ] **Step 1: Write the DTO**

Create `src/main/kotlin/com/example/demo/dto/LikeStateDto.kt`:

```kotlin
package com.example.demo.dto

/**
 * The authoritative like state after a like/unlike. Returned so the client reconciles
 * against the server rather than trusting its own optimistic guess.
 */
data class LikeStateDto(
    val likeCount: Long,
    val likedByMe: Boolean,
)
```

- [ ] **Step 2: Write the service**

Create `src/main/kotlin/com/example/demo/service/PostLikeService.kt`:

```kotlin
package com.example.demo.service

import com.example.demo.dto.LikeStateDto
import com.example.demo.model.PostLike
import com.example.demo.repository.PostLikeRepository
import com.example.demo.repository.PostRepository
import com.example.demo.repository.ProviderAccountRepository
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Service
import org.springframework.web.server.ResponseStatusException

/**
 * Owns every like read and write. Counts are always derived from `post_likes` rows —
 * there is no counter column to drift.
 *
 * Reads and writes resolve the user differently on purpose. A read must never create an
 * account as a side effect of browsing, so [findExistingUserId] looks up an existing
 * provider account and returns null otherwise. A write goes through [UserService], which
 * creates or links exactly as post creation does.
 *
 * Not `@Transactional`, matching [PostService] and [UserService]: each repository call
 * commits independently so the race re-check in [like] is not poisoned by a surrounding
 * transaction.
 */
@Service
class PostLikeService(
    private val postLikeRepository: PostLikeRepository,
    private val postRepository: PostRepository,
    private val providerAccountRepository: ProviderAccountRepository,
    private val userService: UserService,
) {
    /** Read path: resolve an already-known user. Never creates one. */
    fun findExistingUserId(claims: ProviderClaims): Long? =
        providerAccountRepository
            .findByProviderAndProviderAccountId(claims.provider, claims.subject)
            .orElse(null)
            ?.user
            ?.id

    /** Counts for a page of posts. Every requested id gets an entry; unliked posts map to 0. */
    fun countsFor(postIds: Collection<Long>): Map<Long, Long> {
        if (postIds.isEmpty()) return emptyMap()
        val counted = postLikeRepository.countsByPostIds(postIds)
            .associate { (it[0] as Number).toLong() to (it[1] as Number).toLong() }
        // The GROUP BY omits posts with no likes. Backfill them so every requested id has
        // an entry and no caller can mistake "absent" for "unknown".
        return postIds.associateWith { counted[it] ?: 0L }
    }

    /** Which of these posts the viewer liked. Empty when signed out — no query is issued. */
    fun likedBy(userId: Long?, postIds: Collection<Long>): Set<Long> {
        if (userId == null || postIds.isEmpty()) return emptySet()
        return postLikeRepository.likedPostIds(userId, postIds).toSet()
    }

    /** Single-post forms for the detail endpoint. */
    fun countFor(postId: Long): Long = postLikeRepository.countByPostId(postId)

    fun likedByViewer(userId: Long?, postId: Long): Boolean =
        userId != null && postLikeRepository.existsByPostIdAndUserId(postId, userId)

    /** Idempotent. Liking an already-liked post succeeds and changes nothing. */
    fun like(postId: Long, claims: ProviderClaims): LikeStateDto {
        val post = postRepository.findById(postId).orElseThrow {
            ResponseStatusException(HttpStatus.NOT_FOUND, "post $postId not found")
        }
        val user = userService.syncFromClaims(claims)
        if (!postLikeRepository.existsByPostIdAndUserId(postId, user.id!!)) {
            try {
                postLikeRepository.saveAndFlush(PostLike(post = post, user = user))
            } catch (_: DataIntegrityViolationException) {
                // Lost a concurrent insert race. The user's intent is already satisfied.
            }
        }
        return LikeStateDto(likeCount = postLikeRepository.countByPostId(postId), likedByMe = true)
    }

    /** Idempotent. Unliking a post that was never liked succeeds and changes nothing. */
    fun unlike(postId: Long, claims: ProviderClaims): LikeStateDto {
        if (!postRepository.existsById(postId)) {
            throw ResponseStatusException(HttpStatus.NOT_FOUND, "post $postId not found")
        }
        val user = userService.syncFromClaims(claims)
        postLikeRepository.deleteByPostIdAndUserId(postId, user.id!!)
        return LikeStateDto(likeCount = postLikeRepository.countByPostId(postId), likedByMe = false)
    }
}
```

- [ ] **Step 3: Write the failing test**

Create `src/test/kotlin/com/example/demo/service/PostLikeServiceIntegrationTest.kt`:

```kotlin
package com.example.demo.service

import com.example.demo.model.AuthProvider
import com.example.demo.model.Post
import com.example.demo.model.PostStatus
import com.example.demo.model.Song
import com.example.demo.model.User
import com.example.demo.repository.LlmModelRepository
import com.example.demo.repository.PostRepository
import com.example.demo.repository.SongRepository
import com.example.demo.repository.UserRepository
import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.testcontainers.service.connection.ServiceConnection
import org.springframework.test.context.bean.override.mockito.MockitoBean
import org.testcontainers.containers.PostgreSQLContainer
import kotlin.test.assertEquals
import kotlin.test.assertFalse
import kotlin.test.assertNull
import kotlin.test.assertTrue

@SpringBootTest
class PostLikeServiceIntegrationTest {

    companion object {
        @JvmStatic
        @ServiceConnection
        val postgres: PostgreSQLContainer<*> = PostgreSQLContainer("postgres:16").apply { start() }
    }

    @Autowired lateinit var service: PostLikeService
    @Autowired lateinit var posts: PostRepository
    @Autowired lateinit var users: UserRepository
    @Autowired lateinit var songs: SongRepository
    @Autowired lateinit var models: LlmModelRepository

    @MockitoBean lateinit var aiService: AIService

    private fun claims(sub: String, email: String) = ProviderClaims(
        provider = AuthProvider.GOOGLE, subject = sub, email = email, name = "Liker", picture = null,
    )

    private fun newPost(): Post {
        val n = System.nanoTime()
        val author = users.saveAndFlush(User(email = "author-$n@example.com"))
        val model = models.findByProviderAndModel("google", "gemini-2.5-flash").orElseThrow()
        val song = songs.saveAndFlush(
            Song(
                artist = "Artist $n", songName = "Song $n", album = null,
                artistNorm = "artist $n", songNameNorm = "song $n", albumNorm = "",
                llmModel = model, status = PostStatus.PENDING,
            )
        )
        return posts.saveAndFlush(Post(author = author, song = song))
    }

    @Test
    @DisplayName("like is idempotent: liking twice leaves the count at 1")
    fun `like is idempotent`() {
        val post = newPost()
        val c = claims("google-like-1-${System.nanoTime()}", "like1-${System.nanoTime()}@example.com")

        val first = service.like(post.id!!, c)
        val second = service.like(post.id!!, c)

        assertEquals(1L, first.likeCount)
        assertTrue(first.likedByMe)
        assertEquals(1L, second.likeCount)
        assertTrue(second.likedByMe)
    }

    @Test
    @DisplayName("unlike is idempotent: unliking an unliked post reports 0 and not-liked")
    fun `unlike is idempotent`() {
        val post = newPost()
        val c = claims("google-unlike-${System.nanoTime()}", "unlike-${System.nanoTime()}@example.com")

        val state = service.unlike(post.id!!, c)

        assertEquals(0L, state.likeCount)
        assertFalse(state.likedByMe)
    }

    @Test
    @DisplayName("counts and likedBy are viewer-scoped across a batch")
    fun `counts and likedBy are viewer scoped`() {
        val mine = newPost()
        val theirs = newPost()
        val me = claims("google-me-${System.nanoTime()}", "me-${System.nanoTime()}@example.com")
        val other = claims("google-other-${System.nanoTime()}", "other-${System.nanoTime()}@example.com")

        service.like(mine.id!!, me)
        service.like(mine.id!!, other)
        service.like(theirs.id!!, other)

        val ids = listOf(mine.id!!, theirs.id!!)
        val counts = service.countsFor(ids)
        assertEquals(2L, counts[mine.id])
        assertEquals(1L, counts[theirs.id])

        val myId = service.findExistingUserId(me)!!
        assertEquals(setOf(mine.id), service.likedBy(myId, ids))
        // Signed out: no query, nothing liked.
        assertEquals(emptySet(), service.likedBy(null, ids))
    }

    @Test
    @DisplayName("findExistingUserId returns null for an unknown provider account and never creates one")
    fun `findExistingUserId never creates`() {
        val before = users.count()
        val unknown = claims("google-never-seen-${System.nanoTime()}", "ghost@example.com")

        assertNull(service.findExistingUserId(unknown))
        assertEquals(before, users.count())
    }

    @Test
    @DisplayName("like on a missing post throws 404")
    fun `like on missing post throws 404`() {
        val c = claims("google-404-${System.nanoTime()}", "e404-${System.nanoTime()}@example.com")
        val e = runCatching { service.like(99999999L, c) }.exceptionOrNull()
        assertTrue(e is org.springframework.web.server.ResponseStatusException)
        assertEquals(404, e.statusCode.value())
    }
}
```

- [ ] **Step 4: Run the test**

```bash
cd slangdump-pick-n-roll && ./gradlew test --tests "com.example.demo.service.PostLikeServiceIntegrationTest"
```

Expected: PASS, 5 tests.

- [ ] **Step 5: Commit**

```bash
cd slangdump-pick-n-roll
git add src/main/kotlin/com/example/demo/dto/LikeStateDto.kt \
        src/main/kotlin/com/example/demo/service/PostLikeService.kt \
        src/test/kotlin/com/example/demo/service/PostLikeServiceIntegrationTest.kt
git commit -m "feat: add PostLikeService with batched count reads and idempotent like/unlike"
```

---

## Task 3: Like endpoints and security rules

**Repo:** `slangdump-pick-n-roll`

**Files:**
- Modify: `src/main/kotlin/com/example/demo/controller/PostController.kt`
- Modify: `src/main/kotlin/com/example/demo/config/SecurityConfig.kt:50-56`
- Test: `src/test/kotlin/com/example/demo/controller/PostLikeControllerIntegrationTest.kt`

**Interfaces:**
- Consumes: `PostLikeService.like/unlike` and `LikeStateDto` (Task 2); existing `Jwt.toProviderClaims()`.
- Produces: `POST /api/v1/posts/{id}/like` and `DELETE /api/v1/posts/{id}/like`, both authenticated, both returning `LikeStateDto` as JSON.

**Critical:** the new security rules must be registered **above** the existing `/api/v1/posts/**` line. Spring Security applies the first matching rule, and `anyRequest().permitAll()` at the end means a missing rule leaves these endpoints publicly writable.

- [ ] **Step 1: Write the failing test**

Create `src/test/kotlin/com/example/demo/controller/PostLikeControllerIntegrationTest.kt`:

```kotlin
package com.example.demo.controller

import com.example.demo.repository.PostRepository
import com.example.demo.service.AIService
import com.example.demo.support.TestJwt
import com.jayway.jsonpath.JsonPath
import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.testcontainers.service.connection.ServiceConnection
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc
import org.springframework.http.MediaType
import org.springframework.test.context.bean.override.mockito.MockitoBean
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.*
import org.testcontainers.containers.PostgreSQLContainer

@SpringBootTest
@AutoConfigureMockMvc
class PostLikeControllerIntegrationTest {

    companion object {
        @JvmStatic
        @ServiceConnection
        val postgres: PostgreSQLContainer<*> = PostgreSQLContainer("postgres:16").apply { start() }
    }

    @Autowired lateinit var mockMvc: MockMvc
    @Autowired lateinit var postRepository: PostRepository

    @MockitoBean lateinit var aiService: AIService

    // Each created post needs a distinct author (one drop per Taipei day) and a distinct
    // song identity (unique index on the normalized identity).
    private fun createPost(tag: String): Int {
        val n = System.nanoTime()
        val token = TestJwt.token(sub = "google-$tag-$n", email = "$tag-$n@example.com", name = tag)
        val json = mockMvc.perform(
            post("/api/v1/posts")
                .header("Authorization", "Bearer $token")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{ "artist": "Artist $n", "songName": "Song $n", "rawLyrics": "x" }""")
        ).andExpect(status().isOk).andReturn().response.contentAsString
        return JsonPath.read<Int>(json, "$.id")
    }

    private fun liker(tag: String): String {
        val n = System.nanoTime()
        return TestJwt.token(sub = "google-liker-$tag-$n", email = "liker-$tag-$n@example.com", name = tag)
    }

    @Test
    @DisplayName("POST like without a token returns 401 — the endpoint must not fall through to permitAll")
    fun `like without token returns 401`() {
        val id = createPost("anon")
        mockMvc.perform(post("/api/v1/posts/$id/like"))
            .andExpect(status().isUnauthorized)
    }

    @Test
    @DisplayName("DELETE like without a token returns 401")
    fun `unlike without token returns 401`() {
        val id = createPost("anondel")
        mockMvc.perform(delete("/api/v1/posts/$id/like"))
            .andExpect(status().isUnauthorized)
    }

    @Test
    @DisplayName("POST like with a token signed by an untrusted key returns 401")
    fun `like with wrong-key token returns 401`() {
        val id = createPost("wrongkey")
        mockMvc.perform(
            post("/api/v1/posts/$id/like")
                .header("Authorization", "Bearer ${TestJwt.tokenSignedByWrongKey()}")
        ).andExpect(status().isUnauthorized)
    }

    @Test
    @DisplayName("like then unlike round-trips the count and likedByMe")
    fun `like then unlike round trips`() {
        val id = createPost("round")
        val token = liker("round")

        mockMvc.perform(post("/api/v1/posts/$id/like").header("Authorization", "Bearer $token"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.likeCount").value(1))
            .andExpect(jsonPath("$.likedByMe").value(true))

        mockMvc.perform(delete("/api/v1/posts/$id/like").header("Authorization", "Bearer $token"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.likeCount").value(0))
            .andExpect(jsonPath("$.likedByMe").value(false))
    }

    @Test
    @DisplayName("liking twice is idempotent — the count stays at 1")
    fun `liking twice is idempotent`() {
        val id = createPost("twice")
        val token = liker("twice")

        repeat(2) {
            mockMvc.perform(post("/api/v1/posts/$id/like").header("Authorization", "Bearer $token"))
                .andExpect(status().isOk)
                .andExpect(jsonPath("$.likeCount").value(1))
        }
    }

    @Test
    @DisplayName("like on a missing post returns 404")
    fun `like on missing post returns 404`() {
        mockMvc.perform(
            post("/api/v1/posts/99999999/like").header("Authorization", "Bearer ${liker("missing")}")
        ).andExpect(status().isNotFound)
    }
}
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
cd slangdump-pick-n-roll && ./gradlew test --tests "com.example.demo.controller.PostLikeControllerIntegrationTest"
```

Expected: FAIL. The 401 tests fail with 404 or 405 (no such handler), and the round-trip tests fail because the endpoints do not exist.

- [ ] **Step 3: Add the security rules**

In `src/main/kotlin/com/example/demo/config/SecurityConfig.kt`, replace the `authorizeHttpRequests` block:

```kotlin
            .authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/api/v1/posts/eligibility").authenticated()
                // MUST precede the /api/v1/posts/** permitAll below: first match wins, and
                // anyRequest() is permitAll, so without these two lines the like endpoints
                // would be publicly writable.
                it.requestMatchers(HttpMethod.POST, "/api/v1/posts/*/like").authenticated()
                it.requestMatchers(HttpMethod.DELETE, "/api/v1/posts/*/like").authenticated()
                it.requestMatchers(HttpMethod.GET, "/api/v1/posts", "/api/v1/posts/**").permitAll()
                it.requestMatchers(HttpMethod.POST, "/api/v1/posts").authenticated()
                it.requestMatchers(HttpMethod.POST, "/api/v1/users/sync").authenticated()
                it.anyRequest().permitAll()
            }
```

Also update the KDoc above the class — after the sentence ending `the entry point returns 401 otherwise.`, add:

```
 * Liking (`POST`/`DELETE /api/v1/posts/{id}/like`) is authenticated too. Its rules are
 * declared before the `/api/v1/posts/**` permitAll because the first matching rule wins.
```

- [ ] **Step 4: Add the endpoints**

In `src/main/kotlin/com/example/demo/controller/PostController.kt`, add `postLikeService` to the constructor:

```kotlin
class PostController(
    private val postRepository: PostRepository,
    private val postService: PostService,
    private val postLikeService: PostLikeService,
) {
```

Add the import `import com.example.demo.service.PostLikeService` (the wildcard `com.example.demo.dto.*` import already covers `LikeStateDto`).

Then add both handlers immediately after the `detail` function, before the private `nextDelay`:

```kotlin
    // Both are idempotent and return the authoritative state, so a client that lost a
    // response or double-fired converges on the server's answer rather than its own guess.
    @PostMapping("/{id}/like")
    fun like(
        @PathVariable id: Long,
        @AuthenticationPrincipal jwt: Jwt,
    ): LikeStateDto = postLikeService.like(id, jwt.toProviderClaims())

    @DeleteMapping("/{id}/like")
    fun unlike(
        @PathVariable id: Long,
        @AuthenticationPrincipal jwt: Jwt,
    ): LikeStateDto = postLikeService.unlike(id, jwt.toProviderClaims())
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
cd slangdump-pick-n-roll && ./gradlew test --tests "com.example.demo.controller.PostLikeControllerIntegrationTest"
```

Expected: PASS, 6 tests.

- [ ] **Step 6: Run the full suite to confirm nothing regressed**

```bash
cd slangdump-pick-n-roll && ./gradlew test
```

Expected: PASS. `ActuatorExposureTest` and `PostControllerIntegrationTest` must still pass — the reordered rules change no existing path.

- [ ] **Step 7: Commit**

```bash
cd slangdump-pick-n-roll
git add src/main/kotlin/com/example/demo/controller/PostController.kt \
        src/main/kotlin/com/example/demo/config/SecurityConfig.kt \
        src/test/kotlin/com/example/demo/controller/PostLikeControllerIntegrationTest.kt
git commit -m "feat: add authenticated like/unlike endpoints"
```

---

## Task 4: Expose `likeCount` and `likedByMe` on post responses

**Repo:** `slangdump-pick-n-roll`

**Files:**
- Modify: `src/main/kotlin/com/example/demo/dto/PostSummaryDto.kt`
- Modify: `src/main/kotlin/com/example/demo/dto/PostDetailDto.kt`
- Modify: `src/main/kotlin/com/example/demo/controller/PostController.kt` (`list`, `detail`, `create`, `toSummary`, `toDetail`)
- Test: `src/test/kotlin/com/example/demo/controller/PostLikeControllerIntegrationTest.kt` (extend)

**Interfaces:**
- Consumes: `PostLikeService.countsFor/likedBy/countFor/likedByViewer/findExistingUserId` (Task 2).
- Produces: `likeCount: Long` and `likedByMe: Boolean` on every summary and detail JSON response. `Post.toSummary(likeCount, likedByMe)` and `Post.toDetail(likeCount, likedByMe)` now take those values as parameters.

- [ ] **Step 1: Write the failing test**

Append these two tests inside `PostLikeControllerIntegrationTest` (before the closing brace):

```kotlin
    @Test
    @DisplayName("list and detail report likeCount, and likedByMe is false without a token")
    fun `list and detail report like state anonymously`() {
        val id = createPost("view")
        mockMvc.perform(post("/api/v1/posts/$id/like").header("Authorization", "Bearer ${liker("view")}"))
            .andExpect(status().isOk)

        // Anonymous detail: count is visible, likedByMe is false.
        mockMvc.perform(get("/api/v1/posts/$id"))
            .andExpect(jsonPath("$.likeCount").value(1))
            .andExpect(jsonPath("$.likedByMe").value(false))

        // Anonymous list still works and carries the fields.
        mockMvc.perform(get("/api/v1/posts").param("limit", "50"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.items[0].likeCount").isNumber)
            .andExpect(jsonPath("$.items[0].likedByMe").value(false))
    }

    @Test
    @DisplayName("likedByMe is true for the viewer who liked and false for another viewer")
    fun `likedByMe is viewer scoped`() {
        val id = createPost("scoped")
        val mine = liker("scoped-mine")
        val theirs = liker("scoped-theirs")

        mockMvc.perform(post("/api/v1/posts/$id/like").header("Authorization", "Bearer $mine"))
            .andExpect(status().isOk)

        mockMvc.perform(get("/api/v1/posts/$id").header("Authorization", "Bearer $mine"))
            .andExpect(jsonPath("$.likedByMe").value(true))
            .andExpect(jsonPath("$.likeCount").value(1))

        // A signed-in viewer who has never liked anything sees the count but not their like.
        mockMvc.perform(post("/api/v1/posts/${createPost("other")}/like").header("Authorization", "Bearer $theirs"))
            .andExpect(status().isOk)
        mockMvc.perform(get("/api/v1/posts/$id").header("Authorization", "Bearer $theirs"))
            .andExpect(jsonPath("$.likedByMe").value(false))
            .andExpect(jsonPath("$.likeCount").value(1))
    }
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
cd slangdump-pick-n-roll && ./gradlew test --tests "com.example.demo.controller.PostLikeControllerIntegrationTest"
```

Expected: FAIL — `No value at JSON path "$.likeCount"`.

- [ ] **Step 3: Add the DTO fields**

In `src/main/kotlin/com/example/demo/dto/PostSummaryDto.kt`, add these two properties to the data class (keep every existing field):

```kotlin
    val likeCount: Long,
    val likedByMe: Boolean,
```

Do the same in `src/main/kotlin/com/example/demo/dto/PostDetailDto.kt`. Both gain the fields — the TypeScript mirror declares `PostDetailDto extends PostSummaryDto`, and that relationship should stay true.

- [ ] **Step 4: Wire the mappers**

In `PostController.kt`, change the two mapper signatures at the bottom of the file. `toSummary` becomes:

```kotlin
private fun Post.toSummary(likeCount: Long, likedByMe: Boolean) = PostSummaryDto(
    id = id!!,
    authorName = author.displayName,
    authorPhotoUrl = author.photoUrl,
    artist = song.artist,
    songName = song.songName,
    album = song.album,
    lyricUrl = lyricUrl,
    description = description,
    status = song.status,
    createdAt = createdAt,
    likeCount = likeCount,
    likedByMe = likedByMe,
)
```

And `toDetail` takes the same two parameters — add them to the signature and pass them through as `likeCount = likeCount, likedByMe = likedByMe` alongside the existing fields.

- [ ] **Step 5: Wire the handlers**

Add imports to `PostController.kt`:

```kotlin
import org.springframework.security.oauth2.jwt.Jwt
```

(already present) and keep `com.example.demo.web.toProviderClaims` (already imported).

`create` — a new post has no likes:

```kotlin
    @PostMapping
    fun create(
        @Valid @RequestBody request: CreatePostRequest,
        @AuthenticationPrincipal jwt: Jwt,
    ): PostSummaryDto =
        postService.create(request, jwt.toProviderClaims()).toSummary(likeCount = 0, likedByMe = false)
```

`list` — one batched count query and one batched liked-set query per page:

```kotlin
    @GetMapping
    fun list(
        @RequestParam(required = false) cursor: String?,
        @RequestParam(defaultValue = "10") limit: Int,
        @AuthenticationPrincipal jwt: Jwt?,
    ): CursorPage<PostSummaryDto> {
        val capped = min(limit.coerceAtLeast(1), 50)
        val decoded = cursor?.let {
            try { CursorCodec.decode(it) }
            catch (_: Exception) { throw ValidationException("Malformed cursor") }
        }
        val pageable = PageRequest.ofSize(capped + 1)
        val rows = if (decoded == null) {
            postRepository.findFirstPage(pageable)
        } else {
            postRepository.findPageAfter(decoded.createdAt, decoded.id, pageable)
        }
        val hasMore = rows.size > capped
        val page = if (hasMore) rows.subList(0, capped) else rows

        // Two extra queries for the whole page, regardless of page size.
        val ids = page.mapNotNull { it.id }
        val counts = postLikeService.countsFor(ids)
        val liked = postLikeService.likedBy(viewerId(jwt), ids)

        val items = page.map { it.toSummary(counts[it.id] ?: 0L, liked.contains(it.id)) }
        val nextCursor = if (hasMore) {
            val last = items.last()
            CursorCodec.encode(last.createdAt, last.id)
        } else null
        return CursorPage(items = items, nextCursor = nextCursor)
    }
```

`detail` — same shape as before, with the two scalar lookups; the 202/200 and header behaviour is unchanged:

```kotlin
    @GetMapping("/{id}")
    fun detail(
        @PathVariable id: Long,
        @AuthenticationPrincipal jwt: Jwt?,
    ): ResponseEntity<PostDetailDto> {
        val post = postRepository.findById(id).orElseThrow {
            ResponseStatusException(HttpStatus.NOT_FOUND, "post $id not found")
        }
        val view = post.toDetail(
            likeCount = postLikeService.countFor(id),
            likedByMe = postLikeService.likedByViewer(viewerId(jwt), id),
        )
        return if (post.song.status == PostStatus.PENDING) {
            val age = Duration.between(post.song.createdAt, Instant.now()).seconds
            ResponseEntity.status(HttpStatus.ACCEPTED)
                .header(HttpHeaders.RETRY_AFTER, nextDelay(age).toString())
                .cacheControl(CacheControl.noStore())
                .body(view)
        } else {
            // COMPLETED / FAILED are terminal and immutable — let the browser cache them.
            ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(Duration.ofHours(1)).cachePrivate())
                .body(view)
        }
    }

    // These reads stay public, so the token is optional and a malformed-but-verified one
    // must not 422 a browse. An unknown user resolves to null, i.e. "nothing liked".
    private fun viewerId(jwt: Jwt?): Long? {
        val claims = try { jwt?.toProviderClaims() } catch (_: Exception) { null } ?: return null
        return postLikeService.findExistingUserId(claims)
    }
```

- [ ] **Step 6: Run the test to verify it passes**

```bash
cd slangdump-pick-n-roll && ./gradlew test --tests "com.example.demo.controller.PostLikeControllerIntegrationTest"
```

Expected: PASS, 8 tests.

- [ ] **Step 7: Run the full suite**

```bash
cd slangdump-pick-n-roll && ./gradlew test
```

Expected: PASS. `PostControllerIntegrationTest` asserts only on fields that still exist, so it needs no change here.

- [ ] **Step 8: Commit**

```bash
cd slangdump-pick-n-roll
git add src/main/kotlin/com/example/demo/dto/PostSummaryDto.kt \
        src/main/kotlin/com/example/demo/dto/PostDetailDto.kt \
        src/main/kotlin/com/example/demo/controller/PostController.kt \
        src/test/kotlin/com/example/demo/controller/PostLikeControllerIntegrationTest.kt
git commit -m "feat: expose likeCount and likedByMe on post summary and detail"
```

---

## Task 5: Remove LLM identity from the API

**Repo:** `slangdump-pick-n-roll`

**Files:**
- Modify: `src/main/kotlin/com/example/demo/dto/PostDetailDto.kt`
- Modify: `src/main/kotlin/com/example/demo/controller/PostController.kt` (`toDetail`)
- Test: `src/test/kotlin/com/example/demo/controller/PostControllerIntegrationTest.kt` (extend)

**Interfaces:**
- Produces: `PostDetailDto` no longer has `llmProvider` or `llmModel`. Nothing else changes — `LlmModel`, `llm_models`, and `songs.llm_model_id` all stay, since the dedup unique key depends on the model id.

- [ ] **Step 1: Write the failing test**

Append to `PostControllerIntegrationTest` (before the closing brace):

```kotlin
    @Test
    @DisplayName("detail response never leaks which LLM produced the translation")
    fun `detail hides llm identity`() {
        val n = System.nanoTime()
        val token = TestJwt.token(sub = "google-llm-$n", email = "llm-$n@example.com", name = "LLM")
        val createJson = mockMvc.perform(
            postJson("""{ "artist": "Artist $n", "songName": "Song $n", "rawLyrics": "x" }""", token = token)
        ).andExpect(status().isOk).andReturn().response.contentAsString
        val id = JsonPath.read<Int>(createJson, "$.id")

        val detailJson = mockMvc.perform(get("/api/v1/posts/$id"))
            .andExpect(jsonPath("$.llmProvider").doesNotExist())
            .andExpect(jsonPath("$.llmModel").doesNotExist())
            .andReturn().response.contentAsString

        // Nor under any snake_case or nested spelling.
        assert(!detailJson.contains("llm"))
        assert(!detailJson.contains("gemini"))
    }
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
cd slangdump-pick-n-roll && ./gradlew test --tests "com.example.demo.controller.PostControllerIntegrationTest"
```

Expected: FAIL — `llmProvider` exists in the body.

- [ ] **Step 3: Remove the fields**

In `src/main/kotlin/com/example/demo/dto/PostDetailDto.kt`, delete these two lines and the blank line separating them from `lines`:

```kotlin
    val llmProvider: String?,
    val llmModel: String?,
```

In `PostController.kt`'s `toDetail`, delete the two corresponding assignments:

```kotlin
    llmProvider = song.llmModel.provider,
    llmModel = song.llmModel.model,
```

Add a comment above `toDetail` explaining why the model is deliberately absent:

```kotlin
// The LLM behind a translation is deliberately not exposed: users must not learn which
// model ran. `song.llmModel` is still part of the dedup identity — it just never leaves
// the server.
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd slangdump-pick-n-roll && ./gradlew test --tests "com.example.demo.controller.PostControllerIntegrationTest"
```

Expected: PASS.

- [ ] **Step 5: Run the full suite**

```bash
cd slangdump-pick-n-roll && ./gradlew test
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
cd slangdump-pick-n-roll
git add src/main/kotlin/com/example/demo/dto/PostDetailDto.kt \
        src/main/kotlin/com/example/demo/controller/PostController.kt \
        src/test/kotlin/com/example/demo/controller/PostControllerIntegrationTest.kt
git commit -m "feat: stop exposing llmProvider and llmModel in the post detail API"
```

---

## Task 6: Frontend types and BFF like routes

**Repo:** `slangdump-poster`

**Files:**
- Modify: `src/types/api.ts`
- Create: `src/app/api/posts/[id]/like/route.ts`
- Modify: `src/app/api/posts/route.ts:45-63` (the `GET` handler)
- Test: `src/app/api/posts/[id]/like/__tests__/route.test.ts`

**Interfaces:**
- Consumes: the backend contract from Tasks 3-5; existing `auth()`, `mintBffToken`, `UNAUTHORIZED`, `INTERNAL_ERROR`.
- Produces:
  - `PostSummaryDto.likeCount: number`, `PostSummaryDto.likedByMe: boolean`
  - `interface LikeStateDto { likeCount: number; likedByMe: boolean }`
  - `POST /api/posts/{id}/like` and `DELETE /api/posts/{id}/like`

- [ ] **Step 1: Update the shared types**

In `src/types/api.ts`, add to `PostSummaryDto` (after `createdAt`):

```ts
  likeCount: number;
  likedByMe: boolean;
```

Delete these two lines from `PostDetailDto`:

```ts
  llmProvider: string | null;
  llmModel: string | null;
```

And add at the end of the file:

```ts
// Authoritative like state returned by POST/DELETE /api/posts/{id}/like. The client
// reconciles to this rather than trusting its own optimistic guess.
export interface LikeStateDto {
  likeCount: number;
  likedByMe: boolean;
}
```

- [ ] **Step 2: Write the failing test**

Create `src/app/api/posts/[id]/like/__tests__/route.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

const auth = vi.fn();
vi.mock('@/auth', () => ({ auth: () => auth() }));
vi.mock('@/lib/bffToken', () => ({ mintBffToken: async () => 'signed.jwt.token' }));

import { POST, DELETE } from '../route';

const params = Promise.resolve({ id: '7' });

describe('like BFF route', () => {
  beforeEach(() => {
    auth.mockReset();
    vi.stubGlobal('fetch', vi.fn());
  });

  it('returns 401 without a session and never calls the backend', async () => {
    auth.mockResolvedValue(null);

    const res = await POST(new Request('http://localhost/api/posts/7/like'), { params });

    expect(res.status).toBe(401);
    expect(fetch).not.toHaveBeenCalled();
  });

  it('forwards a bearer token and the backend body on POST', async () => {
    auth.mockResolvedValue({ user: { email: 'a@b.c', name: 'A' }, providerSub: 'google-1' });
    vi.mocked(fetch).mockResolvedValue(
      new Response(JSON.stringify({ likeCount: 3, likedByMe: true }), { status: 200 })
    );

    const res = await POST(new Request('http://localhost/api/posts/7/like'), { params });

    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ likeCount: 3, likedByMe: true });
    const [url, init] = vi.mocked(fetch).mock.calls[0];
    expect(url).toContain('/api/v1/posts/7/like');
    expect((init as RequestInit).method).toBe('POST');
    expect((init as RequestInit).headers).toMatchObject({ Authorization: 'Bearer signed.jwt.token' });
  });

  it('sends DELETE upstream when unliking', async () => {
    auth.mockResolvedValue({ user: { email: 'a@b.c', name: 'A' }, providerSub: 'google-1' });
    vi.mocked(fetch).mockResolvedValue(
      new Response(JSON.stringify({ likeCount: 0, likedByMe: false }), { status: 200 })
    );

    await DELETE(new Request('http://localhost/api/posts/7/like'), { params });

    expect((vi.mocked(fetch).mock.calls[0][1] as RequestInit).method).toBe('DELETE');
  });

  it('passes a backend error status through', async () => {
    auth.mockResolvedValue({ user: { email: 'a@b.c', name: 'A' }, providerSub: 'google-1' });
    vi.mocked(fetch).mockResolvedValue(
      new Response(JSON.stringify({ code: 40400, message: 'Not Found' }), { status: 404 })
    );

    const res = await POST(new Request('http://localhost/api/posts/7/like'), { params });

    expect(res.status).toBe(404);
  });
});
```

- [ ] **Step 3: Run it to confirm it fails**

```bash
cd slangdump-poster && npx vitest run src/app/api/posts/\[id\]/like/__tests__/route.test.ts
```

Expected: FAIL — cannot resolve `../route`.

- [ ] **Step 4: Write the route**

Create `src/app/api/posts/[id]/like/route.ts`:

```ts
import { NextResponse } from 'next/server';
import { auth } from '@/auth';
import { mintBffToken } from '@/lib/bffToken';
import { UNAUTHORIZED, INTERNAL_ERROR } from '@/lib/errorCodes';

const BACKEND = process.env.SPRING_BACKEND_URL ?? 'http://localhost:8080';

// Liking requires a signed-in user. Identity comes from the server session and is minted
// into a short-lived token; the backend independently re-verifies it (401 there is the
// real gate — this check just avoids a pointless round-trip).
async function forward(method: 'POST' | 'DELETE', id: string) {
  const session = await auth();
  if (!session?.user || !session.providerSub) {
    return NextResponse.json({ code: UNAUTHORIZED, message: 'Sign in to like' }, { status: 401 });
  }

  const token = await mintBffToken({
    provider: session.provider ?? 'google',
    sub: session.providerSub,
    email: session.user.email,
    name: session.user.name,
    picture: session.user.image,
  });

  const response = await fetch(`${BACKEND}/api/v1/posts/${id}/like`, {
    method,
    headers: { Authorization: `Bearer ${token}` },
  });

  // Success and error bodies pass through verbatim — the backend already speaks the
  // unified { code, message } contract.
  const text = await response.text();
  return new NextResponse(text, {
    status: response.status,
    headers: { 'Content-Type': response.headers.get('Content-Type') ?? 'application/json' },
  });
}

type Ctx = { params: Promise<{ id: string }> };

export async function POST(_request: Request, { params }: Ctx) {
  try {
    const { id } = await params;
    return await forward('POST', id);
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : 'Unknown error';
    return NextResponse.json({ code: INTERNAL_ERROR, message: msg }, { status: 500 });
  }
}

export async function DELETE(_request: Request, { params }: Ctx) {
  try {
    const { id } = await params;
    return await forward('DELETE', id);
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : 'Unknown error';
    return NextResponse.json({ code: INTERNAL_ERROR, message: msg }, { status: 500 });
  }
}
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
cd slangdump-poster && npx vitest run src/app/api/posts/\[id\]/like/__tests__/route.test.ts
```

Expected: PASS, 4 tests.

- [ ] **Step 6: Forward the session token on the list GET**

Without this, `likedByMe` is always `false` even when signed in. In `src/app/api/posts/route.ts`, replace the `GET` handler body's fetch section:

```ts
export async function GET(request: NextRequest) {
  try {
    const url = new URL(request.url);
    const cursor = url.searchParams.get('cursor');
    const limit = url.searchParams.get('limit') ?? '10';
    const params = new URLSearchParams({ limit });
    if (cursor) params.set('cursor', cursor);

    // The feed is public, so the token is OPTIONAL — it only lets the backend fill in
    // likedByMe. A missing or failed session degrades to an anonymous read, never a 401.
    const headers: Record<string, string> = {};
    const session = await auth().catch(() => null);
    if (session?.user && session.providerSub) {
      const token = await mintBffToken({
        provider: session.provider ?? 'google',
        sub: session.providerSub,
        email: session.user.email,
        name: session.user.name,
        picture: session.user.image,
      });
      headers.Authorization = `Bearer ${token}`;
    }

    const response = await fetch(`${BACKEND}/api/v1/posts?${params.toString()}`, { headers });
    const text = await response.text();
    return new NextResponse(text, {
      status: response.status,
      headers: { 'Content-Type': 'application/json' },
    });
  } catch (error: unknown) {
    const msg = error instanceof Error ? error.message : 'Unknown error';
    return NextResponse.json({ code: INTERNAL_ERROR, message: msg }, { status: 500 });
  }
}
```

- [ ] **Step 7: Test the optional token forwarding**

The existing `src/app/api/posts/__tests__/route.test.ts` only exercises `POST`, so nothing there breaks. Add coverage for the new `GET` behaviour — append to that file:

```ts
import { GET } from '../route';

const makeGetReq = (url = 'http://localhost/api/posts?limit=10') =>
  new Request(url) as unknown as NextRequest;

describe('GET /api/posts optional token forwarding', () => {
  beforeEach(() => {
    auth.mockReset();
    vi.mocked(mintBffToken).mockClear();
    vi.unstubAllGlobals();
  });

  it('attaches a bearer token when a session exists so likedByMe can be filled in', async () => {
    auth.mockResolvedValue({
      user: { email: 'a@x.com', name: 'A', image: null },
      provider: 'google',
      providerSub: 'sub1',
    });
    const fetchMock = vi.fn(async () => new Response('{"items":[],"nextCursor":null}', { status: 200 }));
    vi.stubGlobal('fetch', fetchMock);

    const res = await GET(makeGetReq());

    expect(res.status).toBe(200);
    const opts = fetchMock.mock.calls[0][1] as RequestInit;
    expect((opts.headers as Record<string, string>).Authorization).toBe('Bearer signed-token');
  });

  it('still serves the feed anonymously when there is no session', async () => {
    auth.mockResolvedValue(null);
    const fetchMock = vi.fn(async () => new Response('{"items":[],"nextCursor":null}', { status: 200 }));
    vi.stubGlobal('fetch', fetchMock);

    const res = await GET(makeGetReq());

    expect(res.status).toBe(200);
    const opts = fetchMock.mock.calls[0][1] as RequestInit;
    expect((opts.headers as Record<string, string>).Authorization).toBeUndefined();
    expect(mintBffToken).not.toHaveBeenCalled();
  });

  it('degrades to an anonymous read when the session lookup throws', async () => {
    auth.mockRejectedValue(new Error('session store down'));
    const fetchMock = vi.fn(async () => new Response('{"items":[],"nextCursor":null}', { status: 200 }));
    vi.stubGlobal('fetch', fetchMock);

    const res = await GET(makeGetReq());

    expect(res.status).toBe(200);
  });
});
```

Run it:

```bash
cd slangdump-poster && npx vitest run src/app/api/posts/__tests__/route.test.ts
```

Expected: PASS, 5 tests.

- [ ] **Step 8: Run the full frontend suite**

```bash
cd slangdump-poster && npm test
```

Expected: the API route tests pass. `FeedItem.test.tsx` will fail type-checking on the newly required `likeCount`/`likedByMe` fixture fields — that is fixed in Task 8 and is the expected state at this commit. Do not patch it here.

- [ ] **Step 9: Commit**

```bash
cd slangdump-poster
git add src/types/api.ts src/app/api/posts/route.ts \
        src/app/api/posts/__tests__/route.test.ts "src/app/api/posts/[id]/like"
git commit -m "feat: add like BFF routes and optional session token on the feed GET"
```

---

## Task 7: `PostLikeButton`

**Repo:** `slangdump-poster`

**Files:**
- Create: `src/components/PostLikeButton.tsx`
- Test: `src/components/__tests__/PostLikeButton.test.tsx`

**Interfaces:**
- Consumes: `LikeStateDto` from `@/types/api` (Task 6); `POST`/`DELETE /api/posts/{id}/like` (Task 6); `useSession` from `next-auth/react`.
- Produces: `<PostLikeButton postId={number} likeCount={number} likedByMe={boolean} />`. Accessible name is `Like` when not liked and `Unlike` when liked.

- [ ] **Step 1: Write the failing test**

Create `src/components/__tests__/PostLikeButton.test.tsx`:

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// Mock the auth hook so no SessionProvider is required.
const useSession = vi.fn();
vi.mock('next-auth/react', () => ({
  useSession: () => useSession(),
}));

import { PostLikeButton } from '../PostLikeButton';

const signedIn = () => useSession.mockReturnValue({ data: { user: { name: 'A' } }, status: 'authenticated' });
const signedOut = () => useSession.mockReturnValue({ data: null, status: 'unauthenticated' });

describe('PostLikeButton', () => {
  beforeEach(() => {
    useSession.mockReset();
    vi.stubGlobal('fetch', vi.fn());
  });

  it('renders the server count and is disabled when signed out', async () => {
    signedOut();
    render(<PostLikeButton postId={7} likeCount={12} likedByMe={false} />);

    const button = screen.getByRole('button', { name: 'Like' });
    expect(screen.getByText('12')).toBeInTheDocument();
    expect(button).toBeDisabled();
    expect(button).toHaveAttribute('title', 'Sign in to like');
  });

  it('optimistically increments, then reconciles to the server count', async () => {
    signedIn();
    vi.mocked(fetch).mockResolvedValue(
      new Response(JSON.stringify({ likeCount: 99, likedByMe: true }), { status: 200 })
    );
    render(<PostLikeButton postId={7} likeCount={12} likedByMe={false} />);

    await userEvent.click(screen.getByRole('button', { name: 'Like' }));

    // Reconciled to the server's authoritative number, not the optimistic 13.
    await waitFor(() => expect(screen.getByText('99')).toBeInTheDocument());
    expect(screen.getByRole('button', { name: 'Unlike' })).toHaveAttribute('aria-pressed', 'true');
    expect((vi.mocked(fetch).mock.calls[0][1] as RequestInit).method).toBe('POST');
  });

  it('sends DELETE and decrements when already liked', async () => {
    signedIn();
    vi.mocked(fetch).mockResolvedValue(
      new Response(JSON.stringify({ likeCount: 11, likedByMe: false }), { status: 200 })
    );
    render(<PostLikeButton postId={7} likeCount={12} likedByMe />);

    await userEvent.click(screen.getByRole('button', { name: 'Unlike' }));

    await waitFor(() => expect(screen.getByText('11')).toBeInTheDocument());
    expect((vi.mocked(fetch).mock.calls[0][1] as RequestInit).method).toBe('DELETE');
  });

  it('reverts the optimistic update when the request fails', async () => {
    signedIn();
    vi.mocked(fetch).mockResolvedValue(new Response('{}', { status: 500 }));
    render(<PostLikeButton postId={7} likeCount={12} likedByMe={false} />);

    await userEvent.click(screen.getByRole('button', { name: 'Like' }));

    await waitFor(() => expect(screen.getByRole('button', { name: 'Like' })).toHaveAttribute('aria-pressed', 'false'));
    expect(screen.getByText('12')).toBeInTheDocument();
  });

  it('does not fire a second request while one is in flight', async () => {
    signedIn();
    let release: (r: Response) => void = () => {};
    vi.mocked(fetch).mockReturnValue(new Promise<Response>(resolve => { release = resolve; }));
    render(<PostLikeButton postId={7} likeCount={12} likedByMe={false} />);

    const button = screen.getByRole('button', { name: 'Like' });
    await userEvent.click(button);
    await userEvent.click(button);

    expect(vi.mocked(fetch)).toHaveBeenCalledTimes(1);
    release(new Response(JSON.stringify({ likeCount: 13, likedByMe: true }), { status: 200 }));
  });
});
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
cd slangdump-poster && npx vitest run src/components/__tests__/PostLikeButton.test.tsx
```

Expected: FAIL — cannot resolve `../PostLikeButton`.

- [ ] **Step 3: Write the component**

Create `src/components/PostLikeButton.tsx`:

```tsx
'use client';

import React, { useState } from 'react';
import { useSession } from 'next-auth/react';
import type { LikeStateDto } from '@/types/api';

export interface PostLikeButtonProps {
  postId: number;
  likeCount: number;
  likedByMe: boolean;
}

/**
 * The like control. Owns its own optimistic state and network call so FeedItem stays a
 * presentational card.
 *
 * The optimistic update is only a guess: the response carries the authoritative count and
 * the component reconciles to it. On failure it reverts. An in-flight guard means a
 * double-click cannot fire two requests, which would otherwise race the toggle.
 */
export const PostLikeButton: React.FC<PostLikeButtonProps> = ({ postId, likeCount, likedByMe }) => {
  const { status } = useSession();
  const signedIn = status === 'authenticated';

  const [count, setCount] = useState(likeCount);
  const [liked, setLiked] = useState(likedByMe);
  const [busy, setBusy] = useState(false);

  const toggle = async () => {
    if (busy || !signedIn) return;
    const next = !liked;

    setBusy(true);
    setLiked(next);
    setCount(c => c + (next ? 1 : -1));

    try {
      const res = await fetch(`/api/posts/${postId}/like`, { method: next ? 'POST' : 'DELETE' });
      if (!res.ok) throw new Error(`like failed (${res.status})`);
      const state = await res.json() as LikeStateDto;
      setCount(state.likeCount);
      setLiked(state.likedByMe);
    } catch {
      setLiked(!next);
      setCount(c => c + (next ? -1 : 1));
    } finally {
      setBusy(false);
    }
  };

  return (
    <button
      type="button"
      onClick={toggle}
      disabled={!signedIn}
      aria-pressed={liked}
      aria-label={liked ? 'Unlike' : 'Like'}
      title={signedIn ? undefined : 'Sign in to like'}
      className={`flex items-center gap-2 group transition-colors disabled:cursor-not-allowed ${
        liked ? 'text-orange-500' : 'text-zinc-500 hover:text-orange-500 disabled:hover:text-zinc-500'
      }`}
    >
      <span
        className="material-symbols-outlined text-xl group-active:scale-125 transition-transform"
        style={liked ? { fontVariationSettings: "'FILL' 1" } : undefined}
      >
        favorite
      </span>
      <span className="font-inter font-black italic text-sm">{count}</span>
    </button>
  );
};
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd slangdump-poster && npx vitest run src/components/__tests__/PostLikeButton.test.tsx
```

Expected: PASS, 5 tests.

- [ ] **Step 5: Commit**

```bash
cd slangdump-poster
git add src/components/PostLikeButton.tsx src/components/__tests__/PostLikeButton.test.tsx
git commit -m "feat: add PostLikeButton with optimistic toggle and server reconciliation"
```

---

## Task 8: Clean up `FeedItem`

**Repo:** `slangdump-poster`

**Files:**
- Modify: `src/components/FeedItem.tsx` (lines ~171-176, ~215, ~344-348, ~364-389)
- Test: `src/components/__tests__/FeedItem.test.tsx`

**Interfaces:**
- Consumes: `PostLikeButton` (Task 7); `PostSummaryDto.likeCount` / `.likedByMe` (Task 6).
- Produces: no new exports. `FeedItem`'s props are unchanged in shape.

- [ ] **Step 1: Update the existing test fixture and add the new assertions**

In `src/components/__tests__/FeedItem.test.tsx`, add the mock at the top (above the `FeedItem` import), since `PostLikeButton` now calls `useSession`:

```tsx
const useSession = vi.fn(() => ({ data: null, status: 'unauthenticated' }));
vi.mock('next-auth/react', () => ({
  useSession: () => useSession(),
}));
```

Update the `vitest` import to `import { describe, it, expect, vi } from 'vitest';`, and add the two new required fields to the `post()` fixture:

```tsx
    likeCount: 0,
    likedByMe: false,
```

Then append this suite:

```tsx
describe('FeedItem placeholders and disclosures', () => {
  it('shows the real like count from the post, not a hard-coded one', () => {
    render(<FeedItem post={post({ likeCount: 7 })} />);
    expect(screen.getByText('7')).toBeInTheDocument();
    expect(screen.queryByText('2.4K')).not.toBeInTheDocument();
  });

  it('no longer renders the fake VERIFIED badge or the fake comment count', () => {
    render(<FeedItem post={post()} />);
    expect(screen.queryByText('VERIFIED')).not.toBeInTheDocument();
    expect(screen.queryByText('182')).not.toBeInTheDocument();
  });

  it('disables the comment button and explains why', () => {
    render(<FeedItem post={post()} />);
    const comments = screen.getByRole('button', { name: 'Comments' });
    expect(comments).toBeDisabled();
    expect(comments).toHaveAttribute('title', 'Comments arrive in the next phase.');
  });

  it('drops the fake interaction avatar stack', () => {
    render(<FeedItem post={post()} />);
    expect(screen.queryByAltText('Interaction User Avatar')).not.toBeInTheDocument();
    expect(screen.queryByText('+20')).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
cd slangdump-poster && npx vitest run src/components/__tests__/FeedItem.test.tsx
```

Expected: FAIL — `VERIFIED` still present, no button named `Comments`, `2.4K` still rendered.

- [ ] **Step 3: Delete the placeholder constants**

In `src/components/FeedItem.tsx`, replace the cosmetic-placeholder block:

```tsx
  // Cosmetic placeholders (not part of the backend contract).
  const timeLoc = "DAILY DROP";
  const likesCount = "2.4K";
  const commentsCount = "182";
  const interactions = [1, 2, 3];
  const additional = "+20";
  const tagsList = ['#DAILY_DROP'];
```

with:

```tsx
  // Cosmetic placeholders (not part of the backend contract).
  const timeLoc = "DAILY DROP";
  const tagsList = ['#DAILY_DROP'];
```

- [ ] **Step 4: Remove the fake VERIFIED badge**

Delete this line from the header block:

```tsx
              <span className="bg-orange-600 text-black text-[10px] font-black px-2 py-0.5 uppercase italic">VERIFIED</span>
```

- [ ] **Step 5: Replace the LLM attribution with the AI notice**

Replace this block:

```tsx
            {detail.llmProvider && (
              <div className="text-xs text-zinc-500">
                Translated by {detail.llmProvider} / {detail.llmModel}
              </div>
            )}
```

with:

```tsx
            {/* Which model produced this is deliberately never shown. What matters to the
                reader is that a model produced it at all. Wording mirrors the AI sections
                of lib/disclaimer-content.ts so the app says one consistent thing. */}
            <div className="border-l-2 border-zinc-700 pl-3 text-xs leading-relaxed text-zinc-500">
              AI-generated translations may contain errors, omissions, or debatable readings.
              Verify anything you rely on.
            </div>
```

- [ ] **Step 6: Rebuild the interactions footer**

Replace the whole footer block (from `{/* Interactions Footer */}` through its closing `</div>`) with:

```tsx
      {/* Interactions Footer */}
      <div className="p-6 bg-black flex items-center gap-6">
        <PostLikeButton postId={post.id} likeCount={post.likeCount} likedByMe={post.likedByMe} />
        {/* Comments land next phase. Shown disabled rather than hidden so the affordance
            is discoverable and the reason is stated on hover. */}
        <button
          type="button"
          disabled
          aria-label="Comments"
          title="Comments arrive in the next phase."
          className="relative flex items-center gap-2 text-zinc-700 cursor-not-allowed group/comments"
        >
          <span className="material-symbols-outlined text-xl">comment</span>
          <span className="pointer-events-none absolute left-0 bottom-full mb-2 hidden group-hover/comments:block whitespace-nowrap bg-zinc-950 border border-zinc-800 px-2 py-1 text-[10px] font-bold uppercase tracking-widest text-zinc-400">
            Comments arrive in the next phase.
          </span>
        </button>
      </div>
```

Add the import at the top of the file:

```tsx
import { PostLikeButton } from './PostLikeButton';
```

- [ ] **Step 7: Run the test to verify it passes**

```bash
cd slangdump-poster && npx vitest run src/components/__tests__/FeedItem.test.tsx
```

Expected: PASS, 7 tests.

- [ ] **Step 8: Type-check and lint**

```bash
cd slangdump-poster && npx tsc --noEmit && npm run lint
```

Expected: no errors. If `tsc` still reports `llmProvider` or `llmModel`, a reference was missed — search with `grep -rn "llmProvider\|llmModel" src/`.

- [ ] **Step 9: Commit**

```bash
cd slangdump-poster
git add src/components/FeedItem.tsx src/components/__tests__/FeedItem.test.tsx
git commit -m "feat: replace FeedItem placeholders with real likes, AI notice, disabled comments"
```

---

## Task 9: Remove the no-op buttons and make the nav a mobile drawer

**Repo:** `slangdump-poster`

**Files:**
- Modify: `src/components/SideNavBar.tsx`
- Modify: `src/app/page.tsx`
- Modify: `src/hooks/useAppState.ts`
- Test: `src/components/__tests__/SideNavBar.test.tsx`
- Test: `src/hooks/__tests__/useAppState.test.tsx`

**Interfaces:**
- Consumes: nothing new.
- Produces: `SideNavBarProps` becomes `{ isOpen: boolean; onClose?: () => void }`. `useAppState` returns the same keys as before.

- [ ] **Step 1: Write the failing SideNavBar test**

Append to `src/components/__tests__/SideNavBar.test.tsx`:

```tsx
describe('SideNavBar drawer behaviour', () => {
  beforeEach(() => {
    useSession.mockReset();
    useSession.mockReturnValue({ data: null, status: 'unauthenticated' });
  });

  it('no longer renders the dead POST LYRIC button', () => {
    render(<SideNavBar isOpen />);
    expect(screen.queryByText('POST LYRIC')).not.toBeInTheDocument();
  });

  it('slides off-screen when closed and back in when open', () => {
    const closed = render(<SideNavBar isOpen={false} />).container.querySelector('aside')!;
    expect(closed.className).toContain('-translate-x-full');

    // render() builds a fresh container each call, so this is that container's only aside.
    const open = render(<SideNavBar isOpen />).container.querySelector('aside')!;
    expect(open.className).toContain('translate-x-0');
    expect(open.className).not.toContain('-translate-x-full');
  });

  it('calls onClose when a nav item is chosen', async () => {
    const onClose = vi.fn();
    render(<SideNavBar isOpen onClose={onClose} />);

    await userEvent.click(screen.getByText('DAILY DROPS'));

    expect(onClose).toHaveBeenCalledTimes(1);
  });
});
```

Add to the imports at the top of that file:

```tsx
import userEvent from '@testing-library/user-event';
```

- [ ] **Step 2: Write the failing useAppState test**

Create `src/hooks/__tests__/useAppState.test.tsx`:

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { renderHook, waitFor } from '@testing-library/react';
import { useAppState } from '../useAppState';

// matchMedia is not implemented in jsdom; stub it per test with a fixed match result.
function stubMatchMedia(matches: boolean) {
  vi.stubGlobal('matchMedia', vi.fn().mockReturnValue({
    matches,
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
  }));
}

describe('useAppState sidebar default', () => {
  beforeEach(() => {
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue(
      new Response(JSON.stringify({ items: [], nextCursor: null }), { status: 200 })
    ));
  });

  it('closes the sidebar on a narrow viewport', async () => {
    stubMatchMedia(false);
    const { result } = renderHook(() => useAppState());
    await waitFor(() => expect(result.current.sidebarOpen).toBe(false));
  });

  it('leaves the sidebar open on a wide viewport', async () => {
    stubMatchMedia(true);
    const { result } = renderHook(() => useAppState());
    await waitFor(() => expect(result.current.sidebarOpen).toBe(true));
  });
});
```

- [ ] **Step 3: Run both to confirm they fail**

```bash
cd slangdump-poster && npx vitest run src/components/__tests__/SideNavBar.test.tsx src/hooks/__tests__/useAppState.test.tsx
```

Expected: FAIL — `POST LYRIC` still present, no `-translate-x-full` class, `onClose` not a prop, and the narrow-viewport case reports `true`.

- [ ] **Step 4: Update `SideNavBar`**

In `src/components/SideNavBar.tsx`, change the props interface:

```tsx
export interface SideNavBarProps {
  isOpen: boolean;
  /** Called when a nav item is chosen, so the mobile drawer can dismiss itself. */
  onClose?: () => void;
}

export const SideNavBar: React.FC<SideNavBarProps> = ({ isOpen, onClose }) => {
```

Replace the `<aside>` opening tag. Width is now constant and visibility comes from a
transform, which behaves identically at every width — the sidebar slides out when closed
and in when open. No `md:` variant belongs here: the breakpoint only changes whether the
open sidebar *pushes* the feed (`main`'s margin) or floats over it, and that is decided in
`page.tsx`.

```tsx
    <aside
      className={`fixed left-0 top-16 h-[calc(100vh-64px)] w-64 bg-zinc-950 border-r-2 border-zinc-900 transition-transform duration-300 z-40 ${
        isOpen ? 'translate-x-0' : '-translate-x-full'
      }`}
    >
```

Add the close handler to each nav item — change the nav item `<div>` to include:

```tsx
              onClick={onClose}
```

Delete the entire trailing button block:

```tsx
        <div className="px-6 mt-auto">
          <button className="w-full py-4 border-2 border-zinc-800 hover:border-orange-600 hover:text-orange-600 text-white font-black italic tracking-tighter transition-all uppercase text-sm">
            POST LYRIC
          </button>
        </div>
```

- [ ] **Step 5: Update `useAppState`**

In `src/hooks/useAppState.ts`, add this effect immediately after the `toggleSidebar` callback:

```ts
  // The sidebar starts open, which is right for desktop. On a narrow viewport it must
  // start closed — a 256px push on a 375px screen leaves no room for the feed. Layout
  // itself is driven by Tailwind `md:` classes, so this only settles the drawer state and
  // can never cause a reflow. Crossing the breakpoint resets to that mode's default.
  useEffect(() => {
    const mq = window.matchMedia('(min-width: 768px)');
    const apply = () => setSidebarOpen(mq.matches);
    apply();
    mq.addEventListener('change', apply);
    return () => mq.removeEventListener('change', apply);
  }, []);
```

- [ ] **Step 6: Update `page.tsx`**

In `src/app/page.tsx`, replace the sidebar + main block:

```tsx
        <div className="pt-16 flex">
          <SideNavBar isOpen={sidebarOpen} onClose={toggleSidebar} />

          {/* Mobile-only scrim. Below md the drawer floats over the feed, so it needs a
              dismiss target; above md the sidebar shares the layout and none is wanted. */}
          {sidebarOpen && (
            <div
              aria-hidden
              onClick={toggleSidebar}
              className="md:hidden fixed inset-0 top-16 bg-black/60 z-30"
            />
          )}

          {/* Never pushed below md — the drawer overlays instead. */}
          <main className={`flex-1 transition-all duration-300 ${sidebarOpen ? 'md:ml-64' : 'ml-0'}`}>
```

And delete the floating action button entirely:

```tsx
        {/* Floating Action Button */}
        <button className="fixed bottom-8 right-8 w-16 h-16 bg-orange-600 text-black flex items-center justify-center shadow-[0_8px_0_0_rgba(0,0,0,1)] hover:translate-y-[-4px] active:translate-y-[2px] active:shadow-none transition-all z-50">
          <span className="material-symbols-outlined text-3xl font-bold">add</span>
        </button>
```

Both removed buttons were no-ops. `DailyDropEditor`, already at the top of the feed, is the
only real posting entry point.

- [ ] **Step 7: Run the tests to verify they pass**

```bash
cd slangdump-poster && npx vitest run src/components/__tests__/SideNavBar.test.tsx src/hooks/__tests__/useAppState.test.tsx
```

Expected: PASS, 5 + 2 tests.

- [ ] **Step 8: Run the full suite, type-check, and lint**

```bash
cd slangdump-poster && npm test && npx tsc --noEmit && npm run lint
```

Expected: all green.

- [ ] **Step 9: Manually verify the drawer**

```bash
cd slangdump-poster && npm run dev
```

Open `http://localhost:3000` and confirm: at desktop width the sidebar is open and the feed is pushed right; narrowing below 768px closes it; the hamburger opens it over the feed with a dark scrim; tapping the scrim or a nav item closes it; the page body never scrolls horizontally; and there is no floating `+` button in any corner.

- [ ] **Step 10: Commit**

```bash
cd slangdump-poster
git add src/components/SideNavBar.tsx src/app/page.tsx src/hooks/useAppState.ts \
        src/components/__tests__/SideNavBar.test.tsx src/hooks/__tests__/useAppState.test.tsx
git commit -m "feat: mobile overlay drawer for the left nav; remove dead add buttons"
```

---

## Final Verification

- [ ] Backend: `cd slangdump-pick-n-roll && ./gradlew test` — all green.
- [ ] Frontend: `cd slangdump-poster && npm test && npx tsc --noEmit && npm run lint` — all green.
- [ ] `grep -rn "llmProvider\|llmModel" slangdump-poster/src slangdump-pick-n-roll/src/main/kotlin/com/example/demo/dto slangdump-pick-n-roll/src/main/kotlin/com/example/demo/controller` returns nothing.
- [ ] `grep -rn "2.4K\|VERIFIED\|+20" slangdump-poster/src/components/FeedItem.tsx` returns nothing.
- [ ] Full stack up (`docker compose` per the root README), sign in, like a post, reload — the count persists and the heart stays filled.
- [ ] Sign out, reload — the count is still visible and the heart is disabled with a "Sign in to like" tooltip.
