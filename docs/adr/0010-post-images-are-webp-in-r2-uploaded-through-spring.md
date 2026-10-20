# Post images are encoded in the browser and stored in R2 through Spring

An author can put up to five images into a Post's description in the Daily Drop editor.
Each image is encoded by the browser to a WebP (or, on Safari and iOS, a PNG) of at most
2048 px on its long side, uploaded the moment it is inserted, checked by Spring, stored
unchanged in a public Cloudflare R2 bucket under a random key, and embedded in the
description as `![](https://img.slangdump.com/p/<uuid>.webp)`. A
`post_images` table records who uploaded each object and which Post, if any, owns it.

The description already rendered `![alt](url)`; what this adds is a way to author one
that does not point at somebody else's server.

## Through Spring, not presigned URLs

R2 supports presigned PUTs, which would let the browser upload straight to the bucket and
skip every size limit in between. They were rejected for two reasons. Nothing could look at
the bytes before they were public: a presigned PUT stores whatever arrives, so an HTML
page named `cat.jpg` or a photo carrying the author's GPS position would be served from our
domain as-is. And the browser would be talking to a third party, which the architecture
otherwise never allows — the frontend talks to its BFF and the BFF talks to Spring.

The cost is the BFF's body limit. Vercel refuses function bodies over 4.5 MB before our
code runs, while phone photos are routinely larger, and an author may pick up to 10 MB.
The browser's encoding (below) is what gets every image under it: Spring takes at most
4 MB. The 10 MB limit is therefore what an author may *pick*, not what crosses the wire.

## The browser encodes, Spring only checks

The editor (`slangdump-poster/src/lib/encodeImage.ts`) draws every image onto a canvas at
no more than 2048 px on its long edge and encodes it as WebP at quality 0.8, or 0.6 if
that is over 4 MB. Drawing pixels is what strips EXIF, including a phone photo's GPS
position, and what keeps only an animation's first frame. Images are encoded one at a time:
a phone photo decodes to 50-200 MB of pixels, and five at once can make iOS kill the tab.

Safari cannot encode WebP from a canvas, and every browser on iOS is Safari underneath:
asking for WebP quietly returns a PNG. That PNG is accepted as is. PNG has no quality
setting and a 2048 px photo is typically 5-7 MB as PNG, so an oversized one is redrawn at
three quarters of the size until it fits, giving up below 1024 px. Safari photos therefore
land around 1152-1536 px and weigh more in R2 than a WebP would. Bundling a WebAssembly
WebP encoder for Safari was rejected: it would be an extra frontend dependency for a
difference nobody reading a post would notice.

Spring does no image processing and has no image library. `UploadedImage.kt` reads the
container headers only: a still WebP or PNG, at most 2048 px on a side, with nothing after
the image, and no metadata or animation chunks (`EXIF`, `XMP `, `ANIM` in WebP; `eXIf`,
text and APNG chunks in PNG). That makes the browser's stripping enforceable: a
hand-built upload cannot publish GPS data or an animation under our domain. Because
nothing is decoded, there is no decompression bomb to guard against, and the header check
caps what any reader's browser will decode at 2048² pixels.

Encoding on the server was the first design and was rejected: it needed `webp-imageio`, a
JNI wrapper shipping native libwebp inside the jar, plus the decode and encode CPU and
memory on a 1 GB machine. What the browser-side design gives up is a check of the pixels
themselves: a corrupt bitstream in a well-formed container is stored, and browsers fail to
draw it, which harms only the author who uploaded it.

## Images are served by R2, never by Spring

The bucket is public on a custom domain. An image view never reaches a Fly machine, so
reading a feed full of images cannot wake a suspended Spring app, and R2's free egress
covers the traffic. Keys are random UUIDs and never reused, so objects are served
`immutable`. The key's extension is the format, `.webp` or `.png`.

`next/image` is not used. It would spend Vercel's image-optimisation quota re-encoding
images that are already optimised WebP.

## Orphans are cleaned when the author posts, not on a schedule

An image exists before its Post does, so some uploads are never used: removed from the
draft by backspace instead of the remove button, left in a draft that was abandoned, or
uploaded before a submit that failed. The remove button deletes at once. Everything else is
deleted when the author's next Post is created: one Post per Taipei day means no other draft
of theirs can still need an image that the new Post did not embed.

There is deliberately no scheduled sweeper. The residue is an author who uploads and then
never posts again, and it is bounded: uploads are capped at 10 per author per Taipei day —
twice the five a post can hold, so a full post plus replacing every image once —
and the cap counts deleted uploads too, so an upload/delete loop cannot run free. At
most that is 40 MB per author per day, and a typical encoded image is a few hundred KB,
far inside R2's 10 GB free tier. Adding a sweeper would mean a second
scheduled job, run concurrently on both Fly machines, to recover storage that costs nothing.
Revisit if R2 usage ever shows up on a bill.

The cleanup is best-effort and happens after the Post commits, so a storage failure never
fails a Post that already exists; the image stays unattached and is retried on that
author's next Post.

## Consequences

**A new Post can only embed its author's own uploads.** Spring finds every image in the
description with a scanner that mirrors the frontend tokenizer
(`DescriptionImages.kt` ↔ `markdown.tsx`) and refuses any URL that is not one of the
author's live, unattached uploads. The renderer still draws any safe URL, so descriptions
written before this rule keep rendering exactly as they did. The two scanners must agree;
where they could disagree, the backend errs toward seeing an image.

**Images count against the 2000-character description limit.** Each URL is about 60
characters, so five images cost about 300. Not worth a separate field.

**Removing an image with its button is not undoable.** The stored object is gone, so undo
would restore an image whose URL no longer resolves. Removing it any other way stays
undoable, because nothing is deleted until the Post is created.

**Published Posts stay non-editable.** Nothing here adds editing; an attached image is
permanent.

**The permalink does not use an image as `og:image`.** A screenshot of lyrics would
otherwise become the link preview for the Post, which
`slangdump-poster/docs/adr/0003-permalink-publishes-prose-not-lyrics.md` exists to prevent.
