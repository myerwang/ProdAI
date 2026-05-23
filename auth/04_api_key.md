---
form: api_key
topic: auth
applies_to: [backend, security]
data_scale: [any]
decision: machine-to-machine / script / CLI access to your API (no interactive human)
status: stable
last_reviewed: 2026-05-23
---

# Form 4: API Key / Personal Access Token (Machine-to-Machine)

A long-lived secret a non-interactive client sends to call your API. No login flow, no
user session — just a credential identifying a caller (a service, a script, a CI job).

## When to use

- Server-to-server, CLI tools, CI/CD, webhooks, public API platforms
- A developer needs programmatic access on behalf of themselves (Personal Access Token)
- You want per-key rate limiting, scoping, and independent revocation

## When NOT to use

- Authenticating an interactive human in a browser → Form 1/2/3
- Cross-org delegated access with user consent → OAuth2 client-credentials / Form 3
- Anything where the secret would be exposed in frontend code → keys cannot be kept secret in a browser/app bundle

## Pseudocode

```
function issueApiKey(owner, scopes):
  raw = "prefix_" + secureRandom(>=160 bits)   # show prefix for identification
  record = {
    id: uuid(),
    ownerId: owner.id,
    hash: hashSecret(raw),                       # store ONLY the hash, never the raw key
    last4: raw.suffix(4),                        # for display ("...a1b2")
    scopes: scopes,
    createdAt: now(), expiresAt: optional,
    revokedAt: null, lastUsedAt: null,
  }
  store.insert(record)
  return raw                                     # shown to the user EXACTLY ONCE

function authenticate(request):
  raw = request.header("Authorization").stripScheme()   # "Bearer <key>" or "X-API-Key"
  if raw is null: return unauthenticated()
  record = store.findByHash(hashSecret(raw))            # constant-time hash lookup
  if record is null or record.revokedAt != null: return unauthenticated()
  if record.expiresAt != null and now() > record.expiresAt: return unauthenticated()
  store.touchLastUsed(record.id, now())                 # async; don't block the request
  request.caller = record.ownerId
  request.scopes = record.scopes
```

## Backend / storage pattern

```
store keys hashed:
  hashSecret = sha256(raw)  is acceptable here (high-entropy random secret, not a password)
  → fast lookup, and a leaked DB does not reveal usable keys

key shape:
  "<prefix>_<base62 random>"   prefix encodes type/env (e.g. live vs test) for quick triage

controls per key:
  scopes (least privilege) · expiry · per-key rate limit · revoke flag · last-used audit

placement:
  validate at the gateway/edge (Kong) for shared rate limit + auth, or in-app for fine scopes
```

## Pitfalls / Anti-patterns

- ❌ Storing keys in plaintext → DB leak hands out every live key
  - **Fix**: store only `hash(raw)`; show the raw key once at creation
- ❌ Keys that never expire and can't be scoped → one leak = total, permanent access
  - **Fix**: optional expiry, mandatory scopes, easy rotation + revoke
- ❌ Embedding an API key in frontend / mobile app code → it's extractable, treat as public
  - **Fix**: never ship secret keys to clients; use a backend proxy or OAuth flows instead
- ❌ Keys in URLs / query strings → logged by proxies, servers, browser history
  - **Fix**: send in the `Authorization` header, never the query string
- ❌ No per-key rate limit → one compromised/abusive key degrades the whole API
  - **Fix**: rate limit and quota per key, alert on anomalies
- ❌ No "last used" / audit → you can't tell which keys are dead and safe to revoke
  - **Fix**: track `lastUsedAt`; surface unused keys for cleanup
- ❌ Hashing high-entropy keys with slow password hashes (bcrypt) → needless latency per request
  - **Fix**: a fast hash (SHA-256) is correct here precisely because the secret is random and long

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [Kong/kong](https://github.com/Kong/kong) — ~43k★: API gateway with `key-auth` + per-key rate limiting at the edge
- [unkeyed/unkey](https://github.com/unkeyed/unkey) — ~5.3k★: dedicated API key management — issue, scope, rate-limit, revoke
