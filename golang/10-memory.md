# Memory Management in Go

> **Category**: Golang | **Level**: Intermediate

---

## What

Go manages memory using two regions:

| Region | Managed by | Lifetime |
|--------|-----------|---------|
| **Stack** | Compiler — automatically allocated and freed per function call | Lives as long as the function call frame |
| **Heap** | Go garbage collector (GC) | Lives until no more references exist |

The **Go compiler** decides, at compile time, whether an allocation goes on the stack or the heap — a process called **escape analysis**.

---

## Why It Matters

- **Stack allocations are fast** — no GC pressure, automatically freed when the function returns.
- **Heap allocations are slower** — GC must track and eventually collect them.
- Understanding how the compiler allocates memory helps you write code with less GC overhead, especially in performance-sensitive paths.

---

## Stack Allocation

A variable is placed on the **stack** when:

| Condition | Why Stack is Safe |
|-----------|------------------|
| Local variable — not referenced after function returns | Its lifetime ends with the function call |
| Small, fixed-size value (int, bool, small struct) | Known size at compile time |
| Does **not** escape to heap (the compiler verified it) | No external code holds a reference to it |

```go
func stackExample() int {
    x := 42           // stack allocated — x is local, never returned by pointer
    y := x + 1        // stack allocated
    return y           // y's value is copied to caller — no pointer escapes
}
```

---

## Heap Allocation

A variable **escapes to the heap** when:

| Escape Condition | Example |
|-----------------|---------|
| **Pointer returned from function** | `func newFoo() *Foo { return &Foo{} }` — callers hold a reference |
| **Pointer stored in an interface** | Any value assigned to `interface{}` or another interface |
| **Variable captured by a closure** | Goroutine or `defer` closure captures a local variable |
| **Maps and slices whose backing array grows** | Dynamic resizing requires heap allocation |
| **Large objects** | Too large for the current goroutine's stack |
| **Size unknown at compile time** | Dynamic-length slices, `make([]T, n)` where n is a variable |

```go
func heapExample() *int {
    x := 42
    return &x // x ESCAPES — caller holds the pointer, so x must outlive this function
}
```

---

## Escape Analysis — How the Compiler Decides

The Go compiler runs **escape analysis** during compilation to decide where each variable lives.

```bash
# Run escape analysis and print decisions
go build -gcflags="-m" ./...
```

Example output:
```
./main.go:8:12: &Foo{} escapes to heap
./main.go:15:13: x does not escape
```

---

## Practical Examples

### Example 1 — Returning a Value (Stack) vs Pointer (Heap)

```go
type Point struct{ X, Y float64 }

// Value return — Point is stack allocated in callee, copied to caller
func newPoint() Point {
    return Point{1.0, 2.0}
}

// Pointer return — Point escapes to heap, caller holds a reference
func newPointPtr() *Point {
    return &Point{1.0, 2.0}
}
```

> Go is efficient at copying small structs — prefer returning values unless you intentionally want sharing.

---

### Example 2 — Interface Causes Escape

```go
func printValue(v interface{}) {
    fmt.Println(v)
}

func main() {
    x := 42
    printValue(x) // x is boxed into an interface — may escape to heap
}
```

---

### Example 3 — Goroutine Closure Captures Variable

```go
func launchWorker() {
    result := 0          // result escapes because the goroutine closure captures it
    go func() {
        result = compute() // closure holds reference to 'result'
    }()
}
```

---

## Stack vs Heap — Summary

| | Stack | Heap |
|-|-------|------|
| **Allocation speed** | Very fast (pointer bump) | Slower (GC bookkeeping) |
| **GC involvement** | None | Collected when unreachable |
| **Lifetime** | Tied to function call | Until last reference is gone |
| **Size** | Fixed per goroutine (starts ~2 KB, grows dynamically) | Limited by available RAM |
| **Thread safety** | Private per goroutine | Shared — requires synchronisation |

---

## Goroutine Stacks

Each goroutine has its **own stack** that starts small (~2 KB) and **grows dynamically** as needed (up to a configurable limit, default 1 GB). This is what makes it cheap to launch millions of goroutines.

---

## Reducing GC Pressure — Practical Tips

| Tip | Why It Helps |
|-----|-------------|
| **Return values, not pointers** for small structs | Keeps allocation on the stack |
| **Reuse objects with `sync.Pool`** | Reduces heap churn for short-lived objects |
| **Pre-allocate slices with known capacity** | `make([]T, 0, n)` avoids repeated reallocation |
| **Avoid excessive interface boxing** | Passing concrete types where possible avoids escape |
| **Use in-place mutation (pointer receiver)** for large structs | Avoids copying large struct values |

```go
// sync.Pool example — reuse byte buffers
var pool = sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}

func handler() {
    buf := pool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        pool.Put(buf)
    }()
    buf.WriteString("hello")
    // use buf...
}
```

---

## Pitfalls

- **Premature optimisation** — Do not manually try to force stack allocation. Let the compiler decide. Optimise only when profiling shows GC is a bottleneck.
- **`sync.Pool` objects are cleared between GC cycles** — Do not store long-lived state in a Pool.
- **Large goroutine stacks** — Infinite recursion causes stack overflow (goroutine stack grows until the limit is hit).

---

## Memory Aid

> Stack = a *sticky note* on your desk — fast, temporary, gone when you clear it.
> Heap = a *storage room* — shared, durable, cleaned by the GC janitor when nobody needs the item anymore.

---

## What to Learn Next

- [11-testing.md](11-testing.md) — Write tests and benchmarks to validate and measure your code
