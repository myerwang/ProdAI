---
form: oauth2_oidc
topic: auth
applies_to: [frontend, backend, mobile, security]
data_scale: [any]
decision: delegate identity to a 3rd-party IdP, or federate SSO across apps
status: stable
last_reviewed: 2026-05-23
---

# Form 3: OAuth 2.0 / OIDC (Delegated / Federated)

**OAuth 2.0** delegates *authorization* (access to resources). **OIDC** layers
*authentication* (an `id_token` proving who the user is) on top. Use it when you don't
want to own passwords — let Google / GitHub / a corporate IdP authenticate the user.

## When to use

- "Log in with Google / GitHub / Microsoft" social login
- Corporate SSO across multiple internal apps (one IdP, many relying parties)
- You want to **not** store passwords at all (delegate that risk to a dedicated IdP)
- Issuing scoped, consented access to a third-party app calling your API

## When NOT to use

- A single first-party app with its own users and no federation need → Form 1/2 is far simpler
- Pure machine-to-machine with no human → Form 4 (API key) or OAuth2 client-credentials only
- You only need login, not delegated resource access, and you control the IdP anyway → still fine, but don't add OAuth complexity for nothing

## Pseudocode

```
# Authorization Code flow + PKCE (the correct flow for web & mobile & SPA)
function startLogin():
  codeVerifier = secureRandom()
  codeChallenge = base64url(sha256(codeVerifier))
  state = secureRandom()                       # CSRF protection for the redirect
  nonce = secureRandom()                       # binds id_token to this request
  store(session, { codeVerifier, state, nonce })
  redirect(IdP.authorizeUrl, {
    response_type: "code", client_id, redirect_uri,
    scope: "openid profile email",
    state, nonce, code_challenge: codeChallenge, code_challenge_method: "S256",
  })

function handleCallback(params):
  assert params.state == session.state          # reject forged / replayed callbacks
  tokens = await IdP.tokenEndpoint({
    grant_type: "authorization_code",
    code: params.code, redirect_uri,
    code_verifier: session.codeVerifier,        # PKCE proves same client started + finished
    client_id, client_secret?,                  # secret only for confidential clients
  })
  idClaims = verifyIdToken(tokens.id_token, {
    iss: IdP.issuer, aud: client_id,
    nonce: session.nonce, expiry: required,
  })
  user = upsertUserFromClaims(idClaims)         # link by stable subject (sub), not email
  # now establish YOUR OWN local session (Form 1) or token (Form 2)
  establishLocalSession(user)
```

## Backend / deployment pattern

```
roles:
  Authorization Server (IdP)  → keycloak / zitadel / hydra (self-host) or Google/Auth0 (hosted)
  Relying Party (your app)    → starts the flow, verifies id_token, maps to local user
  Resource Server (your API)  → validates access_token (often a JWT, see Form 2)

flow choice:
  web / SPA / mobile  → Authorization Code + PKCE   (never Implicit — deprecated)
  machine-to-machine  → Client Credentials
  never               → Resource Owner Password (ROPC) — defeats the point of delegation
```

## Pitfalls / Anti-patterns

- ❌ Using the **Implicit flow** (tokens in the URL fragment) → token leakage via history/referrer
  - **Fix**: Authorization Code + PKCE everywhere, including SPAs
- ❌ Skipping `state` → **CSRF / login fixation** on the redirect
  - **Fix**: generate, store, and verify `state` on callback
- ❌ Skipping PKCE for "confidential" web apps → code interception on the redirect
  - **Fix**: use PKCE for all client types; it's free defense
- ❌ Linking accounts by **email** → IdP email changes or unverified emails enable takeover
  - **Fix**: link by the IdP's stable `sub` (subject) claim; treat email as a display attribute
- ❌ Trusting an `id_token` without verifying `iss`, `aud`, `nonce`, signature, expiry
  - **Fix**: full validation against the IdP's published JWKS
- ❌ Confusing `access_token` (for APIs) with `id_token` (proof of login) → sending the wrong one
  - **Fix**: `id_token` authenticates the user to your app; `access_token` authorizes calls to resource servers
- ❌ Requesting broad scopes "just in case" → over-permissioned consent, user distrust
  - **Fix**: request the minimum scopes; ask for more incrementally

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [keycloak/keycloak](https://github.com/keycloak/keycloak) — ~35k★: full OAuth2/OIDC + SAML identity and access management server
- [authelia/authelia](https://github.com/authelia/authelia) — ~28k★: OIDC provider + SSO portal with MFA for reverse proxies
- [ory/hydra](https://github.com/ory/hydra) — ~17k★: certified OAuth2/OIDC provider, headless and API-first
- [zitadel/zitadel](https://github.com/zitadel/zitadel) — ~14k★: cloud-native identity, OIDC/OAuth2/SAML
- [logto-io/logto](https://github.com/logto-io/logto) — ~12k★: OIDC-based identity for apps, developer-friendly
