# Slangdump

A social feed where each user drops one song a day. The drop carries an
AI-generated Traditional Chinese translation of the lyrics, with phrase-level
alignment and cultural annotations. Translations are shared: many drops of the
same song reuse one translation.

## Language

### Content

**Song**:
The unit of shared translation — one artist's one track, translated once and
reused by every Post that references it. Identified by artist and song name
only.
_Avoid_: Track, Translation

**Post**:
One user's drop of a Song on a given day: the social wrapper carrying author,
description and likes. Never carries a translation of its own.
_Avoid_: Drop (in code), Entry

**Rendition**:
A distinct recorded version of the same song — radio edit, clean version, live
cut, remix. Deliberately **not** modelled: renditions are distinguished only by
the user writing the variant into the song name, e.g. `HUMBLE.` versus
`HUMBLE. (RADIO EDIT)`. Two users who disagree share one translation, and the
first to post decides it.
_Avoid_: Version, Edit, Variant

**Album**:
The release a Song appears on, and the owner of that release's shared context.
Never part of a Song's identity — the same song on a studio album and on a
greatest-hits compilation is one Song. Optional when a Post is written; when
absent it may be resolved during translation. A Song with no Album is a
standalone single.
_Avoid_: Record, Release, Project

### Generated prose

Each is written by the translator and shown as its own section of a drop.
Their boundaries are load-bearing: prose that strays across them is wrong, not
merely untidy.

**Artist Background**:
Prose about the artist behind a Song — who they are, where they come from, what
they are known for. Belongs to the Song, so it must not describe the individual
track or the release.
_Avoid_: Artist bio, artist info

**Album Context**:
Prose about an Album as a whole — its place in the artist's discography, its
themes, sound and reception. Written once per Album and shown on **every** Song
belonging to it, so it must never describe an individual track.
_Avoid_: Album background, Album notes

**Song Meaning**:
Prose about one Song — its themes, narrative and impact. The correct home for
anything track-specific, including a track's role on its album.
_Avoid_: Song analysis, Interpretation

### Lyric forms

These three are distinct and must not be used interchangeably.

**Raw Lyrics**:
What the user pasted, including scrape chrome — contributor counts, embed
markers, ticketing promos, section headers. Never persisted (copyright).
_Avoid_: Lyrics (unqualified)

**Cleaned Lyrics**:
Raw Lyrics with scrape chrome removed, leaving only sung text. What the
translator actually reads.
_Avoid_: Clean lyrics — collides with "clean version", which is a Rendition

**Canonical Line**:
One lyric line reduced to its match key: NFKC-normalised, trimmed, internal
whitespace collapsed, case and punctuation preserved. Alignment offsets index
it and its hash identifies it, which is how a stored translation is matched to
a viewer's own paste without ever storing the source text.
_Avoid_: Normalized line
