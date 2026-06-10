# Security: CSRF, CORS, TLS, REST Principles & gRPC

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [CSRF (Cross-Site Request Forgery)](#1-csrf-cross-site-request-forgery)
2. [CORS (Cross-Origin Resource Sharing)](#2-cors-cross-origin-resource-sharing)
3. [TLS (Transport Layer Security)](#3-tls-transport-layer-security)
4. [REST Principles](#4-rest-principles)
5. [gRPC and Protocol Buffers](#5-grpc-and-protocol-buffers)

---

## 1. CSRF (Cross-Site Request Forgery)

### What

**CSRF** is an attack where a malicious website tricks a user's browser into making an **unwanted request** to a site where the user is already authenticated. The browser **automatically** includes cookies, so the server thinks the request is legitimate.

### How the Attack Works

```
1. User logs into bank.com → browser stores session cookie
2. User visits evil.com (attacker's site)
3. evil.com has: <img src="https://bank.com/transfer?to=attacker&amount=1000">
4. Browser sends the request to bank.com WITH the session cookie
5. bank.com processes the transfer — thinks it's from the real user
```

```
User (logged into bank.com)      evil.com              bank.com
        │                           │                      │
        │── visits evil.com ───────►│                      │
        │                           │                      │
        │   evil.com triggers:      │                      │
        │   POST bank.com/transfer  │                      │
        │   Cookie: session=abc ────────────────────────► │
        │                           │   Looks legitimate!  │
        │                           │   Transfer executed  │
```

### How to Prevent

| Method | How It Works |
|--------|--------------|
| **CSRF Token** | Server generates a random token per session/form → client must include it in the request → attacker can't guess it |
| **SameSite Cookie** | `SameSite=Strict` or `SameSite=Lax` → browser won't send cookie on cross-site requests |
| **Double Submit Cookie** | Set a random value in both a cookie and a request header → server verifies they match |
| **Check Origin/Referer header** | Reject requests where `Origin` or `Referer` doesn't match your domain |

### Go Example — CSRF Token

```go
package main

import (
    "crypto/rand"
    "encoding/hex"
    "net/http"
)

func generateCSRFToken() string {
    b := make([]byte, 32)
    rand.Read(b)
    return hex.EncodeToString(b)
}

func csrfMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method == "GET" {
            // Generate token and set in cookie + response
            token := generateCSRFToken()
            http.SetCookie(w, &http.Cookie{
                Name: "csrf_token", Value: token,
                HttpOnly: false, // JS needs to read it
                Secure: true, SameSite: http.SameSiteStrictMode,
            })
            next.ServeHTTP(w, r)
            return
        }

        // For POST/PUT/DELETE — validate token
        cookie, err := r.Cookie("csrf_token")
        if err != nil {
            http.Error(w, "CSRF token missing", http.StatusForbidden)
            return
        }
        headerToken := r.Header.Get("X-CSRF-Token")
        if cookie.Value != headerToken {
            http.Error(w, "CSRF token mismatch", http.StatusForbidden)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

---

## 2. CORS (Cross-Origin Resource Sharing)

### What

**CORS** is a security mechanism implemented by browsers that **blocks** web pages from making requests to a different **origin** (domain + port + protocol) than the one that served the page, unless the server explicitly allows it.

### What is an "Origin"?

```
Origin = Protocol + Domain + Port

https://example.com:443  →  origin
https://example.com:8080 →  different origin (different port)
http://example.com:443   →  different origin (different protocol)
https://api.example.com  →  different origin (different subdomain)
```

### How CORS Works

**Simple requests** (GET, POST with simple content types):
```
Browser → [Request + Origin header] → Server
Server → [Response + Access-Control-Allow-Origin header] → Browser
Browser checks → if origin is allowed, process response; otherwise, block it
```

**Preflight requests** (PUT, DELETE, custom headers):
```
Browser → [OPTIONS request with Origin, Method, Headers] → Server
Server → [Response with allowed origins, methods, headers] → Browser
If allowed → Browser sends the actual request
If denied  → Browser blocks the request
```

### Key Headers

| Header | Set By | Purpose |
|--------|--------|---------|
| `Access-Control-Allow-Origin` | Server | Which origins can access (`*` or specific origin) |
| `Access-Control-Allow-Methods` | Server | Allowed HTTP methods |
| `Access-Control-Allow-Headers` | Server | Allowed custom headers |
| `Access-Control-Allow-Credentials` | Server | Allow cookies/auth (`true`/`false`) |
| `Access-Control-Max-Age` | Server | Cache preflight response (seconds) |
| `Origin` | Browser | The requesting origin |

### Go Example — CORS Middleware

```go
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        origin := r.Header.Get("Origin")
        
        // Allow specific origins (never use * with credentials)
        allowedOrigins := map[string]bool{
            "https://myapp.com":     true,
            "https://staging.myapp.com": true,
        }

        if allowedOrigins[origin] {
            w.Header().Set("Access-Control-Allow-Origin", origin)
            w.Header().Set("Access-Control-Allow-Credentials", "true")
        }

        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-CSRF-Token")
        w.Header().Set("Access-Control-Max-Age", "86400")

        // Handle preflight
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusNoContent)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

> **Never use `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true`** — browsers will reject it. Always specify exact origins when credentials are involved.

---

## 3. TLS (Transport Layer Security)

### What

**TLS** (successor to SSL) is a cryptographic protocol that provides **encrypted communication** between a client and a server. It ensures:

| Property | Meaning |
|----------|---------|
| **Confidentiality** | Data is encrypted — no one can eavesdrop |
| **Integrity** | Data is not tampered with during transit |
| **Authentication** | Server (and optionally client) is verified via certificates |

### How — TLS 1.3 Handshake

```
Client                              Server
  │                                    │
  │── ClientHello ────────────────────►│
  │   (supported ciphers, key share)   │
  │                                    │
  │◄── ServerHello ────────────────────│
  │   (chosen cipher, key share,       │
  │    certificate, finished)          │
  │                                    │
  │── Finished ───────────────────────►│
  │                                    │
  │◄══ Encrypted Application Data ════►│
```

**TLS 1.3 requires only 1 round-trip** (1-RTT) vs TLS 1.2's 2 round-trips, and supports **0-RTT** for resumed connections.

### Certificate Chain

```
Root CA (pre-installed in OS/browser)
  └── Intermediate CA
        └── Server Certificate (your domain)
```

1. Server sends its certificate.
2. Client verifies the certificate chain up to a trusted **Root CA**.
3. Client verifies the certificate is not expired and matches the domain.

### TLS Versions

| Version | Status | Key Features |
|---------|--------|--------------|
| SSL 3.0 | ❌ Deprecated | Vulnerable (POODLE) |
| TLS 1.0 | ❌ Deprecated | Vulnerable |
| TLS 1.1 | ❌ Deprecated | Vulnerable |
| TLS 1.2 | ✅ Acceptable | Widely used, 2-RTT handshake |
| TLS 1.3 | ✅ Recommended | 1-RTT, removed weak ciphers, faster |

---

## 4. REST Principles

### What

**REST (Representational State Transfer)** is an architectural style for designing networked APIs. It uses standard HTTP methods and is the most common API design paradigm.

### 6 Constraints

| Constraint | Meaning |
|------------|---------|
| **Client-Server** | Client and server are separate — can evolve independently |
| **Stateless** | Each request contains all info needed — server stores no session state |
| **Cacheable** | Responses must declare if cacheable — reduces load |
| **Uniform Interface** | Consistent URLs, methods, and response formats |
| **Layered System** | Client doesn't know if it's talking to the real server or a proxy/load balancer |
| **Code on Demand** (optional) | Server can send executable code to client (e.g., JavaScript) |

### HTTP Methods

| Method | Action | Idempotent | Safe | Example |
|--------|--------|------------|------|---------|
| `GET` | Read | ✅ Yes | ✅ Yes | `GET /users/123` |
| `POST` | Create | ❌ No | ❌ No | `POST /users` |
| `PUT` | Replace (full update) | ✅ Yes | ❌ No | `PUT /users/123` |
| `PATCH` | Partial update | ❌ No | ❌ No | `PATCH /users/123` |
| `DELETE` | Delete | ✅ Yes | ❌ No | `DELETE /users/123` |

### URL Design Best Practices

```
✅ Good                          ❌ Bad
GET  /users                      GET  /getUsers
GET  /users/123                  GET  /user?id=123
GET  /users/123/orders           GET  /getUserOrders?userId=123
POST /users                      POST /createUser
PUT  /users/123                  POST /updateUser
DELETE /users/123                POST /deleteUser?id=123
```

### Status Codes

| Range | Meaning | Common Codes |
|-------|---------|--------------|
| 2xx | Success | `200 OK`, `201 Created`, `204 No Content` |
| 3xx | Redirect | `301 Moved`, `304 Not Modified` |
| 4xx | Client error | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests` |
| 5xx | Server error | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable` |

### REST Response Format

```json
// Success
{
    "data": { "id": 123, "name": "Alice" },
    "meta": { "page": 1, "total": 50 }
}

// Error
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Email is required",
        "details": [
            { "field": "email", "message": "must not be empty" }
        ]
    }
}
```

---

## 5. gRPC and Protocol Buffers

### What

**gRPC** is a high-performance **Remote Procedure Call (RPC)** framework developed by **Google**. It uses **Protocol Buffers (Protobuf)** for serialization and **HTTP/2** for transport.

### REST vs gRPC

| Feature | REST | gRPC |
|---------|------|------|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/2 only |
| **Format** | JSON (text) | Protobuf (binary) |
| **Contract** | Loose (OpenAPI optional) | Strict (.proto required) |
| **Streaming** | ❌ No (workarounds: SSE, WebSocket) | ✅ Native (4 types) |
| **Code generation** | Optional | Built-in (multiple languages) |
| **Browser support** | ✅ Native | ⚠️ Requires gRPC-Web proxy |
| **Performance** | Good | Better (~10x faster serialization) |
| **Use case** | Public APIs, web | Microservices, internal APIs |

### gRPC Streaming Types

| Type | Client | Server | Use Case |
|------|--------|--------|----------|
| **Unary** | 1 request | 1 response | Normal RPC call |
| **Server streaming** | 1 request | Stream of responses | Real-time updates, feeds |
| **Client streaming** | Stream of requests | 1 response | File upload, batch processing |
| **Bidirectional streaming** | Stream | Stream | Chat, real-time collaboration |

### How gRPC Works

```
1. Define service in .proto file
2. Generate client & server code with protoc
3. Server implements the service interface
4. Client calls methods like local function calls
5. Protobuf handles serialization; HTTP/2 handles transport
```

### Proto File

```protobuf
syntax = "proto3";

package user;

option go_package = "./pb";

// Service definition
service UserService {
    // Unary RPC
    rpc GetUser(GetUserRequest) returns (User);
    
    // Server streaming
    rpc ListUsers(ListUsersRequest) returns (stream User);
    
    // Client streaming
    rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
    
    // Bidirectional streaming
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message GetUserRequest {
    int32 id = 1;
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message CreateUsersResponse {
    int32 created_count = 1;
}

message ChatMessage {
    string from = 1;
    string text = 2;
}
```

### Go — gRPC Server

```go
package main

import (
    "context"
    "log"
    "net"

    pb "myapp/pb"
    "google.golang.org/grpc"
)

type server struct {
    pb.UnimplementedUserServiceServer
}

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // In production: fetch from database
    return &pb.User{
        Id:    req.Id,
        Name:  "Alice",
        Email: "alice@example.com",
    }, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatal(err)
    }

    s := grpc.NewServer()
    pb.RegisterUserServiceServer(s, &server{})

    log.Println("gRPC server listening on :50051")
    if err := s.Serve(lis); err != nil {
        log.Fatal(err)
    }
}
```

### Go — gRPC Client

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    pb "myapp/pb"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    conn, err := grpc.Dial("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    client := pb.NewUserServiceClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("User: %+v\n", user)
    // User: id:1 name:"Alice" email:"alice@example.com"
}
```

### gRPC Error Handling

gRPC uses its own status codes (mapped from HTTP where applicable):

| gRPC Code | HTTP Equivalent | Meaning |
|-----------|-----------------|---------|
| `OK` | 200 | Success |
| `INVALID_ARGUMENT` | 400 | Bad request |
| `NOT_FOUND` | 404 | Resource not found |
| `ALREADY_EXISTS` | 409 | Conflict |
| `PERMISSION_DENIED` | 403 | Forbidden |
| `UNAUTHENTICATED` | 401 | Not authenticated |
| `INTERNAL` | 500 | Server error |
| `UNAVAILABLE` | 503 | Service unavailable |
| `DEADLINE_EXCEEDED` | 504 | Timeout |

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    if req.Id <= 0 {
        return nil, status.Errorf(codes.InvalidArgument, "invalid user ID: %d", req.Id)
    }

    user, err := s.db.FindUser(req.Id)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
    }

    return user, nil
}
```

### When to Use gRPC

| Scenario | Recommended |
|----------|-------------|
| Public API for web/mobile | **REST** (JSON) |
| Microservice-to-microservice | **gRPC** |
| Real-time streaming | **gRPC** |
| Browser clients | **REST** or gRPC-Web |
| High-throughput, low-latency | **gRPC** |
| Third-party integrations | **REST** (wider adoption) |
