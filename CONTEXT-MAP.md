# Context Map

Three services, each its own git repo, sitting side by side under a docs repo at the root.
Two of them keep a glossary; the third borrows the root one.

## Contexts

- [Slangdump](./CONTEXT.md) — the domain shared by every service: what a Song, a Post, an
  Album and a translation *are*. Owned by the Spring API (`slangdump-pick-n-roll`) and the
  AI worker (`slangdump-ai-solation`), neither of which keeps a glossary of its own.
- [Slangdump Poster](./slangdump-poster/CONTEXT.md) — the reader-facing Next.js app. Names
  the things a reader encounters that the domain has no opinion about: reading layouts,
  Follow Mode, Locale, and what a Permalink publishes.

## Relationships

- **Poster → Spring API**: the frontend is a BFF. The browser talks only to Next route
  handlers under `/api`, which mint a short-lived RS256 token and forward to
  `/api/v1/...`. The browser never reaches Spring directly.
- **Spring API → AI worker**: synchronous HTTP over Fly private networking. Spring absorbs
  the 20–60s translation on a background thread so no caller ever waits on it.
- **Shared vocabulary**: `Post`, `Song`, `Album` and `Canonical Line` mean the same thing in
  both glossaries and are wire-level contracts. `Canonical Line` is the load-bearing one —
  both sides must normalise identically or no stored translation will ever match a reader's
  paste.
- **Divergence to know about**: the root glossary lists **Post** with _Avoid: Drop (in
  code)_, while the frontend names the editor a **Daily Drop**. Deliberate — the *act* is a
  Daily Drop, the *thing* is a Post.
