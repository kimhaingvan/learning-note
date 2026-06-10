# API Keys

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [What](#1-what)
2. [How API Keys Work](#2-how-api-keys-work)
3. [Static API Keys](#3-static-api-keys)
4. [Dynamic API Keys](#4-dynamic-api-keys)
5. [Comparison](#5-comparison)
6. [Security Best Practices](#6-security-best-practices)
7. [Code Examples](#7-code-examples)

---

## 1. What

An **API Key** is a unique string used to **identify** and **authenticate** a client (application, service, or user) when calling an API. It acts as a simple credential passed in the request.

```
Client → [Request + API Key] → Server → [Validate Key] → Allow/Deny
```

> **Important**: API keys identify the **caller** (which app), not the **user**. For user-level authentication, use JWT or OAuth.

---

## 2. How API Keys Work

### Typical Flow

```
1. Developer registers on API portal → receives an API key
2. Client includes the key in each request
3. Server validates the key → checks rate limits, permissions
4. Server returns the response or 401/403
```

### Where to Send the Key

| Method | Example | Recommended? |
|--------|---------|--------------|
| **Header** | `X-API-Key: abc123` or `Authorization: Bearer abc123` | ✅ Yes |
| **Query parameter** | `?api_key=abc123` | ⚠️ No — logged in URLs, browser history, server logs |
| **Request body** | `{"api_key": "abc123"}` | ❌ No — not RESTful |

> **Always send API keys in headers** — query parameters are visible in logs, referrer headers, and browser history.

---

## 3. Static API Keys

### What

A **static API key** is a fixed string generated once and used until manually rotated or revoked. It does not change or expire automatically.

### How

```
Server generates a random string → stores hash in DB → gives plaintext to developer
Developer includes key in every request
Server hashes incoming key → compares with stored hash
```

### Characteristics

| Property | Value |
|----------|-------|
| Lifetime | Until manually revoked |
| Rotation | Manual |
| Generation | Once |
| Example | `12321321` (Stripe-style) |

### When to Use

- Internal service-to-service communication.
- Public APIs with rate limiting (e.g., Google Maps API).
- Simple integrations where OAuth is overkill.

---

## 4. Dynamic API Keys

### What

**Dynamic API keys** are **short-lived**, **automatically generated** tokens that expire after a set duration. They are typically obtained through an authentication flow.

### How

```
1. Client authenticates with credentials (or static key)
2. Server generates a time-limited token (e.g., 1 hour)
3. Client uses the token for subsequent requests
4. Token expires → client re-authenticates
```

### Characteristics

| Property | Value |
|----------|-------|
| Lifetime | Short (minutes to hours) |
| Rotation | Automatic (on expiry) |
| Generation | On each authentication |
| Example | JWT, OAuth access token, AWS STS temporary credentials |

### When to Use

- User-facing APIs.
- Multi-tenant platforms.
- When you need fine-grained, time-limited access.
- Compliance requirements (short-lived credentials).

---

## 5. Comparison

| Feature | Static API Key | Dynamic API Key |
|---------|----------------|-----------------|
| **Lifetime** | Long (until revoked) | Short (minutes–hours) |
| **Rotation** | Manual | Automatic |
| **If leaked** | Compromised until noticed | Limited damage (expires soon) |
| **Complexity** | Simple | More complex (auth flow) |
| **User-level access** | ❌ Usually per-app | ✅ Per-user |
| **Revocation** | Must manually revoke | Expires naturally |
| **Use case** | Server-to-server, simple APIs | User-facing, sensitive APIs |

---

## 6. Security Best Practices

| Practice | Why |
|----------|-----|
| **Hash keys before storing** | If DB is breached, attacker gets hashes, not keys |
| **Use HTTPS only** | Prevent keys from being intercepted in transit |
| **Send in headers, not URLs** | URLs are logged everywhere |
| **Set rate limits** | Prevent abuse even with valid keys |
| **Implement key rotation** | Reduce risk window if a key is leaked |
| **Scope permissions** | Give each key the minimum permissions it needs |
| **Monitor usage** | Detect anomalies (unusual IPs, high volume) |
| **Use prefix for identification** | e.g., `sk_live_`, `pk_test_` — helps identify key type without exposing it |
| **Never commit keys to Git** | Use environment variables or secret managers |

---

## 7. Code Examples

### Go — API Key Middleware

```go
package main

import (
    "crypto/sha256"
    "crypto/subtle"
    "encoding/hex"
    "net/http"
)

// hashAPIKey hashes the key before comparison (prevents timing attacks on raw string)
func hashAPIKey(key string) string {
    h := sha256.Sum256([]byte(key))
    return hex.EncodeToString(h[:])
}

// In production: store hashed keys in DB, not hardcoded
var validKeyHash = hashAPIKey("sk_live_abc123xyz789")

func apiKeyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        key := r.Header.Get("X-API-Key")
        if key == "" {
            http.Error(w, `{"error": "missing API key"}`, http.StatusUnauthorized)
            return
        }

        // Constant-time comparison to prevent timing attacks
        incomingHash := hashAPIKey(key)
        if subtle.ConstantTimeCompare([]byte(incomingHash), []byte(validKeyHash)) != 1 {
            http.Error(w, `{"error": "invalid API key"}`, http.StatusForbidden)
            return
        }

        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`{"message": "success"}`))
    })

    http.ListenAndServe(":8080", apiKeyMiddleware(mux))
}
```

### Go — API Key Generation

```go
package main

import (
    "crypto/rand"
    "encoding/hex"
    "fmt"
)

func generateAPIKey(prefix string) (string, error) {
    bytes := make([]byte, 32) // 256 bits of randomness
    if _, err := rand.Read(bytes); err != nil {
        return "", err
    }
    return prefix + hex.EncodeToString(bytes), nil
}

func main() {
    key, _ := generateAPIKey("sk_live_")
    fmt.Println("API Key:", key)
    // qwqewqdsasa
}
```
