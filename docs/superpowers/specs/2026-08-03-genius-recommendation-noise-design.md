# Stripping Genius Recommendation Blocks From Pasted Lyrics

**Date:** 2026-08-03
**Repos touched:** `slangdump-ai-solation` (Python worker) only

## Summary

Genius injects a "You might also like" recommendation widget *inside* the lyrics
container. When a user selects and copies the lyrics, the widget's plain text
lands in the middle of the paste:

```
You might also like
<recommended title 1>
<recommended artist 1>
<recommended title 2>
<recommended artist 2>
<recommended title 3>
<recommended artist 3>
```

`_clean_lyrics` in `services/llm_service.py` already deletes the marker line
(`_LYRIC_NOISE_PATTERNS`, the `^\s*You might also like.*$` entry). It does not
delete the six lines after it, and no literal pattern can — a recommended song
title is textually indistinguishable from a lyric line. Those six lines are sent
to Gemini, translated, stored, and rendered as if they were part of the song.

The fix is a bounded structural rule anchored on the marker, using the one piece
of information a pattern does not have: `req.singer`.

## Why the fix belongs in the worker, and nowhere else

Two other placements were considered and rejected.

**The frontend does not need to change.** `FeedItem.tsx` renders `detail.lines`
and drops any line whose `sourceHash` does not match one of the viewer's pasted
lines (`flatMap` returning `[]`). It never renders the raw paste directly.
So a line the worker declines to return is automatically absent from the display.
Cleaning in `_clean_lyrics` propagates to rendering for free.

**Asking the model to judge was rejected** — see "Rejected: a prompt-level
backstop" below.

## The rule

New `_strip_recommendation_blocks(lyrics, singer)` running **before** the
existing `_LYRIC_NOISE_PATTERNS` loop. The order is load-bearing: the existing
marker pattern deletes the only anchor the structural rule has.

```
req.lyrics
  -> _strip_recommendation_blocks(lyrics, req.singer)   # new: marker + matching pairs
  -> _LYRIC_NOISE_PATTERNS loop                          # existing: remaining chrome
  -> empty-content guard (422)                           # existing
  -> Gemini
```

`_clean_lyrics` therefore takes a new `singer` argument. Spring and the frontend
are untouched; `TranslationRequest.singer` already carries the value.

### Artist comparison

```python
_MAX_RECOMMENDATIONS = 3
_MIN_FOLDED_ARTIST_LEN = 3
_RECOMMENDATION_MARKER = re.compile(r"^\s*You might also like\s*$", re.IGNORECASE)


def _fold_artist(s: str) -> str:
    """NFKC, keep only alphanumerics, casefold.

    Folds "Lil Uzi Vert", "LIL UZI VERT" and "Lil-Uzi-Vert" onto one key. The
    singer value is typed by the user and rarely matches Genius's rendering
    character for character, so a strict comparison would clear almost nothing.
    """
    return "".join(c for c in unicodedata.normalize("NFKC", s) if c.isalnum()).casefold()


def _matches_singer(line: str, singer: str) -> bool:
    folded = _fold_artist(singer)
    if len(folded) < _MIN_FOLDED_ARTIST_LEN:
        # Short stage names (IU, Zico) are exactly the ones that collide with
        # ordinary lyric text once punctuation and case are discarded. Fall back
        # to strict equality rather than folding them.
        return _canonical(line) == _canonical(singer)
    return _fold_artist(line) == folded
```

### Consumption loop

Scanning line by line:

1. Not a marker -> keep the line, advance.
2. Marker -> drop it, then attempt at most `_MAX_RECOMMENDATIONS` pairs.
3. For each attempt: scan forward to the next non-blank line and treat it as the
   title, then scan forward again to the next non-blank line after that and treat
   it as the artist. If `_matches_singer(artist, singer)` holds, consume both —
   along with any blank lines skipped between them — and attempt the next pair.
   Otherwise **stop immediately**, leaving the cursor where it was before the
   attempt; every remaining line is kept verbatim, blank lines included.

Running off the end of the input while looking for either half of a pair is
treated the same as a non-match: stop, keep what remains.

Multiple markers in one paste are handled by the outer scan.

### Why this is safe

Safety comes from two independent bounds, not from the artist comparison alone:

- The window opens only immediately after a literal marker line and closes after
  at most three pairs.
- Inside the window, the second line of each pair must match the known singer.

For a real lyric line to be deleted it must fall within six lines of a marker,
appear in pair position, *and* fold to exactly the singer's name. Relaxing the
comparison (the decision above) raises the clearance rate without widening the
window, which is what carries the risk.

This matters because `tests/test_lyric_input_validation.py` already establishes
the priority: deleting real lyrics is the worse failure, because it is silent —
no error, just a song translated with holes in it.

## Deliberate residue: cross-artist recommendations

When Genius recommends songs by *other* artists, no pair matches, the loop stops
at the first attempt, and the six lines survive. Only the marker is removed
(by the existing pattern, in the second stage).

This is accepted, not overlooked. It is pinned by a test so the behaviour is
recorded rather than assumed.

## Rejected: a prompt-level backstop

Adding a rule to `prompts/translate.md` telling Gemini to ignore recommendation
blocks was considered for the residue case and rejected on two grounds.

**It has no anchor.** By the time the prompt is built, the marker has been
removed by `_LYRIC_NOISE_PATTERNS`. The model would receive six orphan lines
with no more information than the deterministic rule had. Preserving the marker
whenever pairs went unconsumed would restore the anchor, but that is extra
conditional state in the cleaner to serve a minority case.

**The prompt is already overloaded.** The comment at `llm_service.py:32-40`
records a production incident where this prompt drove 22,612 thought tokens
(92% of the ceiling) and returned a candidate with no content parts. `translate.md`
already mandates Google Search grounding, three 150-200 word essays, and
per-line alignments and annotations in a single call. Adding instructions has a
measured cost here.

Revisit if cross-artist recommendations turn out to be common in practice; the
preserve-the-marker variant is the design to reach for then.

## Known limitation

Genius sometimes concatenates chrome onto an adjacent line rather than emitting
it standalone — the same file already documents the
`See <Artist> LiveGet tickets as low as $46` variant. A concatenated marker
defeats the full-line anchor and the rule degrades to today's behaviour. Not
handled; the looser existing pattern still removes what it removes today.

## Testing

Extending `tests/test_lyric_input_validation.py`, following its existing split
between what the cleaner must delete and what it must never delete.

| Case | Expected |
|---|---|
| Three same-artist recommendations, real lyrics on both sides | All seven lines gone, lyrics intact |
| One pair; two pairs | Only the pairs present are consumed |
| Marker followed directly by real lyrics (zero pairs) | Only the marker is removed |
| Cross-artist recommendations | Marker removed, six lines survive (pins the residue) |
| `LIL UZI VERT` / `Lil-Uzi-Vert` casing and punctuation variants | Matched and removed |
| Singer `IU`, lyric line containing `IU` | Folding disabled, lyric preserved |
| A lyric line equal to the singer's name, far from any marker | Preserved |
| Two recommendation blocks in one paste | Both cleared |
| Existing `GENIUS_CHROME` fixture | Still rejected with 422 |

The four existing `_clean_lyrics(...)` call sites in that file need the new
`singer` argument. It is a required parameter rather than one defaulting to `""`,
so every call site is surfaced by the change instead of silently going inert.

Per the project convention, every test carries a `@DisplayName`-equivalent
docstring or `ids=` label describing the case in words.
