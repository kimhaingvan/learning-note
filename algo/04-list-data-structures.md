# List Data Structures

> **Category**: Algorithms | **Level**: Intermediate

---

## What

A **list** is an ordered collection of elements of the same type where:
- Each element can be **accessed, inserted, or deleted**.
- The list size can be **fixed** (static) or **variable** (dynamic).

```
L = (a₁, a₂, a₃, …, aₙ)
```

Unlike a **set**, a list:
- Has a defined **order** — position matters.
- **Allows duplicates** — the same value can appear multiple times.

---

## Why Four Different Representations?

No single structure is best for everything. The choice depends on which operations you perform most:

| Structure | Random Access | Insert/Delete at Head | Insert/Delete at Middle | Memory |
|-----------|:---:|:---:|:---:|:---:|
| **Array list** | ✅ O(1) | ❌ O(n) | ❌ O(n) | Contiguous, fixed |
| **Singly linked list** | ❌ O(n) | ✅ O(1) | ✅ O(1) w/ pointer | Per-node overhead |
| **Doubly linked list** | ❌ O(n) | ✅ O(1) | ✅ O(1) w/ pointer | 2× pointer overhead |
| **Circular list** | ❌ O(n) | ✅ O(1) | ✅ O(1) w/ pointer | Varies |

---

## 1. Array List (Static / Sequential List)

### Concept

Elements are stored **contiguously in memory**. Each element is accessed directly by its **index** in O(1).

```
Index:  0     1     2     3     4
      ┌─────┬─────┬─────┬─────┬─────┐
      │  3  │  7  │  1  │  9  │  5  │
      └─────┴─────┴─────┴─────┴─────┘
```

### Strengths and Weaknesses

| Strengths | Weaknesses |
|-----------|------------|
| Random access in O(1) | Insert/delete in the middle requires shifting: O(n) |
| Cache-friendly (contiguous memory) | Maximum capacity must be known upfront (static form) |
| Simple to iterate | Resizing (dynamic form) is expensive when array is full |

### Implementation (Go)

```go
package main

import "fmt"

type ArrayList struct {
    data []int
}

func NewArrayList() *ArrayList {
    return &ArrayList{data: make([]int, 0)}
}

func (l *ArrayList) Len() int         { return len(l.data) }
func (l *ArrayList) Get(i int) int    { return l.data[i] }        // O(1)
func (l *ArrayList) Append(x int)    { l.data = append(l.data, x) } // amortized O(1)

// Insert at position k — shifts all elements from k onward right: O(n)
func (l *ArrayList) Insert(k, x int) {
    if k < 0 || k > len(l.data) {
        return
    }
    l.data = append(l.data, 0)
    copy(l.data[k+1:], l.data[k:])
    l.data[k] = x
}

// Delete at position k — shifts elements left: O(n)
func (l *ArrayList) Delete(k int) {
    if k < 0 || k >= len(l.data) {
        return
    }
    copy(l.data[k:], l.data[k+1:])
    l.data = l.data[:len(l.data)-1]
}

func (l *ArrayList) Print() {
    fmt.Println(l.data)
}

func main() {
    list := NewArrayList()
    list.Append(10)
    list.Append(20)
    list.Append(30)
    list.Insert(1, 15)   // [10, 15, 20, 30]
    list.Delete(2)        // [10, 15, 30]
    list.Print()
}
```

---

## 2. Singly Linked List

### Concept

Each element is a **node** containing:
- `Val` — the data value
- `Next` — a pointer to the **next node**

The last node's `Next` is `nil`. The list is accessed through the **head pointer**.

```
head
 │
 ▼
┌─────┬──────┐    ┌─────┬──────┐    ┌─────┬──────┐
│  3  │ next │ →  │  7  │ next │ →  │  1  │  nil │
└─────┴──────┘    └─────┴──────┘    └─────┴──────┘
```

### Key Operations

| Operation | Complexity | How |
|-----------|-----------|-----|
| Add at head | O(1) | Create node, point `Next` at old head, update `Head` |
| Add at tail | O(n) | Traverse to last node; O(1) if tail pointer maintained |
| Delete head | O(1) | Advance `Head` to `Head.Next` |
| Delete by value | O(n) | Find node and update predecessor's `Next` |
| Search | O(n) | Walk the chain checking each `Val` |
| Reverse (iterative) | O(n) | Re-link pointers in a single pass |

### Implementation (Go)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type SinglyLinkedList struct {
    Head *Node
}

func (l *SinglyLinkedList) AddHead(x int) {
    l.Head = &Node{Val: x, Next: l.Head}
}

func (l *SinglyLinkedList) AddTail(x int) {
    n := &Node{Val: x}
    if l.Head == nil {
        l.Head = n
        return
    }
    cur := l.Head
    for cur.Next != nil {
        cur = cur.Next
    }
    cur.Next = n
}

func (l *SinglyLinkedList) DeleteHead() {
    if l.Head != nil {
        l.Head = l.Head.Next
    }
}

// DeleteValue removes the first node whose Val == x.
func (l *SinglyLinkedList) DeleteValue(x int) {
    var prev *Node
    cur := l.Head
    for cur != nil && cur.Val != x {
        prev, cur = cur, cur.Next
    }
    if cur == nil {
        return // not found
    }
    if prev == nil {
        l.Head = cur.Next // was the head
    } else {
        prev.Next = cur.Next
    }
}

// ReverseIter reverses the list in-place in O(n).
func (l *SinglyLinkedList) ReverseIter() {
    var prev *Node
    cur := l.Head
    for cur != nil {
        nxt := cur.Next
        cur.Next = prev
        prev, cur = cur, nxt
    }
    l.Head = prev
}

func (l *SinglyLinkedList) Print() {
    for p := l.Head; p != nil; p = p.Next {
        fmt.Printf("%d → ", p.Val)
    }
    fmt.Println("nil")
}

func main() {
    list := &SinglyLinkedList{}
    list.AddTail(1)
    list.AddTail(2)
    list.AddTail(3)
    list.Print()            // 1 → 2 → 3 → nil

    list.DeleteValue(2)
    list.Print()            // 1 → 3 → nil

    list.ReverseIter()
    list.Print()            // 3 → 1 → nil
}
```

### Why Reverse Works

To reverse `A → B → C → nil`:
1. Save `nxt = B`
2. Point `A.Next = nil` (the new tail)
3. Move: `prev = A`, `cur = B`
4. Repeat: `B.Next = A`, `prev = B`, `cur = C`
5. Result: `C → B → A → nil`

---

## 3. Doubly Linked List

### Concept

Each node has **two pointers**:
- `Prev` — points to the **previous** node
- `Next` — points to the **next** node

The list maintains `Head` (front) and `Tail` (back) pointers.

```
nil ← ┌──────┬─────┬──────┐ ↔ ┌──────┬─────┬──────┐ → nil
      │ prev │ val │ next │   │ prev │ val │ next │
      └──────┴─────┴──────┘   └──────┴─────┴──────┘
      Head                                      Tail
```

### Advantages over Singly Linked

| Capability | Singly | Doubly |
|------------|:------:|:------:|
| Traverse forward | ✅ | ✅ |
| Traverse backward | ❌ | ✅ |
| Delete a node given its pointer (O(1)) | ❌ (need predecessor) | ✅ |
| Insert before a given node (O(1)) | ❌ | ✅ |

### Implementation (Go)

```go
package main

import "fmt"

type DNode struct {
    Val       int
    Prev, Next *DNode
}

type DoublyLinkedList struct {
    Head, Tail *DNode
}

func (l *DoublyLinkedList) AddHead(x int) {
    n := &DNode{Val: x, Next: l.Head}
    if l.Head != nil {
        l.Head.Prev = n
    } else {
        l.Tail = n
    }
    l.Head = n
}

func (l *DoublyLinkedList) AddTail(x int) {
    n := &DNode{Val: x, Prev: l.Tail}
    if l.Tail != nil {
        l.Tail.Next = n
    } else {
        l.Head = n
    }
    l.Tail = n
}

// RemoveNode removes a node given its pointer directly — O(1).
func (l *DoublyLinkedList) RemoveNode(p *DNode) {
    if p == nil {
        return
    }
    if p.Prev != nil {
        p.Prev.Next = p.Next
    } else {
        l.Head = p.Next
    }
    if p.Next != nil {
        p.Next.Prev = p.Prev
    } else {
        l.Tail = p.Prev
    }
    p.Prev, p.Next = nil, nil
}

func (l *DoublyLinkedList) PrintForward() {
    for p := l.Head; p != nil; p = p.Next {
        fmt.Printf("%d → ", p.Val)
    }
    fmt.Println("nil")
}

func (l *DoublyLinkedList) PrintBackward() {
    for p := l.Tail; p != nil; p = p.Prev {
        fmt.Printf("%d → ", p.Val)
    }
    fmt.Println("nil")
}
```

### Trade-offs

| | Singly Linked | Doubly Linked |
|-|:---:|:---:|
| Memory per node | 1 pointer | 2 pointers |
| Traverse direction | Forward only | Both directions |
| Delete with node pointer | ❌ O(n) | ✅ O(1) |
| Complexity of pointer updates | Lower | Higher (must update both Prev and Next) |

---

## 4. Circular Linked List

### Concept

The **last node's `Next` points back to the first node** — forming a closed loop. There is no `nil` at the end.

```
            ┌──────────────────────────────────┐
            ↓                                  │
┌─────┬──────┐   ┌─────┬──────┐   ┌─────┬──────┐
│  1  │ next │ → │  2  │ next │ → │  3  │ next │
└─────┴──────┘   └─────┴──────┘   └─────┴──────┘
 Head
```

### When to Use

| Use Case | Why Circular Works Well |
|----------|------------------------|
| Round-robin scheduling | Naturally cycles through all items |
| Circular buffer / ring queue | Head and tail wrap around |
| Multiplayer board games | Players take turns in a cycle |
| Media playlist on repeat | Next song wraps to first |

### Implementation (Go)

```go
package main

import "fmt"

type CNode struct {
    Val  int
    Next *CNode
}

type CircularList struct {
    Head *CNode
}

// NewSingle creates a circular list with one node pointing to itself.
func NewSingle(x int) *CircularList {
    h := &CNode{Val: x}
    h.Next = h
    return &CircularList{Head: h}
}

// InsertAfterHead inserts a new node right after Head.
func (l *CircularList) InsertAfterHead(x int) {
    if l.Head == nil {
        n := &CNode{Val: x}
        n.Next = n
        l.Head = n
        return
    }
    n := &CNode{Val: x, Next: l.Head.Next}
    l.Head.Next = n
}

// PrintN prints n elements starting from Head (prevents infinite loop).
func (l *CircularList) PrintN(n int) {
    if l.Head == nil || n <= 0 {
        return
    }
    cur := l.Head
    for i := 0; i < n; i++ {
        fmt.Printf("%d → ", cur.Val)
        cur = cur.Next
    }
    fmt.Println("(wrap)")
}

func main() {
    cl := NewSingle(1)
    cl.InsertAfterHead(2)
    cl.InsertAfterHead(3)
    cl.PrintN(6) // wraps: 1 → 3 → 2 → 1 → 3 → 2 → (wrap)
}
```

> **Important**: Always use a counter or a sentinel check when traversing a circular list — otherwise the loop never terminates.

---

## Comparison Summary

| Structure | Random Access | Add Head | Add Tail | Delete (with pointer) | Bi-directional | Wrap-around |
|-----------|:---:|:---:|:---:|:---:|:---:|:---:|
| Array List | ✅ O(1) | ❌ O(n) | ✅ amortized O(1) | ❌ O(n) | — | — |
| Singly Linked | ❌ O(n) | ✅ O(1) | O(n) / O(1) w/ tail | ❌ O(n) | ❌ | ❌ |
| Doubly Linked | ❌ O(n) | ✅ O(1) | ✅ O(1) | ✅ O(1) | ✅ | ❌ |
| Circular | ❌ O(n) | ✅ O(1) | O(1) if tail known | ✅ O(1) | variant | ✅ |

---

## Real-World Applications

| Application | Structure Used |
|-------------|---------------|
| Database index (sorted) | Array-backed list |
| Browser history back/forward | Doubly linked list |
| Undo/redo in editors | Doubly linked list |
| OS process scheduling | Circular linked list |
| Implementing Stack and Queue | Singly or doubly linked list |
| LRU Cache | Doubly linked list + Hash Map |

---

## Memory Aid

> Array list = parking spaces in a row — numbered, easy to find, but inserting in the middle means shuffling all cars.
> Linked list = a treasure hunt — each clue (node) tells you where the next one is. Fast to add/remove, slow to find a specific clue without reading from the start.

---

## What to Learn Next

- [05-stack-queue.md](05-stack-queue.md) — Stack and Queue: lists with disciplined access patterns
- [06-tree.md](06-tree.md) — Trees: hierarchical linked structures
