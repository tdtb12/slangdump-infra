# Post images are WebP in R2, uploaded through Spring

An author can put up to five images into a Post's description in the Daily Drop editor.
Each image is uploaded the moment it is inserted, re-encoded by Spring to a WebP of at most
2048 px on its long side, stored in a public Cloudflare R2 bucket under a random key, and
embedded in the description as `![](https://img.slangdump.com/p/<uuid>.webp)`. A
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
code runs, while phone photos are routinely larger, and the stated limit is 10 MB. So the
editor shrinks anything over 4 MB or over 2048 px in the browser before sending it
(`slangdump-poster/src/lib/shrinkImage.ts`). Nothing is lost by this that would have been
kept: Spring scales to the same 2048 px edge regardless. The 10 MB limit is therefore what
an author may *pick*, not what crosses the wire.

## Spring re-encodes every image

Spring reads the format from the bytes, refuses anything that is not JPEG, PNG, GIF or
WebP, refuses a header declaring more than 40 megapixels before decoding it (a few hundred
bytes of PNG can declare a canvas that decodes to gigabytes), draws the first frame onto a
plain ARGB canvas and encodes that as WebP. Handing the encoder pixels rather than the file
is what strips EXIF; there is no metadata-scrubbing step to get wrong.

Animated GIFs keep only their first frame. Animated WebP output was possible but heavier to
encode and to serve, and nothing in a post about lyrics needs it.

The encoder is `webp-imageio`, which ships libwebp for linux x86_64 and aarch64 inside its
jar. Fly runs amd64 and the later ECS plan runs Graviton; neither needs a Dockerfile change.
It is pinned below its latest release because later versions depend on a newer
`kotlin-stdlib` than this build compiles against.

## Images are served by R2, never by Spring

The bucket is public on a custom domain. An image view never reaches a Fly machine, so
reading a feed full of images cannot wake a suspended Spring app, and R2's free egress
covers the traffic. Keys are random UUIDs and never reused, so objects are served
`immutable`.

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
WebP sizes that is far inside R2's 10 GB free tier. Adding a sweeper would mean a second
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
