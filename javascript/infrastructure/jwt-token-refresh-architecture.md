# JWT Token Refresh Architecture: Two Patterns for Long-Running Sessions

**Source:** Planning conversation (AI-assisted architectural design)
**Saved:** 2026-06-17
**Tags:** javascript, security, infrastructure, fundamentals, technology

---

## TL;DR
Two production-ready patterns for keeping JWTs alive in long-running client sessions without breaking user experience or compromising security: the True Refresh Flow (two-token system with short-lived access tokens and long-lived HttpOnly refresh tokens) and the Re-Authentication Flow (single JWT re-minted from an existing session cookie when refresh tokens aren't available). Both use an 80% buffer timer to prevent race conditions on token expiry.

## Key Concepts & Terms
- **JWT (JSON Web Token)**: A signed token containing a payload of claims (user ID, roles, expiry timestamp `exp`). Three parts: header (algorithm), payload (claims), signature. The server verifies by re-computing the signature — if it matches, the token is valid.
- **Access token**: Short-lived JWT (typically 5–15 minutes). Used to authenticate API requests. If stolen, becomes useless quickly.
- **Refresh token**: Long-lived opaque string (typically 7–30 days). Used only to generate new access tokens. Never sent to the API directly.
- **HttpOnly cookie**: A cookie flag that prevents JavaScript from reading the cookie value. XSS attacks can't steal it. The browser attaches it automatically to requests matching the domain — no JS access required.
- **SameSite=Strict**: Cookie flag that prevents the cookie from being sent with cross-site requests. Blocks CSRF attacks.
- **80% buffer**: Setting a `setTimeout` for 80% of a token's lifespan rather than 100%. Provides a safety window to absorb network latency before the token actually expires. Without this buffer, a token could die mid-transit and produce a surprise 401.
- **Silent refresh**: The background token renewal process that runs without user interaction or UI disruption.
- **The OS sleep bug**: When a user closes their laptop, JavaScript execution pauses. `setTimeout` resumes when the lid opens but has no awareness of elapsed wall-clock time. A token that expired 3 hours ago while the laptop was asleep won't be refreshed until the `visibilitychange` event is handled.
- **Exponential backoff**: Retry strategy where each failed attempt waits progressively longer before retrying (e.g., 2s, 4s, 6s). Prevents hammering a failing server.

---

## The Core Problem

A widget or app that stays open indefinitely needs authentication that neither expires mid-session (bad UX) nor persists forever (security risk). The solution: short-lived tokens that refresh silently in the background.

---

## Architecture 1: True Refresh Flow

**When to use:** The API supports refresh tokens (the initial auth response includes both an access token and a refresh token).

**The two-token model:**
- Access token: JWT, expires in ~10 minutes. Used in every API call.
- Refresh token: Opaque string, expires in 7+ days. Used only to get new access tokens.

**Storage strategy:**
- Access token → JavaScript memory (not localStorage, not a cookie)
- Refresh token → HttpOnly, SameSite=Strict cookie (browser attaches automatically, JS can't read it)

**Why this storage split matters:** Storing the refresh token in an HttpOnly cookie means an XSS attack that steals JavaScript state can't get the refresh token. The long-lived credential is protected even if the page is compromised.

```
Client              Backend              API

  |-- 1. Auth req -->|                    |
  |                  |-- 2. Auth req ---->|
  |                  |<-- 3. Access+Refresh token --|
  |<-- 4. Access+Refresh (cookie) --|
  |                                        
  [80% timer fires]

  |-- 5. POST /refresh (cookie auto-sent) -->|
  |                  |-- 6. Rotate refresh -->|
  |                  |<-- 7. New access+refresh --|
  |<-- 8. New tokens --|
```

**JavaScript implementation:**

```javascript
class TokenManager {
  constructor() {
    this.accessToken = null;
    this.refreshTimer = null;
    this.expirationTime = null;
  }

  async initializeSession() {
    const response = await fetch('/api/auth/init', { method: 'POST' });
    const data = await response.json();
    // Expects: { accessToken: "eyJ...", expiresInSeconds: 600 }
    this.handleTokenResponse(data);
  }

  handleTokenResponse(data) {
    this.accessToken = data.accessToken;
    this.expirationTime = Date.now() + (data.expiresInSeconds * 1000);

    const bufferWindowMs = data.expiresInSeconds * 1000 * 0.8;

    if (this.refreshTimer) clearTimeout(this.refreshTimer);
    this.refreshTimer = setTimeout(() => this.refreshSession(), bufferWindowMs);
  }

  async refreshSession() {
    // HttpOnly cookie is attached automatically — JS never sees the refresh token
    const response = await fetch('/api/auth/refresh', { method: 'POST' });
    if (!response.ok) throw new Error('Refresh rejected');
    const data = await response.json();
    this.handleTokenResponse(data);
  }
}
```

---

## Architecture 2: Re-Authentication Flow

**When to use:** The API does not issue refresh tokens. The user is already authenticated to your application via a standard session cookie. At the timer mark, the backend re-mints a new JWT using that session.

```
Client              Backend              API

  |-- 1. GET /api/widget-token -->|          |
  |   (session cookie auto-sent)  |          |
  |                  |-- 2. Server-to-server auth -->|
  |                  |<-- 3. New JWT --|
  |<-- 4. JWT --|

  [80% timer fires — repeat from step 1]
```

**The dependency:** This flow only works while the user's main application session is valid. If they log out of the main app, the widget can no longer re-authenticate.

```javascript
class ReAuthTokenManager {
  constructor() {
    this.jwtToken = null;
    this.timer = null;
  }

  async fetchWidgetToken() {
    const response = await fetch('/api/generate-token', { method: 'GET' });
    if (response.status === 401) throw new Error('Main session expired');

    const data = await response.json();
    // Expects: { jwt: "eyJ...", expiresAtTimestamp: 1719163200000 }
    this.jwtToken = data.jwt;
    this.scheduleNextFetch(data.expiresAtTimestamp);
  }

  scheduleNextFetch(expiresAtTimestamp) {
    const totalLifespanMs = expiresAtTimestamp - Date.now();
    const delayMs = totalLifespanMs * 0.8;
    if (this.timer) clearTimeout(this.timer);
    this.timer = setTimeout(() => this.fetchWidgetToken(), delayMs);
  }
}
```

---

## Production Edge Cases

### Edge Case 1: OS Sleep (The `setTimeout` Bug)

`setTimeout` pauses when the OS sleeps the browser. On wake, the timer continues from where it paused — unaware of how much wall-clock time passed. A token that expired during a 3-hour sleep won't be refreshed until the user interacts with the page.

**Fix:** Listen for the `visibilitychange` event and check the actual timestamp on wake.

```javascript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') {
    this.checkTokenAndRefreshIfNeeded();
  }
});

checkTokenAndRefreshIfNeeded() {
  // If we're past the 80% mark right now, refresh immediately
  const eightyPercentMark = this.expirationTime - (this.expirationTime - Date.now()) * 0.2;
  if (Date.now() >= eightyPercentMark) {
    this.refreshSession();
  }
}
```

### Edge Case 2: Flaky Networks

If the network drops at the 80% mark, the refresh fails and the loop breaks. Without retry logic, the token expires and the user gets a 401.

**Fix:** Exponential backoff with a bounded retry count within the remaining 20% safety window.

```javascript
async refreshSessionWithRetry(attempt = 1) {
  const maxAttempts = 3;
  const backoffMs = attempt * 2000; // 2s, 4s, 6s

  try {
    const response = await fetch('/api/auth/refresh', { method: 'POST' });
    if (!response.ok) throw new Error('Network error');
    const data = await response.json();
    this.handleTokenResponse(data);
  } catch (error) {
    if (attempt <= maxAttempts) {
      // Show "reconnecting..." in UI
      setTimeout(() => this.refreshSessionWithRetry(attempt + 1), backoffMs);
    } else {
      // Hard fail — show "service unavailable"
      this.displayUnavailableUI();
    }
  }
}
```

---

## Decision Tree: Which Pattern to Use?

```
Does the API return a refresh token alongside the access token?
  YES → Use Architecture 1 (True Refresh Flow)
        Store access token in memory, refresh token in HttpOnly cookie
  NO  → Does the user have an active session with your own backend?
          YES → Use Architecture 2 (Re-Authentication Flow)
                Backend re-mints using session cookie + server-to-server API call
          NO  → User must re-authenticate entirely (hard login)
```

---

## Security Summary

| Concern | Mitigation |
|---------|-----------|
| Stolen access token | Short expiry (5–15 min) makes it useless quickly |
| XSS stealing refresh token | HttpOnly cookie — JS can't read it |
| CSRF on refresh endpoint | SameSite=Strict cookie + CSRF token |
| Token expiry during sleep | `visibilitychange` handler + timestamp check |
| Network drop at refresh time | Exponential backoff within 20% safety window |
| Refresh token stolen | Rotate on use — old token immediately invalidated |

---

## Questions & Gaps
- Token rotation: when the refresh token is used, should the server immediately invalidate it and issue a new one? (Yes — this is "refresh token rotation" and is the recommended practice; a stolen refresh token used after the legitimate client has already rotated it will fail.)
- How should the backend handle concurrent refresh requests (e.g., two tabs both firing at 80%)? Deduplicate by user ID with a short lock, or accept that both succeed and the second overwrites the first.
- The `expiresAtTimestamp` vs `expiresInSeconds` response format differs between the two implementations — standardize the API contract before building both.
- What's the cleanup strategy for `refreshTimer` when the widget/page unmounts? Without `clearTimeout` on unmount, the timer can fire into an unmounted component.

## Related Notes
- [Node.js 8 Production Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/nodejs-8-production-patterns.md) — the exponential backoff with jitter pattern here is the same pattern from that note's Circuit Breaker and Rate Limiting sections. The 80% timer + retry-in-remaining-20% is a specific application of the general retry-with-backoff pattern.
- [API Versioning Strategies](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/api-versioning-strategies.md) — auth token handling is a cross-cutting concern for versioned APIs; when an API version change alters the token response format (`expiresInSeconds` vs `expiresAtTimestamp`), both client implementations need updating.
- [Meta's AI Bot Instagram Account Takeover](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/metas-ai-bot-instagram-account-takeover.md) — that incident was partly enabled by insufficient separation between identity verification and account modification authority. The HttpOnly cookie pattern here enforces exactly that separation at the browser layer: the refresh credential travels in a channel JavaScript can't read.
