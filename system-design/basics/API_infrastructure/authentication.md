# Authentication

## The One-Line Summary

> Authentication proves identity after login using tokens so users don't re-enter credentials on every request. Three approaches: Session Tokens, JWT, and OAuth — each with different tradeoffs.
> 

---

## The Problem

User logs in once. Makes 100 subsequent requests. Server needs to know each request is from the authenticated user without asking for the password every time.

**Solution:** After login, server issues a token. User sends token on every request. Server verifies token.

---

## Approach 1 — Session Tokens (Stateful)

1. User logs in with password
2. Server generates a random token, stores it in Redis
3. Server sends token to client
4. Client sends token on every request
5. Server looks up token in Redis — valid? Proceed. Invalid? Reject.

**Advantage:** Instant invalidation — delete token from Redis, user is immediately logged out.

**Disadvantage:** Redis lookup on every request. Redis must be highly available. Multiple server instances need shared Redis.

---

## Approach 2 — JWT (Stateless)

JSON Web Token — a self-contained token that proves identity without any database lookup.

### Structure

```
Header.Payload.Signature
```

- **Header** — signing algorithm (HS256 or RS256)
- **Payload** — user ID, role, expiry time
- **Signature** — Header + Payload signed with secret key

### How verification works

1. Server receives JWT
2. Re-signs Header + Payload with its secret key
3. Signature matches — token valid, no database lookup needed
4. Tampered payload — signature breaks, rejected immediately

### Signing algorithms

|  | HS256 (Symmetric) | RS256 (Asymmetric) |
| --- | --- | --- |
| **Keys** | One secret key — signs and verifies | Private key signs, public key verifies |
| **Use when** | Single server handles everything | Microservices — other services verify without seeing private key |

### The Invalidation Problem

JWT is stateless — server stores nothing. Can't invalidate a token before it expires. Stolen token works until expiry.

**Solutions:**

- **Short expiry (15 minutes)** — minimize damage window
- **Refresh tokens** — long-lived token to get new JWTs without re-login
- **Token blacklist in Redis** — but defeats stateless advantage

### JWT + Refresh Token Flow

1. Login → server issues:
    - **Access token (JWT)** — 15 min expiry, stored in memory/localStorage
    - **Refresh token** — 30 day expiry, stored in **httpOnly cookie**
2. Client uses JWT for API calls
3. JWT expires → browser automatically sends refresh token cookie
4. Server validates refresh token → issues new JWT
5. Attacker steals JWT — has 15 minutes, no refresh token, attack ends

### Why httpOnly cookies for refresh tokens?

httpOnly cookies cannot be read by JavaScript — even malicious JavaScript from XSS attacks. Browser sends them automatically but they're never accessible to code.

- JWT in localStorage = cash in wallet. Pickpocket can steal it.
- Refresh token in httpOnly cookie = cash in a safe. Pickpocket can't reach it.

**Never store refresh tokens in localStorage.**

---

## Approach 3 — OAuth

"Login with Google/GitHub/Apple" — your app never handles or stores the user's password.

### Flow

1. User clicks "Login with Google"
2. App redirects to Google's login page
3. User logs in on Google's servers — your app never sees the password
4. Google asks user to approve access
5. Google sends your app an **authorization code**
6. App exchanges code for **access token**
7. App uses access token to fetch user info from Google

### Why OAuth?

- **User:** One account for everything, no new password to remember
- **Security:** Your database never stores passwords. Breach exposes nothing useful.
- **Trust:** Users trust Google's security more than your app's

### OAuth risks

- Access token stolen — attacker impersonates user until expiry
- Google account compromised — single point of failure for every app using that Google login

---

## Decision Framework

| Scenario | Use |
| --- | --- |
| Simple app, need instant invalidation | Session Token + Redis |
| Stateless microservices, scale matters | JWT + short expiry + refresh tokens |
| Let users login without new accounts | OAuth (Google/GitHub) |
| Internal service-to-service auth | JWT with RS256 asymmetric keys |

---

## Session Token vs JWT

|  | Session Token | JWT |
| --- | --- | --- |
| **State** | Stateful — stored in Redis | Stateless — no storage |
| **Verification** | Redis lookup every request | Signature check — no lookup |
| **Invalidation** | Instant — delete from Redis | Hard — wait for expiry |
| **Scale** | Redis required | No shared storage needed |
| **Use when** | Instant invalidation needed | Stateless microservices, scale |

---

## Three-Question Ritual

**What problem does authentication solve?**

Proving identity on every request without re-entering credentials. After login, a token proves "I am who I say I am" on all subsequent requests.

**What breaks without authentication?**

Any user can access any other user's data. No way to distinguish who is making requests. No privacy, no security.

**When to choose session tokens over JWT?**

When instant token invalidation is required — user logout, account compromise, security breach. JWT can't be invalidated before expiry; session tokens in Redis can be deleted immediately.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| Session token storage | ElastiCache (Redis) |
| JWT issuing and verification | Amazon Cognito |
| OAuth provider | Amazon Cognito (supports Google, Facebook, Apple) |
| API key authentication | API Gateway API Keys |
| Service-to-service auth | IAM roles + STS temporary credentials |