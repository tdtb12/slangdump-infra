# A Song is identified by artist and song name alone

> **Superseded by ADR-0006.** A Song is billed to a *set* of credited artists,
> not one, and `artist_norm` holds them sorted and joined. Everything below about
> why the album and the lyrics stay out of the key is unchanged and still the
> reasoning in force.

A Song is the unit of shared translation, keyed on
`(artist_norm, song_name_norm, llm_model_id)`. Neither the album nor the lyrics
participate. The first user to post a given artist + song name causes the
translation; every later post of that name reuses it verbatim, whatever they
pasted.

Album was originally in the key. It is a bad discriminator in both directions:
the same recording on a studio album and on a greatest-hits compilation is one
translation billed twice, while a radio edit and the explicit cut share an album
and are *not* the same translation. Album is also optional user input, so a
missing album was a participating `''` value rather than a wildcard — meaning
resolving an album afterwards would mutate a row's own identity and could
collide with an existing row. Removing album from the key is what makes the
write-back in ADR-0002 a plain column update.

## Considered options

**Key on a hash of the lyrics.** This was chosen and then reversed. It is the
only option that separates renditions automatically and merges compilation
reissues, but the cache then only hits on byte-identical pastes. Genius scrape
chrome — contributor counts, embed markers, ticketing promos — drifts between
scrapes of the same song, so the hash would have to be taken over noise-stripped
text, which forced the stripping in `_clean_lyrics` to move from Python into
Spring (the cache is checked before the worker is called). A stricter cache than
today's, plus a normalisation contract spanning another language, for a problem
a naming convention solves.

**Key on artist + song name + album.** The status quo. Rejected above.

## Consequences

Renditions are distinguished only by convention: the user writes the variant
into the song name, `HUMBLE.` versus `HUMBLE. (RADIO EDIT)`. Nothing enforces
this and the editor only carries a static hint, so expect it to be missed.

When it is missed the failure is quiet but bounded. Lyrics are never stored; a
viewer pastes their own and lines are matched by `sourceHash`, with unmatched
lines dropped and an existing message shown. So a clean-version drop attached to
the explicit translation loses its censored lines and keeps the rest. The author
gets no signal at all — their `rawLyrics` is discarded uncompared — and the
one-drop-per-Taipei-day limit means a mistake costs them the day.
