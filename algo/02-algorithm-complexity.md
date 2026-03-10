# Algorithm Complexity & Big O Notation

> **Category**: Algorithms | **Level**: Beginner → Intermediate

---

## What

**Algorithm complexity** measures how the **time (or memory) used by an algorithm scales** as the input size `n` grows — independent of hardware, programming language, or system load.

- **Time complexity**: how the number of operations grows with `n`.
- **Space complexity**: how the memory usage grows with `n`.

We express complexity using **Big O notation**: `O(...)`.

---

## Why It Matters

The same problem can have many algorithms. To compare them fairly, we need a **hardware-independent yardstick**. Consider summing the integers 1 through n:

| Approach | Code | Complexity |
|----------|------|------------|
| Loop | `for i := 1; i <= n; i++ { sum += i }` | O(n) |
| Gauss formula | `sum = n * (n+1) / 2` | O(1) |

Both give the same result. But for n = 1,000,000, the formula is 1,000,000× faster (in terms of operations).

> **Key insight:** as `n` gets large, constant factors and lower-order terms become negligible. Only the **dominant growth term** matters.

---

## Big O — Complexity Classes

| Class | Expression | Notation | Growth Behaviour |
|-------|-----------|----------|-----------------|
| Constant | T(n) = 1 | **O(1)** | Same time regardless of input size |
| Logarithmic | T(n) = log n | **O(log n)** | Grows very slowly (doubles input → +1 step) |
| Linear | T(n) = n | **O(n)** | Grows proportionally to input |
| Linearithmic | T(n) = n log n | **O(n log n)** | Common in efficient sort algorithms |
| Quadratic | T(n) = n² | **O(n²)** | Grows fast; problematic for large n |
| Cubic | T(n) = n³ | **O(n³)** | Only viable for small n |
| Exponential | T(n) = 2ⁿ | **O(2ⁿ)** | Impractical for n > 40 or so |
| Factorial | T(n) = n! | **O(n!)** | Catastrophically slow |

### Practical Scale (n = 1,000)

| Complexity | Approx. Operations |
|------------|-------------------|
| O(1) | 1 |
| O(log n) | ~10 |
| O(n) | 1,000 |
| O(n log n) | ~10,000 |
| O(n²) | 1,000,000 |
| O(2ⁿ) | 10³⁰⁰ (impossible) |

---

## Section 3 — Determining Complexity of Code

### The "Active Operation" Principle

Identify the **most-frequently-executed operation** (assignment, comparison, multiplication, etc.) and count how many times it runs as a function of `n`.

**Key rules:**

| Code pattern | Complexity |
|-------------|------------|
| Single statements (no loops) | O(1) |
| One loop running n times | O(n) |
| Two nested loops (each n) | O(n²) |
| Three nested loops | O(n³) |
| Loop that halves/doubles `i` each step | O(log n) |
| Outer loop n, inner loop log n | O(n log n) |
| Recursion T(n) = 2·T(n/2) + n | O(n log n) — Master Theorem |

---

### Example (1) — Sequential Statements

```go
x := a + b
y := x * c
z := y / 2
```

3 operations regardless of input size → **O(1)**.

---

### Example (2) — Single Loop

```go
for i := 0; i < n; i++ {
    a[i] = a[i] + 1  // runs n times
}
```

The assignment runs `n` times → **T(n) = n → O(n)**.

---

### Example (3) — Nested Loops (equal depth)

```go
for i := 0; i < n; i++ {
    for j := 0; j < n; j++ {
        a[i][j] = a[i][j] + 1  // runs n × n = n² times
    }
}
```

→ **T(n) = n² → O(n²)**.

---

### Example (4) — Triangular Nested Loop

```go
for i := 1; i <= n; i++ {
    for j := 1; j <= i; j++ {
        count++  // j runs 1, 2, 3, …, n times
    }
}
```

Total operations:

$$T(n) = 1 + 2 + 3 + \cdots + n = \frac{n(n+1)}{2}$$

The dominant term is n² → **T(n) = O(n²)**.

---

### Example (5) — Dividing the Range Each Step (Logarithmic)

```go
i := 1
for i <= n {
    i = i * 2  // i doubles each iteration
}
```

After k iterations, `i = 2ᵏ`. The loop stops when `2ᵏ > n`, so `k ≈ log₂(n)`.

→ **T(n) = O(log n)**.

> This is the characteristic pattern of **divide and conquer** algorithms like Binary Search.

---

### Example (6) — Outer Loop + Logarithmic Inner Loop

```go
for i := 0; i < n; i++ {
    j := n
    for j > 0 {
        j = j / 2  // j halves each step: log₂(n) iterations
    }
}
```

Outer: n iterations. Inner: log n iterations per outer step.

→ **T(n) = n × log n → O(n log n)**.

---

## Section 4 — How Input Data Affects Complexity

The same algorithm can run faster or slower depending on the **initial state of the data**, not just its size.

| Scenario | Description | Example |
|----------|-------------|---------|
| **Best case** | Data is already in the ideal state | Sorted array in Bubble Sort → O(n) with early exit |
| **Average case** | Random data | Typical analysis assumes random distribution |
| **Worst case** | Data arranged in the most unfavourable way | Reversed array in Insertion Sort → O(n²) |

> In practice, we almost always analyze the **worst case** — it gives us a **guaranteed upper bound** on running time, regardless of what data arrives.

---

## Section 5 — Time vs Space Trade-offs

Besides time, every algorithm has a **space (memory) cost**:

| Resource | What to Measure |
|----------|----------------|
| **Time** | Number of operations (Big O) |
| **Space** | Extra memory used (auxiliary space), also in Big O |

### The Trade-off

Often, using more memory makes an algorithm faster:

- **Caching/Memoization**: store computed results → faster lookups, more memory used.
- **Pre-built index**: build a hash map once → O(1) lookup instead of O(n) scan.
- Conversely, reducing memory may slow the algorithm (e.g., avoiding pre-allocation forces more recomputation).

> Neither time nor space can always win outright — you must optimize **both** for your constraints.

---

## Section 6 — Worked Example: Two Ways to Compute eˣ

Computing the Taylor series approximation of eˣ = 1 + x + x²/2! + x³/3! + …

**Method 1 (Slow — O(n²))**

```go
// Recomputes x^i / i! from scratch each time
S := 1.0
for i := 1; i <= n; i++ {
    p := 1.0
    for j := 1; j <= i; j++ {
        p = p * x / float64(j)  // inner loop runs i times
    }
    S += p
}
```

Total multiplications ≈ 1 + 2 + 3 + … + n = **n(n−1)/2 → O(n²)**.

---

**Method 2 (Fast — O(n))**

```go
// Reuses the previous term: p_i = p_{i-1} * x / i
p := 1.0
S := 1.0
for i := 1; i <= n; i++ {
    p = p * x / float64(i)  // single multiplication per step
    S += p
}
```

Total multiplications = n → **O(n)**.

> Both methods produce the same mathematical result. Method 2 is **O(n)** vs Method 1's **O(n²)** — for n = 1000, that is 1000 operations vs 500,000.

---

## Quick Reference

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(n³) < O(2ⁿ) < O(n!)
```

---

## Memory Aid

> Big O is your "speed limit sign" for algorithms — it tells you how fast your algorithm can go as the data highway (n) grows. Constants are rounding error; the growth class is everything.

---

## What to Learn Next

- [03-recursion.md](03-recursion.md) — Recursive algorithms and their complexity via the Master Theorem
- [07-sorting.md](07-sorting.md) — Comparing O(n²) vs O(n log n) sorting algorithms in practice
