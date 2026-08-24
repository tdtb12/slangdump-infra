# Slangdump

A social feed where each user drops one song a day. The drop carries an
AI-generated Traditional Chinese translation of the lyrics, with phrase-level
alignment and cultural annotations. Translations are shared: many drops of the
same song reuse one translation.

## Language

### Content

**Song**:
The unit of shared translation — one track, translated once and reused by every
Post that references it. Identified by its Credited Artists, its song name, and
the Translator that produced it. The Credited Artists count as a *set*: the same
collaboration billed in either order is one Song. Neither the album, the Featured
Artists, nor the lyrics take part.
_Avoid_: Track, Translation

**Artist**:
A musical act, and the owner of its own Artist Background. One record per act,
shared by every Song it performs on and every Album filed under it.
An act billed as a unit is **one** Artist, not several: Silk Sonic is one Artist;
Bruno Mars and Anderson .Paak billed jointly are two. Nothing can enforce that
distinction — it is a fact about the release, known only to the author — and
getting it wrong forks the translation.
_Avoid_: Singer, Band, Performer (which names a role, not an act)

**Credited Artist**:
An Artist the Song is billed to, before any `feat.`. At least one, and the first
is the **Lead** — the Artist the Album is filed under. Each Credited Artist has
their Artist Background shown on the drop.
_Avoid_: Primary artist, Co-artist (both name a position, not the concept)

**Featured Artist**:
A guest on a Song. An attribute of the Song, like the Album — never part of its
identity, and never given an Artist Background section on that drop. Their part
in the track belongs in the Song Meaning. The same Artist may be Credited on one
Song and Featured on another.
_Avoid_: Feature (the noun for the role), Guest artist

**Performer**:
The link between a Song and an Artist, carrying the role (Credited or Featured)
and the position within that role. Where billing order lives — a Song's identity
deliberately ignores it, but the display does not.
_Avoid_: Credit, Artist role

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
they were made with, permanently. (An Artist Background is the one exception to
that permanence, and it belongs to the Artist rather than to the Song.)
_Avoid_: Default model, Current model

### Generated prose

Each is written by the Translator and shown as its own section of a drop.
Their boundaries are load-bearing: prose that strays across them is wrong, not
merely untidy.

**Artist Background**:
Prose about an Artist — who they are, where they come from, what they are known
for. Belongs to the **Artist**, not to any Song, so it must not describe an
individual track or release, and one drop shows one section per Credited Artist.

The only prose here that is **rewritten**: it is refreshed when it is older than
six months or when a new Album is discovered for that Artist, and the rewrite
changes every Post that Artist appears on, including published permalinks. An
Album Context and a Song Meaning, once written, stand forever.
_Avoid_: Artist bio, artist info

**Album Context**:
Prose about an Album as a whole — its place in the artist's discography, its
themes, sound and reception. Scoped to the **Lead** Credited Artist's
discography. Written once per Album and shown on **every** Song belonging to it,
so it must never describe an individual track.
_Avoid_: Album background, Album notes

**Song Meaning**:
Prose about one Song — its themes, narrative and impact. The correct home for
anything track-specific, including a track's role on its album and what a
Featured Artist contributes to it.
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
