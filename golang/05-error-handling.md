# Error Handling in Go

> **Category**: Golang | **Level**: Intermediate

---

## What

Go treats errors as **values** — not exceptions. Functions signal failure by returning an `error` as the last return value. Callers check it explicitly. This makes error handling visible, deliberate, and composable.

---

## Why

Unlike exceptions (which unwind the call stack invisibly), returning errors as values:
- **Makes failure explicit** — you can't ignore an error without a deliberate decision
- **Composes cleanly** — errors can be wrapped, inspected, and matched
- **Avoids hidden control flow** — no surprise stack unwinding

---

## The `error` Interface

The built-in `error` is a simple interface:

```go
type error interface {
    Error() string
}
```

Any type with an `Error() string` method satisfies `error`.

---

## Creating Errors

### `errors.New` — Simple, Static Error

```go
import "errors"

var ErrNotFound = errors.New("record not found")

func FindUser(id string) (*User, error) {
    if id == "" {
        return nil, ErrNotFound
    }
    // ...
}
```

Use **package-level sentinel errors** (like `ErrNotFound`) for well-known conditions. Callers can compare against them with `errors.Is`.

---

### `fmt.Errorf` — Dynamic, Formatted Error

```go
import "fmt"

func GetOrder(id string) (*Order, error) {
    order, err := db.Query(id)
    if err != nil {
        // %w wraps the original error; callers can unwrap with errors.Is / errors.As
        return nil, fmt.Errorf("GetOrder id=%s: %w", id, err)
    }
    return order, nil
}
```

> `%w` is the **wrapping verb** — it embeds the original error so its type and value are still inspectable.

---

## Inspecting Errors

| Function | Does What | Common Use Case |
|----------|-----------|-----------------|
| `errors.Is(err, target)` | Reports whether `err` or any error in its chain **equals** `target` | Match sentinel errors: `errors.Is(err, ErrNotFound)` |
| `errors.As(err, target)` | Reports whether `err` or any error in its chain can be **assigned to** `target` | Extract the concrete error: `errors.As(err, &myErr)` |
| `errors.Unwrap(err)` | Returns the wrapped error one level up | Rarely used directly; usually `Is`/`As` is enough |

---

## Example — Wrapping and Inspecting

```go
package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("not found")

func findInDB(id int) error {
    if id == 0 {
        return ErrNotFound
    }
    return nil
}

func findUser(id int) error {
    if err := findInDB(id); err != nil {
        return fmt.Errorf("findUser id=%d: %w", id, err)
    }
    return nil
}

func main() {
    err := findUser(0)
    if err != nil {
        fmt.Println(err) // findUser id=0: not found

        // errors.Is walks the entire chain
        if errors.Is(err, ErrNotFound) {
            fmt.Println("→ It's a not-found error")
        }
    }
}
```

---

## Custom Error Types

Define a struct that implements `error` when you need extra context (e.g., HTTP status code, field name).

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: field=%s, msg=%s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    return nil
}

func main() {
    err := validateAge(-1)
    if err != nil {
        var valErr *ValidationError
        if errors.As(err, &valErr) {
            fmt.Println("Field:", valErr.Field)    // Field: age
            fmt.Println("Message:", valErr.Message) // Message: must be non-negative
        }
    }
}
```

---

## Common Pattern — Error Wrapping at Each Layer

Wrap errors with context as they propagate up the call stack, so you get a readable trace:

```go
// database layer
func dbGetUser(id string) (*User, error) {
    return nil, fmt.Errorf("dbGetUser: %w", sql.ErrNoRows)
}

// service layer
func serviceGetUser(id string) (*User, error) {
    u, err := dbGetUser(id)
    if err != nil {
        return nil, fmt.Errorf("serviceGetUser id=%s: %w", id, err)
    }
    return u, nil
}

// output: serviceGetUser id=abc: dbGetUser: sql: no rows in result set
```

---

## Best Practices

| Guideline | Why |
|-----------|-----|
| **Always check errors** | Ignoring `err` silently propagates failure |
| **Wrap with context at each layer** | Tells you *where* the failure happened, not just *what* |
| **Use sentinel errors for well-known conditions** | Lets callers match with `errors.Is` without string comparison |
| **Use custom types for structured error data** | Machine-readable — lets callers extract fields with `errors.As` |
| **Don't swallow errors** | If you can't handle it, wrap and return it |

---

## Pitfalls

- **`if err != nil { return err }`** without wrapping loses call-site context.
- **`fmt.Errorf` without `%w`** creates a new error that breaks `errors.Is`/`errors.As` chain traversal.
- **Checking `err.Error()` string** is fragile — strings change; use `errors.Is` / `errors.As` instead.

---

## Memory Aid

> Go errors are like receipts — return them, pass them up, and wrap them at each counter so the manager knows exactly what went wrong and where.

---

## What to Learn Next

- [06-defer-panic-recover.md](06-defer-panic-recover.md) — Handling truly unexpected failures
