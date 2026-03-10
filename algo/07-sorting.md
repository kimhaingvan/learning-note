# Sorting Algorithms

> **Category**: Algorithms | **Level**: Intermediate

---

## What

**Sorting** rearranges a collection of elements into a defined order (ascending, descending, lexicographic, etc.).

In practice, data is often a collection of **records** with multiple fields. We sort by a designated **key field** and may use an **index array** to avoid physically moving large records.

---

## Algorithm Comparison at a Glance

| Algorithm | Best | Average | Worst | Space | Stable? | Notes |
|-----------|:----:|:-------:|:-----:|:-----:|:-------:|-------|
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | ❌ | Min swaps; good for large records |
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ | Simple; early-exit optimization |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ | Fast on nearly-sorted data |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ | Guaranteed O(n log n) always |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ | Fastest in practice; pivot choice matters |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | Best for stable sort; great for external sort |

---

## 1. Selection Sort

### Idea

On each pass `i`, **find the minimum element** in the unsorted sub-array `a[i..n-1]` and **swap** it into position `i`.

```
Pass 1: find min of [5, 3, 8, 1, 9] → 1, swap with a[0] → [1, 3, 8, 5, 9]
Pass 2: find min of [3, 8, 5, 9] → 3, already in place  → [1, 3, 8, 5, 9]
Pass 3: find min of [8, 5, 9] → 5, swap with a[2]       → [1, 3, 5, 8, 9]
Pass 4: find min of [8, 9] → 8, already in place        → [1, 3, 5, 8, 9]
Done.
```

### Properties

| Property | Detail |
|----------|--------|
| Time complexity | Always **O(n²)** — does not benefit from pre-sorted data |
| Swaps | At most **n−1** — fewest swaps of any O(n²) sort |
| Stable | **No** — a swap can move an equal element over another |
| Best use | Move-expensive records (swap is costly); educational purposes |

### Implementation (Go)

```go
import "golang.org/x/exp/constraints"

func SelectionSort[T constraints.Ordered](a []T) {
    n := len(a)
    for i := 0; i < n-1; i++ {
        minIdx := i
        for j := i + 1; j < n; j++ {
            if a[j] < a[minIdx] {
                minIdx = j
            }
        }
        if minIdx != i {
            a[i], a[minIdx] = a[minIdx], a[i]
        }
    }
}
```

---

## 2. Bubble Sort

### Idea

Repeatedly scan the array, **swapping adjacent elements that are out of order**. After each pass, the largest unsorted element "bubbles up" to its correct position at the end.

```
Pass 1: [5,1,4,2,8] → compare & swap →
        [1,4,2,5,8]  ← 8 is settled

Pass 2: [1,4,2,5,8] → [1,2,4,5,8] ← 5 settled (no more swaps in remaining)
Early exit: done!
```

### Properties

| Property | Detail |
|----------|--------|
| Best case | **O(n)** — if array is already sorted (with early-exit flag) |
| Worst/Average case | **O(n²)** |
| Stable | **Yes** — only swaps when strictly out of order |
| Best use | Very small arrays, educational purposes; not production data |

### Implementation (Go)

```go
// Variant 1: bubble smallest element toward the front each pass
func BubbleSortAsc[T constraints.Ordered](a []T) {
    n := len(a)
    for i := 1; i < n; i++ {
        swapped := false
        for j := n - 1; j >= i; j-- {
            if a[j] < a[j-1] {
                a[j], a[j-1] = a[j-1], a[j]
                swapped = true
            }
        }
        if !swapped { break } // already sorted — early exit
    }
}

// Variant 2: bubble largest element toward the end each pass (more common)
func BubbleSortTail[T constraints.Ordered](a []T) {
    n := len(a)
    for end := n - 1; end > 0; end-- {
        swapped := false
        for j := 0; j < end; j++ {
            if a[j+1] < a[j] {
                a[j], a[j+1] = a[j+1], a[j]
                swapped = true
            }
        }
        if !swapped { break }
    }
}
```

---

## 3. Insertion Sort

### Idea

Like sorting **playing cards in your hand**: for each new card (element), insert it into its correct position among the already-sorted cards to its left.

```
Start:   [5, 2, 4, 6, 1, 3]
i=1: key=2  → [2, 5, 4, 6, 1, 3]
i=2: key=4  → [2, 4, 5, 6, 1, 3]
i=3: key=6  → [2, 4, 5, 6, 1, 3]  (no moves needed)
i=4: key=1  → [1, 2, 4, 5, 6, 3]
i=5: key=3  → [1, 2, 3, 4, 5, 6]
```

### Properties

| Property | Detail |
|----------|--------|
| Best case | **O(n)** — array already sorted |
| Average / Worst | **O(n²)** |
| Stable | **Yes** |
| Best use | Small arrays (< 50 elements); nearly-sorted data; inner loop of hybrid sorts (TimSort) |

### Detailed Step-by-Step

```
[5, 2, 4, 6, 1, 3]

Step i=1: key=2, compare 5>2 → shift 5 right → insert 2 at 0
 → [2, 5, 4, 6, 1, 3]

Step i=2: key=4, compare 5>4 → shift 5 right, 2≤4 → stop, insert 4 at 1
 → [2, 4, 5, 6, 1, 3]

Step i=3: key=6, compare 5≤6 → no shift needed
 → [2, 4, 5, 6, 1, 3]

Step i=4: key=1, shift 6,5,4,2 right → insert 1 at 0
 → [1, 2, 4, 5, 6, 3]

Step i=5: key=3, shift 6,5,4 right → 2≤3 stops → insert 3 at index 2
 → [1, 2, 3, 4, 5, 6]
```

### Implementation (Go)

```go
func InsertionSort(a []int) {
    n := len(a)
    for i := 1; i < n; i++ {
        key := a[i]
        j := i - 1
        // Shift elements greater than key one position to the right
        for j >= 0 && a[j] > key {
            a[j+1] = a[j]
            j--
        }
        a[j+1] = key
    }
}
```

---

## 4. Heap Sort

### Idea

1. **Build a max-heap** from the array — parent ≥ children everywhere.
2. Repeatedly:
   - Swap the root (maximum) with the last element of the heap.
   - Shrink the heap size by 1.
   - **Sift-down** from the root to restore the heap property.

### Why It Works

A max-heap guarantees `a[0]` is always the current maximum. After swapping it to the end, sifting-down restores the heap for the remaining elements. After n−1 rounds, the array is sorted.

```
Initial:  [4, 10, 3, 5, 1]
Build max-heap: [10, 5, 3, 4, 1]

Round 1: swap a[0]↔a[4] → [1,5,3,4,10], sift-down(0,4) → [5,4,3,1,10]
Round 2: swap a[0]↔a[3] → [1,4,3,5,10], sift-down(0,3) → [4,1,3,5,10]
Round 3: swap a[0]↔a[2] → [3,1,4,5,10], sift-down(0,2) → [3,1,4,5,10]
Round 4: swap a[0]↔a[1] → [1,3,4,5,10]
Done: [1, 3, 4, 5, 10]
```

### Properties

| Property | Detail |
|----------|--------|
| All cases | **O(n log n)** — no bad inputs |
| Space | **O(1)** in-place |
| Stable | **No** |
| Best use | When you need guaranteed O(n log n) with no extra memory |

### Implementation (Go)

```go
import "golang.org/x/exp/constraints"

func HeapSort[T constraints.Ordered](a []T) {
    n := len(a)
    if n < 2 { return }

    // 1. Build max-heap bottom-up (start from last internal node)
    for i := n/2 - 1; i >= 0; i-- {
        siftDown(a, n, i)
    }

    // 2. Extract elements from heap one by one
    for end := n - 1; end > 0; end-- {
        a[0], a[end] = a[end], a[0] // move current max to end
        siftDown(a, end, 0)          // restore heap property
    }
}

// siftDown pushes a[i] down to its correct position within a[0:n].
func siftDown[T constraints.Ordered](a []T, n, i int) {
    for {
        left := 2*i + 1
        if left >= n { break }
        right := left + 1

        // Find the larger child
        largest := left
        if right < n && a[right] > a[left] {
            largest = right
        }
        // If parent is already ≥ both children, stop
        if a[i] >= a[largest] { break }

        a[i], a[largest] = a[largest], a[i]
        i = largest
    }
}
```

---

## 5. Quick Sort

### Idea (Divide and Conquer)

1. **Choose a pivot** element.
2. **Partition** the array: all elements ≤ pivot go left, all > pivot go right.
3. **Recursively** sort both sides.
4. No merge step needed — the array is sorted in-place.

```
[8, 3, 5, 2, 7, 4]  pivot=4 (last element)

After partition: [3, 2, 4, 8, 7, 5]
                        ↑ pivot in final position

Recurse left: [3, 2] → [2, 3]
Recurse right: [8, 7, 5] → pivot=5 → [5, 7, 8]

Result: [2, 3, 4, 5, 7, 8]
```

### Properties

| Property | Detail |
|----------|--------|
| Best / Average | **O(n log n)** |
| Worst case | **O(n²)** — when pivot is always the min or max (e.g., sorted input + last-element pivot) |
| Space | **O(log n)** for the recursion stack |
| Stable | **No** |
| Best use | General-purpose; fastest in practice on average; use median-of-3 pivot or random pivot to avoid worst case |

### Lomuto Partition (simple, slightly slower)

```go
func QuickSort(a []int, lo, hi int) {
    if lo < hi {
        p := partition(a, lo, hi)
        QuickSort(a, lo, p-1)
        QuickSort(a, p+1, hi)
    }
}

func partition(a []int, lo, hi int) int {
    pivot := a[hi]  // choose last element as pivot
    i := lo - 1
    for j := lo; j < hi; j++ {
        if a[j] <= pivot {
            i++
            a[i], a[j] = a[j], a[i]
        }
    }
    a[i+1], a[hi] = a[hi], a[i+1] // place pivot in correct position
    return i + 1
}

// Usage: QuickSort(arr, 0, len(arr)-1)
```

### Hoare Partition (faster — fewer swaps)

```go
func hoarePartition(a []int, lo, hi int) int {
    pivot := a[lo+(hi-lo)/2] // median-of-three or middle element
    i, j := lo-1, hi+1
    for {
        for { i++; if a[i] >= pivot { break } }
        for { j--; if a[j] <= pivot { break } }
        if i >= j { return j }
        a[i], a[j] = a[j], a[i]
    }
}
```

---

## 6. Merge Sort

### Idea (Divide and Conquer)

1. **Divide**: split array in half.
2. **Recursively sort** each half.
3. **Merge**: combine two sorted halves into one sorted array.

The "hard work" is in the **merge step** — not the split.

```
[5, 1, 4, 2, 8]

Divide:   [5,1,4] and [2,8]
Divide:   [5] | [1,4]  and  [2] | [8]
Divide:   [5] | [1] | [4]  and  [2] | [8]

Merge up: [1,4] → merge [1][4]
          [1,4,5] → merge [5][1,4]
          [2,8] → merge [2][8]
          [1,2,4,5,8] → merge [1,4,5][2,8]
```

### Properties

| Property | Detail |
|----------|--------|
| All cases | **O(n log n)** — guaranteed |
| Space | **O(n)** auxiliary for the merge buffer |
| Stable | **Yes** — equal elements maintain relative order |
| Best use | When stability is required; external sorting (data on disk); linked list sorting (no random access needed for merge) |

### Implementation (Go)

```go
func MergeSort(a []int) []int {
    if len(a) <= 1 {
        return a
    }
    mid := len(a) / 2
    left := MergeSort(a[:mid])
    right := MergeSort(a[mid:])
    return merge(left, right)
}

func merge(left, right []int) []int {
    result := make([]int, 0, len(left)+len(right))
    i, j := 0, 0
    for i < len(left) && j < len(right) {
        if left[i] <= right[j] {
            result = append(result, left[i])
            i++
        } else {
            result = append(result, right[j])
            j++
        }
    }
    result = append(result, left[i:]...)
    result = append(result, right[j:]...)
    return result
}
```

---

## Quick Sort vs Merge Sort — Head-to-Head

| Aspect | Quick Sort | Merge Sort |
|--------|-----------|-----------|
| Strategy | Hard split (pivot/partition) + easy combine | Easy split (halves) + hard combine (merge) |
| Stability | ❌ Not stable | ✅ Stable |
| Extra memory | O(log n) stack | O(n) merge buffer |
| Cache performance | ✅ In-place, cache-friendly | ❌ Copies to auxiliary array |
| Worst case | O(n²) (avoidable with random pivot) | O(n log n) always |
| Practical speed | **Fastest in average case** | Slightly slower in practice |
| External sort | ❌ | ✅ (merge is streaming) |

---

## When to Use Which

| Situation | Recommendation |
|-----------|---------------|
| General purpose, in-memory, unsorted | **Quick Sort** (random pivot) |
| Must be stable | **Merge Sort** |
| Guaranteed worst-case O(n log n), in-place | **Heap Sort** |
| Data nearly sorted | **Insertion Sort** |
| Very small array (< 10-15 elements) | **Insertion Sort** |
| External sort (disk / stream) | **Merge Sort** |
| Production Go code | `sort.Slice` / `slices.Sort` (uses pattern-defeating quicksort internally) |

---

## Pitfalls

- **Quick Sort with sorted input + last-element pivot** → O(n²). Fix: use random pivot or median-of-3.
- **Merge Sort copying subarrays** on every recursive level wastes allocations. Use in-place merge or pass a reusable buffer.
- **Bubble/Selection/Insertion** for large datasets → avoid; O(n²) becomes unacceptably slow above ~10,000 elements.
- **Heap Sort not stable** — do not use when equal elements must preserve their original relative order.

---

## Memory Aid

> Slow sorts (O(n²)): "look at every pair." Fast sorts (O(n log n)): "divide and conquer — split the work, then combine."

---

## What to Learn Next

- [02-algorithm-complexity.md](02-algorithm-complexity.md) — Understand why O(n log n) is the lower bound for comparison-based sorting
- [08-search.md](08-search.md) — Binary search requires a sorted array — sorting is a prerequisite
