---
form: jwt
topic: auth
applies_to: [frontend, backend, mobile, security]
data_scale: [any]
decision: stateless auth across many services / horizontal scale / cross-domain
status: stable
last_reviewed: 2026-05-23
---

# Form 2: JWT (Stateless Token)

A signed, self-contained token the client sends on every request. The server validates the
**signature** instead of looking up server-side session state.

## When to use

- Many services / horizontal scale where a shared session store is unwanted
- Cross-domain / cross-origin APIs, native mobile, machine clients
- Short-lived **access tokens** paired with a refresh mechanism (see Form 6)
- A trust boundary where one issuer signs and many resource servers verify (microservices)

## When NOT to use

- You need **instant revocation** as a primary requirement → Form 1 (session), or accept JWT + a revocation list (which reintroduces state)
- You're tempted to store mutable, sensitive, or large data in the token → don't; the payload is readable
- A simple single-server first-party web app → Form 1 is simpler and safer by default

## Pseudocode

```
function issueAccessToken(user):
  claims = {
    sub: user.id,
    iss: ISSUER, aud: AUDIENCE,
    iat: now(), exp: now() + ACCESS_TTL,   # keep short: minutes, not days
    # only non-sensitive, slowly-changing claims (e.g. role); never secrets
  }
  return sign(claims, key = currentSigningKey(), alg = "EdDSA" | "RS256")

function verifyAccessToken(token):
  header = decodeHeaderUnverified(token)
  assert header.alg in ALLOWED_ALGS            # pin algorithms; reject "none"
  key = resolveKey(header.kid, ALLOWED_ALGS)   # by key id, from a trusted key set
  claims = verifySignature(token, key)         # throws on bad signature
  assert claims.iss == ISSUER and claims.aud == AUDIENCE
  assert now() < claims.exp
  return claims                                # no DB hit on the happy path
```

## Key management pattern

```
asymmetric (recommended for multi-service):
  issuer holds private key, signs
  resource servers fetch public keys from a published key set (JWKS), verify
  rotate by adding a new key id (kid) while old keys still verify, then retire old

symmetric (single trust domain only):
  shared secret signs + verifies — never expose it to clients or other parties

access vs refresh:
  access token  → short TTL, sent on every request, stateless verify
  refresh token → long TTL, single-use, stored server-side / rotated (see Form 6)
```

## Pitfalls / Anti-patterns

- ❌ Accepting `alg: none` or trusting the token's own `alg` header → **signature bypass**
  - **Why it fails**: attacker sets `alg:none` or swaps RS256→HS256 using the public key as the HMAC secret
  - **Fix**: pin an allow-list of algorithms server-side; never let the token choose
- ❌ Treating JWT as revocable → logout / ban doesn't take effect until expiry
  - **Fix**: short access TTL + refresh rotation (Form 6); or a deny-list keyed by `jti` (reintroduces state)
- ❌ Long-lived access tokens (hours/days) → a stolen token is valid for that whole window
  - **Fix**: access TTL in minutes; rely on refresh for continuity
- ❌ Putting sensitive data in the payload → JWT is **signed, not encrypted**; anyone can base64-decode it
  - **Fix**: store only an id + minimal claims; look up sensitive data server-side (or use JWE if you truly must)
- ❌ Storing the token in `localStorage` → readable by any XSS
  - **Fix**: keep access token in memory; keep the refresh token in an `HttpOnly`, `Secure`, `SameSite` cookie
- ❌ No `iss` / `aud` validation → a token minted for service A is accepted by service B
  - **Fix**: always validate issuer and audience
- ❌ No key rotation plan → one leaked key forces a painful global re-issue
  - **Fix**: `kid`-based JWKS rotation, overlap old+new during transition

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [auth0/node-jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) — ~18k★: the de-facto Node.js JWT sign/verify library
- [golang-jwt/jwt](https://github.com/golang-jwt/jwt) — ~9.1k★: the maintained Go JWT library (successor to dgrijalva/jwt-go)
- [panva/jose](https://github.com/panva/jose) — ~7.6k★: JS JWT/JWE/JWS/JWK with strict alg handling and JWKS
- [jpadilla/pyjwt](https://github.com/jpadilla/pyjwt) — ~5.7k★: the standard Python JWT library
