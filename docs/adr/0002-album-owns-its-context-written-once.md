# The Album owns its context, written once

`album_context` moves off `songs` onto a new `albums` table keyed on
`(artist_norm, album_norm)`. The first song translated for an album stores the
essay; every later song of that album links to the same row and its freshly
generated essay is discarded. Songs also stop carrying `album`/`album_norm` —
the album is read through `album_id`.

The album itself may be absent at post time. The translator resolves it and
returns it in `meta_info.album_name`, which is written back **only when the user
left the field blank** — a typed album is never overwritten, and it also went
into the prompt, so the returned context describes that album and the row stays
self-consistent. `album_name` is nullable so a genuine standalone single returns
nothing rather than inventing a release.

## Consequences

**This buys consistency, not money.** The essay comes from the same single
`generate_content` call that produces the translation, so the model writes one
per song regardless. Discarding the redundant ones stops two songs off one album
showing different prose; it does not reduce the Gemini bill. Cutting tokens
would mean splitting `meta_info` into its own call so a known album can skip it
— which the MAX_TOKENS handler in `llm_service.py` already recommends for
unrelated reasons. Deferred: two calls and more latency during a ship push, and
the split can be added later against this schema without rework.

**Shared prose makes track-specific content a defect.** Per-song, an essay
ending "〈the feeling〉is the album's second single" was untidy. Shared across
every song of the album, it is wrong on all of them but one, permanently, since
the first writer wins. The prompt's field description previously read "the album
and its specific role within the project", whose ambiguous "its" the model read
as the song's role. It is now explicit that the text appears on every song of
the album, with a worked ✗/✓ pair.

**A song with no album shows no album context.** `album_id` is null, so the
section does not render and the prose the model wrote for a single is thrown
away. Accepted over keeping a fallback column on `songs`: one home for the
concept, no precedence rule at read sites.
