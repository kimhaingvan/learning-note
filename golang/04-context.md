# Context in Go

> **Category**: Golang | **Level**: Intermediate

---

## What

`context.Context` is a standard interface for **carrying cancellation signals, deadlines, and request-scoped metadata** across API boundaries and goroutines.

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

---

## Why

Without context, there is no standard way to:
- **Cancel** a long-running operation (e.g., database query, HTTP call) when the caller gives up
- **Propagate timeouts** across goroutine boundaries
- **Pass request-scoped data** (trace ID, user ID) without threading it through every function signature

---

## When

Use `context.Context` as the **first argument** in any function that:
- Makes I/O calls (database, HTTP, gRPC)
- Spawns goroutines that should respect a parent's cancellation
- Runs as part of an HTTP/gRPC request lifecycle

```go
// Convention: ctx is always the first parameter
func GetUser(ctx context.Context, id string) (*User, error) { ... }
```

---

## Core Functions

| Function | Purpose | Notes |
|----------|---------|-------|
| `context.Background()` | Root context — no deadline, no cancel | Use at `main()`, top-level initialization |
| `context.TODO()` | Placeholder — "I'll figure out the right context later" | Use during development; replace before production |
| `context.WithCancel(ctx)` | Returns a child context + `cancel()` function | Caller manually cancels |
| `context.WithTimeout(ctx, d)` | Auto-cancels after duration `d` | Shorthand for `WithDeadline(ctx, time.Now().Add(d))` |
| `context.WithDeadline(ctx, t)` | Auto-cancels at absolute `time.Time` | Useful when you know the exact deadline |
| `context.WithValue(ctx, key, val)` | Attaches request-scoped data | Use typed keys only — never raw strings |

> All derived contexts form a **tree**. Cancelling a parent cancels all children.

---

## Example 1 — WithCancel: Manual Cancellation

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    go func() {
        time.Sleep(2 * time.Second)
        cancel() // trigger cancellation after 2s
    }()

    <-ctx.Done() // block until cancelled
    fmt.Println("Cancelled:", ctx.Err()) // context canceled
}
```

---

## Example 2 — WithTimeout: Deadline on DB Query

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func queryDB(ctx context.Context) error {
    select {
    case <-time.After(5 * time.Second): // simulate slow query
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel() // always call cancel to release resources

    if err := queryDB(ctx); err != nil {
        fmt.Println("Error:", err) // Error: context deadline exceeded
        return
    }
    fmt.Println("Query succeeded")
}
```

> Always `defer cancel()` immediately after `WithTimeout` / `WithCancel` — even if you know the deadline will be hit first. Forgetting leaks resources.

---

## Example 3 — WithValue: Passing Request-Scoped Metadata

```go
package main

import (
    "context"
    "fmt"
)

// Use a typed, unexported key to avoid collisions across packages
type contextKey string

const requestIDKey contextKey = "requestID"

func handler(ctx context.Context) {
    id, ok := ctx.Value(requestIDKey).(string)
    if !ok {
        fmt.Println("No request ID found")
        return
    }
    fmt.Println("Processing request:", id)
}

func main() {
    ctx := context.WithValue(context.Background(), requestIDKey, "req-123")
    handler(ctx) // Processing request: req-123
}
```

---

## Example 4 — Propagating Context through Function Calls

```go
func handleRequest(ctx context.Context, userID string) (*Order, error) {
    // 1. Create a child timeout context for this whole request
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // 2. Pass context down the call chain
    user, err := getUser(ctx, userID)
    if err != nil {
        return nil, err
    }

    orders, err := getOrders(ctx, user.ID)
    if err != nil {
        return nil, err
    }

    return processOrders(ctx, orders)
}
```

---

## Anti-Pattern — "Using But Not Checking"

```go
// ❌ BAD: ignores cancellation — the goroutine keeps working even when ctx is done
func processItems(ctx context.Context, items []Item) {
    for _, item := range items {
        processItem(item) // ctx not passed, no cancellation check
    }
}

// ✅ GOOD: respects cancellation in the loop
func processItems(ctx context.Context, items []Item) error {
    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            processItem(ctx, item)
        }
    }
    return nil
}
```

---

## Best Practices

| Rule | Why |
|------|-----|
| **`ctx` is always the first parameter** | Convention — consistent across all Go code |
| **Never store context in a struct** | Context is request-scoped, not struct-scoped |
| **Always call `cancel()`** | Leaked contexts hold goroutines and memory |
| **Use typed keys for `WithValue`** | Prevents collisions between packages using the same key string |
| **Don't pass `nil`** | Pass `context.Background()` or `context.TODO()` instead |

---

## Pitfalls

- **Storing context in a struct** — breaks the request-scoped contract; pass it explicitly.
- **Forgetting `defer cancel()`** — goroutines and timers leak until the program exits.
- **Using a plain `string` as a WithValue key** — any package can overwrite or read it; use a private custom type.

---

## Memory Aid

> Context flows **downward** through the call tree — like water. Once a parent is cancelled, all children are cancelled too. Never store it, always pass it.

---

## What to Learn Next

- [05-error-handling.md](05-error-handling.md) — Wrapping errors and `ctx.Err()` handling
- [07-concurrency.md](07-concurrency.md) — Cancelling goroutines with context
