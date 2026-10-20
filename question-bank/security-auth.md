# Security & Auth

## Q: OAuth 2.0 vs JWT vs OpenID Connect — how do they relate?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PayPal/PwC rounds)
**Your answer:**

**Refined answer:**

These solve different (related) problems and are often confused because they're commonly used together:

- **OAuth 2.0** is an **authorization** framework — it defines how a client gets permission (a token) to access a resource on a user's behalf, without ever seeing the user's credentials. It says nothing about the token's format or about *who the user is*.
- **JWT (JSON Web Token)** is just a **token format** — a signed (and optionally encrypted) JSON payload with three parts (header, payload/claims, signature), self-contained so a service can verify it without calling back to an issuer. OAuth 2.0 doesn't require JWT specifically — tokens could be opaque strings validated against the issuer — but JWT is the common choice because it's verifiable locally.
- **OpenID Connect (OIDC)** is an **authentication** layer built on top of OAuth 2.0 — it standardizes *who the user is* (an `id_token`, specifically a JWT, with standard claims like `sub`, `email`) on top of OAuth's authorization flow. OAuth alone answers "can this client do X"; OIDC adds "and here's who the user actually is."

**How this maps to your project:** `auth-service` + `api-gateway` (Phase 5) will issue JWTs after authenticating a user — effectively a minimal OIDC-style flow — and every downstream service validates the JWT's signature locally rather than calling back to `auth-service` on every request.

**Follow-ups to expect:**
- Why use JWT instead of a server-side session? (Stateless validation — any service instance can verify the token without a shared session store, which fits a horizontally-scaled microservices architecture better than sticky sessions.)

---

## Q: Explain JWT structure and the authentication/validation flow.
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

A JWT is three base64url-encoded parts joined by dots: `header.payload.signature`.
- **Header** — algorithm used (`alg`, e.g. `HS256`/`RS256`) and token type.
- **Payload (claims)** — the actual data: standard claims (`sub` = subject/user ID, `exp` = expiry, `iat` = issued-at) plus custom claims (roles, scopes).
- **Signature** — a cryptographic signature over the header+payload, using a secret (HMAC, symmetric) or a private key (RSA/EC, asymmetric), so any tampering with the payload invalidates the signature.

**Flow:** user authenticates (credentials) → `auth-service` issues a signed JWT → client sends it as `Authorization: Bearer <token>` on subsequent requests → each receiving service verifies the signature (using the shared secret, or the issuer's public key if asymmetric) and checks `exp`/claims **locally**, without a network call back to `auth-service`. This is exactly why JWTs enable stateless, horizontally scalable auth — verification cost is a local signature check, not a database/session lookup.

**Follow-ups to expect:**
- What happens when a JWT needs to be revoked before it expires (e.g. a compromised token)? JWTs are self-contained and valid until `exp` by design — true instant revocation needs either very short expiries + refresh tokens, or a (small, fast) server-side blocklist check, which partially gives up the "no lookup needed" benefit.
- Symmetric (HMAC) vs asymmetric (RSA) signing — why would multiple services prefer asymmetric? (With asymmetric signing, only `auth-service` holds the private key to *issue* tokens; every other service only needs the public key to *verify* them — a leaked verification key in some downstream service can't be used to forge new tokens, unlike a shared HMAC secret.)
