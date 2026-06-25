# Auth0 Custom Token Exchange (CTE)

**Source:** https://auth0.com/docs/authenticate/custom-token-exchange
**Saved:** 2026-06-24
**Tags:** technology, security, fundamentals, infrastructure, tools

> **Status:** Early Access as of the document date. Available on B2C Professional, B2B Professional, and Enterprise plans. Subject to Free Trial terms in Okta's Master Subscription Agreement.
> **Standard:** Implements RFC 8693 (OAuth 2.0 Token Exchange). Read alongside `technology/fundamentals/oauth2-token-exchange-rfc8693.md`.

---

## TL;DR
Auth0's Custom Token Exchange lets applications exchange any existing token (from any external identity provider, any format) for Auth0 access, ID, and refresh tokens — by calling the standard `/oauth/token` endpoint with a custom Action that validates the incoming token and resolves it to an Auth0 user. The primary production use cases are: external IdP integration, migration to Auth0, delegated authorization for AI agents and service accounts, and cross-audience token acquisition.

## Key Concepts & Terms

- **Custom Token Exchange Profile**: A named configuration that maps a specific token exchange scenario to a single Auth0 Action. Each profile handles one type of incoming subject token. An application can have multiple profiles.
- **Action (CTE trigger)**: The serverless function that runs for each Custom Token Exchange request. Receives the incoming subject/actor tokens and implements all validation logic. Must call `api.authentication.setUser()` or `api.authentication.setUserByConnection()` to complete the transaction. Full control — any validation logic, any token format.
- **`subject_token`**: The incoming token the client is exchanging. Can be any format or type the Action code can interpret — not limited to JWTs or any specific standard. The Action is responsible for decoding and validating it.
- **`secte` log**: Auth0 tenant log event generated on successful Custom Token Exchange transactions. Includes the `actor` property (with `sub` and nested `actor`) when an actor was set via `setActor()`. Used for audit trails.
- **`fecte` log**: Auth0 tenant log event generated on failed Custom Token Exchange transactions. Primary debugging tool.

## How It Works

```
Client application
  │
  POST /oauth/token
  grant_type = urn:ietf:params:oauth:grant-type:token-exchange
  subject_token = <any external token>
  subject_token_type = <URI identifying token type>
  [actor_token = <optional>]
  audience = <target Auth0 API identifier>
  │
  ↓
Auth0 Authorization Server
  │
  1. Validates request format
  2. Matches to a Custom Token Exchange Profile
  3. Executes the Profile's Action
     │
     Action code:
     - Decodes subject_token (any format)
     - Validates signature, expiry, claims
     - Resolves to an Auth0 user
     - Calls api.authentication.setUser(userId)
     - OR setUserByConnection(connection, userId)
     - [Optional] setActor() for delegation semantics
     │
  4. Issues Auth0 tokens for the resolved user
  │
  ↓
Response: access_token, id_token, refresh_token
```

## Primary Use Cases

**1. External IdP integration**: Users authenticated by an external system (SAML IdP, legacy auth system, partner IdP) can exchange their existing token for Auth0 tokens, enabling them to access Auth0-protected resources without re-authenticating.

**2. Migration to Auth0**: During migration, existing sessions from the legacy system can be exchanged for Auth0 sessions transparently — users don't experience a forced logout during the cutover period.

**3. Cross-audience token acquisition**: Get Auth0 access tokens scoped to a specific API audience, starting from a token for a different audience. Standard token exchange use case from RFC 8693.

**4. Delegated authorization for AI agents**: An AI agent, support agent, or service acting on behalf of a user can exchange a user token (with the user's `subject_token`) plus its own `actor_token` to get a delegated Auth0 token that identifies both the service and the user it represents. The issued token carries the `act` claim per RFC 8693 semantics.

## The Action — What You Implement

The Action receives the exchange request and must:
1. Decode the `subject_token` (base64, JWT parsing, SAML parsing — whatever the format requires)
2. Validate it cryptographically (signature verification, expiry, issuer, audience)
3. Validate it against business logic (is this user authorized? is this token for this application?)
4. Resolve it to an Auth0 user via `api.authentication.setUser(userId)` or `api.authentication.setUserByConnection(connection, externalUserId)`
5. Optionally set actor information via `api.authentication.setActor({ sub: actorId })` for delegation semantics

**The security burden**: Auth0 validates the OAuth layer (grant type, profile matching, request format). Your Action is responsible for everything about the subject token itself. If your validation is weak, an attacker can forge a subject token and authenticate as any user.

## Critical Security Warning

Auth0's documentation explicitly calls this out: subject and actor tokens can be any format the Action can interpret, but the Action must implement strong validation. Weak validation opens two specific attack vectors:

- **Spoofing**: Attacker crafts a fake `subject_token` claiming to be another user, and the Action accepts it without signature verification
- **Replay attacks**: Attacker captures a valid token and reuses it after it should have expired

**Required validation at minimum**:
- Cryptographic signature verification (don't just decode — verify the signature against the expected key)
- Expiry check (`exp` claim or equivalent)
- Issuer validation (the token must come from the expected source)
- Audience validation (the token must be intended for this exchange)
- Nonce or jti (JWT ID) tracking for single-use tokens where replay is a concern

## Limitations

- MFA enrollment/challenge APIs (`challengeWith()`, `enrollWith()`) are not available in CTE flows — CTE is non-interactive
- Custom DB Connections with import mode ON cannot use `setUserByConnection()`
- Third-party clients and non-OIDC conformant clients are not supported
- The target API must have "Allow Skipping User Consent" enabled (consent cannot be collected in a non-interactive flow)

## Audit and Debugging

- Every successful CTE transaction generates a `secte` tenant log entry
- Every failed transaction generates a `fecte` tenant log entry
- When delegation semantics are used (`setActor()`), the `actor` property appears in `secte` logs with `sub` and any nested actor chain
- Start debugging CTE issues by filtering tenant logs for `fecte` events and reading the error details

## Questions & Gaps
- The Action has full control over validation — but Auth0 doesn't provide built-in verification utilities for common formats (SAML, vendor JWTs). How much boilerplate validation code is needed per token format?
- "Allow Skipping User Consent" must be enabled on the target API — is there a way to implement CTE while still presenting a consent screen for certain scopes?
- Multiple CTE profiles per application: what's the routing mechanism? Does Auth0 route by `subject_token_type`, or by some other discriminator in the request?
- The `fecte` log provides error details — but how specific are the errors? Does it distinguish "Action threw an error" from "request was malformed" from "profile not found"?

## Related Notes
- [OAuth 2.0 Token Exchange — RFC 8693](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/oauth2-token-exchange-rfc8693.md) — the underlying standard. Auth0's CTE implements RFC 8693's grant type. Read that note for the protocol mechanics; read this note for the Auth0-specific implementation and security requirements.
- [JWT Token Refresh Architecture](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/jwt-token-refresh-architecture.md) — the tokens issued by CTE (access, ID, refresh) follow the same lifecycle patterns described there. The 80% refresh timer applies to CTE-issued tokens exactly as it does to standard Auth0 tokens.
- [30 Core Agentic Engineering Concepts](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/30-core-agentic-engineering-concepts.md) — CTE's delegated authorization use case (AI agent acting on behalf of user) is a production implementation of concept 4 (Agent Patterns — Planner/Executor and Delegation) and directly relevant to concept 17 (Prompt Injection Defense) — if an AI agent can exchange tokens on a user's behalf, the security boundary of what the agent is allowed to do becomes critical.
- [Meta's AI Bot Instagram Account Takeover](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/metas-ai-bot-instagram-account-takeover.md) — the "delegated authorization for AI agents" CTE use case is precisely the scenario that went wrong there. Strong token validation in the CTE Action is what prevents the confused deputy attack pattern.
