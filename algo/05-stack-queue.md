# Stack and Queue

> **Category**: Algorithms | **Level**: Intermediate

---

## What

Both Stack and Queue are **restricted-access lists** — they limit *where* you can insert and remove elements.

| Structure | Order | Insert at | Remove from | Nickname |
|-----------|-------|-----------|-------------|---------|
| **Stack** | LIFO — Last In, First Out | Top | Top | "The plate stack" |
| **Queue** | FIFO — First In, First Out | Rear (back) | Front (head) | "The ticket line" |

---

## Stack (LIFO)

### Core Operations

| Operation | Description | Complexity |
|-----------|-------------|-----------|
| `Push(v)` | Add element to the top | O(1) |
| `Pop()` | Remove and return the top element | O(1) |
| `Peek()` / `Top()` | Read the top without removing | O(1) |
| `IsEmpty()` | Check if stack is empty | O(1) |

### Visual Behaviour

```
Initial:   [ ]
Push(3)  → [ 3 ]         ← Top
Push(5)  → [ 3, 5 ]      ← Top
Push(7)  → [ 3, 5, 7 ]   ← Top
Pop()    → [ 3, 5 ]       ← Top  (returns 7)
Pop()    → [ 3 ]          ← Top  (returns 5)
```

---

### Implementation 1 — Stack using Slice (recommended for Go)

Using a Go slice as the backing array. `append` gives O(1) amortized push; slicing the last element is O(1) pop.

```go
package main

import "fmt"

// Stack is generic — works with any type
type Stack[T any] struct {
    a []T
}

func (s *Stack[T]) Push(v T) {
    s.a = append(s.a, v)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.a) == 0 {
        return zero, false
    }
    v := s.a[len(s.a)-1]
    s.a = s.a[:len(s.a)-1]
    return v, true
}

func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.a) == 0 {
        return zero, false
    }
    return s.a[len(s.a)-1], true
}

func (s *Stack[T]) Len() int { return len(s.a) }

func main() {
    var st Stack[int]
    st.Push(3); st.Push(5); st.Push(7)
    for st.Len() > 0 {
        v, _ := st.Pop()
        fmt.Println(v) // 7, 5, 3
    }
}
```

---

### Implementation 2 — Stack using Linked List

Best when you need guaranteed O(1) operations without any hidden reallocation, or when you want GC-controlled small objects.

```go
type lnode[T any] struct {
    v    T
    next *lnode[T]
}

type LinkedStack[T any] struct {
    head *lnode[T]
    n    int
}

func (s *LinkedStack[T]) Push(v T) {
    s.head = &lnode[T]{v: v, next: s.head}
    s.n++
}

func (s *LinkedStack[T]) Pop() (T, bool) {
    var zero T
    if s.head == nil {
        return zero, false
    }
    v := s.head.v
    s.head = s.head.next
    s.n--
    return v, true
}

func (s *LinkedStack[T]) Len() int { return s.n }
```

---

### Real-World Applications of Stack

| Application | How Stack Is Used |
|-------------|-------------------|
| Function call stack | Each call pushes a frame; return pops it |
| Browser back button | Each page visited is pushed; "Back" pops |
| Undo in text editors | Each edit is pushed; Ctrl+Z pops |
| Expression evaluation | Evaluate `2 + 3 * 4` using operator precedence |
| Balanced brackets check | Push `(`, `[`, `{`; match and pop on `)`, `]`, `}` |
| DFS (Depth-First Search) | Iterative DFS uses an explicit stack |

### Example — Balanced Brackets

```go
func IsBalanced(s string) bool {
    var st Stack[rune]
    match := map[rune]rune{')': '(', ']': '[', '}': '{'}
    for _, ch := range s {
        switch ch {
        case '(', '[', '{':
            st.Push(ch)
        case ')', ']', '}':
            top, ok := st.Pop()
            if !ok || top != match[ch] {
                return false
            }
        }
    }
    return st.Len() == 0
}

// IsBalanced("({[]})") → true
// IsBalanced("({[})") → false
```

---

## Queue (FIFO)

### Core Operations

| Operation | Description | Complexity |
|-----------|-------------|-----------|
| `Enqueue(v)` / `Push(v)` | Add element at the rear | O(1) |
| `Dequeue()` / `Pop()` | Remove and return the front element | O(1) |
| `Front()` / `Peek()` | Read the front without removing | O(1) |
| `IsEmpty()` | Check if queue is empty | O(1) |

### Visual Behaviour

```
Initial:     [ ]
Enq(2)  →    [ 2 ]              Front=2, Rear=2
Enq(4)  →    [ 2, 4 ]           Front=2, Rear=4
Enq(6)  →    [ 2, 4, 6 ]        Front=2, Rear=6
Deq()   →    [ 4, 6 ]           returns 2
Deq()   →    [ 6 ]              returns 4
```

---

### Implementation 1 — Ring Buffer (Circular Array Queue)

A ring buffer uses modular arithmetic on `head` and `tail` indices to avoid shifting elements. This mirrors the "circular array" queue from textbooks.

```
Indices wrap around: (index + 1) % capacity

buf:  [ 4, 5, 6, _, _, 1, 2, 3 ]
                        ↑        ↑
                       head     tail
```

```go
package main

import "fmt"

type RingQueue[T any] struct {
    buf        []T
    head, tail int  // head = next to dequeue, tail = next position to enqueue
    n          int  // current number of elements
}

func NewRingQueue[T any](cap int) *RingQueue[T] {
    return &RingQueue[T]{buf: make([]T, cap)}
}

func (q *RingQueue[T]) Enq(v T) bool {
    if q.n == len(q.buf) {
        return false // full
    }
    q.buf[q.tail] = v
    q.tail = (q.tail + 1) % len(q.buf) // wrap around
    q.n++
    return true
}

func (q *RingQueue[T]) Deq() (T, bool) {
    var zero T
    if q.n == 0 {
        return zero, false // empty
    }
    v := q.buf[q.head]
    q.head = (q.head + 1) % len(q.buf) // wrap around
    q.n--
    return v, true
}

func (q *RingQueue[T]) Len() int  { return q.n }
func (q *RingQueue[T]) Cap() int  { return len(q.buf) }

func main() {
    q := NewRingQueue[int](4)
    q.Enq(1); q.Enq(2); q.Enq(3)
    v, _ := q.Deq()
    fmt.Println(v)       // 1
    q.Enq(4); q.Enq(5)  // reuses the slot freed by Deq
    for q.Len() > 0 {
        v, _ := q.Deq()
        fmt.Print(v, " ") // 2 3 4 5
    }
}
```

> **Why ring buffer?** A naive array-backed queue that removes from the front must shift O(n) elements. The ring buffer uses index arithmetic to avoid all shifting.

---

### Implementation 2 — Linked List Queue

Best when you need **unbounded dynamic capacity** — no fixed size, grows on demand.

```go
type qnode[T any] struct {
    v    T
    next *qnode[T]
}

type LinkedQueue[T any] struct {
    head, tail *qnode[T]
    n          int
}

func (q *LinkedQueue[T]) Enq(v T) {
    nn := &qnode[T]{v: v}
    if q.tail != nil {
        q.tail.next = nn
    } else {
        q.head = nn
    }
    q.tail = nn
    q.n++
}

func (q *LinkedQueue[T]) Deq() (T, bool) {
    var zero T
    if q.head == nil {
        return zero, false
    }
    v := q.head.v
    q.head = q.head.next
    if q.head == nil {
        q.tail = nil // list became empty; tail must also be nil
    }
    q.n--
    return v, true
}

func (q *LinkedQueue[T]) Len() int { return q.n }
```

---

### Implementation 3 — Go Channel Queue (concurrent-safe)

For producer-consumer pipelines, Go's built-in channel is the idiomatic queue:

```go
jobs := make(chan int, 1024) // buffered — producer rarely blocks

// Producer
go func() {
    for _, task := range tasks {
        jobs <- task
    }
    close(jobs)
}()

// Consumer
go func() {
    for task := range jobs {
        process(task)
    }
}()
```

---

### Real-World Applications of Queue

| Application | How Queue Is Used |
|-------------|-------------------|
| CPU / Task scheduling | OS processes tasks in arrival order |
| BFS (Breadth-First Search) | Nodes are enqueued level by level |
| Print spooler | Documents printed in submission order |
| Request buffer in web servers | Incoming HTTP requests queued |
| Message brokers (Kafka, RabbitMQ) | Messages consumed in FIFO order |

### Example — BFS using Queue

```go
func BFS(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var result []int
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        node := queue[0]      // dequeue from front
        queue = queue[1:]

        result = append(result, node.Val)

        if node.Left != nil  { queue = append(queue, node.Left) }  // enqueue
        if node.Right != nil { queue = append(queue, node.Right) } // enqueue
    }
    return result
}
```

---

## Common Pitfalls

| Pitfall | Stack | Queue |
|---------|-------|-------|
| **Underflow** | `Pop()` on empty stack → check `IsEmpty()` first | `Deq()` on empty queue → check `IsEmpty()` first |
| **Overflow** | Only in fixed-capacity implementations | Only in ring buffer with fixed capacity |
| **Naive slice queue** | — | `q = q[1:]` is O(n) due to GC overhead; use ring buffer |
| **GC pressure** | Linked list creates many small objects | Use `sync.Pool` to reuse nodes in hot paths |
| **Unbuffered channels** | — | Can deadlock if producer and consumer run sequentially |

---

## Comparison Summary

| | Array (slice) Stack | Linked Stack | Ring Buffer Queue | Linked Queue | Channel Queue |
|-|:---:|:---:|:---:|:---:|:---:|
| Push/Enq | O(1) amortized | O(1) | O(1) | O(1) | O(1) |
| Pop/Deq | O(1) | O(1) | O(1) | O(1) | O(1) |
| Memory grows dynamically | ✅ | ✅ | ❌ (fixed) | ✅ | ✅ (bounded buffer) |
| Thread-safe | ❌ (add Mutex) | ❌ | ❌ | ❌ | ✅ |
| Cache-friendly | ✅ | ❌ | ✅ | ❌ | ✅ |

---

## Memory Aid

> **Stack** = a stack of plates — you always pick from and add to the top. The last plate you placed is the first one you grab.
> **Queue** = a coffee shop line — you join at the back, get served at the front. No cutting!

---

## What to Learn Next

- [06-tree.md](06-tree.md) — Trees rely on both Stack (DFS) and Queue (BFS) for traversal
- [07-sorting.md](07-sorting.md) — Heap sort builds on the max-heap, which is a tree stored as an array
