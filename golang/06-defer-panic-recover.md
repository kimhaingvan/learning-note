# Defer, Panic, and Recover

> **Category**: Golang | **Level**: Intermediate

---

## What

| Keyword | Purpose |
|---------|---------|
| **`defer`** | Registers a function call to run **after the surrounding function returns** |
| **`panic`** | Stops normal execution — begins unwinding the call stack |
| **`recover`** | Inside a deferred function, **catches** a panic and returns control |

---

## Why

- `defer` ensures cleanup code (closing files, releasing locks, logging) always runs — even on error returns.
- `panic` / `recover` handle **truly unexpected failures** (e.g., nil pointer dereference, invalid array index) without crashing the whole program.

---

## `defer` — Execution Order and Mechanics

### LIFO (Last In, First Out)

Multiple `defer` statements stack up and execute in reverse order:

```go
func example() {
    defer fmt.Println("first")
    defer fmt.Println("second")
    defer fmt.Println("third")
}

// Output:
// third
// second
// first
```

### Variable Capture — Named vs Anonymous Function

This is the most common `defer` gotcha:

| Style | When is the value captured? | Behaviour |
|-------|---------------------------|-----------|
| Direct call `defer f(x)` | **Immediately** when defer is registered | Uses the value at registration time |
| Named return variable | At **function return time** | Sees the final value of a named `return` variable |
| Anonymous closure `defer func() { ... }()` | **At execution time** | Sees the value at the time the deferred function runs |

```go
// ❌ Counter may not be what you expect
i := 0
defer fmt.Println(i)  // captured as 0 right now
i = 42
// Output: 0

// ✅ Closure captures variable by reference — sees current value at execution time
i := 0
defer func() { fmt.Println(i) }()
i = 42
// Output: 42
```

### Common Pattern — Resource Cleanup

```go
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // guaranteed to run regardless of what happens below

    // ... read and process file ...
    return nil
}
```

---

## `panic` — Abnormal Termination

`panic` stops the function, runs all deferred functions in the current goroutine, then propagates **up the call stack** doing the same at each level until the program exits.

### What Causes a Panic?

| Cause | Example |
|-------|---------|
| Explicit call | `panic("something went wrong")` |
| Runtime: nil dereference | `var p *int; _ = *p` |
| Runtime: out-of-bounds index | `a := []int{1}; _ = a[5]` |
| Runtime: divide by zero | `x := 1 / 0` |

### What Happens When Panic Fires

1. Current function **stops immediately**.
2. All **deferred functions** in the panicking goroutine run (LIFO order).
3. If not recovered, **stack trace is printed** and the program exits with **exit code 2**.

---

## `recover` — Catching a Panic

`recover` stops the panic and returns the value passed to `panic()`. It **only works inside a deferred function**:

```go
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()

    return a / b, nil // panics if b == 0
}

func main() {
    result, err := safeDiv(10, 0)
    if err != nil {
        fmt.Println("Error:", err) // Error: recovered from panic: runtime error: integer divide by zero
        return
    }
    fmt.Println("Result:", result)
}
```

> ⚠️ Calling `recover()` outside of a deferred function returns `nil` and has no effect.

---

## Wrapping a Panic into an Error

A common pattern — especially in libraries — to prevent panics from escaping to callers:

```go
func safeExecute(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic caught: %v\n%s", r, debug.Stack())
        }
    }()
    fn()
    return nil
}

func main() {
    err := safeExecute(func() {
        panic("unexpected state")
    })
    fmt.Println(err)
    // panic caught: unexpected state
    // goroutine 1 [running]: ...
}
```

---

## When to Use Each

| Situation | Use |
|-----------|-----|
| Resource that always needs cleanup (file, DB conn, lock) | `defer` |
| Wrapping "before/after" logic around a function | `defer` |
| Truly unrecoverable state — programmer error | `panic` |
| Stopping propagation of a panic at a boundary (e.g., HTTP handler) | `recover` in `defer` |
| Normal expected errors (validation, not found, I/O) | Return `error` — **not** panic |

---

## Pitfalls

- **`defer` in a loop**: the deferred call runs when the **function exits**, not when each loop iteration ends.

  ```go
  // ❌ Closes all files only after the loop finishes — may hold too many open
  for _, f := range files {
      defer f.Close()
  }

  // ✅ Use an inner function to scope the defer per iteration
  for _, f := range files {
      func() {
          defer f.Close()
          process(f)
      }()
  }
  ```

- **Panic in a different goroutine** is unrecoverable from the launching goroutine — each goroutine must `recover` its own panics.

- **Overusing panic/recover** as a substitute for error handling is an anti-pattern. Use returned errors for expected failures.

---

## Memory Aid

> `defer` = "do this on the way out, no matter what."
> `panic` = fire alarm — everything stops, evacuate.
> `recover` = fire extinguisher — only works inside a deferred call, catches the panic before it brings the building down.

---

## What to Learn Next

- [07-concurrency.md](07-concurrency.md) — Goroutines, channels, and recovery in concurrent code
