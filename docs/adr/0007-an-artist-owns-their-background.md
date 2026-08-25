# An Artist owns their background, refreshed rather than rewritten

`artist_background` moves off `songs` onto a new `artists` table keyed on
`name_norm`. One row per artist, one background, shared by every Song they are
credited on. This is the move ADR-0002 made for the Album, with one deliberate
difference recorded below.

Per-song was doubly wrong. Every Drake post paid Gemini to research Drake again,
and the resulting paragraphs disagreed with each other — the reader saw a
different Drake on every card.

The Album table was already an artist string in disguise (`albums.artist_norm`),
so `albums` repoints to `artists.id` in the same migration. Artist names now live
in exactly one place.

## The difference from ADR-0002: a background goes stale

An album context is written once and never revisited — an album's themes, sound
and reception do not change. A person does. They release records, change names,
change stature, die.

So Spring asks the worker for a background only when it is **missing or stale**,
and sends the list as `backgroundNeededFor`. Stale means either:

- `background_updated_at` is older than **6 months**, or
- it is NULL, which `markStale` sets when a **new album** is discovered for that
  artist — the strongest available signal that their story has moved on.

The refresh happens on that artist's **next** translation, not immediately. The
prose for the current one was written in the same pass that identified the album,
so acting on it now would mean a second call to the model for a fact the reader
will not miss for one post.

## Where the money is

`backgroundNeededFor` is **empty in the steady state**, and the prompt then omits
the artist section entirely — no research, no 150-200 grounded words. This is the
first thing in the system that reduces the Gemini bill rather than merely
tidying the output, and ADR-0002's "this buys consistency, not money" does not
apply here.

When several new artists do appear at once the length is tapered: lead 150-200
words, each additional 80-100. A collaboration is not three full essays.

## Considered options

**Snapshot the background onto each Post.** Rejected, and this is the trade-off
that matters. It would keep every published Post exactly as it was written — but
it defeats the sharing entirely: fifty Drake posts means fifty snapshots, which
is fifty researched backgrounds, which is the cost this ADR exists to remove.

**Key the background by Translator, as songs are.** Rejected, for symmetry with
`albums.album_context`, whose key has no `llm_model_id` either. Prose about a
person is not model-specific in any way a reader would notice, and keying it
would multiply the research bill by the number of models.

**Never refresh.** Simplest, and quietly wrong within a year.

## Consequences

**A refresh rewrites already-published prose.** There is one row, so re-writing
Drake's background changes the Artist Background shown on every Post of his,
including server-rendered permalinks that are already indexed. CONTEXT.md
previously promised a Song keeps the prose it was made with permanently; that
promise is narrowed to the album context and the song meaning.

**Nothing stops a guest having a background.** Once the prose belongs to the
person, a featured artist can perfectly well have one — Drake is credited on his
own songs and featured on 「SICKO MODE」. "No background for a guest" is therefore
no longer expressible as a database constraint. It holds because two code paths
are both true, each covered by its own test:

- `backgroundNeededFor` is built from CREDITED performers only, and
- `PostController.toDetail` reads only `role = 'CREDITED'`.

**First writer wins inside the window.** Two songs by one stale artist
translating concurrently would both return prose; the write is a single UPDATE
with the staleness predicate in its WHERE clause, so the second is a no-op rather
than a clobber. A returned null or blank never overwrites prose already on file —
the worker answering "nothing found" must not erase a good background.

**The backfill starts a refresh wave.** V10 sets `background_updated_at` to the
`created_at` of the song the prose came from, so every artist whose background
predates the 6-month window is stale from the first boot and is re-researched on
their next post. Intended, and self-limiting: one background per artist, then
quiet for six months.
