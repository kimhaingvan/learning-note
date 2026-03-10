# Testing in Go

> **Category**: Golang | **Level**: Intermediate

---

## What

Go has a **built-in testing framework** in the `testing` package. No third-party test runner needed. Tests live in `_test.go` files alongside the code they test.

---

## Why

- Tests are **first-class citizens** in Go — `go test` is built into the toolchain.
- The table-driven pattern keeps tests **organised, readable, and easy to extend**.
- Benchmarks (`Benchmark*`) in the same framework let you **measure performance** without extra setup.

---

## Test File Conventions

| Convention | Rule |
|------------|------|
| **File name** | Must end in `_test.go` |
| **Package** | Use `package foo` (white-box) or `package foo_test` (black-box) |
| **Test function name** | Must start with `Test` |
| **Benchmark function name** | Must start with `Benchmark` |
| **Example function name** | Must start with `Example` |
| **Signature** | `func TestXxx(t *testing.T)` |

---

## Running Tests

```bash
# Run all tests in the current module
go test ./...

# Run tests in a specific package
go test ./pkg/users/

# Run a specific test by name (regex)
go test -run TestGetUser ./...

# Run with verbose output
go test -v ./...

# Run with race detector (detect concurrent access bugs)
go test -race ./...

# Run benchmarks
go test -bench=. ./...

# Measure coverage
go test -cover ./...
go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out
```

---

## Table-Driven Tests

The standard Go pattern for testing multiple cases against one function.

```go
// calculator.go
package calc

func Add(a, b int) int { return a + b }
func Div(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}
```

```go
// calculator_test.go
package calc

import (
    "errors"
    "testing"
)

func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -1, -2, -3},
        {"zero", 0, 5, 5},
        {"large numbers", 1000, 2000, 3000},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            result := Add(tc.a, tc.b)
            if result != tc.expected {
                t.Errorf("Add(%d, %d) = %d; want %d", tc.a, tc.b, result, tc.expected)
            }
        })
    }
}

func TestDiv(t *testing.T) {
    tests := []struct {
        name        string
        a, b        int
        expected    int
        expectedErr error
    }{
        {"normal division", 10, 2, 5, nil},
        {"division by zero", 5, 0, 0, errors.New("division by zero")},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            result, err := Div(tc.a, tc.b)

            if tc.expectedErr != nil {
                if err == nil {
                    t.Errorf("expected error %v, got nil", tc.expectedErr)
                }
                return
            }

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if result != tc.expected {
                t.Errorf("Div(%d, %d) = %d; want %d", tc.a, tc.b, result, tc.expected)
            }
        })
    }
}
```

### `t.Run` and Sub-tests

`t.Run(name, func)` creates a **sub-test** — useful for:
- Isolating failures to specific cases
- Running individual cases with `-run TestAdd/zero`
- Parallel sub-tests with `t.Parallel()`

---

## Common `testing.T` Methods

| Method | When to Use |
|--------|-------------|
| `t.Errorf(msg, ...)` | Fail the test and log, but **continue** running |
| `t.Fatalf(msg, ...)` | Fail the test and **stop immediately** |
| `t.Logf(msg, ...)` | Log a message (shown with `-v`) |
| `t.Helper()` | Mark a function as a helper — test output points to the caller, not the helper |
| `t.Parallel()` | Allow this test to run in parallel with other parallel tests |
| `t.Cleanup(func())` | Register cleanup to run after the test (like `defer`, but scoped to the test) |
| `t.Skip(msg)` | Skip the test with an explanation |

---

## Benchmarks

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(123, 456)
    }
}
```

```bash
go test -bench=BenchmarkAdd -benchmem ./...

# Example output:
# BenchmarkAdd-8   1000000000   0.278 ns/op   0 B/op   0 allocs/op
```

- `b.N` is set automatically by the framework to make the benchmark run long enough for stable timing.
- `-benchmem` shows memory allocations per operation.

---

## Mocking with `mockery`

[mockery](https://vektra.github.io/mockery/latest/) generates mock implementations of interfaces automatically.

### Install

```bash
go install github.com/vektra/mockery/v2@latest
```

### Generate a Mock

Given an interface:
```go
// users/repository.go
package users

type Repository interface {
    GetByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, user *User) error
}
```

Run:
```bash
mockery --name=Repository --dir=./users --output=./users/mocks
```

### Use the Mock in Tests

```go
package users_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "yourmodule/users/mocks"
)

func TestGetUser(t *testing.T) {
    mockRepo := mocks.NewRepository(t)

    // Set expectations
    expected := &User{ID: "abc", Name: "Alice"}
    mockRepo.On("GetByID", context.Background(), "abc").Return(expected, nil)

    service := NewUserService(mockRepo)
    user, err := service.GetUser(context.Background(), "abc")

    assert.NoError(t, err)
    assert.Equal(t, expected, user)
    mockRepo.AssertExpectations(t)
}
```

> `mockery` v2 generates mocks that integrate with [testify/mock](https://github.com/stretchr/testify#mock-package). Pass `t` to `mocks.NewRepository(t)` — the mock auto-fails the test if unexpected calls are made.

---

## Test Helpers Pattern

```go
// helpers_test.go
func assertError(t *testing.T, err error, target error) {
    t.Helper() // marks this as a helper — error points to caller line
    if !errors.Is(err, target) {
        t.Errorf("expected error %v, got %v", target, err)
    }
}
```

---

## Best Practices

| Practice | Why |
|----------|-----|
| **Table-driven tests** | Clear, extensible, avoids repeated boilerplate |
| **Sub-tests with `t.Run`** | Isolate failures; run individual cases |
| **`t.Helper()` in helper functions** | Accurate error line numbers |
| **Test the public API (black-box)** | `package foo_test` — tests what callers actually see |
| **Use `t.Cleanup`** over manual defer | Cleanup runs even if test panics |
| **`-race` in CI** | Catch race conditions before they reach production |

---

## Pitfalls

- **`t.Fatal` in a goroutine** — only the goroutine panics; the test doesn't fail as expected. Use `t.Error` + channels or sync primitives.
- **Not calling `AssertExpectations`** on mocks — unexpectedly uncalled mocks pass silently.
- **Shared state between table test cases** — make sure each `tc` in the loop is captured correctly (use `tc := tc` inside the loop when not using `t.Run`).

---

## Memory Aid

> A test is a contract: given these inputs, the code must produce these outputs. Table-driven tests are the contract's terms, written in a table so the contract is easy to read, extend, and enforce.

---

## What to Learn Next

- [07-concurrency.md](07-concurrency.md) — Testing concurrent code with `-race` flag
- [05-error-handling.md](05-error-handling.md) — Testing error paths and using `errors.Is` in assertions
