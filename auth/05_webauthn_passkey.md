---
form: webauthn_passkey
topic: auth
applies_to: [frontend, backend, mobile, security]
data_scale: [any]
decision: phishing-resistant / passwordless login using device-bound or synced credentials
status: stable
last_reviewed: 2026-05-23
---

# Form 5: WebAuthn / Passkeys (Phishing-Resistant, Passwordless)

Public-key authentication built into browsers and OSes. The user proves possession of a
private key (held in a device authenticator or synced keychain) via a biometric/PIN gesture.
There is **no shared secret to phish, leak, or reuse**.

## When to use

- You want to eliminate passwords (passkeys as primary) or add a phishing-resistant factor
- Consumer apps wanting one-tap biometric login synced across a user's devices
- High-value accounts needing strong, phishing-resistant MFA (security-key second factor)

## When NOT to use

- Headless / machine clients → Form 4 (API key); WebAuthn needs a user gesture + authenticator
- You can't yet handle account recovery for users who lose all authenticators → ship a recovery path first
- Environments with no platform authenticator and no security keys → keep a fallback (Form 1/3)

## Pseudocode

```
# Registration (create a credential)
function startRegistration(user):
  challenge = secureRandom()                     # single-use, server-stored, time-boxed
  store(session, challenge)
  return {
    rp: { id: RP_ID, name: RP_NAME },            # RP_ID is the registrable domain
    user: { id: user.handle, name: user.email, displayName: user.name },
    challenge,
    pubKeyCredParams: [ES256, RS256],
    authenticatorSelection: { residentKey: "preferred", userVerification: "preferred" },
  }

function finishRegistration(response):
  assert response.challenge == session.challenge   # anti-replay
  assert response.origin == EXPECTED_ORIGIN        # origin binding = phishing resistance
  assert response.rpIdHash == sha256(RP_ID)
  cred = verifyAttestation(response)
  store.saveCredential(user.id, {
    credentialId: cred.id,
    publicKey: cred.publicKey,
    signCount: cred.signCount,                     # clone-detection counter
  })

# Authentication (assert an existing credential)
function startLogin():
  challenge = secureRandom(); store(session, challenge)
  return { challenge, rpId: RP_ID, userVerification: "preferred" }

function finishLogin(response):
  assert response.challenge == session.challenge
  assert response.origin == EXPECTED_ORIGIN
  cred = store.getCredential(response.credentialId)
  assert verifySignature(response, cred.publicKey)
  if response.signCount <= cred.signCount and response.signCount != 0:
    flagPossibleClone()                            # counter went backwards
  store.updateSignCount(cred.id, response.signCount)
  establishLocalSession(cred.userId)               # then Form 1 or Form 2
```

## Backend / storage pattern

```
per user: a list of registered credentials { credentialId, publicKey, signCount, label, addedAt }
  - allow multiple authenticators per user (phone + laptop + hardware key)
  - never store a private key — you only ever hold public keys

passkey vs hardware security key:
  synced passkey   → backed up to the platform keychain, survives device loss (best UX)
  device-bound key → never leaves the authenticator (highest assurance, needs backup factor)

RP_ID + origin are the anti-phishing core: a credential for example.com cannot be used on a look-alike domain.
```

## Pitfalls / Anti-patterns

- ❌ Not verifying `origin` / `rpId` server-side → loses the phishing resistance that's the whole point
  - **Fix**: strictly assert expected origin and RP id hash on every ceremony
- ❌ Reusing or not expiring the challenge → **replay attacks**
  - **Fix**: single-use, server-generated, short-lived challenge per ceremony
- ❌ No account recovery → user loses their device and is permanently locked out
  - **Fix**: register multiple authenticators, plus a recovery factor (recovery codes, email/Form 3 fallback)
- ❌ Ignoring the signature counter → miss cloned-authenticator detection
  - **Fix**: persist and compare `signCount`; flag regressions
- ❌ Treating passkeys as a second factor only → misses the big win (passwordless primary)
  - **Fix**: support passkey as a primary credential where your risk model allows
- ❌ Wrong `rpId` (e.g. full URL or wrong subdomain) → credentials silently fail to match
  - **Fix**: set `rpId` to the registrable domain you actually serve auth from

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [teamhanko/hanko](https://github.com/teamhanko/hanko) — ~8.9k★: passkey-first authentication backend + web components
- Note: large IdPs also implement WebAuthn/passkeys — e.g. [keycloak/keycloak](https://github.com/keycloak/keycloak) ~35k★ and [zitadel/zitadel](https://github.com/zitadel/zitadel) ~14k★ support passkeys as part of Form 3.

Below 5k★ (useful but not primary references): [MasterKale/SimpleWebAuthn](https://github.com/MasterKale/SimpleWebAuthn) ~2.2k (popular JS server+browser library), [go-webauthn/webauthn](https://github.com/go-webauthn/webauthn) ~1.3k (Go).
