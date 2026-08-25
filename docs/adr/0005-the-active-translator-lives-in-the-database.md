# The Active Translator lives in the database and is pushed to the worker

One row of `llm_models` carries `enabled = true`, and that row is the Active
Translator. Spring reads it when a Post is created, keys the Song on it, and
sends `provider` and `model` in the `/api/v1/process` body. The worker looks the
model up in its own `MODEL_CONFIGS` registry, dials exactly that model, and
echoes back the one it resolved. Switching is a Flyway migration run as
`release_command`.

Two things decided the Translator before this, and nothing reconciled them.
Spring read `ai.model.provider` / `ai.model.model` from configuration and wrote
the result onto `songs.llm_model_id` — a **label** recording which model should
have run. The worker hardcoded `MODEL = "gemini-2.5-flash"`, which is what
actually ran. When they disagreed the persist step logged a warning and kept the
label, so a Song could be permanently keyed to a Translator that never touched
it. Because a Song is a shared cache, that wrong key was then reused by every
later Post of the same track. Making the database decide and having the worker
obey is what collapses the two into one.

The `enabled` flag is not merely a label either: **at most one** is enforced by
a partial unique index, and **at least one** by a `RAISE EXCEPTION` guard in
each switch migration. Flyway wraps a migration in a transaction, so a guard
that fires rolls the migration back and fails the deploy, rather than leaving
production with no Translator and surfacing as a 500 on a user's Post.

## Considered options

**Have the worker keep hardcoding it and drop the column.** Simplest, and it
would have removed the drift, but it puts the choice inside a Python constant —
so a Song could no longer record which Translator made it, and the identity in
ADR-0001 loses a third of its key. Switching would also become a worker deploy
rather than a reviewable one-line change.

**Point both at one environment variable.** Removes the disagreement without
removing the second source of truth: the value is still not recorded per Song,
so the archive cannot say what produced an old translation. It also means the
Translator can change without any migration, which is exactly the audit trail
this is for.

**An admin endpoint to flip the flag.** Rejected on cost, not on principle.
There is no admin surface and no role model — `SecurityConfig` has no `hasRole`
anywhere — so a one-row `UPDATE` would pull in a whole authorisation feature.
A migration is already reviewed, already audited in git, and already fails the
deploy when it is wrong.

**Make `model` required on the wire.** Deferred. It would need a fourth deploy
during the rollout for no benefit, so the field is optional with a warning log
on the fallback path. The warning is the signal for when it can be tightened.

## Consequences

**Switching re-translates the catalogue, gradually, and that is the point.** A
Song is keyed on `(artist_norm, song_name_norm, llm_model_id)`, so enabling a
different Translator makes every cache lookup miss: the next Post of a song
creates a second `songs` row and pays for a fresh translation. Nothing is
migrated or rewritten. Existing Posts keep rendering their original translation
because `posts.song_id` is single-valued, and search dedupes hits by post, so
the two rows never appear side by side. Songs stuck permanently `FAILED` get a
fresh chance for free, since they are behind the same key.

**Old rows are never cleaned up.** They are the archive backing every Post made
before the switch. A Translator row can be disabled but must not be deleted.

**Rollback is one migration and no worker deploy** — provided the worker keeps a
registry entry for the previous model. Dropping an entry the moment it stops
being active would trade that away, and would also strand any Song still
`PENDING` at the moment of a switch, since those complete under their own
Translator rather than the active one.

**The rollout has to be worker-first, with no overlap.** Pydantic ignores fields
it does not know, so a new Spring against an old worker would key Songs to the
new Translator while the worker quietly ran the old one — reintroducing the
original bug silently and permanently, on real users' Songs. Worker first, then
Spring, then the flip.
