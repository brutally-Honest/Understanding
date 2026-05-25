# Authentication — Evolution

> One thread: **stateless ↔ revocable ↔ confidential**. Every scheme is a negotiation between these three. None wins all three at once.

---

## 1. Basic Auth (1990s)

**Idea:** Send credentials on every single request.

```
Authorization: Basic dXNlcjpwYXNzd29yZA==
                      ↑ base64("user:password") — not encrypted, just encoded
```

**Storage**
- Client: credentials in memory / hardcoded
- Server: hashed passwords in DB

**Tradeoffs**
| ✅ Pros | ❌ Cons |
|--------|--------|
| Zero server state | Credentials on every request |
| Trivial to implement | No logout, no expiry |
| Stateless by nature | One leak = permanent compromise |

---

## 2. Session-Based Auth (early 2000s)

**Idea:** Login once → server creates a session → client gets a session ID cookie → only the ID travels after that.

```
Login → server stores { sessionId: "abc123" → userId: 42 } in Redis
      → sets cookie: Set-Cookie: sessionId=abc123; HttpOnly
```

**Storage**
- Client: session ID in `httpOnly` cookie
- Server: session store (Redis / DB) mapping `sessionId → user data`

**Tradeoffs**
| ✅ Pros | ❌ Cons |
|--------|--------|
| Credentials travel once | Server must store state |
| True logout (delete session) | Doesn't scale horizontally without shared Redis |
| Server-side revocation | Cookie = CSRF vulnerability |
| Battle-tested | Bad fit for mobile / cross-domain APIs |

---

## 3. JWT — JSON Web Token (~2015)

**Idea:** Server stores nothing. Issue a signed token containing the user data itself. Any server that knows the secret can verify it.

> Like a notarised document — anyone who trusts the notary can verify it without calling them.

```
header.payload.signature
  ↑ algo    ↑ { userId, role, exp }   ↑ HMAC-SHA256(header+payload, secret)
```

All three parts are base64-encoded — **visible to anyone, but unforgeable without the secret.**

**Storage**
- Client: `localStorage` (XSS risk) or `httpOnly` cookie (preferred)
- Server: nothing — just the secret key

**Tradeoffs**
| ✅ Pros | ❌ Cons |
|--------|--------|
| Fully stateless | Cannot revoke before expiry |
| Scales horizontally | Payload is visible (base64, not encrypted) |
| Works across services / mobile | Secret leak = all tokens compromised |
| No session store | Short expiry = bad UX; long expiry = security risk |

---

## 4. Access Token + Refresh Token

**Idea:** Split the JWT into two tokens with different lifespans.

> Like a day pass + a long-term membership card. The day pass expires fast; you use the membership card to renew it.

```
Access token  — short-lived (15 min) — used for API calls
Refresh token — long-lived (7–30d)  — used only to get a new access token
```

**Storage**
- Client: access token in **JS memory** (not localStorage), refresh token in `httpOnly; SameSite=Strict` cookie
- Server: refresh token **stored in DB** (so it can be invalidated)

**Flow**
```
Login  → issue both tokens → store refresh token hash in DB
Request → send access token → server verifies signature only (no DB hit)
Access expires → call /refresh → server checks DB, issues new access token
Logout → delete refresh token from DB → access token expires naturally
```

**Tradeoffs**
| ✅ Pros | ❌ Cons |
|--------|--------|
| Revocation possible (via refresh token) | Access token payload still visible |
| Short blast radius on access token leak | Slightly more complex flow |
| Good UX — silent refresh keeps user logged in | |

---

## 5. JWE — JSON Web Encryption

**Idea:** Everything above *signed* the token (tamper-proof) but didn't *hide* the payload. JWE encrypts the payload itself.

> JWT = sealed envelope with a window (can't change the letter, but can see through).  
> JWE = opaque box — can't see inside without the private key.

```
protected_header . encrypted_key . iv . ciphertext . auth_tag
```

- `encrypted_key` — the symmetric AES key, wrapped with recipient's RSA public key
- `ciphertext` — AES-256-GCM encrypted payload
- `auth_tag` — integrity check

**Tradeoffs**
| ✅ Pros | ❌ Cons |
|--------|--------|
| Payload invisible to logs, proxies, intermediaries | Heavy computation (asymmetric crypto) |
| Required for regulated industries (banking, healthcare) | Real key management overhead |
| Claims safe even if token is intercepted | Overkill for most apps |

---

## At a Glance

| | Basic Auth | Session | JWT | Access+Refresh | JWE |
|--|--|--|--|--|--|
| **Server state** | None | Yes (session store) | None | Refresh token in DB | Same |
| **Client storage** | Credentials | Cookie (httpOnly) | Token | Access: memory; Refresh: cookie | Same |
| **Revocable** | N/A | ✅ | ❌ | ✅ (refresh token) | ✅ |
| **Payload visible** | Yes | No | Yes | Yes | **No** |
| **Scales horizontally** | ✅ | Needs shared Redis | ✅ | ✅ | ✅ |
| **Use case** | Internal tools, CLIs | Traditional web apps | APIs, SPAs, microservices | Production consumer apps | Regulated / sensitive data |

---

> The evolution is a single thread: **stateless vs revocable vs confidential**.  
> Production systems often combine approaches — e.g. access+refresh with JWE for the access token.
