# Algorithm Problem-Solving: 6 Essential Steps

> **Category**: Algorithms | **Level**: Beginner → Intermediate

---

## What

When facing an algorithmic problem, you need a **systematic process** — not just a guess at code. These 6 steps guide you from understanding the problem all the way to a production-quality solution.

```
Problem → Understand → Data Structure → Algorithm → Code → Test → Optimize
```

---

## Step 1 — Define the Problem

### Goal

Clearly understand **what needs to be solved**, identify the **input**, the **output**, and the **relationship between them**.

### What to Do

- **Read the problem statement carefully.** Understand every constraint, assumption, and edge case — do not skim.
- Every algorithmic problem follows the model:

  ```
  Input → Processing → Output
  ```

- Your job is to **find the rule or formula** that transforms Input into Output.
- Distinguish:
  - **Exact problems**: must produce a mathematically precise answer.
  - **Approximate problems**: acceptable to return a "good enough" answer if it is far more efficient.
- Sometimes you need to **restate the problem in formal notation** (math or logic) to make it easier to model.

### Key Questions to Ask

| Question | Why It Matters |
|----------|---------------|
| What is the input? What are the constraints on it? | Shapes your algorithm choice |
| What exactly is the expected output? | Defines correctness |
| What are the edge cases? (empty input, single element, duplicates) | Reveals boundary bugs |
| Are there multiple valid outputs? | Determines if we need exact or approximate solution |

### Example

> **Problem**: Given an array of integers a₁, a₂, …, aₙ, find the maximum element.
>
> - **Input**: an array of n integers
> - **Output**: the maximum value
> - **Goal**: traverse the array, compare elements, identify the largest

---

## Step 2 — Choose the Right Data Structure

### Goal

Find the best **way to represent and store data** so that operations are fast, memory-efficient, and easy to reason about.

### What to Do

A **data structure** is a way of organizing, storing, and accessing data in computer memory.

Each type of problem calls for a different structure:

| Scenario | Best Structure |
|----------|---------------|
| Ordered sequence of elements | Array / Slice |
| Dynamic collection (frequent insert/delete) | Linked List |
| Process in arrival order | Queue (FIFO) |
| Undo / backtrack / call stack | Stack (LIFO) |
| Pair relationships between elements | Graph |
| Hierarchical parent-child relationships | Tree |
| Fast lookup by key | Hash Map |
| Sorted lookup with range queries | Balanced BST |

> **Rule of thumb:** choosing the right data structure often reduces algorithm complexity by an order of magnitude — before you've written a single line of logic.

### Example

- Finding the maximum in a sequence → a plain **array** is sufficient.
- Processing tasks in order of arrival → use a **queue**.
- Undo history in a text editor → use a **stack**.

---

## Step 3 — Find the Algorithm

### Goal

Determine the **step-by-step procedure** for transforming input into output — this is the "heart" of your solution.

### What a Good Algorithm Must Have

| Property | Meaning |
|----------|---------|
| **Correctness** | Always produces the right result for any valid input |
| **Efficiency** | Uses minimal time and memory |
| **Generality** | Works for all valid inputs, not just examples |

### Ways to Describe an Algorithm

1. **Natural language** — describe each step in plain English
2. **Flowchart** — visual diagram of the control flow
3. **Pseudocode** — structured, language-agnostic code notation

### Example: Find Maximum in an Array

```
Step 1: Set max = a[0]
Step 2: For i = 1 to n-1:
           if a[i] > max, then max = a[i]
Step 3: Return max
```

This is a correct, O(n) linear algorithm. It is also **optimal** — you must look at every element at least once, so you cannot do better than O(n).

---

## Step 4 — Implement (Code It)

### Goal

Translate the algorithm into **working code** in a specific programming language.

### Approach: Stepwise Refinement

1. Write the **general structure** (function signature, major phases).
2. **Break down each phase** into smaller, concrete steps.
3. **Translate progressively** into actual language syntax.

> Keep the implementation faithful to your algorithm. Avoid mixing the "design" and "coding" phases — bugs often sneak in when you improvise during coding.

### Best Practices

| Practice | Why |
|----------|-----|
| Break the program into **functions / modules** | Easier to test, debug and reason about each piece |
| Preserve the **algorithm's structure** in code | Bugs from "clever shortcuts" are hard to trace |
| Use **meaningful names** | Future-you will thank present-you |

### Example (Go)

```go
func findMax(arr []int) int {
    if len(arr) == 0 {
        panic("empty array")
    }
    maxVal := arr[0]
    for _, x := range arr[1:] {
        if x > maxVal {
            maxVal = x
        }
    }
    return maxVal
}
```

### Practice

- Implement a function to compute the sum of all positive numbers in a slice.
- Compare two versions: a manual `for` loop vs using `slices.Max` from the standard library.

---

## Step 5 — Test

### Goal

Ensure the program **runs correctly and handles all situations** — including the ones you did not think of.

### Three Categories of Tests

| Category | What It Checks | Example |
|----------|---------------|---------|
| **Functional testing** | Correct output for normal input | `[1, 2, 3] → 3` |
| **Boundary testing** | Smallest, largest, or empty input | `[7] → 7`, `[] → error` |
| **Invalid input testing** | Program handles bad data gracefully | `nil input → panic/sentinel handled` |

> Testing is not just bug-finding. It is **confidence-building** — you verify that your algorithm is trustworthy for all valid inputs.

### Example (Table-Driven Test in Go)

```go
func TestFindMax(t *testing.T) {
    tests := []struct {
        name     string
        input    []int
        expected int
    }{
        {"normal",    []int{1, 2, 3},     3},
        {"negative",  []int{-1, -5, -2}, -1},
        {"single",    []int{7},            7},
        {"all-equal", []int{4, 4, 4},      4},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            got := findMax(tc.input)
            if got != tc.expected {
                t.Errorf("findMax(%v) = %d; want %d", tc.input, got, tc.expected)
            }
        })
    }
}
```

---

## Step 6 — Optimize

### Goal

After the program is **correct**, improve its **performance** — reduce time and memory usage.

### Two Directions of Optimization

| Direction | Technique | Example |
|-----------|-----------|---------|
| **Algorithm** | Replace with a faster algorithm | O(n²) → O(n log n) |
| **Data structure** | Use a more efficient structure for the dominant operation | Array → Segment Tree for range queries |

### Additional Techniques

- **Caching / Memoization** — avoid re-computing identical sub-problems.
- **Reduce auxiliary memory** — use in-place algorithms where possible.
- **Use appropriate types** — choose `int32` vs `int64`, `bool` over string flags, etc.

### Example

- **Finding max in a single pass** → already O(n), cannot be improved (lower bound: must look at every element).
- **Finding max over many sub-ranges** repeatedly → replace naïve O(n) per query with a **Segment Tree** for O(log n) per query after O(n) preprocessing.

---

## Summary

| Step | Question to Answer |
|------|-------------------|
| 1. Define the problem | What is input? What is output? What are the constraints? |
| 2. Choose data structure | How should data be stored and accessed? |
| 3. Find the algorithm | What is the step-by-step procedure? |
| 4. Implement | How does this translate to code? |
| 5. Test | Does it work for all cases, including edge cases? |
| 6. Optimize | Is it fast enough? Can we do better? |

---

## What to Learn Next

- [02-algorithm-complexity.md](02-algorithm-complexity.md) — Measuring and comparing algorithm efficiency
