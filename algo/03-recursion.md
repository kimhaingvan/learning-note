# Recursion

> **Category**: Algorithms | **Level**: Intermediate

---

## What

**Recursion** is a technique where a function **calls itself** to solve a smaller version of the same problem, combining results to solve the original.

```
Problem P → break into P' (same type, smaller) → solve P' → build solution of P from P'
```

If the solution to problem `P` can be expressed in terms of the solution to a **smaller, simpler problem P' of the same type**, then recursion is applicable.

---

## Why

Many problems have a naturally **self-similar structure** — trees, nested directories, mathematical sequences, divide-and-conquer algorithms. Recursion lets you express these solutions in a form that mirrors the problem's structure directly, making the code short, readable, and provably correct.

---

## When

Use recursion when:
- The problem can be **divided into smaller sub-problems of the same type**.
- There is a clear **base case** (the smallest version of the problem — solvable directly).
- The problem involves **hierarchical structures** (trees, graphs, nested data).
- You are implementing **backtracking, divide-and-conquer, or dynamic programming**.

Avoid recursion when:
- The depth is very large (risk of **stack overflow**).
- The same sub-problem is re-computed many times without memoization (consider DP instead).
- An iterative approach is equally clear and faster.

---

## Structure of a Recursive Function

Every correct recursive function has exactly two parts:

| Part | Also Called | Description | Purpose |
|------|-------------|-------------|---------|
| **Base case (anchor)** | Termination condition | The smallest version of the problem — solved directly, **no recursive call** | Makes the function **stop** — prevents infinite loops |
| **Recursive case** | Inductive step | Breaks the problem into smaller sub-problems and **calls itself** | **Shrinks** the problem toward the base case |

> ⚠️ A recursive function without a base case will call itself indefinitely, eventually causing a **stack overflow** (runtime crash).

### Generic Template

```go
func solve(problem Problem) Solution {
    if isBaseCase(problem) {        // base case — answer is trivial
        return directSolution(problem)
    }
    smaller := reduceProblem(problem) // make the problem smaller
    subSolution := solve(smaller)     // recursive call
    return buildSolution(subSolution) // combine to get the full answer
}
```

---

## Classic Example — Factorial

### Mathematical Definition

$$n! = n \times (n-1)!$$

with base case: $0! = 1$

### Call Chain for `Factorial(3)`

```
Factorial(3)
  → 3 × Factorial(2)
        → 2 × Factorial(1)
              → 1 × Factorial(0)
                    → 1  (base case)
              ← 1 × 1 = 1
        ← 2 × 1 = 2
  ← 3 × 2 = 6
```

### Implementation (Go)

```go
func Factorial(n int) int {
    if n == 0 {          // base case
        return 1
    }
    return n * Factorial(n-1) // recursive case
}

func main() {
    fmt.Println(Factorial(5))  // 120
    fmt.Println(Factorial(0))  // 1
}
```

---

## Example — Fibonacci Numbers

### Definition

$$F(n) = F(n-1) + F(n-2)$$

with $F(0) = 0$, $F(1) = 1$.

```go
// Naive recursive — O(2ⁿ), re-computes many sub-problems
func FibNaive(n int) int {
    if n <= 1 {
        return n
    }
    return FibNaive(n-1) + FibNaive(n-2)
}

// Memoized — O(n) time, O(n) space
func Fib(n int, memo map[int]int) int {
    if n <= 1 {
        return n
    }
    if v, ok := memo[n]; ok {
        return v
    }
    memo[n] = Fib(n-1, memo) + Fib(n-2, memo)
    return memo[n]
}
```

> Without memoization, `FibNaive(40)` computes over 1 billion calls. With memoization, it computes just 40.

---

## Example — Binary Search (Recursive)

Binary search is a classic divide-and-conquer algorithm with a clean recursive form:

```go
func binarySearch(a []int, target, lo, hi int) int {
    if lo > hi {
        return -1  // base case: not found
    }
    mid := lo + (hi-lo)/2
    switch {
    case a[mid] == target:
        return mid              // base case: found
    case a[mid] < target:
        return binarySearch(a, target, mid+1, hi) // recurse right
    default:
        return binarySearch(a, target, lo, mid-1) // recurse left
    }
}
```

Complexity: **T(n) = T(n/2) + 1 → O(log n)** (by the Master Theorem).

---

## Example — Tree Traversal

Recursive traversal is arguably the most natural expression of tree operations:

```go
type Node struct {
    Val   int
    Left  *Node
    Right *Node
}

// Postorder: count all nodes in the subtree
func CountNodes(root *Node) int {
    if root == nil {    // base case: empty subtree
        return 0
    }
    return 1 + CountNodes(root.Left) + CountNodes(root.Right)
}

// Preorder: compute height
func Height(root *Node) int {
    if root == nil {
        return 0
    }
    left := Height(root.Left)
    right := Height(root.Right)
    if left > right {
        return left + 1
    }
    return right + 1
}
```

---

## How the Call Stack Works

Each recursive call is a new **stack frame** pushed onto the call stack. The stack stores:
- Function parameters
- Local variables
- Return address

```
Stack during Factorial(3):
┌────────────────────┐  ← top
│ Factorial(0) = 1   │
├────────────────────┤
│ Factorial(1) = 1×1 │
├────────────────────┤
│ Factorial(2) = 2×1 │
├────────────────────┤
│ Factorial(3) = 3×2 │
└────────────────────┘  ← bottom (initial call)
```

Results are computed as the stack **unwinds** (top to bottom).

---

## Eliminating Recursion

Every recursive algorithm can be rewritten iteratively using an **explicit stack** to simulate what the call stack was doing automatically.

### Reasons to Eliminate Recursion

| Reason | Detail |
|--------|--------|
| Avoid stack overflow | Very deep recursion (depth > ~10,000) can exhaust the goroutine stack |
| Performance | Avoid function call overhead in tight loops |
| Explicit control | Sometimes iterative version is easier to debug step-by-step |

### Factorial — Iterative Equivalent

```go
func FactorialIterative(n int) int {
    result := 1
    for i := 2; i <= n; i++ {
        result *= i
    }
    return result
}
```

### Tree Traversal — Iterative Using Explicit Stack

```go
// Iterative Inorder (LNR) using an explicit stack
func InorderIterative(root *Node) []int {
    var result []int
    stack := []*Node{}

    for root != nil || len(stack) > 0 {
        for root != nil {
            stack = append(stack, root)
            root = root.Left
        }
        root = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, root.Val)
        root = root.Right
    }
    return result
}
```

---

## Advantages and Disadvantages

### Advantages

| Advantage | When It Matters |
|-----------|----------------|
| Short, readable code | Problems with naturally recursive structure (trees, grammars, backtracking) |
| Easy to express mathematical recurrences | Factorial, Fibonacci, Towers of Hanoi |
| Natural fit for backtracking / DFS | Maze solving, permutations, subsets |

### Disadvantages

| Disadvantage | Mitigation |
|--------------|------------|
| Stack memory per call (O(depth) space) | Convert to iterative or increase stack size |
| Repeated sub-problem computation | Add memoization → dynamic programming |
| Risk of stack overflow for deep recursion | Convert to iterative; use tail-call style where possible |

---

## Summary

| Concept | Key Point |
|---------|-----------|
| **Base case** | Where the function stops — must always be reachable |
| **Recursive case** | Reduces the problem toward the base case |
| **Call stack** | Frames accumulate as recursion deepens, unwind on return |
| **Memoization** | Cache sub-problem results to avoid exponential re-computation |
| **Iterative equivalent** | Any recursion can be replaced with an explicit stack |

> **"Recursion is the art of shrinking a problem — but a master programmer knows when to eliminate it."**

---

## Memory Aid

> A recursive function is like a Russian nesting doll. Each call opens a smaller doll inside. When you hit the smallest doll (base case), you start closing them back up, building the answer from the inside out.

---

## What to Learn Next

- [02-algorithm-complexity.md](02-algorithm-complexity.md) — Master Theorem for analyzing recursive complexity
- [04-list-data-structures.md](04-list-data-structures.md) — Recursive operations on linked lists
- [06-tree.md](06-tree.md) — Trees: the most natural application of recursion
