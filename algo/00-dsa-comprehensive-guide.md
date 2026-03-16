# DSA Comprehensive Guide

> A structured reference for Data Structures & Algorithms — from fundamentals to interview-ready patterns.
> Written in Go. Topics are ordered from foundational to advanced.

---

## Table of Contents

1. [Problem-Solving Framework](#1-problem-solving-framework)
2. [Algorithm Complexity & Big O](#2-algorithm-complexity--big-o)
3. [Recursion](#3-recursion)
4. [List Data Structures](#4-list-data-structures)
5. [Stack](#5-stack)
6. [Queue](#6-queue)
7. [Trees & Binary Trees](#7-trees--binary-trees)
8. [Sorting Algorithms](#8-sorting-algorithms)
9. [Search Algorithms](#9-search-algorithms)

---

## 1. Problem-Solving Framework

### Definition / Idea

- **Definition**: A systematic 6-step process for solving any algorithmic problem correctly and efficiently.
- **Idea**: Treat every problem as a pipeline — `Input → Understanding → Model → Algorithm → Code → Verify`.
- **Purpose**: Avoid jumping to code before fully understanding the problem. Most bugs come from skipping the first two steps.

### When to Use

Always. This is the meta-framework that wraps every other topic.

### Main Approach

```
Step 1 — Define the Problem
  - What is the input? What are constraints?
  - What exactly is the expected output?
  - What are the edge cases?

Step 2 — Choose the Right Data Structure
  - Ordered sequence → Array
  - Dynamic insert/delete → Linked List
  - FIFO → Queue
  - LIFO / backtrack → Stack
  - Key-based lookup → Hash Map
  - Hierarchical → Tree
  - Relationship graph → Graph

Step 3 — Find the Algorithm
  - Must be correct, efficient, and general

Step 4 — Implement
  - Stepwise refinement; break into functions

Step 5 — Test
  - Functional: normal input
  - Boundary: empty, single element, max
  - Invalid: bad data, nil

Step 6 — Optimize
  - Identify bottleneck; apply better algorithm or data structure
```

### Data Structure Cheat-Sheet

| Scenario | Best Structure |
|----------|---------------|
| Ordered, random access | Array / Slice |
| Frequent insert/delete | Linked List |
| FIFO | Queue |
| Undo / DFS backtrack | Stack |
| Key → value lookup | Hash Map |
| Parent-child hierarchy | Tree |
| Range query on sorted data | Balanced BST |
| Undirected/directed relationships | Graph |

### Template (Go)

```go
func solve(input []int) int {
    // Step 1: edge case
    if len(input) == 0 {
        return 0
    }
    // Step 2: pick data structure (array here)
    // Step 3: algorithm (findMax below)
    maxVal := input[0]
    for _, v := range input[1:] {
        if v > maxVal {
            maxVal = v
        }
    }
    return maxVal
}
```

### Common Mistakes

- Jumping to code without understanding the problem fully.
- Choosing the wrong data structure (e.g., slice for frequent lookups → use map).
- Forgetting edge cases: empty input, single element, all-equal, negative values.
- Testing only the happy path.

### Quick Revision Points

- Every problem maps to: Input → Processing → Output.
- Data structure choice often reduces complexity by an order of magnitude — before writing any logic.
- Test in three categories: functional, boundary, invalid.

---

## 2. Algorithm Complexity & Big O

### Definition / Idea

- **Definition**: A measure of how time (or space) used by an algorithm scales as input size `n` grows.
- **Idea**: Ignore constants and hardware — focus only on the **growth rate** as n → ∞. Only the dominant term matters.
- **Purpose**: Compare algorithms fairly and predict performance at scale.

### When to Use

Whenever comparing two algorithm choices, or evaluating whether your solution will pass within time limits.

### Complexity Classes

| Class | Notation | Growth | Practical limit |
|-------|----------|--------|----------------|
| Constant | O(1) | Fixed | Always viable |
| Logarithmic | O(log n) | Very slow | Always viable |
| Linear | O(n) | Proportional | n ≤ 10⁸ |
| Linearithmic | O(n log n) | Moderate | n ≤ 10⁷ |
| Quadratic | O(n²) | Fast | n ≤ 10⁴ |
| Cubic | O(n³) | Very fast | n ≤ 500 |
| Exponential | O(2ⁿ) | Explosive | n ≤ 25 |
| Factorial | O(n!) | Catastrophic | n ≤ 12 |

### Complexity from Code Patterns

| Code Pattern | Complexity |
|-------------|------------|
| Single statement / no loop | O(1) |
| One loop (n iterations) | O(n) |
| Two nested loops (n each) | O(n²) |
| Loop that halves/doubles `i` | O(log n) |
| Outer n + inner log n | O(n log n) |
| Recursion T(n) = 2·T(n/2) + n | O(n log n) |

### Template (Go)

```go
// O(1) — constant
func first(a []int) int { return a[0] }

// O(n) — single scan
func sum(a []int) int {
    s := 0
    for _, v := range a { s += v }
    return s
}

// O(log n) — halving each step
func logExample(n int) int {
    count := 0
    for i := 1; i <= n; i *= 2 { count++ }
    return count
}

// O(n²) — nested scan
func hasDup(a []int) bool {
    for i := 0; i < len(a); i++ {
        for j := i + 1; j < len(a); j++ {
            if a[i] == a[j] { return true }
        }
    }
    return false
}
```

### Common Mistakes

- Forgetting that nested loops multiply, not add: O(n) × O(n) = O(n²).
- Using `(lo + hi) / 2` in binary search — overflows for large indices. Use `lo + (hi-lo)/2`.
- Treating O(n log n) as "almost O(n)" — it's significantly slower at large n.
- Forgetting space complexity: an O(n) space usage can be a knockout for memory-constrained environments.

### Related Patterns

- **Divide and Conquer** → O(n log n) via Master Theorem.
- **Greedy** → usually O(n log n) or O(n).
- **Dynamic Programming** → polynomial time (O(n²), O(n³)) instead of exponential.
- **Two Pointers / Sliding Window** → O(n) from O(n²).

### Quick Revision Points

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(n³) < O(2ⁿ) < O(n!)
```

- **Best case**: ideal input (e.g., array already sorted).
- **Average case**: random input.
- **Worst case**: most unfavourable input — what we almost always analyze.
- Time-space trade-off: more memory often means faster time (caching, memoization).

---

## 3. Recursion

### Definition / Idea

- **Definition**: A function that calls itself to solve a smaller version of the same problem.
- **Idea**: `P → break into P' (same type, smaller) → solve P' → build solution of P from P'`.
- **Purpose**: Naturally express self-similar problems — trees, divide-and-conquer, backtracking, DP.

### When to Use

- Problem can be divided into **smaller sub-problems of the same type**.
- Clear **base case** exists.
- Involves hierarchical/nested structures (trees, graphs, directories).
- Implementing backtracking, divide-and-conquer, or DP.

**Avoid when**: depth is very large (stack overflow risk), or same sub-problem re-computed (use memoisation or DP).

### Main Approach

Every recursive function has exactly two parts:

| Part | Role |
|------|------|
| **Base case** | Smallest version — solved directly, no recursive call. Terminates recursion. |
| **Recursive case** | Breaks into smaller sub-problem, calls itself, combines result. |

### Complexities

| Algorithm | Time | Space |
|-----------|------|-------|
| Factorial | O(n) | O(n) stack |
| Fibonacci (naïve) | O(2ⁿ) | O(n) stack |
| Fibonacci (memoised) | O(n) | O(n) |
| Binary Search (recursive) | O(log n) | O(log n) |
| Tree traversal | O(n) | O(h) where h = height |

### Template (Go)

```go
// Generic recursive skeleton
func solve(problem Problem) Solution {
    if isBaseCase(problem) {
        return directSolution(problem)
    }
    smaller := reduceProblem(problem)
    subSolution := solve(smaller)
    return buildSolution(subSolution)
}

// Factorial — O(n)
func Factorial(n int) int {
    if n == 0 { return 1 }       // base case
    return n * Factorial(n-1)    // recursive case
}

// Fibonacci — memoised O(n)
func Fib(n int, memo map[int]int) int {
    if n <= 1 { return n }
    if v, ok := memo[n]; ok { return v }
    memo[n] = Fib(n-1, memo) + Fib(n-2, memo)
    return memo[n]
}

// Binary Search — recursive O(log n)
func binarySearch(a []int, target, lo, hi int) int {
    if lo > hi { return -1 }
    mid := lo + (hi-lo)/2
    switch {
    case a[mid] == target: return mid
    case a[mid] < target:  return binarySearch(a, target, mid+1, hi)
    default:               return binarySearch(a, target, lo, mid-1)
    }
}

// Convert recursion → iterative using explicit stack
func FactorialIter(n int) int {
    result := 1
    for i := 2; i <= n; i++ { result *= i }
    return result
}
```

### Example Problems (LeetCode)

- [509 — Fibonacci Number](https://leetcode.com/problems/fibonacci-number/)
- [206 — Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)
- [104 — Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/)
- [700 — Search in a Binary Search Tree](https://leetcode.com/problems/search-in-a-binary-search-tree/)

### Common Mistakes

- Missing or incorrect base case → infinite recursion → stack overflow.
- Naïve Fibonacci: `FibNaive(40)` makes > 1 billion calls — always memoize.
- Deep recursion on untrusted input (trees of n nodes can be height n) → use iterative with explicit stack in production.
- Forgetting to combine/build the sub-solution correctly.

### Related Patterns

- **Divide and Conquer**: MergeSort, QuickSort, Binary Search.
- **Backtracking**: N-Queens, permutations, subsets.
- **Dynamic Programming**: recursion + memoization.
- **Tree Traversal**: pre/in/postorder.

### Quick Revision Points

- Base case = anchor. Recursive case = inductive step.
- Every recursive algorithm has an iterative equivalent using an explicit stack.
- Memoization converts O(2ⁿ) Fibonacci → O(n).
- Recursion depth = stack space = O(depth). Deep trees → iterative is safer.

---

## 4. List Data Structures

### Definition / Idea

- **Definition**: An ordered collection of elements where each element can be accessed, inserted, or deleted.
- **Idea**: Four representations — array (sequential), singly linked, doubly linked, circular — each with different trade-offs.
- **Purpose**: Foundation for almost every data structure that follows.

### When to Use

| Structure | Use When |
|-----------|----------|
| Array List | Fast random access O(1); few insertions/deletions |
| Singly Linked | Frequent head insert/delete; unknown final size |
| Doubly Linked | Need backward traversal; O(1) delete by node pointer |
| Circular | Round-robin scheduling; buffer with wrap-around |

### Complexities

| Operation | Array | Singly Linked | Doubly Linked |
|-----------|:-----:|:-------------:|:-------------:|
| Access by index | O(1) | O(n) | O(n) |
| Insert at head | O(n) | O(1) | O(1) |
| Insert at tail | O(1) amortized | O(n) or O(1) w/ tail ptr | O(1) w/ tail ptr |
| Insert at middle | O(n) | O(1) w/ pointer | O(1) w/ pointer |
| Delete by index | O(n) | O(n) | O(1) w/ pointer |
| Search | O(n) | O(n) | O(n) |
| Reverse | O(n) | O(n) | O(n) |

### Template (Go)

```go
// ---- Array List ----
type ArrayList struct{ data []int }
func (l *ArrayList) Get(i int) int        { return l.data[i] }     // O(1)
func (l *ArrayList) Append(x int)          { l.data = append(l.data, x) }
func (l *ArrayList) Insert(k, x int) {
    l.data = append(l.data, 0)
    copy(l.data[k+1:], l.data[k:])
    l.data[k] = x
}

// ---- Singly Linked List ----
type Node struct { Val int; Next *Node }
type SLL struct { Head *Node }

func (l *SLL) AddHead(x int) {
    l.Head = &Node{Val: x, Next: l.Head}
}
func (l *SLL) ReverseIter() {
    var prev *Node
    cur := l.Head
    for cur != nil {
        nxt := cur.Next
        cur.Next = prev
        prev, cur = cur, nxt
    }
    l.Head = prev
}

// ---- Doubly Linked List ----
type DNode struct { Val int; Prev, Next *DNode }
type DLL struct { Head, Tail *DNode }

func (l *DLL) AddTail(x int) {
    n := &DNode{Val: x, Prev: l.Tail}
    if l.Tail != nil { l.Tail.Next = n } else { l.Head = n }
    l.Tail = n
}
func (l *DLL) Delete(n *DNode) {
    if n.Prev != nil { n.Prev.Next = n.Next } else { l.Head = n.Next }
    if n.Next != nil { n.Next.Prev = n.Prev } else { l.Tail = n.Prev }
}
```

### Example Problems (LeetCode)

- [206 — Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)
- [21 — Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/)
- [141 — Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)
- [19 — Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/)
- [146 — LRU Cache](https://leetcode.com/problems/lru-cache/) ← doubly linked + hash map

### Common Mistakes

- Accessing `.Next` without checking for `nil` → nil pointer panic.
- Forgetting to update the `Tail` pointer in doubly linked list operations.
- Using a slice as a queue with `q = q[1:]` — this is O(n). Use a ring buffer.
- Losing the `Head` pointer before the reverse is complete.

### Related Patterns

- **Two Pointers (fast/slow)**: cycle detection, find middle.
- **Sentinel / Dummy Nodes**: simplify edge cases for insert at head/tail.
- **Hash Map + DLL**: LRU Cache (O(1) get/put).

### Quick Revision Points

- Array = cache-friendly, O(1) random access, O(n) insert/delete.
- Linked list = O(1) insert/delete at known position, no random access.
- Singly linked reverse: `prev, cur, nxt` three-pointer technique.
- Doubly linked delete is O(1) if you already hold the node pointer.

---

## 5. Stack

### Definition / Idea

- **Definition**: A restricted list where insertion and removal occur only at one end (the **top**).
- **Idea**: **LIFO** — Last In, First Out. Like a stack of plates.
- **Purpose**: Manages ordered state that must be "unwound" (function calls, undo history, DFS, expression evaluation).

### When to Use

- **Undo / redo** — each action is a pushed state; undo pops.
- **DFS** — explicit stack replaces the call stack.
- **Balanced brackets** — push opens, pop on close.
- **Expression evaluation** — operator precedence.
- **Backtracking** — store choices, pop to backtrack.
- **Monotonic stack** — next greater/smaller element problems.

### Complexities

| Operation | Complexity |
|-----------|-----------|
| Push | O(1) |
| Pop | O(1) |
| Peek | O(1) |
| IsEmpty | O(1) |
| Space | O(n) |

### Template (Go)

```go
// Generic slice-based Stack (recommended)
type Stack[T any] struct{ a []T }

func (s *Stack[T]) Push(v T)       { s.a = append(s.a, v) }
func (s *Stack[T]) IsEmpty() bool  { return len(s.a) == 0 }
func (s *Stack[T]) Len() int       { return len(s.a) }
func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.a) == 0 { return zero, false }
    return s.a[len(s.a)-1], true
}
func (s *Stack[T]) Pop() (T, bool) {
    v, ok := s.Peek()
    if ok { s.a = s.a[:len(s.a)-1] }
    return v, ok
}

// Application: Balanced Brackets
func IsBalanced(str string) bool {
    var st Stack[rune]
    match := map[rune]rune{')': '(', ']': '[', '}': '{'}
    for _, ch := range str {
        switch ch {
        case '(', '[', '{': st.Push(ch)
        case ')', ']', '}':
            top, ok := st.Pop()
            if !ok || top != match[ch] { return false }
        }
    }
    return st.IsEmpty()
}

// Application: Iterative DFS on tree
func DFS(root *Node) {
    if root == nil { return }
    st := []*Node{root}
    for len(st) > 0 {
        n := st[len(st)-1]; st = st[:len(st)-1]
        fmt.Println(n.Val)
        if n.Right != nil { st = append(st, n.Right) }
        if n.Left != nil  { st = append(st, n.Left)  }
    }
}
```

### Example Problems (LeetCode)

- [20 — Valid Parentheses](https://leetcode.com/problems/valid-parentheses/)
- [155 — Min Stack](https://leetcode.com/problems/min-stack/)
- [739 — Daily Temperatures](https://leetcode.com/problems/daily-temperatures/) ← monotonic stack
- [84 — Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) ← monotonic stack
- [150 — Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation/)

### Common Mistakes

- Calling `Pop()` or `Peek()` on an empty stack without checking → panic. Always check `IsEmpty()` first.
- Using a stack when a queue is needed (mixing LIFO and FIFO logic).
- Forgetting to push **Right before Left** in iterative preorder DFS (so Left is processed first when popped).

### Related Patterns

- **Monotonic Stack**: maintains elements in increasing or decreasing order to find next-greater/smaller in O(n).
- **Two Stacks = Queue**: use two stacks to simulate a FIFO queue.
- **Stack + Recursion equivalence**: any recursion can be iterativeized with an explicit stack.

### Quick Revision Points

- LIFO — last pushed is first popped.
- Use Go slice: `append` = push, `s[:n-1]` = pop, `s[n-1]` = peek.
- Balanced brackets: push open, pop and match on close, empty at end = valid.
- DFS = Stack; BFS = Queue.

---

## 6. Queue

### Definition / Idea

- **Definition**: A restricted list where insertion happens at the rear and removal at the front.
- **Idea**: **FIFO** — First In, First Out. Like a ticket line.
- **Purpose**: Process work in arrival order — scheduling, BFS, rate limiting, message passing.

### When to Use

- **BFS** — explore nodes level by level.
- **Task scheduling** — CPU process queue, print spooler.
- **Sliding window** — deque variant for max/min in window.
- **Producer-consumer** — Go channels.

### Complexities

| Operation | All implementations |
|-----------|-------------------|
| Enqueue | O(1) |
| Dequeue | O(1) |
| Peek/Front | O(1) |
| IsEmpty | O(1) |
| Space | O(n) |

### Template (Go)

```go
// Ring Buffer Queue — fixed capacity, O(1) all ops
type RingQueue[T any] struct {
    buf        []T
    head, tail int
    n          int
}

func NewRingQueue[T any](cap int) *RingQueue[T] {
    return &RingQueue[T]{buf: make([]T, cap)}
}
func (q *RingQueue[T]) Enq(v T) bool {
    if q.n == len(q.buf) { return false }
    q.buf[q.tail] = v
    q.tail = (q.tail + 1) % len(q.buf)
    q.n++
    return true
}
func (q *RingQueue[T]) Deq() (T, bool) {
    var zero T
    if q.n == 0 { return zero, false }
    v := q.buf[q.head]
    q.head = (q.head + 1) % len(q.buf)
    q.n--
    return v, true
}

// Idiomatic Go queue: use a slice (rebuild if needed) or channel
// Simple slice queue (fine for coding interviews):
queue := []int{}
queue = append(queue, 1)   // enqueue
front := queue[0]           // peek
queue = queue[1:]           // dequeue — O(n) in worst case for slice

// BFS example
func BFS(root *TreeNode) []int {
    if root == nil { return nil }
    var result []int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        node := queue[0]; queue = queue[1:]
        result = append(result, node.Val)
        if node.Left != nil  { queue = append(queue, node.Left) }
        if node.Right != nil { queue = append(queue, node.Right) }
    }
    return result
}

// Go Channel Queue (concurrent, bounded)
jobs := make(chan int, 64)
jobs <- 1           // enqueue
v := <-jobs         // dequeue
```

### Example Problems (LeetCode)

- [102 — Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) ← BFS
- [200 — Number of Islands](https://leetcode.com/problems/number-of-islands/) ← BFS
- [239 — Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) ← monotonic deque
- [232 — Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/)
- [225 — Implement Stack using Queues](https://leetcode.com/problems/implement-stack-using-queues/)

### Common Mistakes

- `q = q[1:]` on a plain slice is **not O(1)** — it reallocates/shifts under the hood in large programs. Use a ring buffer for performance-critical code.
- Mixing up which end is front vs rear.
- Forgetting to check for empty queue before dequeuing.
- Unbuffered Go channels → deadlock when producer/consumer run sequentially.

### Related Patterns

- **BFS**: queue-driven level-by-level exploration.
- **Monotonic Deque**: double-ended queue maintaining order for sliding window max/min.
- **Priority Queue (Heap)**: process highest-priority element first instead of FIFO.

### Quick Revision Points

- FIFO — first enqueued is first dequeued.
- BFS = Queue; DFS = Stack.
- Ring buffer avoids O(n) shifts with modular arithmetic on head/tail indices.
- Go channels are thread-safe queues.

---

## 7. Trees & Binary Trees

### Definition / Idea

- **Definition**: A hierarchical data structure of nodes connected by edges, with a single root and no cycles.
- **Idea**: Recursive — a tree is either empty, or a root node with zero or more subtrees.
- **Purpose**: Model hierarchical data (DOM, file systems, org charts); enable O(log n) search (BST); underpin heaps, segment trees.

### When to Use

- **Hierarchy** — file system, DOM, org chart.
- **Sorted data + O(log n) operations** — BST / AVL.
- **Priority queue** — Heap (complete binary tree stored as array).
- **Range queries** — Segment tree, Fenwick tree.
- **Prefix search** — Trie.

### Key Terminology

| Term | Meaning |
|------|---------|
| Root | Top node, no parent |
| Leaf | Node with no children |
| Height | Max level count from root to deepest leaf |
| Depth | Level of a node from the root |
| Complete tree | All levels full except possibly last (filled L→R) |
| Perfect tree | All leaves at same level; 2ʰ−1 nodes for height h |
| Degree | Number of children a node has |

### Binary Tree Properties

- Max nodes at height h: $2^h - 1$
- Min nodes at height h: $h$ (skewed)
- Height of complete tree with n nodes: $\lfloor \log_2 n \rfloor + 1$

### Complexities

| Operation | BST (balanced) | BST (worst/skewed) |
|-----------|:--------------:|:--------------:|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |
| Traversal | O(n) | O(n) |
| Space | O(n) | O(n) |

### Template (Go)

```go
type Node[T any] struct { Val T; Left, Right *Node[T] }

// ── Traversals ─────────────────────────────────────────────────────────
// Preorder (NLR) — serialize tree; clone; prefix expression
func Preorder[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    visit(r.Val)
    Preorder(r.Left, visit)
    Preorder(r.Right, visit)
}

// Inorder (LNR) — BST gives sorted output
func Inorder[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    Inorder(r.Left, visit)
    visit(r.Val)
    Inorder(r.Right, visit)
}

// Postorder (LRN) — delete tree; compute size/height; postfix
func Postorder[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    Postorder(r.Left, visit)
    Postorder(r.Right, visit)
    visit(r.Val)
}

// Level-order (BFS) — print level by level
func LevelOrder[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    q := []*Node[T]{r}
    for len(q) > 0 {
        n := q[0]; q = q[1:]
        visit(n.Val)
        if n.Left != nil  { q = append(q, n.Left) }
        if n.Right != nil { q = append(q, n.Right) }
    }
}

// Iterative Inorder (avoids stack overflow on deep trees)
func InorderIter[T any](r *Node[T], visit func(T)) {
    var stack []*Node[T]
    for r != nil || len(stack) > 0 {
        for r != nil { stack = append(stack, r); r = r.Left }
        r = stack[len(stack)-1]; stack = stack[:len(stack)-1]
        visit(r.Val)
        r = r.Right
    }
}

// Height / Max Depth
func Height[T any](r *Node[T]) int {
    if r == nil { return 0 }
    l, ri := Height(r.Left), Height(r.Right)
    if l > ri { return l + 1 }
    return ri + 1
}

// Array representation (for heaps — 1-based index)
// parent(i) = i/2, leftChild(i) = 2*i, rightChild(i) = 2*i+1
```

### Example Problems (LeetCode)

- [104 — Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/)
- [102 — Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/)
- [98 — Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/)
- [226 — Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/)
- [543 — Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/)
- [235 — Lowest Common Ancestor of BST](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/)
- [105 — Construct Tree from Preorder + Inorder](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)

### Common Mistakes

- Missing `if r == nil { return }` base case in traversal → nil pointer panic.
- Using 0-based array indexing for heaps → off-by-one; use 1-based.
- Deep recursive traversal on skewed tree → stack overflow. Use iterative for production.
- BST validation: comparing only parent and child is wrong. Pass min/max bounds down the tree.

### Related Patterns

- **DFS + Stack**: iterative tree traversal.
- **BFS + Queue**: level-order traversal.
- **Divide and Conquer**: most tree problems decompose as root result = combine(left result, right result).
- **Morris Traversal**: O(1) space inorder traversal.
- **Heap (Max/Min)**: complete binary tree stored as array; sift-up/sift-down operations.

### Traversal Quick Reference

```
Tree:      A
          / \
         B   C
        / \   \
       D   E   F

Preorder  (NLR): A B D E C F   ← clone, serialize
Inorder   (LNR): D B E A C F   ← sorted output from BST
Postorder (LRN): D E B F C A   ← delete tree, compute size
Level-order:     A B C D E F   ← BFS
```

### Quick Revision Points

- Inorder on BST = sorted sequence.
- Preorder = serialize / clone a tree.
- Postorder = safe deletion (process children before parent).
- Level-order (BFS) = shortest path in unweighted tree.
- Array tree: `parent(i) = i/2`, `left(i) = 2i`, `right(i) = 2i+1` (1-based).

---

## 8. Sorting Algorithms

### Definition / Idea

- **Definition**: Rearranging a collection into a defined order (ascending, descending, lexicographic).
- **Idea**: Comparison-based sorting has a theoretical lower bound of O(n log n). Different algorithms make different trade-offs among stability, memory, and worst-case guarantees.
- **Purpose**: Pre-processing step that unlocks O(log n) search, simplifies range queries, and enables merge-based operations.

### When to Use Which

| Situation | Algorithm |
|-----------|-----------|
| General purpose, in-memory | Quick Sort (random pivot) |
| Stability required | Merge Sort |
| Guaranteed O(n log n), in-place | Heap Sort |
| Nearly sorted data | Insertion Sort |
| Very small array (< 15 elements) | Insertion Sort |
| External sort (disk/stream) | Merge Sort |
| Production Go code | `slices.Sort` (pattern-defeating quicksort) |

### Algorithm Comparison

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|:----:|:-------:|:-----:|:-----:|:------:|
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | ❌ |
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ |

### Templates (Go)

```go
// ── Insertion Sort — O(n²), stable, great for small/nearly-sorted ──────
func InsertionSort(a []int) {
    for i := 1; i < len(a); i++ {
        key := a[i]
        j := i - 1
        for j >= 0 && a[j] > key { a[j+1] = a[j]; j-- }
        a[j+1] = key
    }
}

// ── Merge Sort — O(n log n), stable, O(n) space ────────────────────────
func MergeSort(a []int) []int {
    if len(a) <= 1 { return a }
    mid := len(a) / 2
    return merge(MergeSort(a[:mid]), MergeSort(a[mid:]))
}
func merge(l, r []int) []int {
    res := make([]int, 0, len(l)+len(r))
    i, j := 0, 0
    for i < len(l) && j < len(r) {
        if l[i] <= r[j] { res = append(res, l[i]); i++ } else { res = append(res, r[j]); j++ }
    }
    res = append(res, l[i:]...); res = append(res, r[j:]...)
    return res
}

// ── Quick Sort — O(n log n) avg, O(n²) worst, in-place ────────────────
func QuickSort(a []int, lo, hi int) {
    if lo < hi { p := partition(a, lo, hi); QuickSort(a, lo, p-1); QuickSort(a, p+1, hi) }
}
func partition(a []int, lo, hi int) int {
    pivot := a[hi]; i := lo - 1
    for j := lo; j < hi; j++ {
        if a[j] <= pivot { i++; a[i], a[j] = a[j], a[i] }
    }
    a[i+1], a[hi] = a[hi], a[i+1]
    return i + 1
}

// ── Heap Sort — O(n log n), in-place, not stable ──────────────────────
func HeapSort(a []int) {
    n := len(a)
    for i := n/2 - 1; i >= 0; i-- { siftDown(a, n, i) }   // build max-heap
    for end := n - 1; end > 0; end-- {
        a[0], a[end] = a[end], a[0]                          // move max to end
        siftDown(a, end, 0)                                   // restore heap
    }
}
func siftDown(a []int, n, i int) {
    for {
        left := 2*i+1; if left >= n { break }
        right, largest := left+1, left
        if right < n && a[right] > a[left] { largest = right }
        if a[i] >= a[largest] { break }
        a[i], a[largest] = a[largest], a[i]; i = largest
    }
}
```

### Example Problems (LeetCode)

- [912 — Sort an Array](https://leetcode.com/problems/sort-an-array/)
- [75 — Sort Colors](https://leetcode.com/problems/sort-colors/) ← Dutch National Flag / 3-way partition
- [148 — Sort List](https://leetcode.com/problems/sort-list/) ← Merge Sort on linked list
- [215 — Kth Largest Element in Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) ← QuickSelect
- [56 — Merge Intervals](https://leetcode.com/problems/merge-intervals/)

### Common Mistakes

- **Quick Sort + sorted input + last-element pivot** → O(n²). Fix: use random pivot or median-of-3.
- **Merge Sort**: allocating a new slice on every recursive call is expensive. Pass a reusable buffer.
- **Not recognising stability** requirements — Heap Sort and Quick Sort are NOT stable.
- Using O(n²) sorts (bubble/selection/insertion) on large datasets (> 10,000 elements).
- Forgetting that `slices.Sort` / `sort.Slice` in Go use pdqsort (pattern-defeating quicksort) — effectively O(n log n) in practice.

### Related Patterns

- **Divide and Conquer**: Merge Sort, Quick Sort.
- **Heap / Priority Queue**: Heap Sort, Kth largest via partial heap.
- **Counting Sort / Radix Sort**: O(n) for bounded integer keys.
- **QuickSelect**: O(n) average to find the Kth element without full sort.

### Quick Revision Points

- **Stable sorts** (preserve equal element order): Insertion, Bubble, Merge.
- **Unstable sorts**: Selection, Heap, Quick.
- Slowest guaranteed O(n log n) worst case with O(1) space = **Heap Sort**.
- Fastest in practice = **Quick Sort** (with random pivot).
- Best for nearly-sorted = **Insertion Sort** (O(n) best case).
- Best for linked lists = **Merge Sort** (no random access needed).

---

## 9. Search Algorithms

### Definition / Idea

- **Definition**: Finding an element (or its position) in a collection.
- **Idea**: The right algorithm depends entirely on whether the data is sorted, the size, and how dynamic the structure is.
- **Purpose**: Core operation for almost every program — databases, compilers, UI, networking.

### When to Use

| Data State | Query Type | Best Algorithm |
|------------|-----------|----------------|
| Unsorted, small | Exact | Linear Search |
| Unsorted, any size | Exact | Hash Map O(1) |
| Sorted, static | Exact / Range | Binary Search |
| Sorted, unknown length | Exact | Exponential → Binary |
| Sorted, uniform numeric | Exact | Interpolation Search |
| Dynamic, ordered | Exact / Range / Rank | Balanced BST |

### Complexities

| Algorithm | Time | Space | Prerequisite |
|-----------|------|-------|--------------|
| Linear Search | O(n) | O(1) | None |
| Binary Search | O(log n) | O(1) | Sorted |
| Exponential Search | O(log i)* | O(1) | Sorted |
| Interpolation Search | O(log log n) avg / O(n) worst | O(1) | Sorted + uniform |
| Hash Map Lookup | O(1) avg / O(n) worst | O(n) | None |
| BST Search | O(log n) balanced / O(n) skewed | O(n) | None |

*where i is the result index

### Templates (Go)

```go
// ── Linear Search — O(n) ───────────────────────────────────────────────
func LinearSearch[T comparable](a []T, x T) int {
    for i, v := range a { if v == x { return i } }
    return -1
}

// ── Binary Search — O(log n), sorted array ────────────────────────────
// LowerBound: first index where a[i] >= x
func LowerBound[T constraints.Ordered](a []T, x T) int {
    lo, hi := 0, len(a)
    for lo < hi {
        mid := lo + (hi-lo)/2
        if a[mid] < x { lo = mid + 1 } else { hi = mid }
    }
    return lo
}

// UpperBound: first index where a[i] > x
func UpperBound[T constraints.Ordered](a []T, x T) int {
    lo, hi := 0, len(a)
    for lo < hi {
        mid := lo + (hi-lo)/2
        if a[mid] <= x { lo = mid + 1 } else { hi = mid }
    }
    return lo
}

// Exact BinarySearch using LowerBound
func BinarySearch[T constraints.Ordered](a []T, x T) int {
    pos := LowerBound(a, x)
    if pos < len(a) && a[pos] == x { return pos }
    return -1
}

// Count elements equal to x:
//   count := UpperBound(a, x) - LowerBound(a, x)
// Range [lo_val, hi_val]:
//   slice := a[LowerBound(a, lo_val) : UpperBound(a, hi_val)]

// ── Exponential Search — O(log i), unbounded/infinite array ──────────
func ExponentialSearch[T constraints.Ordered](a []T, x T) int {
    n := len(a)
    if n == 0 { return -1 }
    if a[0] == x { return 0 }
    b := 1
    for b < n && a[b] < x { b *= 2 }
    lo, hi := b/2, b+1
    if hi > n { hi = n }
    pos := lo + LowerBound(a[lo:hi], x)
    if pos < n && a[pos] == x { return pos }
    return -1
}

// ── Hash-Based Search — O(1) average ──────────────────────────────────
// Go built-in map = hash table
freq := make(map[int]int)
for _, v := range data { freq[v]++ }
if count, ok := freq[42]; ok { /* found */ }

// ── BST Search — O(log n) balanced ────────────────────────────────────
func bstSearch(root *Node[int], target int) *Node[int] {
    if root == nil || root.Val == target { return root }
    if target < root.Val { return bstSearch(root.Left, target) }
    return bstSearch(root.Right, target)
}
```

### Example Problems (LeetCode)

- [704 — Binary Search](https://leetcode.com/problems/binary-search/)
- [35 — Search Insert Position](https://leetcode.com/problems/search-insert-position/) ← LowerBound
- [34 — Find First and Last Position of Element](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/) ← LowerBound + UpperBound
- [153 — Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/)
- [162 — Find Peak Element](https://leetcode.com/problems/find-peak-element/)
- [1095 — Find in Mountain Array](https://leetcode.com/problems/find-in-mountain-array/)

### Common Mistakes

- **Binary search on unsorted data** — silently returns wrong answers.
- **Mid overflow**: `(lo + hi) / 2` overflows for large values. Always use `lo + (hi-lo)/2`.
- **Mixing `[lo, hi]` (closed) and `[lo, hi)` (half-open) intervals** in the same code — causes off-by-one bugs. Prefer half-open consistently.
- **Interpolation search on non-uniform data** — degrades to O(n); unsafe without additional bounds checking.
- **Forgetting that hash map lookup is O(1) average but O(n) worst case** due to hash collisions.

### Related Patterns

- **Binary Search on Answer**: search over a space of possible answers instead of the array (e.g., "minimum speed to reach destination").
- **Two Pointers**: effectively O(n) "search" for a pair with a given sum.
- **Sliding Window**: O(n) search for a subarray matching a condition.

### Binary Search Template Variants

```
Half-open [lo, hi):
  lo = 0, hi = len(a)
  while lo < hi:
      mid = lo + (hi-lo)/2
      if condition(mid): hi = mid
      else: lo = mid + 1
  return lo   ← first index satisfying condition

Classic exact [lo, hi]:
  while lo <= hi:
      mid = lo + (hi-lo)/2
      if a[mid] == x: return mid
      if a[mid] < x: lo = mid + 1
      else: hi = mid - 1
  return -1
```

### Quick Revision Points

- Linear = O(n), always works.
- Binary = O(log n), requires **sorted** data.
- Hash = O(1) average, no ordering.
- BST = O(log n) balanced, supports range + sorted queries.
- LowerBound finds insertion point / first occurrence.
- UpperBound finds end of range / count of element.
- "Binary search on answer" pattern: search over the answer domain, not the data.

---

## Master Quick-Reference

### Data Structure Decision Tree

```
Need fast lookup by key?      → Hash Map O(1)
Need sorted order?            → Sorted Array + BinarySearch  OR  BST
Need insert/delete anywhere?  → Linked List
Need LIFO?                    → Stack
Need FIFO?                    → Queue
Need hierarchical?            → Tree
Need priority ordering?       → Heap (Priority Queue)
Need weighted/graph problem?  → Graph + Dijkstra/BFS/DFS
```

### Algorithm Complexity Summary

```
Sorting:
  O(n²):       Selection, Bubble, Insertion (simple, small n)
  O(n log n):  Merge, Heap, Quick (efficient)
  O(n):        Counting, Radix (special cases)

Searching:
  O(n):        Linear
  O(log n):    Binary, BST (balanced)
  O(1) avg:    Hash table

Recursion:
  O(2ⁿ) → O(n) with memoization (Fibonacci)
  O(n log n)   Merge Sort, Quick Sort (average)
  O(log n)     Binary Search, balanced BST ops
```

### Problem Pattern → Algorithm Mapping

| Problem Signal | Look For |
|----------------|----------|
| "Find if exists" in unsorted | Hash map |
| "Find if exists" in sorted array | Binary search |
| "Kth largest/smallest" | Heap or QuickSelect |
| "Subarray sum / window" | Sliding window, prefix sum |
| "All combinations / permutations" | Backtracking (recursion) |
| "Shortest path (unweighted)" | BFS |
| "Shortest path (weighted)" | Dijkstra |
| "Detect cycle in linked list" | Fast/slow pointers |
| "Next greater element" | Monotonic stack |
| "Intervals / scheduling" | Sort by start, greedy |
| "Count occurrences" | Hash map / sorting |
| "Balanced brackets" | Stack |
| "Tree path sum" | DFS recursion |
| "Level-by-level tree" | BFS (queue) |
