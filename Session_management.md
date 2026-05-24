# Session Management with JWT

## Table of Contents
1. [The Core Problem](#1-the-core-problem)
2. [Production Standard — Access Token + Refresh Token](#2-production-standard--access-token--refresh-token)
3. [Implementing the 15-Minute Inactivity Window](#3-implementing-the-15-minute-inactivity-window)
4. [Token Lifecycle Flow — Concrete Examples](#4-token-lifecycle-flow--concrete-examples)
5. [How the Client Decides When to Refresh](#5-how-the-client-decides-when-to-refresh)
6. [Alternative — New JWT on Every Request](#6-alternative--new-jwt-on-every-request)
7. [Security Rules — What Not To Do](#7-security-rules--what-not-to-do)
8. [Quick Reference Summary](#8-quick-reference-summary)

---

## 1. The Core Problem

JWTs are **stateless** — once issued, the server has no record of them. The expiry (`exp` claim) is baked into the token itself. This creates two problems:

- You cannot extend a JWT that is about to expire — you must issue a new one.
- You cannot invalidate a JWT before it expires — the server will always accept a valid signature.

If you set a JWT expiry to 15 minutes and the user is actively clicking every 2 minutes, their session still dies at minute 15. Issuing a new JWT on every request solves the expiry problem but introduces race conditions and security gaps (covered in [Section 6](#6-alternative--new-jwt-on-every-request)).

The industry-standard solution is the **Access Token + Refresh Token** pattern.

---

## 2. Production Standard — Access Token + Refresh Token

### Token Roles

| Token | Expiry | Stored Where | Purpose |
|---|---|---|---|
| **Access Token** | 15 min | JS memory (not localStorage) | Sent with every API request in `Authorization` header |
| **Refresh Token** | 7 days | HttpOnly cookie + server DB/Redis | Only sent to `/refresh` endpoint to get a new access token |

### Why Two Tokens?

- The **access token** is short-lived, so even if stolen, it expires quickly.
- The **refresh token** is long-lived but stored server-side, so it can be invalidated at any time (logout, forced logout, breach detection).
- Refresh tokens support **rotation** — each use issues a new refresh token and deletes the old one, making token theft detectable.

### High-Level Flow

```
Client                         API Server                      Redis / DB
  │                                │                                │
  │── POST /login ────────────────►│                                │
  │◄── accessToken1 (15min)        │                                │
  │    refreshToken1 (cookie) ─────│──── store refreshToken1 ──────►│
  │                                │                                │
  │── GET /api/data                │                                │
  │   Authorization: accessToken1 ►│                                │
  │◄── 200 OK ─────────────────────│                                │
  │                                │                                │
  │── POST /refresh                │                                │
  │   cookie: refreshToken1 ──────►│──── verify refreshToken1 ─────►│
  │                                │◄─── found, lastUsedAt updated ─│
  │◄── accessToken2 ───────────────│                                │
  │    refreshToken2 (cookie) ─────│──── store refreshToken2 ──────►│
  │                                │──── delete refreshToken1 ──────►│
```

---

## 3. Implementing the 15-Minute Inactivity Window

The inactivity window is tracked via a `lastUsedAt` field on the refresh token, **not** on the access token. Every time `/refresh` is called successfully, `lastUsedAt` is updated — this resets the inactivity clock.

### Refresh Token Entity

```java
public class RefreshToken {
    private String token;          // hashed UUID stored in DB
    private String userId;
    private Instant expiresAt;     // absolute max lifetime (e.g., 7 days)
    private Instant lastUsedAt;    // updated on every successful refresh
}
```

### Refresh Endpoint Logic

```java
public ResponseEntity<?> refresh(String refreshToken) {
    RefreshToken stored = refreshTokenRepo.findByToken(hash(refreshToken))
        .orElseThrow(() -> new SessionExpiredException("Invalid token"));

    // Check absolute expiry (max session lifetime)
    if (stored.getExpiresAt().isBefore(Instant.now())) {
        refreshTokenRepo.delete(stored);
        throw new SessionExpiredException("Session expired");
    }

    // Check inactivity window
    Duration inactive = Duration.between(stored.getLastUsedAt(), Instant.now());
    if (inactive.toMinutes() >= 15) {
        refreshTokenRepo.delete(stored);
        throw new SessionExpiredException("Session expired due to inactivity");
    }

    // Rotate: delete old token, issue new ones
    refreshTokenRepo.delete(stored);
    String newAccessToken  = jwtService.generateAccessToken(stored.getUserId());
    RefreshToken newRefresh = refreshTokenRepo.save(new RefreshToken(
        generateAndHashToken(), stored.getUserId(),
        Instant.now().plus(7, DAYS),
        Instant.now()  // lastUsedAt = now → inactivity clock reset
    ));

    return ResponseEntity.ok(new TokenResponse(newAccessToken, newRefresh.getToken()));
}
```

**Key rule:** The 15-minute clock resets every time `/refresh` succeeds. The user only gets logged out if they make zero requests for a full 15-minute window.

---

## 4. Token Lifecycle Flow — Concrete Examples

### Scenario A — Active User (Happy Path)

```
T=0    Login
       │
       ▼
 accessToken1  (expires T=15)
 refreshToken1 (lastUsedAt=T=0)

T=5    GET /api/data { Authorization: accessToken1 }
       accessToken1 valid → 200 OK → NO new tokens

T=14   Client sees token expiring in 1 min → triggers silent refresh
       │
       ▼
 POST /refresh { cookie: refreshToken1 }
 lastUsedAt=T=0, now=T=14, inactive=14min < 15min → VALID ✅
       │
       ▼
 DELETE refreshToken1
 CREATE accessToken2  (expires T=29)
 CREATE refreshToken2 (lastUsedAt=T=14)

T=28   Client refreshes again
       lastUsedAt=T=14, now=T=28, inactive=14min < 15min → VALID ✅
       │
       ▼
 DELETE refreshToken2
 CREATE accessToken3  (expires T=43)
 CREATE refreshToken3 (lastUsedAt=T=28)
```

---

### Scenario B — User Goes Inactive (Session Expires)

```
T=0    Login
       │
       ▼
 accessToken1  (expires T=15)
 refreshToken1 (lastUsedAt=T=0)

       ── user stops doing anything ──

T=15   accessToken1 expires. Client triggers silent refresh.
       │
       ▼
 POST /refresh { cookie: refreshToken1 }
 lastUsedAt=T=0, now=T=15, inactive=15min ≥ 15min → EXPIRED ❌
       │
       ▼
 DELETE refreshToken1
 return 401 Unauthorized
       │
       ▼
 Client: redirect user to /login → "Session expired"
```

---

### Scenario C — Narrowly Saved by Last-Second Activity

```
T=0    Login
       accessToken1 (expires T=15), refreshToken1 (lastUsedAt=T=0)

T=14   User clicks → accessToken1 still valid (1 min left)
       Client proactively refreshes:
       lastUsedAt=T=0, now=T=14, inactive=14min → VALID ✅

       DELETE refreshToken1
       CREATE accessToken2  (expires T=29)
       CREATE refreshToken2 (lastUsedAt=T=14)  ← inactivity clock RESET

       ── user goes inactive again ──

T=29   accessToken2 expires. Client triggers refresh.
       lastUsedAt=T=14, now=T=29, inactive=15min ≥ 15min → EXPIRED ❌

       DELETE refreshToken2
       return 401 → redirect to /login
```

---

### Scenario D — Logout

```
T=10   User clicks Logout
       │
       ▼
 POST /logout { cookie: refreshToken1 }
       │
       ▼
 DELETE refreshToken1 from DB/Redis
 Clear HttpOnly cookie on response
       │
       ▼
 Client discards accessToken1 from JS memory
 Redirect to /login

 Even if attacker had stolen accessToken1 earlier,
 it becomes useless at T=15 with no refresh path available.
```

---

### Decision Tree

```
Incoming request
      │
      ├── accessToken valid? ──YES──► Process request. No new tokens.
      │
      └── accessToken expired?
                │
                ▼
          POST /refresh
                │
                ├── refreshToken not in DB?
                │       └──► 401. Redirect to login.
                │
                ├── lastUsedAt ≥ 15min ago?
                │       └──► DELETE token. 401. "Session expired due to inactivity."
                │
                └── lastUsedAt < 15min ago?
                        └──► DELETE old refreshToken
                             CREATE new accessToken
                             CREATE new refreshToken (lastUsedAt = now)
                             Return new accessToken.
```

---

## 5. How the Client Decides When to Refresh

### Reading Token Expiry Without a Network Call

A JWT payload is base64-encoded. The client can decode it locally to read the `exp` claim:

```javascript
function getTokenExpiry(token) {
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.exp * 1000; // exp is Unix seconds; JS uses milliseconds
}
// accessToken1 payload: { userId: "123", exp: 1716000000 }
```

No server call needed. The client always knows exactly when the token expires.

---

### Strategy 1 — Proactive Interceptor (Primary Strategy)

An interceptor runs before every outgoing request. If the token is within 60 seconds of expiry, it refreshes first, then sends the request.

```javascript
axios.interceptors.request.use(async (config) => {
    const timeLeft = getTokenExpiry(accessToken) - Date.now();

    if (timeLeft < 60_000) {  // less than 60 seconds left
        accessToken = await callRefreshEndpoint();
    }

    config.headers.Authorization = `Bearer ${accessToken}`;
    return config;
});
```

**Race condition fix:** If 5 parallel requests all see the token expiring, they must not each trigger a separate `/refresh` call. Use a shared promise:

```javascript
let refreshPromise = null;

axios.interceptors.request.use(async (config) => {
    const timeLeft = getTokenExpiry(accessToken) - Date.now();

    if (timeLeft < 60_000) {
        if (!refreshPromise) {
            refreshPromise = callRefreshEndpoint()
                .finally(() => refreshPromise = null);
        }
        accessToken = await refreshPromise; // all 5 requests wait on same promise
    }

    config.headers.Authorization = `Bearer ${accessToken}`;
    return config;
});
```

```
5 requests fire simultaneously:
  Request 1: refreshPromise is null → creates POST /refresh call
  Requests 2-5: refreshPromise exists → wait on it

POST /refresh completes → all 5 get accessToken2 → all 5 proceed
Only 1 refresh call was made. ✅
```

---

### Strategy 2 — Reactive 401 Handling (Safety Net)

Make the request. If the server returns 401, refresh and retry. Handles edge cases like clock skew between client and server.

```javascript
axios.interceptors.response.use(
    (response) => response,
    async (error) => {
        const original = error.config;

        if (error.response?.status === 401 && !original._retried) {
            original._retried = true;  // prevent infinite retry loop
            accessToken = await callRefreshEndpoint();
            original.headers.Authorization = `Bearer ${accessToken}`;
            return axios(original);  // retry original request
        }

        return Promise.reject(error);
    }
);
```

```
GET /api/orders → 401 (token expired mid-flight)
      │
      ▼
Interceptor catches 401 → POST /refresh → accessToken2
      │
      ▼
Retry GET /api/orders with accessToken2 → 200 OK
User sees their data. Never knew anything happened.
```

---

### Strategy 3 — Background Timer (Optional)

On login, schedule a refresh to fire 1 minute before expiry.

```javascript
function scheduleRefresh(token) {
    const refreshAt = getTokenExpiry(token) - Date.now() - 60_000;

    setTimeout(async () => {
        accessToken = await callRefreshEndpoint();
        scheduleRefresh(accessToken);  // schedule next refresh recursively
    }, refreshAt);
}

scheduleRefresh(accessToken1); // called after login
```

**Limitation:** Timer lives in memory. A browser tab refresh kills it. Strategy 2's interceptor catches the resulting 401. Use this as an enhancement, not as your primary mechanism.

---

### What Production Uses

**Strategies 1 + 2 together:**

| Strategy | Handles |
|---|---|
| Proactive Interceptor | Normal case — refreshes before token expires |
| Reactive 401 | Edge cases — clock skew, backgrounded tab, server-side invalidation |

---

## 6. Alternative — New JWT on Every Request

### What This Looks Like

Issue a fresh JWT (15-min expiry) on every successful API response, set via `Set-Cookie`. This creates a rolling session without a separate refresh token.

```
T=0    POST /login → server issues jwt1 (exp=T+15) → Set-Cookie: jwt1
T=5    GET /api/orders (cookie: jwt1) → valid → issues jwt2 (exp=T+20) → Set-Cookie: jwt2
T=18   GET /api/profile (cookie: jwt2) → valid → issues jwt3 (exp=T+33) → Set-Cookie: jwt3

[User inactive from T=18]

T=33   jwt3 expires. Next request → 401 → redirect to login ✅
```

The sliding window works. **So why not use this approach?**

---

### Problem 1 — Race Condition With Parallel Requests

Modern page loads fire 5–10 API calls simultaneously. Each gets a new JWT in the response. The browser stores whichever response arrives last.

```
T=5   Browser fires 3 requests simultaneously (cookie: jwt1):

  Response A → Set-Cookie: jwt2a  (arrives T=5.0)  → browser stores jwt2a
  Response B → Set-Cookie: jwt2b  (arrives T=5.3)  → browser stores jwt2b
  Response C → Set-Cookie: jwt2c  (arrives T=9.8)  → browser stores jwt2c

jwt2a and jwt2b are orphaned. All three are valid JWT signatures.
Server will accept any of them — messy, unpredictable state.
```

With many slow or retried requests, the browser's active token becomes hard to predict.

---

### Problem 2 — Token Theft Is Undetectable

With no rotation, two parties can independently extend their own sessions simultaneously.

```
Attacker steals jwt2 from cookie.

T=6   Real user   (cookie: jwt2) → server issues jwt3  → Set-Cookie: jwt3
T=6   Attacker    (cookie: jwt2) → server issues jwt3' → Set-Cookie: jwt3'

Both jwt3 and jwt3' are valid. Both parties have live sessions.
Server has no way to detect anything is wrong. ❌
```

With refresh token rotation, theft is detectable:

```
Attacker uses refreshToken1 → server issues refreshToken2, deletes refreshToken1.
Real user uses refreshToken1 → NOT FOUND → server kills all sessions, triggers alert. ✅
```

---

### Problem 3 — Crypto Cost on Every Request

JWT signing uses HMAC-SHA256. It's cheap but not free.

```
Standard approach:  ~1 sign operation per 14 minutes per user
This approach:      1 sign operation per request per user

At 10,000 req/sec → 10,000 sign ops/sec  vs  ~11 sign ops/sec
```

On a small service this is negligible. On a high-traffic API it adds measurable CPU cost.

---

### Problem 4 — Cookie Overhead on Every Request

A JWT is typically 200–500 bytes. With this approach, this overhead is added to every request and every response.

```
Standard approach:  refreshToken cookie sent ONLY to /refresh
This approach:      JWT cookie sent on every request (upload) and every response (download)
```

---

### When This Approach Is Acceptable

- Internal admin dashboards or tooling
- Low traffic, single-user sessions
- Predominantly sequential (not parallel) requests
- Token theft detection is not a concern

---

### Comparison

| Issue | New JWT Per Request | Standard Access + Refresh |
|---|---|---|
| Sliding window works | Yes | Yes |
| Parallel request race condition | Yes — messy state | No — interceptor handles it |
| Token theft detectable | No | Yes — via rotation |
| Crypto cost | Every request | Every ~14 min |
| Implementation complexity | Low | Medium |

---

## 7. Security Rules — What Not To Do

### Never Store Access Tokens in `localStorage`

JavaScript can read `localStorage`. An XSS vulnerability lets an attacker steal the token with `localStorage.getItem('token')`. Store access tokens in JS memory (a variable). Store refresh tokens in `HttpOnly; Secure; SameSite=Strict` cookies — JavaScript cannot read these.

### Never Use Long-Lived Access Tokens

A 24-hour access token cannot be invalidated if the user logs out or their account is compromised. It stays valid until expiry. Keep access tokens short — 5 to 15 minutes.

### Always Rotate Refresh Tokens

Each time a refresh token is used, delete the old one and issue a new one. If an attacker steals and uses a refresh token, the next legitimate refresh attempt by the real user will fail — the server detects the reuse and can invalidate all sessions for that user.

### Never Put Sensitive Data in the JWT Payload

A JWT payload is base64-encoded, not encrypted. Anyone who intercepts the token can decode it with no key. Only put `userId`, `roles`, and `exp` in the payload. Never put SSN, email, passwords, or any PII.

---

## 8. Quick Reference Summary

### Token Lifecycle Rules

- **Login** → create accessToken + refreshToken. Store refreshToken in DB with `lastUsedAt = now`.
- **API request** → validate accessToken. If valid, serve response. No new tokens.
- **Token expiring** → client calls `/refresh` with refreshToken cookie.
- **`/refresh` success** → delete old refreshToken, create new accessToken and refreshToken, update `lastUsedAt`.
- **`/refresh` failure** (inactive ≥ 15 min, or token not found) → delete token, return 401, client redirects to login.
- **Logout** → delete refreshToken from DB, clear cookie, discard accessToken from memory.

### The 15-Minute Inactivity Rule

The inactivity clock lives on `refreshToken.lastUsedAt`. It resets every time `/refresh` succeeds. The user is only logged out if they make no requests for a full 15 minutes — meaning their accessToken expires and then the refresh check finds `lastUsedAt ≥ 15 min ago`.

### Client Refresh Strategy

1. **Proactive interceptor** — check token expiry before every request; refresh if < 60 sec remaining. Use a shared promise to prevent parallel refresh calls.
2. **Reactive 401 handler** — catch 401 responses, refresh once, retry the original request. Safety net for clock skew and edge cases.
