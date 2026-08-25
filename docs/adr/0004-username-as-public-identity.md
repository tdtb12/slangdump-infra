# The Username is the only public identity

Every User has a Username: unique across the site, lowercase, 3–20 characters of
`[a-z0-9_]`, chosen by the User and shown as `@name`. It is the byline on every
Post, resolved by a live join at read time, so a rename rewrites the name on all
of that User's past drops at once.

It replaces the name the OAuth provider supplied — `users.display_name` is
renamed to `users.username` rather than joined by a second column, and the
provider's name is no longer stored at all. This was the bug that forced the
decision: sign-in refreshed `display_name` from the provider every time, so a
name the User had chosen silently reverted at their next login. The name branch
of `refreshProfile` is deleted rather than gated, because `陳大文` is no longer a
legal value for the column.

A new User does not choose their first Username. It is derived from their email
local-part — sanitised the same way, with a numeric suffix on collision — and
they are redirected once to `/profile` to change it if they want to. Renames are
then limited to one per rolling 24 hours, tracked in `username_changed_at`,
which starts `NULL` so the first change is free and a typo at that first prompt
is correctable.

## Considered options

**A separate handle beside the display name.** Recommended, and rejected by the
user. Two names means every read site picks one, every screen has to decide, and
the two drift — a User who renames their handle and not their display name is
findable under one name and shown under another. One name is one answer.

**Case-insensitive uniqueness with the typed case preserved**, so `AdaLovelace`
displays as typed but cannot coexist with `adalovelace`. Also recommended, also
rejected. It needs a functional unique index on `lower(username)` and a second,
raw column's worth of care at every comparison. Forcing lowercase on save makes
the plain `UNIQUE` constraint the whole rule, at the cost of a name nobody can
capitalise. Note this deliberately contradicts the project's other normalisation
rule — a Canonical Line is NFKC-normalised with case preserved — because that
one exists so a hash matches a reader's paste, and this one exists so two
Usernames cannot collide.

**Quarantining a vacated Username.** Rejected in favour of releasing it the
instant it is given up. A quarantine needs a `retired_usernames` table, an
expiry, and a sweeper, to protect an audience of a few dozen users.

## Consequences

**Instant release is an impersonation surface, and it is accepted.** Somebody
who renames away from `@ada` frees the name for anyone to take, and the taker
then owns the byline on their own future drops under a name readers associate
with someone else. The only thing standing in the way is the 24h rename limit,
which caps how fast a squatter can cycle. This gets worse, not better, once
`/@username` becomes a public profile page — revisit it then, not now.

**The provider's human name is gone.** Nothing displays "Ada Lovelace" anymore,
and nothing can: the column that held it now holds something with a different
shape and a different meaning. Restoring it means a new column and a new
migration, not a revert.

**Renames are retroactive by construction.** The byline was already a live join
on the author row, so nothing had to change to make this work — which also means
there is no way to opt a Post out of it. A drop cannot keep the name it was
published under.

**The existing production rows were rewritten in place.** V8 derives a Username
from each `display_name` — trimmed, lowercased, spaces to underscores, illegal
characters dropped, collisions suffixed — inside the same transaction that adds
the constraint, so a derivation that fails the CHECK aborts the deploy instead
of half-applying. Those users were not consulted about their new name; they can
change it, once, for free.
