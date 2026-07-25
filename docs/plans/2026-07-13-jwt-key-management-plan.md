# JWT Key Management & Parameter Store Readiness — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adopt the new dev RSA keypair across the frontend and backend, and make the Spring backend able to load its JWT public key from an environment variable so production can supply it from AWS SSM Parameter Store with no further code change.

**Architecture:** The Next.js BFF signs a 2-minute RS256 JWT with a private key; Spring verifies it as an OAuth2 resource server with the matching public key. The frontend already reads its PEM from an env var and needs no code change. The backend reads its public key from a Spring `Resource` *location*, which cannot carry PEM *content* — so it gains a higher-precedence `auth.jwt.public-key` property holding a raw PEM, with the existing classpath location retained as the zero-config default for local dev and tests. PEM parsing is extracted from `SecurityConfig` into a standalone `RsaPublicKeyReader` shared by both paths.

**Tech Stack:** Kotlin 2.1 / Java 21, Spring Boot 4.0.6 (`spring-boot-starter-oauth2-resource-server`, Nimbus), Gradle, JUnit 5 + `kotlin.test`, Testcontainers; Next.js with `jose`.

**Design doc:** `docs/plans/2026-07-13-jwt-key-management-design.md`

## Global Constraints

- **Repo paths.** Root: `/Users/jimhsu/Documents/slangdump`. Backend: `slangdump-pick-n-roll/`. Frontend: `slangdump-poster/`. AI worker: `slangdump-ai-solation/`. **These are three separate git repos** — the sub-directories are gitignored by the root repo. Every `git commit` must be run from inside the correct repo, and no commit spans repos.
- **The three dev-key files are one atomic set.** Backend `src/main/resources/keys/jwt-dev-public.pem`, backend `src/test/resources/keys/jwt-dev-private.pem`, and frontend `.env.local` → `AUTH_JWT_PRIVATE_KEY` must all hold the *same* new dev keypair. A partial swap makes the signer and verifier disagree and every authenticated request 401s.
- **Key fingerprints** (SHA-256 of DER public key), for verifying you touched the right files:
  - New dev pair (adopt): `e5e5ec8c7325b77716ff8642a9607b297679d4458d383304cb22af27cdab2049`
  - New prod pair (Parameter Store only, never committed): `0ebca568f95bcddeb7f2e99017175741a08ce15cde28f260838fa8834cb34d04`
  - Old dev pair (retire, must appear nowhere when done): `2431440862a8140c67e27e37e7ddbc43a59e868c2839394aa18723180be55803`
- **Never commit `jwt-keys/`, `jwt-private.pem`, or the production key in any form.** The root repo already ignores `jwt-keys/` and `*.pem`. The *dev* private key is committed to the backend on purpose — it is a throwaway key whose entire job is zero-setup local tests. The *production* private key is a secret and lives only in Parameter Store.
- **Every JUnit test gets a `@DisplayName`.** This is a standing project convention; match the existing style in `PostControllerIntegrationTest.kt` — a `@DisplayName` describing behavior plus a backticked function name.
- **Backend tests need Docker running** (Testcontainers starts a real `postgres:16`). The two new test classes in Task 3 are pure unit tests with no Spring context and no Docker requirement.
- **Do not add dual-key rotation.** It was explicitly deferred. The backend verifies with exactly one public key.

---

### Task 1: Adopt the new dev keypair in the backend

Swaps both halves of the backend's dev keypair together. The existing integration tests are the proof: they sign with the committed test private key and verify against the committed public key, so they pass only if both files are the matched new pair.

**Files:**
- Modify (replace contents): `slangdump-pick-n-roll/src/main/resources/keys/jwt-dev-public.pem`
- Modify (replace contents): `slangdump-pick-n-roll/src/test/resources/keys/jwt-dev-private.pem`

**Interfaces:**
- Consumes: the generated keypair in `/Users/jimhsu/Documents/slangdump/jwt-keys/` (gitignored, local only).
- Produces: a backend whose committed dev keypair is `e5e5ec8c…`. `TestJwt` loads the private key from the classpath at `/keys/jwt-dev-private.pem` and `SecurityConfig` loads the public key from `classpath:keys/jwt-dev-public.pem` — **neither needs a code change**, only the file contents change.

- [ ] **Step 1: Confirm the tests currently pass on the OLD key**

This establishes a green baseline, so that a failure after the swap unambiguously means the swap is wrong rather than something being pre-broken. Docker must be running.

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test
```

Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 2: Swap both dev key files**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
cp ../jwt-keys/jwt-public-dev.pem  src/main/resources/keys/jwt-dev-public.pem
cp ../jwt-keys/jwt-private-dev.pem src/test/resources/keys/jwt-dev-private.pem
```

- [ ] **Step 3: Verify the two files are a matched pair, and are the NEW pair**

Both commands must print the same fingerprint, and it must be the new dev fingerprint `e5e5ec8c…`. If they differ from each other, you copied mismatched files; if they equal `2431…`, the copy silently didn't happen.

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
echo -n "public : "; openssl pkey -pubin -in src/main/resources/keys/jwt-dev-public.pem -pubout -outform DER | openssl sha256 | awk '{print $2}'
echo -n "private: "; openssl pkey -in src/test/resources/keys/jwt-dev-private.pem -pubout -outform DER | openssl sha256 | awk '{print $2}'
```

Expected — both lines identical:
```
public : e5e5ec8c7325b77716ff8642a9607b297679d4458d383304cb22af27cdab2049
private: e5e5ec8c7325b77716ff8642a9607b297679d4458d383304cb22af27cdab2049
```

- [ ] **Step 4: Run the full test suite against the new key**

This is the real gate. `PostControllerIntegrationTest` mints tokens with the new private key and the resource server verifies them with the new public key; it also asserts `TestJwt.tokenSignedByWrongKey()` is still rejected, proving verification did not become permissive.

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test
```

Expected: `BUILD SUCCESSFUL`. If you see `401` failures in `PostControllerIntegrationTest`, the two PEM files are not a matched pair — go back to Step 3.

- [ ] **Step 5: Commit**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
git add src/main/resources/keys/jwt-dev-public.pem src/test/resources/keys/jwt-dev-private.pem
git commit -m "chore(auth): adopt new dev JWT keypair

Both halves swap together; the integration tests verify the signer and
verifier still agree. Dev-only key, committed deliberately so local runs
and CI need zero setup. Production uses a separate key from Parameter Store."
```

---

### Task 2: Complete the lockstep swap in the frontend, and document the env var

Until this lands, the backend trusts the new dev key while the frontend still signs with the old one, so local sign-in is broken. This task closes that. It also adds the `.env.example` the repo never had, so the `AUTH_JWT_PRIVATE_KEY` requirement is discoverable rather than tribal knowledge.

**Files:**
- Modify: `slangdump-poster/.env.local` (value of `AUTH_JWT_PRIVATE_KEY` only — this file is gitignored and is **not** committed)
- Create: `slangdump-poster/.env.example`
- Modify: `slangdump-poster/.gitignore:34`

**Interfaces:**
- Consumes: `jwt-keys/jwt-private-dev.pem` — the same private key Task 1 committed to the backend's test resources.
- Produces: no code change. `src/lib/bffToken.ts` already reads `process.env.AUTH_JWT_PRIVATE_KEY` and already converts single-line `\n`-escaped PEMs via `pem.replace(/\\n/g, '\n')`. It is Parameter-Store-native as written; only the *value* changes.

- [ ] **Step 1: Render the new dev private key as a single-line `\n`-escaped PEM**

`.env.local` stores the PEM on one line with escaped newlines, which is the format `bffToken.ts` already handles and the format an ECS-injected env var can carry.

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-poster
perl -pe 's/\n/\\n/' ../jwt-keys/jwt-private-dev.pem
```

Expected: one long line beginning `-----BEGIN PRIVATE KEY-----\nMII…` and ending `…\n-----END PRIVATE KEY-----\n`.

- [ ] **Step 2: Replace the `AUTH_JWT_PRIVATE_KEY` value in `.env.local`**

Edit `slangdump-poster/.env.local` and replace the **entire existing** `AUTH_JWT_PRIVATE_KEY="…"` line with the Step 1 output wrapped in double quotes:

```
AUTH_JWT_PRIVATE_KEY="<paste the single line from Step 1 here>"
```

Keep the surrounding comment lines. Do not commit this file — it is gitignored.

- [ ] **Step 3: Verify the frontend's key now matches the backend's**

The private key in `.env.local` must derive the same public key the backend verifies with.

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-poster
grep '^AUTH_JWT_PRIVATE_KEY' .env.local \
  | sed 's/^AUTH_JWT_PRIVATE_KEY="//; s/"$//' \
  | perl -pe 's/\\n/\n/g' \
  | openssl pkey -pubout -outform DER | openssl sha256 | awk '{print $2}'
```

Expected: `e5e5ec8c7325b77716ff8642a9607b297679d4458d383304cb22af27cdab2049` — identical to the backend fingerprint from Task 1, Step 3. Any other value means the paste was truncated or mangled.

- [ ] **Step 4: Allow `.env.example` past the ignore rule**

`.gitignore:34` currently ignores `.env*`, which would silently swallow `.env.example`. Add a negation rather than relying on `git add -f`, so a future contributor cannot drop the file by accident.

In `slangdump-poster/.gitignore`, find:

```gitignore
# env files (can opt-in for committing if needed)
.env*
```

and change it to:

```gitignore
# env files (can opt-in for committing if needed)
.env*
!.env.example
```

- [ ] **Step 5: Create `.env.example`**

Create `slangdump-poster/.env.example` with **empty values** — this file is committed, so it must never carry real key material:

```bash
# Copy to .env.local and fill in. Never commit .env.local.

# RSA private key (PKCS#8 PEM) the BFF uses to sign the short-lived RS256 token that the
# Spring backend verifies with the matching public key. Single line, newlines escaped as \n.
# Local dev: use the dev private key from jwt-keys/jwt-private-dev.pem — it is a throwaway
# key, and its public half is committed to the backend so no setup is needed.
# Production: injected from SSM Parameter Store (/slangdump/prod/auth_jwt_private_key,
# SecureString) by the ECS task definition `secrets:` block.
AUTH_JWT_PRIVATE_KEY=
```

- [ ] **Step 6: Verify `.env.example` is tracked and `.env.local` is still ignored**

This guards the one thing that would be a genuine incident: committing the real key.

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-poster
git check-ignore -v .env.local        # expect: a match — still ignored
git status --short .env.example       # expect: ?? .env.example — now visible to git
```

Expected: `.env.local` prints an ignore rule; `.env.example` shows as untracked (not ignored).

- [ ] **Step 7: Manually verify the end-to-end token round-trip**

The unit tests cannot prove the frontend and backend agree, because they live in different repos. Start the backend (Docker + Postgres running) and the frontend, sign in with Google, and create a post.

```bash
# terminal 1
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll && ./gradlew bootRun
# terminal 2
cd /Users/jimhsu/Documents/slangdump/slangdump-poster && npm run dev
```

Expected: sign-in succeeds and creating a post returns 2xx. A `401` on post-create means the two keys still disagree — recheck Step 3. Stop both servers when done.

- [ ] **Step 8: Commit**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-poster
git add .gitignore .env.example
git commit -m "chore(auth): document AUTH_JWT_PRIVATE_KEY, add .env.example

Adopt the new dev signing key locally (.env.local, not committed) to match
the dev public key the backend now verifies with. bffToken.ts needs no
change: it already reads the PEM from an env var, which is exactly how
Parameter Store will deliver it in production."
```

---

### Task 3: Let the backend load its public key from an environment variable

The core change. ECS resolves Parameter Store entries into **environment variables**, but `SecurityConfig` reads a Spring `Resource` *location* — and no location string can express "the PEM sitting in `AUTH_JWT_PUBLIC_KEY`". Without this, the Parameter Store requirement is unmeetable regardless of how the infrastructure is written.

PEM parsing moves out of `SecurityConfig` (which today does both HTTP security config and hand-rolled base64/`X509EncodedKeySpec` decoding) into `RsaPublicKeyReader`, which has one job, needs no Spring context, and is therefore directly unit-testable — which the inline parsing was not.

**Files:**
- Create: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/config/RsaPublicKeyReader.kt`
- Modify: `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/config/SecurityConfig.kt`
- Modify: `slangdump-pick-n-roll/src/main/resources/application.yml`
- Test (create): `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/config/RsaPublicKeyReaderTest.kt`
- Test (create): `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/config/SecurityConfigKeyResolutionTest.kt`

**Interfaces:**
- Consumes: nothing from Tasks 1–2 beyond the committed dev public key at `classpath:keys/jwt-dev-public.pem`, which remains the default.
- Produces:
  - `object RsaPublicKeyReader { fun fromPem(pem: String): RSAPublicKey }` — throws `IllegalArgumentException` on malformed input. Tolerates literal `\n` escape sequences in the PEM, mirroring the frontend, so a `\n`-escaped Parameter Store value works.
  - `SecurityConfig(publicKeyPem: String, publicKeyResource: Resource)` with `internal fun resolvePublicKey(): RSAPublicKey` — `publicKeyPem` (non-blank) wins over `publicKeyResource`.
  - New property `auth.jwt.public-key`, bound to env var `AUTH_JWT_PUBLIC_KEY`, default empty.

- [ ] **Step 1: Write the failing tests for `RsaPublicKeyReader`**

Create `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/config/RsaPublicKeyReaderTest.kt`. Note `src/main/resources` is on the test classpath, so the committed dev public key is readable here — that makes the first test assert against the *real* key the app ships with, not a synthetic one.

```kotlin
package com.example.demo.config

import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import java.security.KeyPairGenerator
import java.security.interfaces.RSAPublicKey
import java.util.Base64
import kotlin.test.assertEquals
import kotlin.test.assertFailsWith

class RsaPublicKeyReaderTest {

    private fun generatedKey(): RSAPublicKey =
        (KeyPairGenerator.getInstance("RSA").apply { initialize(2048) }
            .generateKeyPair().public) as RSAPublicKey

    private fun toPem(key: RSAPublicKey): String {
        val body = Base64.getMimeEncoder(64, "\n".toByteArray()).encodeToString(key.encoded)
        return "-----BEGIN PUBLIC KEY-----\n$body\n-----END PUBLIC KEY-----\n"
    }

    @Test
    @DisplayName("parses the committed dev public key the application ships with")
    fun `parses the committed dev public key`() {
        val pem = javaClass.getResourceAsStream("/keys/jwt-dev-public.pem")!!
            .use { it.readBytes().decodeToString() }

        val key = RsaPublicKeyReader.fromPem(pem)

        assertEquals(2048, key.modulus.bitLength())
    }

    @Test
    @DisplayName("round-trips a PEM back to the same RSA key")
    fun `round-trips a PEM back to the same key`() {
        val original = generatedKey()

        val parsed = RsaPublicKeyReader.fromPem(toPem(original))

        assertEquals(original.modulus, parsed.modulus)
        assertEquals(original.publicExponent, parsed.publicExponent)
    }

    @Test
    @DisplayName("parses a single-line PEM whose newlines are escaped as backslash-n")
    fun `parses a PEM with escaped newlines`() {
        val original = generatedKey()
        val escaped = toPem(original).replace("\n", "\\n")

        val parsed = RsaPublicKeyReader.fromPem(escaped)

        assertEquals(original.modulus, parsed.modulus)
    }

    @Test
    @DisplayName("rejects a PEM containing no key material")
    fun `rejects an empty PEM`() {
        assertFailsWith<IllegalArgumentException> {
            RsaPublicKeyReader.fromPem("-----BEGIN PUBLIC KEY-----\n-----END PUBLIC KEY-----")
        }
    }

    @Test
    @DisplayName("rejects input that is not a valid RSA public key")
    fun `rejects malformed input`() {
        assertFailsWith<IllegalArgumentException> {
            RsaPublicKeyReader.fromPem("-----BEGIN PUBLIC KEY-----\nbm90IGEga2V5\n-----END PUBLIC KEY-----")
        }
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test --tests 'com.example.demo.config.RsaPublicKeyReaderTest'
```

Expected: compilation failure — `Unresolved reference: RsaPublicKeyReader`. That is the correct "red".

- [ ] **Step 3: Implement `RsaPublicKeyReader`**

Create `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/config/RsaPublicKeyReader.kt`:

```kotlin
package com.example.demo.config

import java.security.GeneralSecurityException
import java.security.KeyFactory
import java.security.interfaces.RSAPublicKey
import java.security.spec.X509EncodedKeySpec
import java.util.Base64

/**
 * Parses an SPKI ("BEGIN PUBLIC KEY") PEM into an RSA public key.
 *
 * Kept out of [SecurityConfig] so parsing is testable without a Spring context, and so the
 * two key sources — the AUTH_JWT_PUBLIC_KEY env var (production, via SSM Parameter Store)
 * and the committed classpath dev key (local, tests) — share one parser.
 */
object RsaPublicKeyReader {

    fun fromPem(pem: String): RSAPublicKey {
        // An env-injected PEM may arrive on a single line with newlines escaped; the BFF
        // accepts the same form for its private key.
        val base64 = pem
            .replace("\\n", "\n")
            .replace("-----BEGIN PUBLIC KEY-----", "")
            .replace("-----END PUBLIC KEY-----", "")
            .replace(Regex("\\s"), "")

        require(base64.isNotEmpty()) { "PEM contains no key material" }

        val der = try {
            Base64.getDecoder().decode(base64)
        } catch (e: IllegalArgumentException) {
            throw IllegalArgumentException("PEM body is not valid base64", e)
        }

        return try {
            KeyFactory.getInstance("RSA")
                .generatePublic(X509EncodedKeySpec(der)) as RSAPublicKey
        } catch (e: GeneralSecurityException) {
            throw IllegalArgumentException("PEM is not a valid RSA public key", e)
        }
    }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test --tests 'com.example.demo.config.RsaPublicKeyReaderTest'
```

Expected: `BUILD SUCCESSFUL`, 5 tests passing.

- [ ] **Step 5: Write the failing test for key-source precedence**

This rule *is* the entire production code path — in production the env var must beat the classpath default, or the backend silently keeps trusting the dev key and rejects every real token. It must not be the one thing that ships untested.

`SecurityConfig` is a plain Kotlin class, so it can be constructed directly with no Spring context and no Docker.

Create `slangdump-pick-n-roll/src/test/kotlin/com/example/demo/config/SecurityConfigKeyResolutionTest.kt`:

```kotlin
package com.example.demo.config

import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import org.springframework.core.io.ByteArrayResource
import java.security.KeyPairGenerator
import java.security.interfaces.RSAPublicKey
import java.util.Base64
import kotlin.test.assertEquals

/**
 * The env-var PEM (production, from SSM Parameter Store) must take precedence over the
 * classpath location (the committed dev key). Getting this backwards would make production
 * verify with the dev key.
 */
class SecurityConfigKeyResolutionTest {

    private fun generatedKey(): RSAPublicKey =
        (KeyPairGenerator.getInstance("RSA").apply { initialize(2048) }
            .generateKeyPair().public) as RSAPublicKey

    private fun toPem(key: RSAPublicKey): String {
        val body = Base64.getMimeEncoder(64, "\n".toByteArray()).encodeToString(key.encoded)
        return "-----BEGIN PUBLIC KEY-----\n$body\n-----END PUBLIC KEY-----\n"
    }

    @Test
    @DisplayName("uses the env-var PEM when it is set, ignoring the key location")
    fun `env var PEM wins over the location`() {
        val envKey = generatedKey()
        val locationKey = generatedKey()

        val config = SecurityConfig(
            publicKeyPem = toPem(envKey),
            publicKeyResource = ByteArrayResource(toPem(locationKey).toByteArray()),
        )

        assertEquals(envKey.modulus, config.resolvePublicKey().modulus)
    }

    @Test
    @DisplayName("falls back to the key location when the env-var PEM is blank")
    fun `blank env var PEM falls back to the location`() {
        val locationKey = generatedKey()

        val config = SecurityConfig(
            publicKeyPem = "",
            publicKeyResource = ByteArrayResource(toPem(locationKey).toByteArray()),
        )

        assertEquals(locationKey.modulus, config.resolvePublicKey().modulus)
    }

    @Test
    @DisplayName("treats a whitespace-only env-var PEM as unset")
    fun `whitespace env var PEM falls back to the location`() {
        val locationKey = generatedKey()

        val config = SecurityConfig(
            publicKeyPem = "   \n  ",
            publicKeyResource = ByteArrayResource(toPem(locationKey).toByteArray()),
        )

        assertEquals(locationKey.modulus, config.resolvePublicKey().modulus)
    }
}
```

- [ ] **Step 6: Run the test to verify it fails**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test --tests 'com.example.demo.config.SecurityConfigKeyResolutionTest'
```

Expected: compilation failure — `SecurityConfig` has no `publicKeyPem` parameter and no `resolvePublicKey` function.

- [ ] **Step 7: Rewrite `SecurityConfig`**

Replace the whole of `slangdump-pick-n-roll/src/main/kotlin/com/example/demo/config/SecurityConfig.kt`. The security rules are unchanged — only key resolution changes, and the PEM parsing is delegated. Note the removed imports (`KeyFactory`, `X509EncodedKeySpec`, `Base64`): the hand-rolled parsing is gone.

```kotlin
package com.example.demo.config

import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.io.Resource
import org.springframework.http.HttpMethod
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.oauth2.jwt.JwtDecoder
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder
import org.springframework.security.web.SecurityFilterChain
import java.security.interfaces.RSAPublicKey

/**
 * Spring acts as an OAuth2 resource server. The Next.js BFF signs a short-lived RS256 JWT
 * (claims: provider, sub, email, name, picture) with an RSA private key it alone holds;
 * this backend verifies it with only the matching public key — it can validate identity
 * but never mint tokens.
 *
 * Reads are public (anyone can browse the feed). Writes — creating a post and syncing the
 * user profile — require a valid token; the entry point returns 401 otherwise.
 *
 * The public key comes from one of two places, in order:
 *   1. `auth.jwt.public-key` — a raw SPKI PEM. Production sets this from AWS SSM Parameter
 *      Store, which ECS delivers as the AUTH_JWT_PUBLIC_KEY environment variable.
 *   2. `auth.jwt.public-key-location` — a Spring resource location, defaulting to the
 *      committed dev key so local runs and tests need no configuration.
 * A location cannot carry PEM content, which is why (1) exists.
 */
@Configuration
class SecurityConfig(
    @Value("\${auth.jwt.public-key:}")
    private val publicKeyPem: String,
    @Value("\${auth.jwt.public-key-location:classpath:keys/jwt-dev-public.pem}")
    private val publicKeyResource: Resource,
) {
    @Bean
    fun jwtDecoder(): JwtDecoder =
        NimbusJwtDecoder.withPublicKey(resolvePublicKey()).build()

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/api/v1/posts", "/api/v1/posts/**").permitAll()
                it.requestMatchers(HttpMethod.POST, "/api/v1/posts").authenticated()
                it.requestMatchers(HttpMethod.POST, "/api/v1/users/sync").authenticated()
                it.anyRequest().permitAll()
            }
            .oauth2ResourceServer { rs -> rs.jwt { } }
        return http.build()
    }

    internal fun resolvePublicKey(): RSAPublicKey {
        val pem = if (publicKeyPem.isNotBlank()) {
            publicKeyPem
        } else {
            publicKeyResource.inputStream.use { it.readBytes().decodeToString() }
        }
        return RsaPublicKeyReader.fromPem(pem)
    }
}
```

- [ ] **Step 8: Add the new property to `application.yml`**

In `slangdump-pick-n-roll/src/main/resources/application.yml`, replace the existing `auth:` block:

```yaml
auth:
  jwt:
    # SPKI PEM public key used to verify the RS256 bearer token the Next.js BFF signs.
    # Defaults to the committed dev key; production sets a file: location with its own key.
    public-key-location: ${AUTH_JWT_PUBLIC_KEY_LOCATION:classpath:keys/jwt-dev-public.pem}
```

with:

```yaml
auth:
  jwt:
    # Raw SPKI PEM. Production sets this from SSM Parameter Store
    # (/slangdump/prod/auth_jwt_public_key), which ECS injects as AUTH_JWT_PUBLIC_KEY.
    # Takes precedence over public-key-location. Blank locally, so the dev key is used.
    public-key: ${AUTH_JWT_PUBLIC_KEY:}
    # Fallback: a resource location, defaulting to the committed dev key so local runs
    # and tests need no configuration.
    public-key-location: ${AUTH_JWT_PUBLIC_KEY_LOCATION:classpath:keys/jwt-dev-public.pem}
```

- [ ] **Step 9: Run the new tests, then the whole suite**

The full run matters: it proves the refactor did not weaken the real filter chain. `PostControllerIntegrationTest` boots the actual Spring context with no `AUTH_JWT_PUBLIC_KEY` set, so it exercises the fallback path, and still asserts that a token signed by an untrusted key is rejected.

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
./gradlew test --tests 'com.example.demo.config.*'
./gradlew test
```

Expected: both `BUILD SUCCESSFUL`.

- [ ] **Step 10: Verify the env var actually overrides the dev key at runtime**

Tests construct `SecurityConfig` directly; this proves the property is really wired to the environment variable end-to-end — the one thing a direct-construction test cannot show. Point the backend at the *production* public key and confirm a dev-signed token is now rejected.

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
AUTH_JWT_PUBLIC_KEY="$(cat ../jwt-keys/jwt-public.pem)" ./gradlew bootRun
```

In another terminal, send a create-post request with **no** token:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://localhost:8080/api/v1/posts \
  -H 'Content-Type: application/json' -d '{}'
```

Expected: `401`. And the startup log must show no key-parsing error — a `PEM is not a valid RSA public key` at boot means the env var is not being read correctly. Stop the server.

- [ ] **Step 11: Commit**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-pick-n-roll
git add src/main/kotlin/com/example/demo/config/RsaPublicKeyReader.kt \
        src/main/kotlin/com/example/demo/config/SecurityConfig.kt \
        src/main/resources/application.yml \
        src/test/kotlin/com/example/demo/config/RsaPublicKeyReaderTest.kt \
        src/test/kotlin/com/example/demo/config/SecurityConfigKeyResolutionTest.kt
git commit -m "feat(auth): load JWT public key from AUTH_JWT_PUBLIC_KEY

ECS resolves SSM Parameter Store entries into environment variables, but the
key was only loadable from a Spring resource location, which cannot carry PEM
content. Add auth.jwt.public-key (raw PEM, higher precedence) and keep the
classpath location as the zero-config default for local dev and tests.

Extract PEM parsing into RsaPublicKeyReader so both sources share one parser
and the parsing is unit-testable without a Spring context."
```

---

### Task 4: Correct the secrets documentation

`slangdump-ai-solation/docs/secrets-management.md` is the house convention doc, and it is currently wrong about auth in two ways: it lists a planned `JWT_SECRET` (stale HS256-era thinking — the system uses an RS256 *keypair*), and it promises the app "must accept the previous key for verification during rotation window — plan dual-key support into the auth implementation from day one." That promise will not be kept. A security doc that overstates the system's guarantees is worse than a blunt one.

**Files:**
- Modify: `slangdump-ai-solation/docs/secrets-management.md` (the "Stored secrets" table and the "Rotation" section)

**Interfaces:**
- Consumes: the parameter names and types established in Task 3 (`AUTH_JWT_PUBLIC_KEY`) and Task 2 (`AUTH_JWT_PRIVATE_KEY`).
- Produces: the authoritative parameter names, types, and env-var bindings that AWS provisioning must honour later. No code.

- [ ] **Step 1: Replace the `JWT_SECRET` row in the "Stored secrets" table**

Find:

```markdown
| `JWT_SECRET` | no (planned) | — | Optional until auth lands; rotation requires dual-key support |
```

Replace with:

```markdown
| `AUTH_JWT_PRIVATE_KEY` | yes (frontend) | `slangdump-poster` `src/lib/bffToken.ts` | RSA private key (PKCS#8 PEM). The BFF signs the RS256 token with it. **SecureString.** |
| `AUTH_JWT_PUBLIC_KEY` | yes (backend) | `slangdump-pick-n-roll` `SecurityConfig.kt` | RSA public key (SPKI PEM). Verify-only. Not a secret — stored as a plain `String`, no KMS. |
```

Then, immediately below the table, add:

```markdown
Auth uses an **RS256 keypair**, not a shared secret: the frontend BFF holds the private key
and signs; the backend holds only the public key and verifies, so it can validate identity
but never mint tokens. Only the private key is confidential — the public key is stored in
Parameter Store as a plain `String` (rather than baked into the JAR) purely so it stays in
lockstep with the private key and can be replaced by a config change and restart rather than
an artifact rebuild.

The **dev** keypair is deliberately committed (`jwt-dev-public.pem` in the backend's main
resources, `jwt-dev-private.pem` in its test resources) so local runs and CI need zero
setup. It is a throwaway key and is never used in production.
```

- [ ] **Step 2: Add the parameter names under "Parameter naming"**

In the "Production (AWS)" → "Parameter naming" section, extend the examples list:

```markdown
Examples:
- `/slangdump/prod/gemini_api_key`
- `/slangdump/staging/gemini_api_key`
- `/slangdump/prod/auth_jwt_private_key` — `SecureString`, consumed by the **frontend** task
- `/slangdump/prod/auth_jwt_public_key` — plain `String`, consumed by the **backend** task
```

Then add this subsection immediately after the existing `### IAM` section (the outer fence
below is quadruple-backtick only so the inner bash block survives copy-paste — the text you
paste into the doc starts at `### Uploading`):

````markdown
### Uploading the JWT keypair

Run once per environment, at provisioning time. The private key is generated locally in
`jwt-keys/` (gitignored) and **deleted from the laptop once uploaded** — Parameter Store is
then the single source of truth.

```bash
aws ssm put-parameter --name /slangdump/prod/auth_jwt_private_key \
  --type SecureString --key-id alias/slangdump \
  --value "$(cat jwt-keys/jwt-private.pem)"

aws ssm put-parameter --name /slangdump/prod/auth_jwt_public_key \
  --type String \
  --value "$(cat jwt-keys/jwt-public.pem)"

# Parameter Store now holds the only copy of the production private key.
rm jwt-keys/jwt-private.pem
```

The execution role needs `kms:Decrypt` only for the private key's CMK; the public key is a
plain `String` and needs no KMS grant.
````

- [ ] **Step 3: Replace the JWT bullet in the "Rotation" section**

Find:

```markdown
- `JWT_SECRET` (when introduced): application must accept the previous key for verification during rotation window. Plan dual-key support into the auth implementation from day one.
```

Replace with:

```markdown
- **JWT keypair**: the backend verifies with exactly **one** public key, so rotation is a
  coordinated deploy and **is not zero-downtime**. Update both parameters, then restart the
  backend and frontend tasks together. Authentication fails for the overlap window — bounded
  by the 2-minute token TTL plus task restart time — because the two services deploy as
  separate ECS tasks, so there is a period where one side has rotated and the other has not.

  Dual-key verification (backend accepting a previous key alongside the current one) is the
  known upgrade path if that gap ever becomes unacceptable. It was deliberately not built.
```

- [ ] **Step 4: Verify the stale references are gone**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-ai-solation
grep -n "JWT_SECRET\|dual-key support into the auth" docs/secrets-management.md
```

Expected: **no output**. Any hit means a stale HS256-era claim survived the edit.

- [ ] **Step 5: Commit**

```bash
cd /Users/jimhsu/Documents/slangdump/slangdump-ai-solation
git add docs/secrets-management.md
git commit -m "docs: correct JWT secrets and rotation for the RS256 keypair

The doc described a planned symmetric JWT_SECRET and promised dual-key
rotation support. Auth actually uses an RS256 keypair, and dual-key
verification was deliberately deferred — so rotation is a coordinated
deploy with a gap bounded by the 2-minute token TTL. Document what the
system really does, and record the real parameter names and types."
```

---

## Done when

- The old dev key fingerprint `2431440862…` appears nowhere in either repo.
- `./gradlew test` is green in the backend, including the two new test classes.
- Booting the backend with `AUTH_JWT_PUBLIC_KEY` set overrides the committed dev key; booting without it falls back to the dev key.
- Signing in on the frontend and creating a post works end-to-end on the new dev key.
- `slangdump-poster/.env.example` is committed; `.env.local` and `jwt-keys/` are still ignored.
- `secrets-management.md` documents the two real parameters and the real (non-zero-downtime) rotation procedure.

## Deferred to AWS provisioning day

Not in this plan — the app is made ready, the infrastructure is not built. When provisioning:
upload the two parameters (commands in `secrets-management.md`), delete
`jwt-keys/jwt-private.pem`, add both to the ECS task definitions' `secrets:` blocks, and grant
the **execution** role `ssm:GetParameters` on `/slangdump/prod/*` plus `kms:Decrypt` on the CMK.
