# Search Algorithms

> **Category**: Algorithms | **Level**: Intermediate

---

## What

**Searching** is the process of finding an element (or a related position) in a collection.

### Search Models

| Model | Goal | Example |
|-------|------|---------|
| **Exact match** | Find position where `a[i] == x` | "Is 'alice' in this list?" |
| **Lower bound** | First position where `a[i] >= x` | "Where does 42 first appear (or would fit)?" |
| **Upper bound** | First position where `a[i] > x` | "Where does the range for 42 end?" |
| **Range query** | All elements in `[lo, hi]` | "Who scored between 80 and 90?" |

### Data Characteristics That Determine the Right Algorithm

| Factor | Implication |
|--------|------------|
| **Unsorted** | Only linear search works reliably |
| **Sorted** | Binary, exponential, interpolation search all apply |
| **Static** | Pre-sort once; fast lookup forever |
| **Dynamic (insertions/deletions)** | Use BST or hash map; re-sorting after every change is expensive |
| **Numeric, uniform distribution** | Interpolation search shines |
| **Unknown length / stream** | Exponential search first, then binary |

---

## 1. Linear (Sequential) Search

### Idea

Check elements one by one from left to right until the target is found or the end is reached.

### Variants

```go
// 1. Basic — works on any slice; O(n) always
func LinearSearch[T comparable](a []T, x T) int {
    for i, v := range a {
        if v == x { return i }
    }
    return -1
}

// 2. Early-exit on sorted data — O(n) worst, but stop early
func SortedSearch[T constraints.Ordered](a []T, x T) int {
    for i, v := range a {
        if v == x { return i }
        if v > x  { break }  // x cannot be further right (sorted)
    }
    return -1
}

// 3. Sentinel — eliminates the bounds check inside the loop
// Append x temporarily; the loop ALWAYS terminates.
func SentinelSearch(a []int, x int) int {
    a = append(a, x) // sentinel at index len(a)-1
    i := 0
    for a[i] != x { i++ }
    a = a[:len(a)-1] // remove sentinel
    if i < len(a) { return i }
    return -1
}

// 4. Find ALL occurrences
func FindAll[T comparable](a []T, x T) []int {
    var indices []int
    for i, v := range a {
        if v == x { indices = append(indices, i) }
    }
    return indices
}

// 5. Predicate-based (find first match)
func FindIf[T any](a []T, pred func(T) bool) int {
    for i, v := range a {
        if pred(v) { return i }
    }
    return -1
}
```

### When to Use

| ✅ Use | ❌ Avoid |
|--------|---------|
| Unsorted data | Large sorted collections (use binary search) |
| Very small collections (< 16 elements) | Frequent repeated searches on the same data |
| One-off search where pre-sorting is too expensive | Data that changes often but is always queried in range |

---

## 2. Binary Search

### Idea

On a **sorted** array, compare the target with the **middle element**. Discard the half that cannot contain the target. Repeat — halving the search space each step.

**Invariant**: the target is always in the half-open interval `[lo, hi)`.

```
Sorted array: [1, 3, 5, 7, 9, 11, 13]
Search for 7:

lo=0, hi=7, mid=3 → a[3]=7 == 7 → FOUND at index 3

Search for 6:
lo=0, hi=7, mid=3 → a[3]=7 > 6 → hi=3
lo=0, hi=3, mid=1 → a[1]=3 < 6 → lo=2
lo=2, hi=3, mid=2 → a[2]=5 < 6 → lo=3
lo=3, hi=3 → loop ends → NOT FOUND
```

### LowerBound / UpperBound / BinarySearch

These three functions form the backbone of binary search:

| Function | Returns |
|----------|---------|
| `LowerBound(a, x)` | First index `i` where `a[i] >= x` |
| `UpperBound(a, x)` | First index `i` where `a[i] > x` |
| `BinarySearch(a, x)` | Exact match index (uses LowerBound) |

```go
import "golang.org/x/exp/constraints"

// LowerBound: first position where a[i] >= x
// Returns len(a) if all elements < x.
func LowerBound[T constraints.Ordered](a []T, x T) int {
    lo, hi := 0, len(a) // half-open: [lo, hi)
    for lo < hi {
        mid := lo + (hi-lo)/2 // avoids integer overflow
        if a[mid] < x {
            lo = mid + 1
        } else {
            hi = mid
        }
    }
    return lo
}

// UpperBound: first position where a[i] > x
// Returns len(a) if all elements <= x.
func UpperBound[T constraints.Ordered](a []T, x T) int {
    lo, hi := 0, len(a)
    for lo < hi {
        mid := lo + (hi-lo)/2
        if a[mid] <= x {
            lo = mid + 1
        } else {
            hi = mid
        }
    }
    return lo
}

// BinarySearch: returns index of exact match, or -1 if not found.
func BinarySearch[T constraints.Ordered](a []T, x T) int {
    pos := LowerBound(a, x)
    if pos < len(a) && a[pos] == x {
        return pos
    }
    return -1
}
```

### Why Half-Open Intervals `[lo, hi)`?

- `lo == hi` means the range is empty — clean termination.
- `LowerBound(a, x) == len(a)` unambiguously means "not found (too large)".
- The math `lo + (hi-lo)/2` is overflow-safe even for very large indices.

### Which Items Fall in Range [a, b]?

```go
all := []int{1, 2, 3, 3, 4, 5, 6}

// Count elements equal to 3:
count := UpperBound(all, 3) - LowerBound(all, 3) // = 4 - 2 = 2

// Extract elements in [3, 5]:
lo := LowerBound(all, 3)
hi := UpperBound(all, 5)
result := all[lo:hi] // [3, 3, 4, 5]
```

---

## 3. Exponential Search

### Idea

Useful when the array is **sorted but its length is unknown or very large** (e.g., streaming data, infinite arrays). 

1. Keep doubling an index `b` until `a[b] >= x` or `b >= n`.
2. Binary search within `[b/2, min(b, n-1)]` — the window where x can live.

```go
func ExponentialSearch[T constraints.Ordered](a []T, x T) int {
    n := len(a)
    if n == 0 { return -1 }
    if a[0] == x { return 0 }

    // Double bound until we overshoot x
    b := 1
    for b < n && a[b] < x {
        b *= 2
    }

    // Binary search in [b/2, min(b, n-1)]
    lo := b / 2
    hi := b + 1
    if hi > n { hi = n }
    pos := lo + LowerBound(a[lo:hi], x)
    if pos < n && a[pos] == x {
        return pos
    }
    return -1
}
```

| Property | Value |
|----------|-------|
| Time | O(log i) where i is the result index |
| Space | O(1) |
| Requirement | Sorted array |
| Best use | Unbounded/infinite array; result index is small |

---

## 4. Interpolation Search

### Idea

For **uniformly distributed numeric data**, instead of always checking the midpoint, **estimate the likely position** of `x` using linear interpolation:

$$pos = lo + \frac{(x - a[lo]) \times (hi - lo)}{a[hi] - a[lo]}$$

If the distribution is truly uniform (like a fully sequential list of integers), the estimated position is often exact, yielding **O(log log n)** average time.

```go
func InterpolationSearch(a []int, x int) int {
    lo, hi := 0, len(a)-1
    for lo <= hi && x >= a[lo] && x <= a[hi] {
        if lo == hi {
            if a[lo] == x { return lo }
            return -1
        }
        // Estimate position
        pos := lo + (x-a[lo])*(hi-lo)/(a[hi]-a[lo])
        if a[pos] == x { return pos }
        if a[pos] < x {
            lo = pos + 1
        } else {
            hi = pos - 1
        }
    }
    return -1
}
```

| Property | Value |
|----------|-------|
| Average time | **O(log log n)** for uniform data |
| Worst time | **O(n)** for heavily skewed data |
| Requirement | Sorted + numeric + uniform distribution |
| Fragile | Yes — avoid for string keys or non-uniform data |

---

## 5. Hash-Based Search

### Idea

Compute a **hash code** from the key; use it as a direct index into a hash table. On average, lookup is O(1) — no comparison loop needed.

```go
// Go's built-in map is a hash table
index := map[string]int{
    "alice": 1,
    "bob":   2,
    "carol": 3,
}

// O(1) average lookup
if i, ok := index["bob"]; ok {
    fmt.Println("Found at logical index:", i) // 2
}

// Build frequency map
freq := make(map[int]int)
for _, v := range []int{1, 2, 2, 3, 3, 3} {
    freq[v]++
}
// freq = {1:1, 2:2, 3:3}
```

| Property | Value |
|----------|-------|
| Lookup time | **O(1) average**, O(n) worst (hash collisions) |
| Space | O(n) |
| Ordered? | **No** — use a BST or sorted slice for ordered operations |
| Best use | Exact lookup by key; frequency counting; deduplication |

---

## 6. Tree-Based Search (BST / Balanced BST)

### Idea

A **Binary Search Tree (BST)** stores keys such that, for every node:
- All keys in the **left subtree** are smaller.
- All keys in the **right subtree** are larger.

Searching traverses left or right at each node, discarding half the tree each step.

```
      5
    /   \
   3     8
  / \   / \
 1   4 7   9

Search 7: 7 < root(5)? No → go right (8)
          7 < 8? Yes → go left (7) → FOUND
```

### Go's `sort` Package as Sorted Index

For static data, building a sorted slice + binary search is often simpler and faster than maintaining a BST:

```go
import (
    "sort"
    "fmt"
)

data := []int{5, 2, 9, 1, 7, 3}
sort.Ints(data) // sort once

// Exact search
i := sort.SearchInts(data, 7)
if i < len(data) && data[i] == 7 {
    fmt.Println("Found at", i)
}

// Range query: elements in [3, 7]
lo := sort.SearchInts(data, 3)
hi := sort.SearchInts(data, 8) // upper bound of 7
fmt.Println("In [3,7]:", data[lo:hi]) // [3 5 7]
```

### BST vs Balanced BST vs B-Tree

| Structure | Lookup | Insert/Delete | Best Use |
|-----------|:------:|:-------------:|----------|
| Unsorted array | O(n) | O(1) append | Infrequent search |
| Sorted array + binary search | O(log n) | O(n) shift | Static data, reads >> writes |
| BST (unbalanced) | O(n) worst | O(n) worst | Avoid for production |
| AVL / Red-Black Tree | O(log n) | O(log n) | In-memory ordered map/set |
| B-Tree / B+ Tree | O(log n) | O(log n) | Database indexes, disk storage |

---

## Choosing the Right Search Algorithm

| Data | Query Type | Best Choice |
|------|-----------|-------------|
| Unsorted, small | Exact | `LinearSearch` |
| Unsorted, any size | Exact | Hash map |
| Sorted, fixed | Exact, range | `BinarySearch` / `LowerBound` + `UpperBound` |
| Sorted, unknown length | Exact | `ExponentialSearch` |
| Sorted, uniform numeric | Exact | `InterpolationSearch` (with fallback) |
| Dynamic, ordered operations | Exact, range, rank | Balanced BST (`sort.Search` for simple cases) |

---

## Pitfalls

- **Binary search on unsorted data** — produces wrong results silently. Sort first, always.
- **Mid overflow**: use `lo + (hi-lo)/2`, not `(lo+hi)/2` (the latter overflows for large indices).
- **Interpolation search on non-uniform data** — degrades to O(n); unsafe for production without bounds checking.
- **Hash map key collisions** — handled internally by Go runtime, but be aware of worst case and avoid using mutable or unhashable types as keys.
- **Half-open vs closed intervals**: be consistent — mixing `[lo, hi]` and `[lo, hi)` in the same code causes off-by-one bugs.

---

## Memory Aid

> "Search is just asking: where does X belong?" — Linear says "check everywhere". Binary says "cut in half". Hash says "compute the address directly". Tree says "follow the sign at each fork."

---

## What to Learn Next

- [06-tree.md](06-tree.md) — Binary Search Trees are trees + search combined
- [07-sorting.md](07-sorting.md) — Most fast searches require sorted data first
