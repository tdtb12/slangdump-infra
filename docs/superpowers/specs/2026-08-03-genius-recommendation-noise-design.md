# Stripping Genius Recommendation Blocks From Pasted Lyrics

**Date:** 2026-08-03 (revised 2026-08-04)
**Repos touched:** `slangdump-ai-solation` (Python worker) only

## Summary

Genius injects a "You might also like" recommendation widget *inside* the lyrics
container. When a user selects and copies the lyrics, the widget's plain text
lands in the middle of the paste — a marker line followed by alternating song
title and artist name, one pair per recommendation.

`_clean_lyrics` in `services/llm_service.py` already deletes the marker line
(`_LYRIC_NOISE_PATTERNS`, the `^\s*You might also like.*$` entry). It does not
delete the lines after it, and no literal pattern can — a recommended song title
is textually indistinguishable from a lyric line. Those lines are sent to Gemini,
translated, stored, and rendered as if they were part of the song.

The fix is a structural rule anchored on the marker: delete the marker and the
next six non-blank lines.

## Revision: why this is not the singer-anchored rule

The first version of this design used `req.singer` — the artist the user typed —
to decide where the block ends. Test every artist slot, consume through the last
one that names the singer, consume nothing if none do. The reasoning was that a
lyric line does not name the singer, so the boundary could never run past the
widget into the song.

**A real paste falsified it.** Searching for *Christina Perri — A Thousand Years*
produced this block:

```
You might also like
"Slut!" (Taylor's Version) [From the Vault]
Taylor Swift
Hate You
Jung Kook (정국)
Now That We Don't Talk (Taylor's Version) [From the Vault]
Taylor Swift
```

Not one of the three recommendations is by Christina Perri. Genius recommends
from a global pool, not from the song's own artist, so on a typical page **no
slot matches the singer and the rule consumes nothing** — it left every noise
line in place. The earlier sample where all three recommendations were by the
song's own artist (Lil Uzi Vert) was the exception, not the pattern.

The same finding removes the objection that produced the singer rule. A fixed
six-line delete was rejected because Genius might render fewer than three
recommendations and the delete would run into the song. That risk belongs to a
*per-artist* recommender, which can run out of material. A global one always has
three items to show. Three independent samples carry three pairs each, and the
markup agrees: `RecommendedSongs__Body` renders three `<a>` elements.

So the singer machinery — `_fold_artist`, `_names_the_singer`, the parenthetical
stripping for `Jung Kook (정국)`, the short-name floor for `IU` — is deleted
rather than kept alongside the new rule. It is a disproven heuristic; leaving it
in would only be complexity that never fires.

## Why the fix belongs in the worker, and nowhere else

**The frontend does not need to change.** `FeedItem.tsx` renders `detail.lines`
and drops any line whose `sourceHash` does not match one of the viewer's pasted
lines (`flatMap` returning `[]`). It never renders the raw paste directly. So a
line the worker declines to return is automatically absent from the display.
Cleaning in `_clean_lyrics` propagates to rendering for free.

Spring needs no change either; it forwards the paste verbatim.

## The rule

`_strip_recommendation_blocks(lyrics)` runs **before** the existing
`_LYRIC_NOISE_PATTERNS` loop. The order is load-bearing: the existing marker
pattern deletes the only anchor the structural rule has.

```
req.lyrics
  -> _strip_recommendation_blocks(lyrics)   # new: marker + six lines
  -> _LYRIC_NOISE_PATTERNS loop             # existing: remaining chrome
  -> empty-content guard (422)              # existing
  -> Gemini
```

Scanning line by line:

1. Not a marker (`^\s*You might also like\s*$`, full line) → keep it, advance.
2. Marker → drop it, then drop the next **six non-blank** lines. Blank lines
   encountered inside the block go too and do not count toward the six: Genius's
   plain text sometimes separates a title from its artist, and counting the gap
   would close the block early and leave the tail of the widget in the song.
3. Fewer than six lines available — a block ending the paste — means drop what is
   there.

Multiple markers in one paste are handled by the outer scan.

No comparison against the singer at any point. `_clean_lyrics` takes only
`lyrics`.

The `You might also like` entry stays in `_LYRIC_NOISE_PATTERNS`. It is no longer
the main path, but it still catches the concatenated variant described under
"Known limitation", where the full-line anchor misses.

### Logging

One INFO line per call when anything was removed:

```
Removed %d Genius recommendation line(s) across %d block(s)
```

Counts only — the project forbids writing lyric text to logs. A per-block count
below six means a widget rendered short, which is the one condition under which
this rule reaches into the song. Since the loss is otherwise silent, this line is
the only trace it would leave.

## Known cost, stated plainly

If Genius ever renders fewer than three recommendations, the delete runs into the
song and takes lyric lines. **There is no longer any check that would stop it** —
the singer comparison was that check, and it did not work. This is accepted, not
overlooked, and it is pinned by
`test_a_block_with_fewer_than_three_recommendations_takes_lyrics_with_it` so it
stays a recorded decision rather than resurfacing later as a bug report.

What bounds it:

- the window opens only immediately after a literal full-line marker;
- it closes after six lines;
- it logs a count when it fires.

The project's stated priority is that deleting real lyrics is the worse failure,
because it is silent. That priority is unchanged; what changed is the evidence
about how likely the deletion is. A rule that reliably deletes nothing is not
safer than one that occasionally over-deletes — it just fails in a way that is
easier to ignore.

## Rejected alternatives

**Singer-anchored boundary.** Shipped, falsified, removed. See the revision
section above.

**Per-pair gating** (consume pairs while the artist matches, stop at the first
that does not). Rejected before the singer rule shipped: the interleaved
`artist / other / artist` block would consume one pair, stop at the second, and
leave four lines of noise. Moot now, since no artist comparison happens at all.

**A prompt-level backstop.** Adding a rule to `prompts/translate.md` telling
Gemini to ignore recommendation blocks, rejected on two grounds.

*It has no anchor.* By the time the prompt is built, the marker has been removed.
The model would receive orphan lines with no more information than the
deterministic rule had.

*The prompt is already overloaded.* The comment at `llm_service.py:32-40` records
a production incident where this prompt drove 22,612 thought tokens (92% of the
ceiling) and returned a candidate with no content parts. `translate.md` already
mandates Google Search grounding, three 150-200 word essays, and per-line
alignments and annotations in a single call. Adding instructions has a measured
cost here.

## Known limitation

Genius sometimes concatenates chrome onto an adjacent line rather than emitting
it standalone — the same file already documents the
`See <Artist> LiveGet tickets as low as $46` variant. A concatenated marker
defeats the full-line anchor and the rule degrades to today's behaviour: the
looser `_LYRIC_NOISE_PATTERNS` entry removes the marker text and the six
recommendation lines survive. Not handled.

## Testing

In `tests/test_lyric_input_validation.py`, following its existing split between
what the cleaner must delete and what it must never delete.

| Case | Expected |
|---|---|
| Three recommendations, all by the song's own artist (Lil Uzi Vert sample) | Marker and all six lines gone, lyrics intact |
| Three recommendations, none by the page's artist (Christina Perri sample) | Same — the rule does not distinguish them |
| A block ending the paste, one recommendation only | Marker and the two lines it has removed |
| Two recommendations followed by real lyrics | Six lines taken, **the two lyric lines are lost** — the known cost, pinned |
| Lines that look like recommendations with no marker before them | All preserved |
| Two recommendation blocks in one paste | Both handled independently |
| Blank lines interleaved inside a block | Consumed with it, not counted toward six |
| Existing `GENIUS_CHROME` fixture | Still cleans to nothing, still 422 |

The two singer-shaped samples are one parametrized test, because with the singer
comparison gone they exercise the same code path; the `ids=` record where each
came from.

Seven tests asserting singer-matching behaviour were deleted with the machinery
they covered (casing and punctuation folding, the parenthesised alias, the
two-letter stage name floor, trailing-residue and short-block boundaries).

### Mutation results

The tests were checked against four mutants, each killed:

| Mutant | Result |
|---|---|
| `_RECOMMENDATION_LINES = 5` | 5 failed |
| `_RECOMMENDATION_LINES = 7` | 5 failed |
| Blank lines count toward the six | 1 failed |
| Consume without requiring a marker | 21 failed |
