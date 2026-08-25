# Lexical search is trigrams, not Postgres full-text search

`search_documents.search_text` is matched with `pg_trgm` — a GIN index with
`gin_trgm_ops` accelerating `LIKE '%q%'`, plus the `<%` word-similarity operator
for misspellings. There is no `tsvector` column, no `to_tsvector`, and no
`ts_rank` anywhere in the search path.

The corpus is mostly Traditional Chinese: translated lyric lines, annotation
explanations, `song_meaning`, `album_context`. Chinese is written without spaces,
and stock Postgres text search splits on whitespace, so `to_tsvector('simple',
'這首歌講的是炫富文化與貧窮的拉扯')` produces roughly one token — the whole
sentence. Nothing a user could type would match it. The three extensions that
tokenize Chinese (`zhparser`, `pg_jieba`, `pgroonga`) are all absent from Neon's
extension allowlist, so this is not a configuration we can change.

Trigrams need no tokenizer. `炫富文化` and `designer shoe` are both just
overlapping three-character windows, and one index serves both languages.

## Considered options

**Elasticsearch.** What the original plan named, and what a search feature
normally reaches for: real CJK analyzers (ICU, kuromoji, smartcn), phrase and
proximity queries, native highlighting. It is a paid managed service, or an
operated cluster with its own memory floor, plus an ingestion path to keep in
sync with Postgres. `CLAUDE.md` requires justifying why the current approach is
insufficient before adding a paid service, and at this corpus size — thousands
of documents, not millions — it is not insufficient. Deferred, not rejected on
merit.

**A bigram `tsvector` for Chinese.** Emitting overlapping character pairs into a
`simple` tsvector recovers indexed matching for 1–2 character CJK queries, which
is the one case trigrams genuinely cannot serve. It means maintaining a second
index and a second query path for the same text, and buys nothing for the 3+
character queries that make up nearly all searches. Left as the fix if short-
query latency ever becomes visible.

**Vectors only, no lexical pass.** Embeddings tokenize Chinese perfectly well,
so this would sidestep the whole problem. Rejected: exact search must be exact.
Someone typing an artist's name wants that artist, not the five songs nearest to
them in embedding space, and every query would then cost an HTTP round trip to a
suspended Fly Machine.

## Consequences

**The normalized key is materialized in a column, not generated.** Postgres 13+
could compute it — `lower(normalize(text, NFKC))` plus a whitespace collapse — so
a generated column or an expression index is technically available. It is still
Kotlin's job, because a *query* is not a row: the query side has to call
`SearchText.normalize` whatever the column does, and a SQL copy would be a second
definition of the same rule that must agree with the first forever. One
definition, two callers — the indexer for documents, the service for queries. A
document written any other way silently matches nothing.

**Nothing in `search_documents` can be displayed.** `search_text` is lowercased
and whitespace-collapsed, so a title comes back as `kendrick lamar`. Snippets are
read back from the table that owns the row, by `(kind, ref_id)`. The table
deliberately keeps no second, displayable copy — the same call V2 made when it
dropped `alignments.src`.

**Ranking is hand-weighted, because there is no `ts_rank`.** Score is
`similarity × kind weight`, with a bonus for verbatim containment; the weights
live in `SearchDocKind` and the SQL `CASE` is generated from them. There is no
term frequency, no document-length normalization, and no phrase or proximity
matching: "designer shoe" and "shoe designer" are the same query.

**One- and two-character CJK queries have no index selectivity.** Such a pattern
yields only padded boundary trigrams, which a mid-string `%…%` match cannot use,
so the index scan matches every row and the recheck discards them one by one.
Bounded two ways: the endpoint rejects queries below the minimum, and every
search sets `statement_timeout = 2000ms` so no query can hold a pooled Neon
connection.

**Fuzzy matching in Chinese is the embeddings' job, not the trigram index's.**
`word_similarity` scores against the best-matching extent within a term, which
gives real typo tolerance for space-delimited names ("kendirck" → 0.44 against
"kendrick lamar"). Chinese has no word boundaries, so it degenerates to
whole-string `similarity`, and a one-character slip inside a sentence scores
about 0.2 — indistinguishable from noise at any usable threshold. That case is
covered by the semantic pass instead, which is a large part of why the cascade
exists at all.

**Chinese matching is script-exact, so Simplified input is the semantic pass's
job too.** There was briefly a Traditional→Simplified fold in `SearchText`,
carried by ICU4J, that let a query in either script match a corpus in either. It
was removed: the corpus and the audience are Traditional (`zh-TW`), it cost a
14 MB jar for one function, and it was lossy in the direction that mattered — 髮
and 發 both fold to 发, so a search for 頭髮 also matched 頭發. A Simplified query
now finds nothing lexically and falls through to the cascade, which is to say it
degrades from exact to approximate rather than to empty. Re-adding the fold is
cheap in code but not free in data: stored keys and query keys must agree, so
every `search_text` row would need re-normalizing.
