---
form: refresh_token_rotation
topic: auth
applies_to: [frontend, backend, mobile, security]
data_scale: [any]
decision: harden token/session lifetime — short access + rotating, revocable refresh
status: stable
last_reviewed: 2026-05-23
---

# Form 6: Refresh Token Rotation & Session Lifecycle (Cross-Cutting)

Not a login method — the **lifecycle layer** that makes Form 1/2/3 safe over time. A short-lived
access credential is continuously renewed by a long-lived, **single-use, rotating, revocable**
refresh token. Solves the core tension: short access TTL (safety) vs. not re-prompting the user (UX).

## When to use

- Any JWT setup (Form 2) — short access tokens need a renewal mechanism
- Mobile / SPA that must stay logged in for days without re-entering credentials
- You need "log out everywhere" / per-device revocation on top of stateless access tokens

## When NOT to use

- A purely stateful server session (Form 1) already gives instant revoke — you may not need a separate refresh token
- Truly stateless machine-to-machine with short-lived client-credential tokens → just re-fetch; no refresh token needed

## Pseudocode

```
function onLogin(user, device):
  family = uuid()                                  # one refresh-token "family" per device/login
  return rotate(user.id, family, prev = null)

function rotate(userId, family, prev):
  accessToken = issueAccessToken(userId)           # short TTL (minutes), see Form 2
  refreshRaw = secureRandom(>=160 bits)
  store.insertRefresh({
    id: uuid(), userId, family,
    hash: hashSecret(refreshRaw),
    prev: prev,                                     # chain for reuse detection
    expiresAt: now() + REFRESH_TTL,                # absolute cap on the family
    usedAt: null, revoked: false,
  })
  if prev != null: store.markUsed(prev)            # old refresh token is now spent
  return { accessToken, refreshRaw }               # refreshRaw → HttpOnly Secure cookie

function onRefresh(refreshRaw):
  rec = store.findRefreshByHash(hashSecret(refreshRaw))
  if rec is null or rec.revoked or now() > rec.expiresAt:
    return reauthenticate()
  if rec.usedAt != null:                           # REUSE of an already-spent token → theft
    store.revokeFamily(rec.family)                 # nuke the whole family, force re-login
    return reauthenticate()
  return rotate(rec.userId, rec.family, prev = rec.id)

function logout(refreshRaw):       store.revokeFamily(lookup(refreshRaw).family)
function logoutEverywhere(userId): store.revokeAllFamilies(userId)
```

## Backend / storage pattern

```
store refresh tokens (hashed), one row per issued token, grouped by family:
  family = a device/session line; rotation advances it; reuse detection revokes it

TTL design (two clocks):
  access token   → minutes (stateless verify, see Form 2)
  refresh token  → sliding rotation, with a hard absolute family expiry (days/weeks)

transport:
  refresh token  → HttpOnly + Secure + SameSite cookie (web), or secure storage / keychain (mobile)
  access token   → memory only (web), Authorization header on each call

reuse detection = the security core:
  a valid-but-already-used refresh token means it was stolen and replayed → revoke the family
```

## Pitfalls / Anti-patterns

- ❌ Non-rotating refresh tokens → a stolen refresh token works until its long expiry, undetected
  - **Fix**: rotate on every use; make each refresh token single-use
- ❌ No reuse detection → theft is invisible; both attacker and victim refresh happily
  - **Fix**: detect use of a spent token and revoke the entire family
- ❌ Refresh token in `localStorage` → XSS steals long-lived access
  - **Fix**: `HttpOnly` `Secure` cookie (web) / OS secure storage (mobile)
- ❌ Sliding expiry with no absolute cap → a continuously-refreshed session never dies
  - **Fix**: enforce an absolute family lifetime regardless of activity
- ❌ Two concurrent refreshes (e.g. parallel tabs) both rotate → one gets a "used" token and gets logged out
  - **Fix**: short grace window / single-flight on refresh, or accept the immediate predecessor briefly
- ❌ Access token TTL set long "to avoid refresh complexity" → defeats the whole stateless-but-safe design
  - **Fix**: keep access minutes-short; let rotation handle continuity
- ❌ No "log out everywhere" → can't contain a known compromise
  - **Fix**: family/user-level revocation, surfaced as a user-facing "active sessions" screen

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [better-auth/better-auth](https://github.com/better-auth/better-auth) — ~28k★: framework-agnostic auth with sessions, rotation, multi-device
- [nextauthjs/next-auth](https://github.com/nextauthjs/next-auth) — ~28k★ (Auth.js): session + token rotation across providers
- [supertokens/supertokens-core](https://github.com/supertokens/supertokens-core) — ~15k★: implements rotating refresh tokens with reuse detection
- [ory/kratos](https://github.com/ory/kratos) — ~14k★: identity + session lifecycle management, API-first
