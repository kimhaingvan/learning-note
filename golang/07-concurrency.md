# Concurrency in Go

> **Category**: Golang | **Level**: Intermediate

---

## What

Go's concurrency model is built on three primitives:

| Primitive | Purpose |
|-----------|---------|
| **Goroutine** | Lightweight thread — spawned with `go func()` |
| **Channel** | Typed conduit for communication between goroutines |
| **Sync primitives** | `WaitGroup`, `Mutex`, `RWMutex` — coordinate shared access |

**Go's philosophy:** *"Do not communicate by sharing memory; instead, share memory by communicating."*

---

## Goroutines

A goroutine is a function running **concurrently** with other goroutines in the same address space.

### How Go Manages Goroutines (M:N Scheduling)

- **Multiple goroutines** (G) are multiplexed onto **multiple OS threads** (M) by the **Go scheduler** (P = logical processor).
- `GOMAXPROCS` controls how many OS threads run in parallel (default = number of CPU cores).
- Each goroutine starts with a **small stack (~2 KB)** that grows dynamically — you can run millions concurrently.

```go
func printGreeting(name string) {
    fmt.Printf("Hello, %s!\n", name)
}

func main() {
    go printGreeting("Alice") // spawns goroutine
    go printGreeting("Bob")   // spawns goroutine

    time.Sleep(time.Second) // wait for goroutines (use WaitGroup in production)
}
```

---

## Channels

A channel is a **typed, goroutine-safe** queue for passing values between goroutines.

```go
ch := make(chan int)      // unbuffered
ch := make(chan int, 10)  // buffered with capacity 10
```

### Unbuffered vs Buffered

| Property | Unbuffered `make(chan T)` | Buffered `make(chan T, n)` |
|----------|--------------------------|---------------------------|
| **Synchronisation** | **Synchronous** — sender blocks until receiver reads | **Asynchronous** — sender blocks only when buffer is full |
| **Use case** | Handoff / signal between goroutines | Rate limiting, producer-consumer with burst |
| **Send blocks when** | No receiver ready | Buffer is full |
| **Receive blocks when** | No sender ready | Buffer is empty |

### Closing and Ranging

```go
ch := make(chan int, 3)
ch <- 1; ch <- 2; ch <- 3
close(ch) // signals no more values will be sent

// range exits when channel is closed and drained
for v := range ch {
    fmt.Println(v) // 1, 2, 3
}

// Two-value receive: ok == false means channel is closed and empty
v, ok := <-ch
```

> ⚠️ Only the **sender** should close a channel. Closing from the receiver side or closing twice causes a panic.

### Direction-Typed Channels

```go
func producer(out chan<- int) { /* send-only */ }
func consumer(in <-chan int)  { /* receive-only */ }
```

---

## sync.WaitGroup — Wait for All Goroutines

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1) // increment counter before launching goroutine
        go func(n int) {
            defer wg.Done() // decrement on exit
            fmt.Printf("Worker %d done\n", n)
        }(i)
    }

    wg.Wait() // block until counter reaches 0
    fmt.Println("All workers finished")
}
```

> Always call `wg.Add(n)` **before** launching the goroutine — calling it inside the goroutine is a race condition.

---

## sync.Mutex and sync.RWMutex — Protect Shared State

Use when multiple goroutines read or write the same data.

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock() // released even if Increment panics
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

### Mutex vs RWMutex

| Scenario | Use |
|----------|-----|
| Mixed reads and writes | `sync.Mutex` |
| **Many reads, rare writes** | `sync.RWMutex` — `RLock`/`RUnlock` allow concurrent readers |

```go
var rwmu sync.RWMutex

// Multiple goroutines can hold RLock simultaneously
rwmu.RLock()
defer rwmu.RUnlock()
// read data...

// Only one goroutine can hold Lock at a time
rwmu.Lock()
defer rwmu.Unlock()
// write data...
```

---

## Concurrency vs Parallelism

| | Concurrency | Parallelism |
|-|-------------|-------------|
| **Definition** | Dealing with many things at once (structure) | Doing many things at the same time (execution) |
| **Requires** | A single core is enough | Multiple CPU cores |
| **In Go** | Goroutines are always concurrent | Parallelism happens when `GOMAXPROCS > 1` |

---

## Concurrency Patterns

### 1. Worker Pool

Limit the number of concurrent goroutines processing a queue of jobs.

```go
func workerPool(jobs <-chan int, results chan<- int, numWorkers int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- job * 2 // process job
            }
        }()
    }
    go func() {
        wg.Wait()
        close(results)
    }()
}

func main() {
    jobs := make(chan int, 10)
    results := make(chan int, 10)

    workerPool(jobs, results, 3)

    for i := 1; i <= 6; i++ {
        jobs <- i
    }
    close(jobs)

    for r := range results {
        fmt.Println(r)
    }
}
```

---

### 2. Fan-Out / Fan-In

**Fan-Out**: distribute work across multiple goroutines.
**Fan-In**: merge multiple output channels into one.

```go
func fanOut(input <-chan int, n int) []<-chan int {
    outputs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        ch := make(chan int)
        outputs[i] = ch
        go func(out chan<- int) {
            for v := range input {
                out <- v * v
            }
            close(out)
        }(ch)
    }
    return outputs
}

func fanIn(channels ...<-chan int) <-chan int {
    merged := make(chan int)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                merged <- v
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(merged)
    }()
    return merged
}
```

---

### 3. Pipeline

Chain stages where each stage reads from the previous channel and writes to the next.

```go
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    c := generate(2, 3, 4)
    out := square(c)
    for v := range out {
        fmt.Println(v) // 4, 9, 16
    }
}
```

---

### 4. Cancellation with Context

```go
func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d: stopping — %v\n", id, ctx.Err())
            return
        default:
            fmt.Printf("Worker %d: working...\n", id)
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    var wg sync.WaitGroup
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            worker(ctx, id)
        }(i)
    }
    wg.Wait()
}
```

---

### 5. Future / Promise

Run a computation asynchronously; retrieve result later via channel.

```go
func asyncCompute(x int) <-chan int {
    result := make(chan int, 1)
    go func() {
        time.Sleep(time.Second)
        result <- x * x
        close(result)
    }()
    return result
}

func main() {
    future := asyncCompute(5)
    // do other work...
    fmt.Println("Result:", <-future) // 25
}
```

---

### 6. Pub/Sub (Publisher / Subscriber)

A hub broadcasts messages to multiple subscriber channels.

```go
type PubSub struct {
    mu          sync.RWMutex
    subscribers map[string][]chan string
}

func (ps *PubSub) Subscribe(topic string) <-chan string {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    ch := make(chan string, 1)
    ps.subscribers[topic] = append(ps.subscribers[topic], ch)
    return ch
}

func (ps *PubSub) Publish(topic, msg string) {
    ps.mu.RLock()
    defer ps.mu.RUnlock()
    for _, ch := range ps.subscribers[topic] {
        go func(c chan string) { c <- msg }(ch)
    }
}
```

---

### 7. Scatter / Gather

Scatter a request to N workers; gather the first (or all) results.

```go
func scatter(ctx context.Context, query string, sources []string) <-chan string {
    results := make(chan string, len(sources))
    for _, src := range sources {
        go func(url string) {
            // simulate fetch
            time.Sleep(time.Duration(rand.Intn(300)) * time.Millisecond)
            select {
            case results <- fmt.Sprintf("result from %s", url):
            case <-ctx.Done():
            }
        }(src)
    }
    return results
}

func gather(ctx context.Context, n int, results <-chan string) []string {
    out := make([]string, 0, n)
    for i := 0; i < n; i++ {
        select {
        case r := <-results:
            out = append(out, r)
        case <-ctx.Done():
            return out
        }
    }
    return out
}
```

---

### 8. Event Loop

Process events from multiple sources with `select`.

```go
func eventLoop(ctx context.Context, events <-chan string, ticks <-chan time.Time) {
    for {
        select {
        case event, ok := <-events:
            if !ok {
                return
            }
            fmt.Println("Event:", event)
        case t := <-ticks:
            fmt.Println("Tick at", t.Format("15:04:05"))
        case <-ctx.Done():
            fmt.Println("Shutting down event loop")
            return
        }
    }
}
```

---

## Pitfalls

- **Goroutine leak**: starting a goroutine without a way for it to exit (always pair with context or channel close).
- **Closing a channel twice** or **closing from the receiver** panics.
- **Sending to a nil channel** blocks forever; **receiving from a nil channel** blocks forever.
- **Copying a Mutex** (e.g., passing struct by value) — always use a pointer to the struct containing a Mutex.
- **`wg.Add` inside the goroutine** — the parent may call `wg.Wait()` before all goroutines have had a chance to `Add`.

---

## Memory Aid

> **"Don't communicate by sharing memory; share memory by communicating."**
> Channels pass ownership of data — only one goroutine touches it at a time. Mutex guards data that truly must be shared.

---

## What to Learn Next

- [08-generics.md](08-generics.md) — Type-safe reusable code
- [04-context.md](04-context.md) — Cancelling goroutines gracefully
