# A Song is identified by the SET of its credited artists

Supersedes ADR-0001, whose reasoning about the album is unchanged and still
applies.

A Song is billed to one or more **credited artists** and may carry any number of
**featured artists**. Only the credited ones are part of its identity. The key
stays `(artist_norm, song_name_norm, llm_model_id)`; `artist_norm` now holds the
credited names normalised individually, **sorted**, and joined with U+001F.

Sorting is what makes it a set. 「Ni**as in Paris」 typed as `JAY-Z, Kanye West`
and as `Kanye West, JAY-Z` is one song and one translation — two authors must not
pay for the same work twice because they disagreed about who to list first. The
order the author typed is kept separately, in `performers.position`, and is what
`songs.artist` renders for display.

U+001F rather than a comma or a slash: `Tyler, The Creator` contains one, and a
printable separator would let one artist's name forge a collision with a
different pair of artists. U+001F cannot be typed into a form.

A **featured artist** is an attribute of the song, like the album. `SICKO MODE`
featuring Drake is a Travis Scott song; if a guest were part of the key, the same
recording would translate twice as soon as one author listed the guest and
another did not. Guests are still sent to the Translator — the prompt uses them
to place a verse in the Song Meaning — they simply do not identify anything.

`albums` keys on the **lead** credited artist, `performers.position = 0` among
the credited ones, now as a real `albums.artist_id`. An album belongs to one act
even when a track on it does not.

## Considered options

**Key on the ordered list.** Rejected: it makes billing order load-bearing for
authors who have no reason to know it, and the failure is invisible — a second
translation of a song that is already translated, silently billed.

**Key on the lead artist only, with co-artists as attributes.** Simpler, and
wrong: `Watch the Throne` is not a JAY-Z album with Kanye West attached, and a
co-artist earning no Artist Background section was the complaint that started
this work.

**Include featured artists in the key.** Rejected above. It also makes the key
depend on how thorough the author was, which is not a property of the song.

## Consequences

**One-element joins are identical to the old key**, which is what let V10 leave
every existing `artist_norm` untouched. `SongIdentityTest` pins that.

**A song billed to a group is one credited artist, not several.** Silk Sonic is
an act; Bruno Mars and Anderson .Paak billed jointly are two. Nothing enforces
the distinction and nothing can — it is a fact about the release, and the author
is the only one who knows it. Getting it wrong forks the translation.

**`artist_norm` is now up to four names wide**, so the column moved to
`VARCHAR(1024)`. The worst-case btree index row is ~1287 bytes, well under the
limit, and `MAX_CREDITED_ARTISTS = 4` is what keeps it bounded.

**Search indexes credited and featured names alike.** A guest is not part of the
identity but is very much part of why someone searches for the track.
