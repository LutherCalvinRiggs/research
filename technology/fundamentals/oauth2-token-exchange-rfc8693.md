# OAuth 2.0 Token Exchange — RFC 8693

**Source:** https://www.rfc-editor.org/rfc/rfc8693
**Saved:** 2026-06-24
**Tags:** technology, fundamentals, security, infrastructure, javascript

> IETF Standards Track. Published January 2020. Authors: M. Jones, A. Nadalin (Microsoft), B. Campbell (Ping Identity), J. Bradley (Yubico), C. Mortimore (Visa).

---

## TL;DR
RFC 8693 defines how to request and obtain security tokens from an OAuth 2.0 authorization server acting as a Security Token Service (STS). It enables two key patterns: **impersonation** (service A gets a token that makes it look like user B to a downstream service) and **delegation** (service A gets a token that identifies both itself and the user B it's acting on behalf of). A new grant type (`urn:ietf:params:oauth:grant-type:token-exchange`) and four new JWT claims (`act`, `scope`, `client_id`, `may_act`) are the core additions.

## Key Concepts & Terms

- **Security Token Service (STS)**: A service that accepts security tokens, validates them, and issues new security tokens in response. RFC 8693 defines how to interact with an OAuth 2.0 authorization server acting in this role.
- **Token exchange**: Trading one security token for another — typically trading a broad-scope access token for a narrower one suited to a specific downstream service, or converting a user's identity token into a service-callable token.
- **Impersonation**: Service A is issued a token that identifies it as user B. Recipients of the token cannot distinguish A from B — A has all the rights B has within the token's scope. The `sub` claim in the issued token is B's identity; A's identity is not present.
- **Delegation**: Service A is issued a token that identifies both user B (the subject) and A (the actor). The `sub` claim is B's identity; the `act` claim is A's identity. Recipients know explicitly that A is acting on behalf of B. A still has its own distinct identity.
- **`act` claim** (Actor): A JWT claim that expresses delegation — identifies the party currently wielding the token on behalf of the subject. Value is a JSON object with identity claims about the actor. Chains of delegation are expressed by nesting `act` claims inside each other (outermost = current actor, deeper = prior actors in the chain).
- **`may_act` claim**: A JWT claim that pre-authorizes a specific party to act on behalf of the token's subject. The authorization server can read this claim in a `subject_token` to decide whether the requesting party is permitted to impersonate or receive delegated rights from the subject.
- **`subject_token`**: The token representing the party on whose behalf the new token is being requested. Required in the token exchange request.
- **`actor_token`**: Optional token representing the party that will wield the issued token. Providing both `subject_token` and `actor_token` signals delegation semantics; providing only `subject_token` signals impersonation.

## The Token Exchange Flow

```
Client (or Resource Server)          Authorization Server

POST /token
  grant_type = urn:ietf:params:oauth:grant-type:token-exchange
  subject_token = <existing token>
  subject_token_type = <URI identifying token type>
  [actor_token = <acting party's token>]
  [actor_token_type = <URI>]
  [audience = <logical target service name>]
  [resource = <target service URL>]
  [scope = <desired scope on issued token>]
  [requested_token_type = <desired output token type>]
                                    ↓
                          Validates subject_token
                          Checks policy (audience, resource, scope)
                          Issues new security token
                                    ↓
HTTP 200
  access_token = <new token>
  issued_token_type = <URI>
  token_type = "Bearer" (or "N_A" if not an access token)
  expires_in = <seconds>
  [scope = <if different from requested>]
```

## The Three Semantic Cases

**1. Service acting on its own behalf** — no token exchange needed. Not delegation or impersonation.

**2. Impersonation** — service A needs to call service C as user B:
- Request: `subject_token` = B's token, no `actor_token`
- Issued token: `sub = B` (A's identity absent)
- Effect: C cannot distinguish A from B

**3. Delegation** — service A needs to call service C on behalf of user B, while remaining identifiably A:
- Request: `subject_token` = B's token, `actor_token` = A's token
- Issued token: `sub = B`, `act.sub = A`
- Effect: C knows it's dealing with A acting for B

## Token Type URI Identifiers

| URI | Meaning |
|-----|---------|
| `urn:ietf:params:oauth:token-type:access_token` | OAuth 2.0 access token from this auth server |
| `urn:ietf:params:oauth:token-type:refresh_token` | OAuth 2.0 refresh token |
| `urn:ietf:params:oauth:token-type:id_token` | OpenID Connect ID token |
| `urn:ietf:params:oauth:token-type:saml1` | Base64url-encoded SAML 1.1 assertion |
| `urn:ietf:params:oauth:token-type:saml2` | Base64url-encoded SAML 2.0 assertion |
| `urn:ietf:params:oauth:token-type:jwt` | A JWT specifically (format indicator, not use indicator) |

**The access_token vs. JWT distinction**: An access token represents a delegated authorization decision; a JWT is a token format. An access token *can* be a JWT but doesn't have to be. When `subject_token_type` is `access_token`, it means "a typical OAuth access token from this server, opaque to the client." When it's `jwt`, it means specifically a JWT format is expected, useful in cross-domain scenarios.

## The `act` Claim — Delegation Chains

Simple delegation — A acting for B:
```json
{
  "sub": "user@example.com",
  "act": {
    "sub": "admin@example.com"
  }
}
```

Chained delegation — service16 acting for user, via service77:
```json
{
  "sub": "user@example.com",
  "act": {
    "sub": "https://service16.example.com",
    "act": {
      "sub": "https://service77.example.com"
    }
  }
}
```

The outermost `act` is the current actor. Nested `act` claims are history only — they are informational and must NOT be used in access control decisions.

## The `may_act` Claim — Pre-Authorization

Allows a token to pre-authorize a specific party to act on its subject's behalf:
```json
{
  "sub": "user@example.com",
  "may_act": {
    "sub": "admin@example.com"
  }
}
```

When the auth server sees a `subject_token` containing `may_act`, it can use this to verify the requesting `actor_token` is authorized — without requiring separate configuration for every possible delegation relationship.

## Practical Use Cases

**1. Microservice-to-microservice with user context**: Frontend receives user access token. Backend service A needs to call backend service C on behalf of the same user. A exchanges the user token for a narrower-scoped token suitable for C, using delegation so C knows A is acting for the user.

**2. Service account impersonation for internal tooling**: Admin tool needs to check what a specific user can access. Auth server issues an impersonation token; the tool acts as that user within a strictly limited scope and time window.

**3. Cross-domain identity bridging**: System A has a SAML 2.0 identity assertion from one domain. System B operates in an OAuth 2.0 domain. Token exchange converts the SAML assertion (`subject_token_type: saml2`) into an OAuth 2.0 access token that B understands.

**4. Scope narrowing for downstream calls**: A resource server receives a broad-scope user token. Before calling a backend billing service, it exchanges the token for one with only `billing:read` scope, reducing the blast radius if the narrower token is compromised.

## Security Considerations

- **Omitting client authentication** allows any party holding a compromised token to use the STS to exchange it into other tokens. Client authentication at the token exchange endpoint is strongly recommended.
- **Scope restriction** is the primary mitigation for delegation/impersonation abuse. The issued token should have the minimum scope required for the intended downstream call.
- **Limited token lifetime** reduces the window for misuse.
- **Impersonation power**: When A impersonates B, A has all of B's rights within the token's scope. Authorization servers should apply policy carefully before issuing impersonation tokens.
- Tokens must only travel over TLS. Tokens containing sensitive identity information should be encrypted to their intended recipient.

## Error Codes

- `invalid_request`: The token exchange request is malformed, or either token is invalid or unacceptable per policy.
- `invalid_target`: The requested `resource` or `audience` targets too many services simultaneously, or the auth server cannot issue a token for those targets. Client should narrow the request.

## Questions & Gaps
- The spec explicitly leaves trust model details out of scope — how should implementers decide when to allow impersonation vs. requiring delegation? This is a policy decision with significant security implications not addressed here.
- The `may_act` claim pre-authorizes delegation, but the spec doesn't define how the authorization server should handle revocation of that pre-authorization if the `may_act` claim was included in a long-lived token.
- Refresh tokens in token exchange responses are described as unusual — the spec says "typically not issued." In what specific scenarios would a refresh token from a token exchange response be appropriate and safe?
- The distinction between `resource` (URI of target service) and `audience` (logical name of target service) is subtle — when should an implementer use one vs. the other, and how does the auth server policy map between them?

## Related Notes
- [JWT Token Refresh Architecture](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/jwt-token-refresh-architecture.md) — the refresh token flow described there is the simpler single-service JWT lifecycle. RFC 8693 extends this to multi-service environments where a token must be traded for a different token suited to a different audience.
- [System Design Playbook — How Apple Pay Works](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/system-design-playbook-neo-kim.md) — Apple Pay's DAN/cryptogram model is a practical implementation of delegation semantics: the payment network acts on behalf of the card issuer, with a narrowly-scoped credential (one-time cryptogram) instead of the full card number.
- [Meta's AI Bot Instagram Account Takeover](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/metas-ai-bot-instagram-account-takeover.md) — the confused deputy problem in that incident is structurally related to impersonation gone wrong: the bot was granted the ability to act as the user without adequate verification, which is exactly what RFC 8693's security considerations warn against when client authentication is omitted.
