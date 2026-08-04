# Stripping Genius Recommendation Blocks From Pasted Lyrics

**Date:** 2026-08-03
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

The fix is a bounded structural rule anchored on the marker, using the one piece
of information a pattern does not have: `req.singer`.

### What the real samples show

Two pastes observed in the wild, both three recommendations / six lines:

- All three by the song's own artist.
- Artists **interleaved**: the song's artist, a different artist, the song's
  artist again.

The second shape is what drives the design. A different artist is not a separate
category of block to handle elsewhere — it appears *inside* an otherwise ordinary
block, so any rule that stops at the first unrecognised artist stops in the
middle of the noise.

The second sample also showed an artist rendered as `Name (본명)` — an English
stage name with a parenthesised native-script alias. The user typed only the
stage name.

## Why the fix belongs in the worker, and nowhere else

**The frontend does not need to change.** `FeedItem.tsx` renders `detail.lines`
and drops any line whose `sourceHash` does not match one of the viewer's pasted
lines (`flatMap` returning `[]`). It never renders the raw paste directly. So a
line the worker declines to return is automatically absent from the display.
Cleaning in `_clean_lyrics` propagates to rendering for free.

**Asking the model to judge was rejected** — see "Rejected alternatives" below.

## The rule

New `_strip_recommendation_blocks(lyrics, singer)` running **before** the
existing `_LYRIC_NOISE_PATTERNS` loop. The order is load-bearing: the existing
marker pattern deletes the only anchor the structural rule has.

```
req.lyrics
  -> _strip_recommendation_blocks(lyrics, req.singer)   # new: marker + confirmed pairs
  -> _LYRIC_NOISE_PATTERNS loop                          # existing: remaining chrome
  -> empty-content guard (422)                           # existing
  -> Gemini
```

`_clean_lyrics` therefore takes a new `singer` argument. Spring and the frontend
are untouched; `TranslationRequest.singer` already carries the value.

### Artist comparison

```python
_MIN_FOLDED_ARTIST_LEN = 3

# NFKC folds fullwidth parens onto ASCII; the CJK bracket forms it leaves alone
# are listed explicitly.
_PARENTHETICAL = re.compile(r"[(\[【〔][^)\]】〕]*[)\]】〕]")


def _fold_artist(s: str) -> str:
    """NFKC, drop parenthesised asides, keep only alphanumerics, casefold.

    Folds "Lil Uzi Vert", "LIL UZI VERT" and "Lil-Uzi-Vert" onto one key. The
    singer value is typed by the user and rarely matches Genius's rendering
    character for character, so a strict comparison would clear almost nothing.

    Dropping the parenthetical is what makes "Jung Kook (정국)" and
    "BTS (방탄소년단)" match a user who typed only the stage name. Substring
    containment would do that too, but it also matches a short artist name
    buried inside an unrelated word, so equality on the reduced form is used
    instead. If removing the parenthetical would leave nothing, it is kept.
    """
    s = unicodedata.normalize("NFKC", s)
    stripped = _PARENTHETICAL.sub(" ", s)
    if any(c.isalnum() for c in stripped):
        s = stripped
    return "".join(c for c in s if c.isalnum()).casefold()


def _matches_singer(line: str, singer: str) -> bool:
    folded = _fold_artist(singer)
    if len(folded) < _MIN_FOLDED_ARTIST_LEN:
        # Short stage names (IU, Zico) are exactly the ones that collide with
        # ordinary lyric text once punctuation and case are discarded. Fall back
        # to strict equality rather than folding them.
        return _canonical(line) == _canonical(singer)
    return _fold_artist(line) == folded
```

### Consumption boundary

Scanning line by line:

1. Not a marker -> keep the line, advance.
2. Marker -> drop it, then examine the next **six non-blank lines** as three
   candidate `(title, artist)` pairs. Fewer than six available means no block;
   keep everything.
3. Test `_matches_singer` on each of the three artist slots. If none match, keep
   all six lines.
4. Otherwise consume everything from the marker through **the last matching
   artist slot** — both halves of every pair up to it, plus any blank lines that
   fell between them. Lines beyond it are kept verbatim, blank lines included.

Multiple markers in one paste are handled by the outer scan.

Step 4 is the whole design. A fixed six-line delete was considered and is
strictly worse:

| Block | Slots | Consumed | Outcome |
|---|---|---|---|
| Artist / other / artist | ✓ ✗ ✓ | 6 | Fully cleared (real sample) |
| Artist / artist / other | ✓ ✓ ✗ | 4 | Cleared to the last confirmed pair; two lines residue |
| Only two recommendations, then real lyrics | ✓ ✓ ✗ | 4 | Recommendations cleared, **lyrics untouched** |

The third row is why the boundary is data-driven rather than fixed. Genius is
not contractually obliged to render three recommendations, and a fixed six-line
delete would silently eat two lyric lines whenever it renders fewer. Trailing
by the last confirmed match makes that failure mode structurally impossible: a
lyric line does not match the singer's name, so the consumption stops before it.

The cost is the second row — a block whose *trailing* recommendations are by
other artists leaves residue. Residue is always preferable to deleting lyrics.

### Why this is safe

Safety comes from three independent bounds:

- The window opens only immediately after a literal marker line.
- It closes after at most three pairs.
- It closes earlier still, at the last line positively confirmed as the known
  artist.

For a real lyric line to be deleted it must fall within six lines of a marker,
sit in an artist slot, fold to exactly the singer's name, **and** have a later
slot in the same window also match.

This matters because `tests/test_lyric_input_validation.py` already establishes
the priority: deleting real lyrics is the worse failure, because it is silent —
no error, just a song translated with holes in it. The project also forbids
writing lyrics to logs, so there is no post-hoc way to discover what was lost;
only a count could be logged.

## Deliberate residue

Two cases leave noise behind, both accepted and both pinned by tests:

- **Trailing recommendations by other artists** — consumption stops at the last
  confirmed pair.
- **No recommendation by the song's own artist** — nothing is confirmed, so only
  the marker is removed.

These are recorded rather than assumed, so a future change can see what the rule
does and does not cover.

## Rejected alternatives

**Per-pair gating** (consume pairs while the artist matches, stop at the first
that does not). This was the original design and the real samples killed it: the
interleaved `artist / other / artist` block would consume one pair, stop at the
second, and leave four lines of noise. Testing every slot before choosing a
boundary is what handles interleaving.

**A prompt-level backstop.** Adding a rule to `prompts/translate.md` telling
Gemini to ignore recommendation blocks was considered for the residue cases and
rejected on two grounds.

*It has no anchor.* By the time the prompt is built, the marker has been removed
by `_LYRIC_NOISE_PATTERNS`. The model would receive orphan lines with no more
information than the deterministic rule had. Preserving the marker whenever a
block went unconfirmed would restore the anchor, but that is extra conditional
state in the cleaner to serve a minority case.

*The prompt is already overloaded.* The comment at `llm_service.py:32-40` records
a production incident where this prompt drove 22,612 thought tokens (92% of the
ceiling) and returned a candidate with no content parts. `translate.md` already
mandates Google Search grounding, three 150-200 word essays, and per-line
alignments and annotations in a single call. Adding instructions has a measured
cost here.

Revisit if residue turns out to be common in practice; preserving the marker for
unconfirmed blocks is the design to reach for then.

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
| Three recommendations, all by the song's artist, real lyrics on both sides | All seven lines gone, lyrics intact |
| Interleaved `artist / other / artist` | All seven lines gone (the real sample) |
| Trailing `artist / artist / other` | Marker + four lines gone, two lines residue |
| No slot matches the singer | Only the marker is removed |
| Two recommendations followed by real lyrics | Four lines gone, **both lyric lines preserved** |
| Fewer than six non-blank lines after the marker | Only the marker is removed |
| Marker followed directly by real lyrics | Only the marker is removed |
| `LIL UZI VERT` / `Lil-Uzi-Vert` casing and punctuation variants | Matched |
| Artist rendered `Jung Kook (정국)`, singer typed `Jung Kook` | Matched |
| Singer `IU`, lyric line containing `IU` | Folding disabled, lyric preserved |
| A lyric line equal to the singer's name, far from any marker | Preserved |
| Two recommendation blocks in one paste | Both handled independently |
| Blank lines interleaved inside a block | Consumed along with their pair |
| Existing `GENIUS_CHROME` fixture | Still rejected with 422 |

The four existing `_clean_lyrics(...)` call sites in that file need the new
`singer` argument. It is a required parameter rather than one defaulting to `""`,
so every call site is surfaced by the change instead of silently going inert.

Following the project's test-naming convention, each case carries a docstring or
`ids=` label describing it in words.
