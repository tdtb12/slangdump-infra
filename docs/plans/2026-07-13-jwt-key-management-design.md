# JWT Key Management and Parameter Store Readiness

**Date:** 2026-07-13
**Status:** Approved, pending implementation
**Repos affected:** `slangdump` (root infra), `slangdump-pick-n-roll` (backend), `slangdump-poster` (frontend)

## Context

The Next.js BFF signs a 2-minute RS256 JWT with an RSA private key it alone holds
(`slangdump-poster/src/lib/bffToken.ts`). Spring acts as an OAuth2 resource server and
verifies that token with only the matching public key (`SecurityConfig.kt`) — it can
validate identity but never mint tokens. This asymmetry is deliberate and is preserved.

Two new RSA-2048 keypairs were generated into `jwt-keys/`: a dev pair and a production
pair. Investigation found three things worth stating plainly, because they drive the whole
design:

1. **`jwt-keys/` was not gitignored.** The production private key was one `git add .` away
   from being committed permanently. Nothing had been committed yet. *(Fixed ahead of this
   spec; see "Already done".)*

2. **Neither new keypair is the pair the system runs on today.** A third, older dev pair
   (fingerprint `2431440862…`) is currently wired up: its public half is committed at
   `src/main/resources/keys/jwt-dev-public.pem`, its private half at
   `src/test/resources/keys/jwt-dev-private.pem` (used by `TestJwt.kt`), and the same
   private key sits in the frontend's `.env.local`. Adopting the new dev pair therefore
   means changing all three in lockstep — a partial swap breaks authentication.

3. **The backend is not Parameter-Store-native.** ECS resolves Parameter Store entries into
   **environment variables**, but `SecurityConfig` reads the public key from a Spring
   `Resource` *location* (`classpath:` / `file:`). A location string cannot carry PEM
   content, so the backend cannot consume a Parameter Store value as it stands. This is the
   substantive engineering work in this spec.

Key fingerprints (SHA-256 of the DER-encoded public key):

| Keypair | Fingerprint | Disposition |
|---|---|---|
| New dev (`jwt-private-dev.pem` / `jwt-public-dev.pem`) | `e5e5ec8c…` | Adopt |
| New prod (`jwt-private.pem` / `jwt-public.pem`) | `0ebca568…` | Upload to Parameter Store |
| Old dev, currently in use | `2431440862…` | Retire |

## Goals

- Adopt the new dev keypair across both repos without breaking local dev or tests.
- Make the backend able to receive its public key from an environment variable, so that a
  future AWS deployment can supply it from SSM Parameter Store with no further code change.
- Keep the production private key out of git and off the laptop once AWS is provisioned.
- Leave the documented secrets convention accurate rather than aspirational.

## Non-goals

- Provisioning AWS. No Terraform, ECS task definitions, or IAM policies are created here.
  This spec makes the *application* ready; the infrastructure is a later, separate piece of
  work. What this spec does produce is the exact parameter names, types, and env-var
  bindings that infrastructure will have to honour.
- Dual-key (zero-downtime) rotation. Explicitly deferred — see "Rotation".
- Any change to how identity claims are produced or consumed.

## Design

### Only the private key is a secret

The public key is, by definition, safe to disclose. This is why the two keys are treated
differently below: the private key is a `SecureString` encrypted with a customer-managed KMS
key, while the public key is a plain `String`. Storing the public key as a `SecureString`
would pay KMS decrypt calls and key-policy complexity to protect a value that is public by
design. It is still kept *in* Parameter Store rather than baked into the JAR, because it
must stay in lockstep with the private key and must be replaceable by a config change and
restart rather than an artifact rebuild.

### Key inventory and lifecycle

**Dev pair (`e5e5ec8c…`) — non-sensitive, committed.** Its whole purpose is that every
developer and CI runner can sign and verify tokens with zero setup, so it is committed
deliberately, exactly as the old dev pair was. Three files change as one atomic commit set:

| File | Repo | Content |
|---|---|---|
| `src/main/resources/keys/jwt-dev-public.pem` | backend | new dev **public** key |
| `src/test/resources/keys/jwt-dev-private.pem` | backend | new dev **private** key (read by `TestJwt.kt`) |
| `.env.local` → `AUTH_JWT_PRIVATE_KEY` | frontend | new dev **private** key, PEM |

These three are a matched set. They must land together; a partial swap yields a signer and a
verifier that disagree, and every authenticated request 401s.

**Prod pair (`0ebca568…`) — private half is secret, never committed.** `jwt-keys/` is
gitignored and serves as a staging area only. At provisioning time both halves are uploaded
to Parameter Store, after which `jwt-keys/jwt-private.pem` is deleted from the laptop and
Parameter Store becomes the single source of truth.

### Backend: accept a PEM from an environment variable

`SecurityConfig` gains a second, higher-precedence property. `auth.jwt.public-key` carries
raw PEM **content** (bound to `AUTH_JWT_PUBLIC_KEY`); the existing
`auth.jwt.public-key-location` continues to carry a `Resource` **location** and continues to
default to the committed dev key.

Resolution rule, in order:

1. If `auth.jwt.public-key` is non-blank, parse it as PEM and use it. *(production)*
2. Otherwise load `auth.jwt.public-key-location`. *(local dev, tests — zero configuration)*

Rejected alternatives: a container entrypoint that writes the env var to a file and points
the existing `file:` location at it (hides the mechanism in infrastructure, and round-trips
the key through the container filesystem); and a single property that sniffs whether its own
value is a PEM or a location (one name meaning two things is the kind of cleverness that
makes the next reader stop and squint).

`application.yml`:

```yaml
auth:
  jwt:
    # Raw SPKI PEM. Set in production from SSM Parameter Store via the ECS `secrets:` block.
    # When blank, the key is loaded from public-key-location instead.
    public-key: ${AUTH_JWT_PUBLIC_KEY:}
    public-key-location: ${AUTH_JWT_PUBLIC_KEY_LOCATION:classpath:keys/jwt-dev-public.pem}
```

### Backend: guard against the dev-key fallback in production

The resolution rule above means a blank `AUTH_JWT_PUBLIC_KEY` in production — a missing SSM
parameter, a typo'd ARN, a misconfigured ECS task definition — resolves silently to the
committed dev key. Because the dev key's private half is also committed (in the backend's
test resources), that failure mode is not a crash: the app boots healthy and keeps serving
traffic while verifying tokens against a key anyone who can read the repo can sign with.

`ProductionKeyGuard` (`com.example.demo.config.ProductionKeyGuard`) closes this: at bean
construction it re-resolves the public key exactly as `SecurityConfig` would, and throws —
failing application startup outright — if that key is the dev key. It is annotated
`@Configuration @Profile("prod")`, so it is inert in local dev and in the test suite, both of
which rely on the dev-key fallback being reachable, and only active — and load-bearing — when
the Spring `prod` profile is active for the process.

That makes `SPRING_PROFILES_ACTIVE=prod` itself a required, security-relevant setting rather
than an operational nicety: without it, `ProductionKeyGuard` does not exist as a bean and its
check never runs, and the silent downgrade described above is possible again exactly as if
the guard had never been written. The backend's `Dockerfile` now sets
`SPRING_PROFILES_ACTIVE=prod` as the image default for this reason, so a deployment cannot
accidentally run unguarded just because a task definition forgot to set it. The root
`docker-compose.yml` explicitly overrides that back to a non-`prod` profile for local
development, since local compose does not supply `AUTH_JWT_PUBLIC_KEY` and needs the dev-key
fallback to stay reachable.

### Backend: extract PEM parsing

`SecurityConfig` currently does two unrelated jobs — HTTP security configuration and
hand-rolled PEM parsing (header stripping, base64 decode, `X509EncodedKeySpec`). Both
resolution paths above need that parsing, so it is extracted into a small unit with one
responsibility:

- **`RsaPublicKeyReader`** — `fromPem(pem: String): RSAPublicKey`. Depends on nothing but the
  JDK, is callable without a Spring context, and is therefore directly unit-testable, which
  the current inline parsing is not.

`SecurityConfig` keeps its `jwtDecoder` bean and its filter chain, and delegates parsing.
No change to the security rules themselves (reads public; post-create and user-sync
authenticated).

### Frontend: no code change

`bffToken.ts` already reads its PEM from the `AUTH_JWT_PRIVATE_KEY` environment variable and
already tolerates single-line values with `\n` escapes — it is Parameter-Store-native as
written. It needs only:

- the new dev private key swapped into `.env.local` (value only), and
- a new committed `.env.example` documenting `AUTH_JWT_PRIVATE_KEY` with an empty value, so
  the requirement is discoverable.

The frontend's `.gitignore` currently ignores `.env*`, which would swallow `.env.example`. It
gains a negation rather than relying on `git add -f`, so the file cannot be silently dropped
by a future contributor:

```gitignore
.env*
!.env.example
```

### Parameter Store layout

Follows the existing `/slangdump/{env}/{key_lowercase}` convention from
`slangdump-ai-solation/docs/secrets-management.md`.

| Parameter | Type | Consumed by | Env var |
|---|---|---|---|
| `/slangdump/prod/auth_jwt_private_key` | `SecureString` (customer-managed KMS key) | frontend task | `AUTH_JWT_PRIVATE_KEY` |
| `/slangdump/prod/auth_jwt_public_key` | `String` | backend task | `AUTH_JWT_PUBLIC_KEY` |

Both are delivered by the ECS task definition `secrets:` block, which resolves the parameter
ARN at task start and sets the environment variable inside the container. No application code
runs against the AWS SDK; nothing needs to change in the app to move between environments.

The ECS **task execution role** (not the task role — the agent resolves secrets before the
container starts) requires `ssm:GetParameters` on `/slangdump/prod/*` and `kms:Decrypt` on
the CMK protecting the private key.

### Rotation

Dual-key verification was considered and **deliberately deferred**. The backend accepts
exactly one public key, so rotation is a coordinated deploy:

1. Update both parameters in Parameter Store.
2. Restart the backend and frontend tasks together.
3. Authentication fails for the overlap window — bounded by the 2-minute token TTL plus task
   restart time.

This is a real, accepted cost: because the two services deploy as separate ECS tasks, there
is a window in which one side has rotated and the other has not, and every request 401s.

`slangdump-ai-solation/docs/secrets-management.md` does not exist on this branch's base — it
was only ever added on the unrelated `feat/lyrics-translation-chain` branch, never on
`master`. That branch's version lists a planned `JWT_SECRET` (stale HS256-era thinking; the
system uses an RS256 keypair) and promises that the application "must accept the previous key
for verification during rotation window — plan dual-key support into the auth implementation
from day one." That promise will not be true, so the doc is not simply carried over as-is: it
is created on this branch — recovered from that other branch's content as a starting point —
and rewritten to describe the real procedure above, list the two real parameters, and record
dual-key verification as the known upgrade path for zero-downtime rotation, rather than a
promise. A security doc that overstates the system's guarantees is worse than a blunt one.

## Testing

- **The existing backend integration tests must stay green against the new dev key.** This is
  the primary evidence that the three-file lockstep swap is complete and consistent: they
  sign with the committed test private key and verify against the committed public key, so
  they fail loudly if any one of the three files is missed.
- `TestJwt.tokenSignedByWrongKey` must still be rejected — proves the resource server has not
  become permissive during the refactor.
- **New:** unit tests for `RsaPublicKeyReader.fromPem` — valid PEM, whitespace/newline
  variations, malformed input.
- **New:** a test asserting `auth.jwt.public-key` takes precedence over
  `auth.jwt.public-key-location` when both are set. That precedence rule *is* the entire
  production code path, and it must not be the one thing that ships untested.
- Manual: sign in through the frontend against a locally running backend and create a post,
  confirming an end-to-end token round-trip on the new dev key.

## Already done (ahead of this spec)

The gitignore hole was closed immediately rather than waiting for implementation, since it
was actively exposing the production private key and any commit to this repo risked it.
`jwt-keys/` and `*.pem` are now ignored in the root repo; `git check-ignore` confirms all
three private keys are ignored and `jwt-keys/` no longer appears in `git status`. The rule is
scoped to the root repo, so the backend — a separate repo — can still commit the dev keys it
needs.

## Risks

- **Partial dev-key swap.** The failure mode is total auth failure locally, which the
  integration tests catch immediately. Mitigated by treating the three files as one change.
- **Prod private key on the laptop until AWS exists.** Accepted for MVP. The key is
  gitignored now and deleted after upload. If AWS provisioning slips a long way out, prefer
  regenerating the prod pair at deploy time from a trusted environment rather than letting
  this file age on a developer machine.
- **Rotation gap.** Accepted, documented above, with dual-key as the escape hatch if the
  downtime ever becomes unacceptable.
