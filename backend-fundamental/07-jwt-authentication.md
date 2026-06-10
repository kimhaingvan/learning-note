# JWT, Authentication & Authorization

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [Authentication vs Authorization](#1-authentication-vs-authorization)
2. [JWT (JSON Web Token)](#2-jwt-json-web-token)
3. [Cookie-Based Authentication](#3-cookie-based-authentication)
4. [Token-Based vs Cookie-Based](#4-token-based-vs-cookie-based)
5. [OAuth 2.0](#5-oauth-20)

---

## 1. Authentication vs Authorization

| | Authentication (AuthN) | Authorization (AuthZ) |
|-|------------------------|------------------------|
| **What** | Verify **who you are** | Verify **what you can do** |
| **Question** | "Are you who you claim to be?" | "Do you have permission to do this?" |
| **When** | Happens **first** | Happens **after** authentication |
| **How** | Password, biometrics, token, SSO | Roles, permissions, policies, ACLs |
| **Example** | Login with email/password | Admin can delete users, viewer cannot |
| **HTTP Status** | `401 Unauthorized` (actually means unauthenticated) | `403 Forbidden` |

```
Request → [Authentication: Who are you?] → [Authorization: Can you do this?] → Resource
```

---

## 2. JWT (JSON Web Token)

### What

**JWT** is an open standard (RFC 7519) for securely transmitting information between parties as a **Base64-encoded JSON** string. It consists of **3 parts** separated by dots:

```
header.payload.signature
```

### How — Token Creation

**Step 1**: Create the Header.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Base64URL encode → `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

**Step 2**: Create the Payload (claims).

```json
{
  "userId": 123,
  "role": "admin",
  "exp": 1700000000
}
```

Base64URL encode → `eyJ1c2VySWQiOjEyMywicm9sZSI6ImFkbWluIiwiZXhwIjoxNzAwMDAwMDB9`

**Step 3**: Create the Signature.

```
signature = HMAC-SHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    secretKey
)
```

**Final token**:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEyMywicm9sZSI6ImFkbWluIiwiZXhwIjoxNzAwMDAwMDB9.signature_here
```

### How — Token Verification

1. **Split** the token into 3 parts: header, payload, signature.
2. **Recompute** the signature using the same algorithm and secret key.
3. **Compare** the computed signature with the received signature.
4. If they match → token is **valid** and has not been tampered with.
5. Check `exp` claim → reject if expired.

### Standard Claims

| Claim | Name | Description |
|-------|------|-------------|
| `iss` | Issuer | Who issued the token |
| `sub` | Subject | Who the token is about (user ID) |
| `aud` | Audience | Intended recipient |
| `exp` | Expiration | Token expiry time (UNIX timestamp) |
| `nbf` | Not Before | Token is not valid before this time |
| `iat` | Issued At | When the token was created |
| `jti` | JWT ID | Unique token identifier (for revocation) |

### Go — JWT from Scratch (no library)

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "strings"
)

var apiKey = "a-string-secret-at-least-256-bits-long123"

func hmacSHA256(message, key string) string {
    h := hmac.New(sha256.New, []byte(key))
    h.Write([]byte(message))
    return base64.RawURLEncoding.EncodeToString(h.Sum(nil))
}

func encodeBase64URL(data interface{}) string {
    jsonData, _ := json.Marshal(data)
    return base64.RawURLEncoding.EncodeToString(jsonData)
}

func generateJWT() string {
    header := map[string]string{"alg": "HS256", "typ": "JWT"}
    payload := map[string]interface{}{
        "sub": "1234567890", "name": "John Doe", "admin": true,
    }

    h := encodeBase64URL(header)
    p := encodeBase64URL(payload)
    signature := hmacSHA256(h+"."+p, apiKey)

    return fmt.Sprintf("%s.%s.%s", h, p, signature)
}

func verifyJWT(token string) bool {
    parts := strings.Split(token, ".")
    if len(parts) != 3 {
        return false
    }
    expectedSig := hmacSHA256(parts[0]+"."+parts[1], apiKey)
    return expectedSig == parts[2]
}

func main() {
    token := generateJWT()
    fmt.Println("Token:", token)
    fmt.Println("Valid:", verifyJWT(token)) // true
}
```

### Go — JWT Full Implementation

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "strings"
)

type Header struct {
    Alg string `json:"alg"`
    Typ string `json:"typ"`
}

type Payload struct {
    Sub   string `json:"sub"`
    Name  string `json:"name"`
    Admin bool   `json:"admin"`
}

func EncodeToBase64URL(data interface{}) string {
    jsonData, _ := json.Marshal(data)
    return base64.RawURLEncoding.EncodeToString(jsonData)
}

func HmacSHA256(message, key string) string {
    h := hmac.New(sha256.New, []byte(key))
    h.Write([]byte(message))
    return base64.RawURLEncoding.EncodeToString(h.Sum(nil))
}

func GenerateJWT(apiKey string) string {
    header := Header{Alg: "HS256", Typ: "JWT"}
    payload := Payload{Sub: "1234567890", Name: "John Doe", Admin: true}

    headerBase64 := EncodeToBase64URL(header)
    payloadBase64 := EncodeToBase64URL(payload)
    signature := HmacSHA256(headerBase64+"."+payloadBase64, apiKey)

    return fmt.Sprintf("%s.%s.%s", headerBase64, payloadBase64, signature)
}

func VerifyJWT(token, apiKey string) bool {
    parts := strings.Split(token, ".")
    if len(parts) != 3 {
        return false
    }

    headerBase64, payloadBase64, signature := parts[0], parts[1], parts[2]
    expectedSignature := HmacSHA256(headerBase64+"."+payloadBase64, apiKey)

    return signature == expectedSignature
}

func main() {
    apiKey := "a-string-secret-at-least-256-bits-long123"
    token := GenerateJWT(apiKey)
    fmt.Println("Generated Token:", token)
    fmt.Println("Verification:", VerifyJWT(token, apiKey)) // true
}
```

### Security Considerations

| Issue | Mitigation |
|-------|------------|
| Token theft | Use short expiry + refresh tokens |
| No revocation | Maintain a blacklist or use short-lived tokens |
| Sensitive data in payload | Payload is **only encoded, NOT encrypted** — never store secrets |
| Algorithm confusion attack | Always validate `alg` header; reject `"alg": "none"` |

---

## 3. Cookie-Based Authentication

### How

```
1. Client sends credentials (POST /login)
2. Server verifies credentials
3. Server creates a session → stores in DB/Redis (session_id → user data)
4. Server sends session_id in Set-Cookie header
5. Browser auto-sends cookie with every request
6. Server looks up session_id → identifies user
```

```
Client                          Server
  │                                │
  │── POST /login (credentials) ──►│
  │                                │  create session in DB
  │◄── Set-Cookie: sid=abc123 ─────│
  │                                │
  │── GET /profile                 │
  │   Cookie: sid=abc123 ─────────►│
  │                                │  lookup session abc123
  │◄── 200 OK { user data } ──────│
```

### Cookie Attributes

| Attribute | Purpose |
|-----------|---------|
| `HttpOnly` | Cannot be accessed by JavaScript (prevents XSS) |
| `Secure` | Only sent over HTTPS |
| `SameSite=Strict` | Not sent in cross-site requests (prevents CSRF) |
| `Path=/` | Cookie scope |
| `Max-Age=3600` | Expiry in seconds |

---

## 4. Token-Based vs Cookie-Based

| Feature | Cookie-Based (Session) | Token-Based (JWT) |
|---------|------------------------|---------------------|
| **State** | Stateful (session stored on server) | Stateless (token contains all data) |
| **Storage** | Cookie (browser-managed) | localStorage, cookie, or header |
| **Scalability** | Need shared session store (Redis) | No server-side storage needed |
| **Mobile support** | ❌ Cookies are web-centric | ✅ Works with any client |
| **CSRF** | ⚠️ Vulnerable (cookie auto-sent) | ✅ Not vulnerable (if in header) |
| **XSS** | ✅ HttpOnly cookie prevents JS access | ⚠️ localStorage is accessible by JS |
| **Revocation** | ✅ Easy (delete session) | ❌ Hard (token is self-contained) |
| **Size** | Small (just session ID) | Larger (contains claims) |

---

## 5. OAuth 2.0

### What

**OAuth 2.0** is an **authorization framework** that allows a third-party application to access a user's resources **without exposing their credentials**. It defines how to obtain **access tokens**.

### Roles

| Role | Description | Example |
|------|-------------|---------|
| **Resource Owner** | The user who owns the data | You (the user) |
| **Client** | The app requesting access | A third-party app |
| **Authorization Server** | Issues tokens after authentication | Google, GitHub |
| **Resource Server** | Hosts the protected resources | Google API, GitHub API |

### Authorization Code Flow (Most Common)

```
User        Client App        Auth Server        Resource Server
 │              │                  │                    │
 │─ Click "Login with Google" ────►│                    │
 │              │── Redirect to ──►│                    │
 │              │   Auth Server    │                    │
 │◄─── Login page ────────────────│                    │
 │─── Enter credentials ─────────►│                    │
 │              │                  │                    │
 │◄── Redirect with auth_code ────│                    │
 │──► auth_code │                  │                    │
 │              │── Exchange ─────►│                    │
 │              │   code for token │                    │
 │              │◄─ access_token ──│                    │
 │              │                  │                    │
 │              │── GET /api ──────────────────────────►│
 │              │   Authorization: Bearer <token>       │
 │              │◄─ Protected data ────────────────────│
```

### Grant Types

| Grant Type | Use Case |
|------------|----------|
| **Authorization Code** | Server-side apps (most secure) |
| **Authorization Code + PKCE** | SPAs and mobile apps |
| **Client Credentials** | Machine-to-machine (no user) |
| **Refresh Token** | Obtain new access token without re-login |

### Access Token vs Refresh Token

| | Access Token | Refresh Token |
|-|--------------|---------------|
| **Purpose** | Access protected resources | Get a new access token |
| **Lifetime** | Short (15 min – 1 hour) | Long (days – months) |
| **Stored** | Memory or short-lived cookie | Secure, HttpOnly cookie |
| **Sent to** | Resource server | Authorization server only |
