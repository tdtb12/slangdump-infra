# Likes API and Feed Cleanup

**Date:** 2026-07-21
**Repos touched:** `slangdump-pick-n-roll` (Spring), `slangdump-poster` (Next.js)

## Summary

Four changes to the feed. One is a real feature — post likes, which need a new
table, endpoints, and auth rules. The other three remove hard-coded placeholder
UI and stop leaking which LLM produced a translation.

Comments are explicitly **not** in scope. The comment button ships disabled with
a tooltip saying so.

---

## 1. Likes

### Model

Authenticated users only, one like per user per post. Signed-out visitors see
the count but cannot like.

New migration `V5__post_likes.sql` in `slangdump-pick-n-roll`:

```sql
CREATE TABLE post_likes (
  id         BIGSERIAL PRIMARY KEY,
  post_id    BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id    BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  UNIQUE (post_id, user_id)
);
CREATE INDEX idx_post_likes_user ON post_likes(user_id);
```

`UNIQUE (post_id, user_id)` is the dedup rule and also serves count-by-post
lookups. There is no denormalized counter column: counts are always derived, so
they cannot drift from the rows.

Per the project's Flyway rule, this migration runs as a distinct deployment step
against the **direct** Neon connection, never at application startup.

### Reading counts without N+1

The cursor paging queries in `PostRepository.kt` (`findFirstPage`,
`findPageAfter`) are **not** modified. Their keyset logic is subtle and gains
nothing from an aggregate join.

Instead a new `PostLikeService` answers two batched questions per page, a fixed
cost regardless of page size:

- counts: `SELECT l.post.id, count(l) FROM PostLike l WHERE l.post.id IN :ids GROUP BY l.post.id` -> `Map<Long, Long>`
- viewer's likes: `SELECT l.post.id FROM PostLike l WHERE l.user.id = :me AND l.post.id IN :ids` -> `Set<Long>`

The second query is skipped entirely when no user is resolved. Posts absent from
the count map have zero likes.

`Post.toSummary()` in `PostController.kt` takes the resolved `(likeCount,
likedByMe)` pair for that post rather than deriving it, so the list handler looks
each post up in the batch results once. `Post.toDetail()` — a single-post path —
takes the same pair, resolved by two scalar queries (`countByPostId`,
`existsByPostIdAndUserId`) instead of the batch form.

### Endpoints

| Method | Path | Auth | Behavior |
|---|---|---|---|
| `POST` | `/api/v1/posts/{id}/like` | required | Idempotent. Inserts if absent. |
| `DELETE` | `/api/v1/posts/{id}/like` | required | Idempotent. Deletes if present. |

Both return `LikeStateDto(likeCount: Long, likedByMe: Boolean)` so the client
reconciles against the server rather than trusting its own optimistic guess.
Both return 404 if the post does not exist.

A concurrent-insert race surfaces as a unique-constraint violation; that is
caught and treated as success, since the user's intent (be in the liked set) is
already satisfied.

### Security

`SecurityConfig.kt` currently reads:

```kotlin
it.requestMatchers(HttpMethod.GET, "/api/v1/posts", "/api/v1/posts/**").permitAll()
it.requestMatchers(HttpMethod.POST, "/api/v1/posts").authenticated()
...
it.anyRequest().permitAll()
```

The existing wildcard is `GET`-scoped and the `POST` rule matches the exact
collection path only, so **without new rules the like endpoints would fall
through to `anyRequest().permitAll()` and be publicly writable.** Add:

```kotlin
it.requestMatchers(HttpMethod.POST, "/api/v1/posts/*/like").authenticated()
it.requestMatchers(HttpMethod.DELETE, "/api/v1/posts/*/like").authenticated()
```

These must be registered **before** the existing `/api/v1/posts/**` permitAll
line, since Spring Security applies the first matching rule.

`GET /api/v1/posts` and `GET /api/v1/posts/{id}` stay public and take a nullable
`@AuthenticationPrincipal jwt: Jwt?`. A present token resolves `likedByMe`; an
absent or unresolvable one yields `false`. Read paths look up an existing user
only and never create one — user creation stays on the `/api/v1/users/sync` and
post-create paths.

### DTO contract

`PostSummaryDto` and `PostDetailDto` each gain:

```kotlin
val likeCount: Long,
val likedByMe: Boolean,
```

Both gain the fields, not just the summary, because `api.ts` declares
`PostDetailDto extends PostSummaryDto` and that relationship should stay true.
The mirrored TypeScript interfaces in `src/types/api.ts` are updated to match.

A newly created post returns `likeCount: 0, likedByMe: false`.

### BFF (`slangdump-poster`)

New route `src/app/api/posts/[id]/like/route.ts` exporting `POST` and `DELETE`.
Each mints a token via `mintBffToken` using the server session exactly as
`src/app/api/posts/route.ts` does, and returns `401 { code: UNAUTHORIZED }`
without a session rather than making a doomed round-trip.

`GET` in `src/app/api/posts/route.ts` gains **optional** token forwarding: if a
session exists, mint and attach `Authorization`; otherwise send the request
unchanged. Without this, `likedByMe` is always `false` even when signed in.

`src/app/api/posts/[id]/route.ts` is left alone apart from inheriting the new
DTO fields — it already forwards `Retry-After` and `Cache-Control` verbatim, and
that polling contract must not change.

### UI

A new `src/components/PostLikeButton.tsx` owns the entire interaction:

- props: `postId`, `likeCount`, `likedByMe`
- optimistic toggle on click, with an in-flight guard so double-clicks cannot
  fire two requests
- reconciles to the `LikeStateDto` in the response; on error, reverts to the
  pre-click state and leaves the count untouched
- when signed out: rendered disabled with a "Sign in to like" tooltip

`FeedItem` passes props and renders it. Network and optimistic state stay out of
`FeedItem`, which is already 393 lines, and the button becomes testable alone.

---

## 2. LLM identity removed from the API

Users must never learn which model ran. Remove:

- `llmProvider` / `llmModel` from `PostDetailDto.kt`
- both assignments from `Post.toDetail()` in `PostController.kt`
- both fields from `PostDetailDto` in `src/types/api.ts`
- the conditional render block in `FeedItem.tsx` that prints
  "Translated by {provider} / {model}"

The `LlmModel` entity, the `llm_models` table, and `songs.llm_model_id` all stay
exactly as they are. `llm_model_id` participates in the dedup unique key
`(artist_norm, song_name_norm, album_norm, llm_model_id)`. This is an
API-surface change only; no migration.

In place of the removed block, `FeedItem` renders an unconditional notice
whenever a translation is shown:

> AI-generated translations may contain errors, omissions, or debatable
> readings. Verify anything you rely on.

The wording deliberately echoes `src/lib/disclaimer-content.ts` (sections 4 and
6) so the app states one consistent thing. It is styled as a bordered zinc note,
not a yellow/red warning box — it is permanent, so it should not read as an
alarm.

---

## 3. FeedItem hard-coded placeholders

Delete the placeholder constants `likesCount`, `commentsCount`, `interactions`,
and `additional`, and delete the `VERIFIED` badge span from the header.

`timeLoc` and `tagsList` are also cosmetic placeholders but are **out of scope**
and stay as-is.

The interactions footer is kept as the interaction bar, with its contents
replaced:

- left: `PostLikeButton` with the real count
- beside it: the comment button, `disabled` and `aria-disabled`, with
  `cursor-not-allowed`, no count, and the tooltip **"Comments arrive in the next
  phase."** — implemented as both a native `title` attribute and a styled hover
  bubble, so it works on hover and for assistive tech
- removed: the dicebear avatar stack and the `+20` bubble

---

## 4. Dead no-op buttons

Both are removed:

- the floating `add` FAB in `src/app/page.tsx` (`fixed bottom-8 right-8`)
- the `POST LYRIC` button pinned to the bottom of `SideNavBar.tsx`

Neither had an `onClick`. `DailyDropEditor`, already at the top of the feed, is
the only real posting entry point.

`src/components/LeftBar.tsx` and `src/components/TopBar.tsx` are dead code
imported by nothing. They are **out of scope** and left in place.

---

## 5. Mobile navigation

The sidebar is `fixed w-64` and `main` is pushed by `ml-64` at every viewport
width, which squeezes the feed badly on a phone. Below the `md` breakpoint it
becomes an overlay drawer.

The breakpoint is expressed in Tailwind classes, not in JavaScript, so server
and client render identically and there is no hydration mismatch:

- `SideNavBar` is always `w-64`. Visibility comes from a transform:
  `-translate-x-full` when closed, `translate-x-0` when open, plus
  `md:translate-x-0`. Existing desktop collapse behavior is preserved via the
  width toggle.
- `main` uses `md:ml-64` when open and is never pushed below `md`, so the drawer
  floats over the feed.
- A backdrop (`md:hidden fixed inset-0 bg-black/60 z-30`) renders only while the
  drawer is open on mobile. Tapping it closes the drawer.
- Clicking a nav item closes the drawer on mobile.

`useAppState` keeps `sidebarOpen = true` as its initial value, which is correct
for desktop, and a mount effect sets it to `false` when
`matchMedia('(min-width: 768px)')` does not match. Because layout is driven by
`md:` classes rather than that boolean, the worst case on mobile is one brief
drawer slide on first paint — never a reflow.

---

## Testing

Updated:

- `slangdump-poster/src/components/__tests__/FeedItem.test.tsx` — placeholders
  gone, LLM line gone, AI notice present, comment button disabled
- `slangdump-poster/src/components/__tests__/SideNavBar.test.tsx` — POST LYRIC
  button removed, drawer classes
- `slangdump-poster/src/app/api/posts/__tests__/route.test.ts` — optional token
  forwarding on the list GET
- `slangdump-pick-n-roll/.../PostControllerIntegrationTest.kt` — detail response
  no longer carries LLM fields; summary and detail carry like fields

New:

- `PostLikeButton` component tests: optimistic toggle, revert on error,
  double-click guard, signed-out disabled state
- like endpoint integration tests: idempotent like and unlike, count accuracy,
  `likedByMe` per viewer, 404 on missing post
- security tests: unauthenticated `POST` and `DELETE` on
  `/api/v1/posts/{id}/like` return 401 — this guards the `anyRequest()`
  fall-through described above
- BFF like route tests: 401 without session, token attached with session

## Out of scope

Comments (next phase), `timeLoc` and `tagsList` placeholders, deleting
`LeftBar.tsx` / `TopBar.tsx`, notification and settings buttons in `TopNavBar`,
and any change to the polling contract.
