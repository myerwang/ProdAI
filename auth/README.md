---
topic: auth
applies_to: [frontend, backend, mobile, security]
forms: 6
decision: choose by (who authenticates, stateful vs stateless, first-party vs federated, credential strength)
status: stable
last_reviewed: 2026-05-23
---

# Auth Patterns — Index

Six production-grade authentication forms. Each form lives in its own file with
metadata, pseudocode, pitfalls, and measured high-star references (≥5,000★, per
`../AGENTS.md` §3.6).

"Auth" here = **authentication** (proving who you are) + **session/token lifecycle**.
Fine-grained **authorization** (RBAC/ABAC/policy engines) is a separate topic, not
covered here.

---

## Decision Tree (read first)

```
who / what is authenticating?
├─ a human, logging into YOUR app with YOUR credentials
│   ├─ must revoke a session instantly (logout-all, ban, pwd change)  → 01_session_cookie
│   ├─ stateless, many services, horizontal scale, cross-domain        → 02_jwt
│   └─ whichever you pick, harden token lifetime + rotation            → 06_refresh_token_rotation
├─ a human, via a 3rd-party identity (Google / GitHub / corporate SSO) → 03_oauth2_oidc
├─ a machine / script / CLI / server-to-server                         → 04_api_key
└─ you want phishing-resistant / passwordless login                    → 05_webauthn_passkey

credential strength?
├─ password (legacy baseline)     → still hash with argon2id/bcrypt (see 01 pitfalls)
├─ phishing-resistant / no password → 05_webauthn_passkey
└─ no password, delegate to IdP   → 03_oauth2_oidc
```

The first real decision for most apps is **stateful (01) vs stateless (02)**; everything
else layers on top.

---

## Summary

| # | Form | File | State | Best for | Top OSS (verified 2026-05-23) |
|---|---|---|---|---|---|
| 1 | Session + Cookie | [01_session_cookie.md](./01_session_cookie.md) | stateful | human login, instant revoke | passport ~24k★ / devise ~24k★ |
| 2 | JWT | [02_jwt.md](./02_jwt.md) | stateless | multi-service, scale, cross-domain | node-jsonwebtoken ~18k★ / jose ~7.6k★ |
| 3 | OAuth 2.0 / OIDC | [03_oauth2_oidc.md](./03_oauth2_oidc.md) | — | 3rd-party identity, SSO | keycloak ~35k★ / authelia ~28k★ |
| 4 | API Key / PAT | [04_api_key.md](./04_api_key.md) | stateful | machine-to-machine | Kong ~43k★ / unkey ~5.3k★ |
| 5 | WebAuthn / Passkeys | [05_webauthn_passkey.md](./05_webauthn_passkey.md) | — | phishing-resistant, passwordless | hanko ~8.9k★ |
| 6 | Refresh Token Rotation | [06_refresh_token_rotation.md](./06_refresh_token_rotation.md) | stateful | token lifecycle hardening | better-auth ~28k★ / supertokens ~15k★ |

---

## Common Combinations

- **Server-rendered web app**: 01 (session) + argon2id passwords; add 03 for social login
- **SPA + JSON API**: 02 (short-lived access JWT) + 06 (refresh rotation in httpOnly cookie)
- **Enterprise SSO**: 03 (OIDC) fronted by keycloak / zitadel; downstream services trust 02 (JWT)
- **Public API platform**: 04 (API key / PAT) at the gateway (Kong) + per-key rate limit
- **Consumer app going passwordless**: 05 (passkeys) as primary + 03 (social) as fallback
- **Mobile native**: 02 (JWT) + 06 (refresh in secure storage / keychain)

---

## Related but NOT separate forms here

- **Password hashing** (argon2id / bcrypt / scrypt) — a baseline requirement, covered in 01 pitfalls
- **MFA / TOTP** — a second factor layered on any form; see 05 for the phishing-resistant variant
- **SAML 2.0** — enterprise federation; legacy peer of OIDC, fold into 03 when needed
- **Authorization** (RBAC / ABAC / Cedar / OPA) — a different topic (what you may do, not who you are)

---

## How to extend

This folder was created upfront as a recognized multi-form taxonomy (`../AGENTS.md` §2.3).
When a sub-pattern grows to a 3rd related document, deepen it:

```
03_oauth2_oidc.md          ← form overview (current)
oauth2_oidc/               ← when deep-dives accumulate
├── pkce.md
├── token_introspection.md
└── ...
```

See `../CONTRIBUTING.md` §3 / §3.1 for the on-demand vs upfront-taxonomy rules.

---

## History

- **2026-05-23**: Initial taxonomy. 6 forms, created upfront per AGENTS.md §2.3. All
  references measured via GitHub API (≥5,000★ bar).
