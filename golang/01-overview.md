# Golang — Learning Roadmap & Overview

> **Category**: Golang | **Level**: Intermediate

---

## Core Concepts to Master

| Topic | File |
|-------|------|
| OOP in Go | [02-oop-in-go.md](02-oop-in-go.md) |
| Pointer vs Value Methods | [03-pointer-value-methods.md](03-pointer-value-methods.md) |
| Context | [04-context.md](04-context.md) |
| Error Handling | [05-error-handling.md](05-error-handling.md) |
| Defer, Panic, Recover | [06-defer-panic-recover.md](06-defer-panic-recover.md) |
| Concurrency (Goroutines, Channels, WaitGroup, Mutex) | [07-concurrency.md](07-concurrency.md) |
| Generics | [08-generics.md](08-generics.md) |
| Reflection | [09-reflection.md](09-reflection.md) |
| Memory Management | [10-memory.md](10-memory.md) |
| Testing & Benchmarking | [11-testing.md](11-testing.md) |

---

## Packages Worth Exploring

- `sync` — Mutex, RWMutex, WaitGroup, Once, Map
- `net` — TCP/UDP servers and clients
- `context` — cancellation, deadlines, request-scoped values
- `reflect` — runtime type inspection and manipulation
- `testing` — unit tests, benchmarks, examples

---

## Practice Exercises

| Exercise | Concepts Covered |
|----------|-----------------|
| Asynchronous Order Processing | Goroutines, channels, worker pools |
| Logging System with Concurrent Processing | Mutex, WaitGroup, buffered channels |
| Real-Time Inventory Management | Context cancellation, goroutines, sync |

---

## gRPC in Go

- Use `google.golang.org/grpc` for building RPC services.
- Define service contracts in `.proto` files with Protocol Buffers.
- Key patterns: unary RPC, server streaming, client streaming, bidirectional streaming.

---

## Recommended Reading

- **"Concurrency in Go: Tools and Techniques for Developers"** — Katherine Cox-Buday
  - Deep coverage of goroutines, channels, sync primitives, and concurrency patterns.

---

## What to Learn Next

- Start with [02-oop-in-go.md](02-oop-in-go.md) — Go's approach to OOP concepts.
