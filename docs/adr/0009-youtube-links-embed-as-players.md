# A YouTube link in a description embeds as a player

A Post's `description` is markdown in the narrow three-construct dialect defined by
`slangdump-poster/src/lib/markdown.tsx`. That dialect gains a fourth construct: a
YouTube URL renders a player *beneath* its link rather than in place of it. The URL
may be a markdown link or bare text, and may stand alone on its line or sit inside a
sentence; every form embeds.

The field was always meant to carry media. `src/types/api.ts` says so — "media
URLs are embedded inline in the markdown `description`. File upload is out of
scope this phase." What was missing was the read path. Until now an author who
linked the song they were writing about produced an anchor, and the reader had to
leave the page to hear it. On a site whose entire subject is what a song's lyrics
mean, the song is the one thing the post was already about.

## The link survives the player

The player is added to the link, not substituted for it. Replacing it was the first
shape of this and it deletes authored text: `[聽這首](https://youtu.be/ID)` rendered a
player and 聽這首 was simply gone, with nothing in the output to say a word had been
dropped. That is the same silent failure this document objects to below — a result
the author cannot see coming and cannot see happening.

Showing it costs close to nothing, because the label is usually the URL. The editor
configures `autolink: true, linkOnPaste: true` (`DescriptionEditor.tsx`), so an
author who pastes a URL gets a link node whose text *is* that URL, which serializes
to `[https://youtu.be/ID](https://youtu.be/ID)`. A bare URL is therefore anchored on
its own text too — otherwise the same visible act of pasting a link would render two
different ways depending on whether the row predates the editor.

It also has a job when the player does not. An uploader who disabled embedding gets
YouTube's own error inside the frame, and the way out of that is a link. The player
keeps its corner link-out for the case where the failure is in the frame itself, and
the anchor above it is the ordinary route.

## Position is not a test the author can pass

Requiring the URL to stand alone on its line is the tidier rule and it is not the
one here. The description editor is WYSIWYG: markdown "is an implementation detail
the author never sees" (`DescriptionEditor.tsx`), so a rule keyed to where a line
ends in a serialization they are never shown is a rule they cannot aim at. It fires
or it does not, for reasons invisible from the composer, and the failure is silent
— an orange anchor where a player was wanted.

The cost is real: a 16:9 frame can land mid-sentence. It lands as a block, so the
words before it and after it survive on their own lines rather than being lost, and
an author who does not want that can put the link on its own line — which is the
thing they *can* see themselves doing.

What this does not cost is the promise that motivated the narrow dialect in the
first place. Descriptions already in the database were written as plain text, and
the guard against silently reflowing them is the YouTube-only scan below, not the
line boundary.

## A YouTube-only URL scan, not general autolinking

Bare URLs embed, which means the tokenizer now reads text it previously passed
through untouched, and a bare YouTube URL does become an anchor. What it is not is
*general* autolinking. A bare URL is parsed, and if it is not a YouTube video the
characters stay literal exactly as before — no anchor, no player.

This is the narrowest change that reaches the legacy rows. Descriptions written
before the WYSIWYG editor existed hold raw URLs as plain text — the editor
autolinks on paste, so only new posts produce link nodes. Making all bare URLs
into anchors would have been the tidier grammar and a far wider blast radius:
every `https://genius.com/...` already in the database would change appearance.

Where a bare URL *ends* is then a question the line boundary used to answer for
free. The scan runs to the first character that cannot appear in a URL — printable
ASCII, not `\S` — because the corpus is mostly Traditional Chinese and Chinese
prose has no spaces: `聽這個https://youtu.be/ID很讚` would otherwise carry 很讚 into
the video id and embed nothing at all. Trailing `.,;:!?')]}"` is then trimmed back
into the prose, since a URL may legally end with any of it but a sentence more
often does. Trimming too much is harmless: a URL that no longer parses as a video
falls through and stays literal text, which is what it did before.

## The player is a facade

The feed renders several posts at once and each live YouTube embed pulls roughly a
megabyte of third-party JavaScript on mount. What renders is a thumbnail from
`i.ytimg.com` behind a play button; the `youtube-nocookie.com` iframe is mounted
on click. Nothing is requested from YouTube until the reader asks for it.

The video id is validated to eleven characters against an exact-match host
allowlist, and the embed URL is **rebuilt from that id** — the author's string
never reaches `src`. The renderer's guarantee is that markup injection is
structurally impossible rather than filtered out, with URLs as the one audited
exception, and an iframe only keeps that guarantee if the untrusted string is
never the thing that gets loaded.

## Consequences

**Follow Mode now has an audio source on the page, and nobody is driving it.**
Follow Mode listens to the *microphone* — `scribeEar.ts` opens it with
`echoCancellation: false` because "in the same-device case the speaker's output is
the entire signal." Putting a player in the feed makes same-device playback the
obvious thing to do, and same-device is the configuration that file documents as
fragile: a phone can force echo cancellation back on and subtract its own speaker
from its own microphone, cancelling exactly the signal Sync exists to hear. The
embed and Follow Mode are deliberately left unaware of each other for now. A
reader who starts both gets whatever the device does.

**Playhead sync is deferred, not rejected.** The YouTube IFrame Player API exposes
`getCurrentTime()`. A Follow Mode driven off the playhead would need no
microphone, would sidestep the echo-cancellation trap entirely, and would drop an
ElevenLabs call per session — it is strictly better than what exists. It is not
here because it is a larger change that overlaps unmerged work on
`feat/sync-follow-mode` and deserves its own decision. `YouTubeEmbed` holds the id
and the mounted player behind one component boundary so that work has a seam to
attach to rather than a rewrite to perform.

**Third-party shorteners stay links.** `bit.ly`, `t.co` and smart links hide their
target, and a browser cannot read a cross-origin redirect, so resolving one means
a server-side route in the BFF — with a host allowlist and a redirect-hop cap to
avoid turning the frontend into an SSRF proxy, plus a cache and a loading state in
the feed. YouTube's own `youtu.be` needs none of that: the id is in the path.

**Responsibility for the video is unchanged.** An official iframe is the sanctioned
way to surface someone else's recording, and it carries YouTube's own licensing
and ad path. The specific upload an author links may still be unofficial; that
liability sits with the uploader and with YouTube, exactly as it does for the
anchor rendered today. This does not weaken
`slangdump-poster/docs/adr/0003-permalink-publishes-prose-not-lyrics.md` — no
lyric line is published by embedding a player.

**The permalink does not advertise the video.** No `og:video`, no video
`og:image`. That page is deliberately narrow, and the video is a third party's
upload rather than the post's own content; unfurling it as a video card would be
the first time the permalink published something it does not own.

**The editor still shows a link, not a preview.** The author sees what they always
saw, and now so does the reader — the asymmetry is narrower than it was, since the
feed shows the link too, but the preview is still missing. This is the same gap
images already have — renderable but not authorable — and closing it means a new
Tiptap node.

**If a Content-Security-Policy is ever added, it has to know about this.** There
is none today. One would need `frame-src https://www.youtube-nocookie.com` and
`img-src https://i.ytimg.com`, and getting it wrong fails as a blank frame rather
than an error.
