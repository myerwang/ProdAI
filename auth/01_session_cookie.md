---
form: session_cookie
topic: auth
applies_to: [frontend, backend, security]
data_scale: [any]
decision: human login where the server must be able to revoke a session instantly
status: stable
last_reviewed: 2026-05-23
---

# Form 1: Server Session + Cookie (Stateful)

The default for first-party web apps. The server holds session state; the browser holds
only an **opaque session id** in an `HttpOnly` cookie.

## When to use

- First-party web app (server-rendered, or SPA on the same site)
- You must be able to **revoke instantly**: logout-everywhere, ban a user, kill sessions on password change
- Smallest client attack surface — the id is opaque and not readable by JS
- Session-bound data (roles, permissions, cart) lives server-side and may change mid-session

## When NOT to use

- Many independent services with no shared session store, and you refuse sticky sessions → Form 2 (JWT)
- Third-party / cross-domain API consumers → Form 2 (JWT) or Form 4 (API key)
- Native mobile where cookie jars are awkward → token-based (Form 2)

## Pseudocode

```
function login(credentials):
  user = lookupUser(credentials.identifier)
  if user is null: return genericAuthError()          # don't reveal which field was wrong
  if not verifyPasswordHash(user.passwordHash, credentials.password):
    return genericAuthError()

  # rotate session id on privilege change to prevent fixation
  sessionId = secureRandom(>=128 bits)
  store.put(sessionId, {
    userId: user.id,
    createdAt: now(),
    lastSeenAt: now(),
    absoluteExpiry: now() + ABSOLUTE_TTL,
  }, ttl = IDLE_TTL)

  setCookie("sid", sessionId, {
    httpOnly: true, secure: true, sameSite: "Lax", path: "/",
    maxAge: IDLE_TTL,
  })
  return ok()

function authMiddleware(request):
  sessionId = request.cookie("sid")
  if sessionId is null: return unauthenticated()
  session = store.get(sessionId)
  if session is null or now() > session.absoluteExpiry:
    return unauthenticated()
  session.lastSeenAt = now()
  store.touch(sessionId, ttl = IDLE_TTL)             # sliding idle window
  request.user = loadUser(session.userId)            # roles read fresh, can change live

function logout(request):
  store.delete(request.cookie("sid"))                # instant server-side revoke
  clearCookie("sid")
```

## Backend / storage pattern

```
session store: map of sessionId -> { userId, createdAt, lastSeenAt, absoluteExpiry }
  - in-memory map        → single instance only (lost on restart, no scale)
  - shared cache / KV     → multi-instance, supports "revoke all my sessions"
  - DB table              → durable, queryable ("list my active sessions"), slower

cookie attributes (all required):
  HttpOnly   → JS cannot read it (blocks XSS token theft)
  Secure     → HTTPS only
  SameSite   → Lax (default) or Strict; None requires Secure + explicit CSRF defense
  Max-Age / Expires → idle timeout
```

## Pitfalls / Anti-patterns

- ❌ Storing passwords in plaintext / fast hashes (`md5`, `sha256`)
  - **Why it fails**: leaked DB → instant credential compromise; fast hashes are brute-forceable
  - **Fix**: `argon2id` (preferred) or `bcrypt`; per-user salt is built in; never roll your own
- ❌ Not rotating the session id on login / privilege elevation → **session fixation**
  - **Fix**: issue a fresh id at every authentication and privilege change; delete the old one
- ❌ `SameSite=None` (or cross-site forms) without CSRF protection → **CSRF**
  - **Fix**: `SameSite=Lax/Strict`, plus a per-session CSRF token for state-changing requests
- ❌ Missing `HttpOnly` / `Secure` → token readable by JS, or sent over HTTP
  - **Fix**: always set both; test with the cookie inspector
- ❌ In-memory session store behind a load balancer → users randomly logged out
  - **Fix**: shared store (cache/DB), or sticky sessions as a stopgap only
- ❌ Only an idle timeout, no absolute timeout → a stolen session lives forever if kept warm
  - **Fix**: enforce both a sliding idle TTL and a hard absolute expiry
- ❌ Treating "delete cookie" as logout without deleting server state → session still valid if cookie copied
  - **Fix**: revoke server-side first, then clear the cookie

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [heartcombo/devise](https://github.com/heartcombo/devise) — ~24k★: the Rails session-auth standard (Warden-based)
- [jaredhanson/passport](https://github.com/jaredhanson/passport) — ~24k★: Node.js auth middleware, session strategy is the canonical pattern
- [spring-projects/spring-security](https://github.com/spring-projects/spring-security) — ~9.5k★: JVM session management, fixation protection, CSRF built in
