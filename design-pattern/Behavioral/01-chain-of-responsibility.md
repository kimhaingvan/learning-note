# Chain of Responsibility

> **Category**: Behavioral | **Language**: Go

---

## What

**Chain of Responsibility** lets you pass requests along a chain of handlers. Each handler decides either to **process the request** or to **pass it to the next handler** in the chain.

---

## Why / When

- When you need to execute several handlers in a **particular order**.
- When your program must process different kinds of requests in various ways, but the exact types and sequences are unknown beforehand.
- Classic example: HTTP middleware pipelines.

---

## How

1. **Define the Handler interface** — a method that wraps a "next" handler.
2. **Create and implement concrete handlers.**
3. **Wire them up** — a Server/Chain assembles handlers in order.
4. **Kick off the chain** — client triggers the first handler, which cascades through.

---

## Example

```go
package main

import (
	"errors"
	"fmt"
)

// ── Core types ──

type ResponseWriter struct {
	Code string
	Body string
}

type Request struct {
	Path    string
	Payload string
}

type HandlerFunc func(w ResponseWriter, r *Request) error

// Handler interface — one link in the chain
type IHandler interface {
	Handle(next HandlerFunc) HandlerFunc
}

// ── Concrete handlers (middleware) ──

type LoggingMiddleware struct{}

func (lm *LoggingMiddleware) Handle(nextFunc HandlerFunc) HandlerFunc {
	return func(w ResponseWriter, r *Request) error {
		fmt.Printf("→ [LOG] Start %s\n", r.Path)
		defer fmt.Printf("← [LOG] End   %s\n", r.Path)
		return nextFunc(w, r)
	}
}

type AuthMiddleware struct{}

func (am *AuthMiddleware) Handle(nextFunc HandlerFunc) HandlerFunc {
	return func(w ResponseWriter, r *Request) error {
		fmt.Printf("→ [AUTH] Checking %s\n", r.Path)
		if r.Path == "/secret" {
			fmt.Println("✗ Unauthorized access attempt")
			return errors.New("unauthorized")
		}
		return nextFunc(w, r)
	}
}

// ── Application logic ──

func HandleSignInAPI(w ResponseWriter, r *Request) error {
	fmt.Println("Sign in successful")
	return nil
}

func HandleSecretAPI(w ResponseWriter, r *Request) error {
	fmt.Println("Secret API accessed")
	return nil
}

// ── Server wiring ──

type Server struct {
	middlewares []IHandler
}

func NewServer(middlewares []IHandler) *Server {
	return &Server{middlewares: middlewares}
}

// Then wraps final business logic with all middleware, last → first
func (c *Server) Then(finalFunc HandlerFunc) HandlerFunc {
	h := finalFunc
	for i := len(c.middlewares) - 1; i >= 0; i-- {
		h = c.middlewares[i].Handle(h)
	}
	return h
}

func (s *Server) HandleAPI(path string, req *Request) {
	var finalFunc HandlerFunc
	switch path {
	case "/signin":
		finalFunc = HandleSignInAPI
	case "/secret":
		finalFunc = HandleSecretAPI
	default:
		fmt.Printf("No handler for %s\n", path)
		return
	}

	handlerFunc := s.Then(finalFunc)
	if err := handlerFunc(ResponseWriter{}, req); err != nil {
		fmt.Printf("Error: %s\n", err)
	}
}

// ── Client ──

func main() {
	logging := &LoggingMiddleware{}
	auth := &AuthMiddleware{}

	server := NewServer([]IHandler{logging, auth})

	server.HandleAPI("/secret", &Request{Path: "/secret", Payload: "..."})
	// → [LOG] Start /secret
	// → [AUTH] Checking /secret
	// ✗ Unauthorized access attempt
	// ← [LOG] End   /secret
	// Error: unauthorized

	server.HandleAPI("/signin", &Request{Path: "/signin", Payload: "..."})
	// → [LOG] Start /signin
	// → [AUTH] Checking /signin
	// Sign in successful
	// ← [LOG] End   /signin
}
```

---

## Pitfalls

- A broken handler that forgets to call `next` silently drops the request.
- Order matters — `[auth, logging]` behaves differently from `[logging, auth]`.
- Hard to debug long chains — add logging at each link.

---

## Memory Aid

> Chain of Responsibility = assembly line. Each worker inspects the item, does their part (or rejects it), then passes it to the next.
