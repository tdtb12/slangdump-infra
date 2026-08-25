# Slangdump

A social feed where each user drops one song a day. The drop carries an
AI-generated Traditional Chinese translation of the lyrics, with phrase-level
alignment and cultural annotations. Translations are shared: many drops of the
same song reuse one translation.

## Language

### Content

**Song**:
The unit of shared translation — one artist's one track, translated once and
reused by every Post that references it. Identified by artist and song name,
together with the Translator that produced it. Neither the album nor the lyrics
take part.
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

**Translator**:
The model that produced a Song's translation. Part of a Song's identity, so the
same track under two Translators is two Songs, each with its own prose and its
own Posts.
_Avoid_: Engine, Provider (that names only half of a Translator)

**Active Translator**:
The single Translator every new Song is made with. Exactly one exists at a time.
Changing it never rewrites an existing Song or Post — those keep the Translator
they were made with, permanently.
_Avoid_: Default model, Current model

### People

**User**:
One person's account, identified by their email address. Two sign-ins with
different providers but the same email are one User, not two.
_Avoid_: Account, Member, Profile

**Username**:
The public name a User is known by, written `@name`. Chosen by the User, unique
across the site, and the only name of theirs anyone else sees — the name their
provider knows them by is never shown. Changing it changes it everywhere at
once, including on drops made before the change.
_Avoid_: User id (that is the internal key), Handle, Display name, Screen name

**Author**:
A User in relation to a Post they dropped. Not a separate kind of person —
every User is the Author of their own drops and of nothing else.
_Avoid_: Poster, Owner, Creator

### Generated prose

Each is written by the Translator and shown as its own section of a drop.
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
