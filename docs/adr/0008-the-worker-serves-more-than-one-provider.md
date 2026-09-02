# The worker serves more than one provider, and the Translator names which one

The AI worker gains a backend per API vendor and a registry keyed on
`(provider, model)` rather than on a model id. `provider` names **the vendor
dialled, not the lab that made the weights** — a Gemini reached through
OpenRouter is `openrouter/google/gemini-3.7-flash`, a different Translator from
`google/gemini-3.6-flash`. The Active Translator becomes
`openrouter/deepseek/deepseek-v4-flash-0731`. This amends ADR-0005, which
established the Active Translator but assumed one vendor.

The trigger was a quota, not a model preference. Gemini's free tier allows
gemini-2.5-flash 5 RPM and 20 RPD; production ran at 8 of 5 and 14 of 20, so the
whole product was capped at a handful of songs a day and the overflow reached
users as permanently `FAILED` songs. The worker's own retry policy was part of
the breach: `RETRY_ATTEMPTS = 4` composed with `EMPTY_RESPONSE_ATTEMPTS = 3` is
up to twelve requests for one translation, and every retry of a 429 spends more
of the quota that just refused it. Retrying a per-minute quota rejection cannot
work — the identical request meets the identical ceiling one second later.

## Why the request for a fallback became a switch

The change was asked for as "fall back to OpenRouter on a 429". It cannot be
built that way, and the reason is in ADR-0001, not in the worker.

A Song's identity is `(artist_norm, song_name_norm, llm_model_id)`, and
`PostService` reads the enabled Translator and writes it onto the row **before**
the worker is called. A worker that substituted a different vendor mid-request
would store OpenRouter's prose under a row claiming `google/gemini-2.5-flash`.
That is not a mislabelled log line: the row is permanent, and it is shared with
every later Post of the same track. It is precisely the label-versus-what-ran
drift that ADR-0005 and `V9__llm_model_enabled.sql` exist to end, rebuilt
deliberately this time.

`TranslationPersistenceService` already has the detector for it — it logs
`worker model {}/{} differs from song {} target {}/{}; keeping target` — and
"keeping target" is the wrong half to keep. A fallback would make that warning
fire on every rate-limited translation, by design.

So there is no fallback. There is one Active Translator, it is now an OpenRouter
model, and OpenRouter's own `allow_fallbacks` absorbs a rate-limited endpoint by
routing around it **inside the same request** — the thing a retry could never do,
and the reason the 429 retry is dropped rather than reimplemented.

## Why the provider is the vendor, not the lab

`openrouter/google/gemini-3.7-flash` reads oddly and is correct. What the pair
has to identify is *what produced this prose*, and reaching Gemini through
OpenRouter differs from reaching it directly in the price paid, the endpoint
served, the quantization the weights were run at, and — most visibly in the
output — the search engine the research went through, Exa rather than Google
Search. Recording the lab instead would make two genuinely different Translators
share one key, which is the identity failure this ADR is here to avoid.

It also keeps the field mechanical: `provider` selects a backend, and `model` is
the string that backend puts on the wire. Nothing has to interpret either.

## Why search is a server tool

`translate.md` is built around spending nothing on research it does not need:
with every Credited Artist's background already on file, Spring sends no
`需要背景的歌手` line and the prompt tells the model to spend **no** search budget
at all. That rule is what makes the steady-state translation cheap (ADR-0007).

OpenRouter's `web` plugin — and the `:online` suffix, both now deprecated —
would have quietly destroyed it. The plugin searches unconditionally, once per
request, on a query derived from a message that is 95% lyrics: a billed search on
every translation, for a question nobody asked. The `openrouter:web_search`
server tool is offered to the model and used only when the model decides it needs
it, so the prompt's rule keeps working.

It also keeps the worker at **one** HTTP request, because OpenRouter runs the
tool loop server-side. That matters historically: a two-pass research-then-
translate design was already tried in this worker and rolled back.

## Why routing is pinned

One model id at OpenRouter is served by many endpoints, whose input prices spread
better than 12x (roughly $0.035 to $0.44 per million) and whose quantizations run
down to int4. Left alone the router optimises for throughput and may land
anywhere in that range. So `sort: price` takes the cheap end, and `quantizations`
sets a floor at int8. The floor is not frugality in reverse — this prompt asks
for three essays of zh-TW prose plus per-line alignment, where quantization
damage shows up as quietly worse writing rather than as an error any test could
catch.

`max_tool_calls` is pinned for the same reason: OpenRouter's default is 30, which
is thirty billable searches on a single song.

## Considered options

**Fall back inside the worker on a 429.** What was asked for. Rejected above: it
writes one Translator's prose under another's row, permanently and shared.

**Fall back in Spring, re-keying the Song to the Translator that actually ran.**
Honest about identity, and still rejected. The Song row is created before the
translation starts, so re-keying means an UPDATE that can collide with an
existing row for the fallback Translator, and the dedupe key (`V6`) has to be
resolved mid-flight. It buys nothing that switching does not, and it needs
`llm_models_one_enabled` relaxed to express "two Translators, one preferred" —
turning a database-enforced invariant into a convention.

**Pay for Gemini.** Removes the quota with no code change at all, and was
declined on cost: OpenRouter is roughly 7x cheaper per translation for this
workload, against a project running on a $10-15/month budget.

**Route to a Gemini through OpenRouter instead of DeepSeek.** Moves one variable
rather than four, which was the recommendation. Overruled deliberately: the
quota that broke production is Google's, and reaching the same model through a
different door does not move it. `openrouter/google/gemini-3.7-flash` is
registered inert as the rollback target instead.

**Adopt OpenRouter's structured outputs.** Rejected for now. There is no
`response_schema` anywhere in this worker; the shape contract is a markdown block
in the prompt plus `json_repair`, and every downstream tolerance (name-matched
backgrounds, single-element array unwrapping, fence stripping) is built on that.
Adding a schema on one provider's path only would make the two backends disagree
about what a malformed response even is.

## Consequences

**Switching forks the catalogue, and that is what repairs the outage.** Every
cache lookup now misses, so the next Post of a song creates a second `songs` row
and pays for a fresh translation. Nothing is migrated or rewritten. The songs
stranded `FAILED` by the quota breach are keyed to `google/gemini-2.5-flash` and
`FAILED` is permanent, so a new Translator is the only thing that gives them
another chance — for free, as a side effect of the key changing.

**Gemini stays required, not vestigial.** `GEMINI_API_KEY` remains a mandatory
setting and every retired Gemini entry stays in `MODEL_CONFIGS`. Songs translated
before the switch are keyed to those Translators, Spring sends them back, and a
Song still `PENDING` across the switch completes under its own. Rollback also
depends on it, per ADR-0005.

**A missing `OPENROUTER_API_KEY` is a 503, not a startup failure.** Refusing to
boot would turn one missing secret into a total outage, including for the Gemini
Translators that do not need it. The worker starts, serves what it can, and
refuses only what it cannot — which is also the exact shape of the window where
V11 has enabled the Translator before the secret reached the app.

**Quality is now unattributable.** Provider, model, search engine and moderation
regime all changed in one step, so a prose regression cannot be traced to any one
of them. The mitigation is a manual read of real output against the deployed
worker before the migration runs, and a rollback target registered inert.

**Moderation is no longer ours to switch off.** The Gemini path sets every safety
threshold to `BLOCK_NONE`; on OpenRouter, moderation belongs to the routed
endpoint and there is no request-level equivalent. A `content_filter` finish is
mapped to 502 and, like every error, marks the Song `FAILED` permanently.

**`openrouter:web_search` is Beta.** Its behaviour may change under us, and it is
the one part of this design with no fallback of its own.
