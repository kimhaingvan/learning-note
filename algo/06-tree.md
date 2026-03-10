# Trees

> **Category**: Algorithms | **Level**: Intermediate

---

## What

A **tree** is a hierarchical data structure consisting of **nodes** connected by **edges**, with a single designated **root node**.

**Recursive definition:**
- A single node is a tree (its own root).
- Taking a node `n` as the parent of roots `n₁, n₂, …, nₖ` of `k` disjoint trees produces a new tree rooted at `n`, with `T₁, T₂, …, Tₖ` as its **subtrees**.
- An **empty tree** has no nodes.

---

## Terminology

| Term | Definition |
|------|-----------|
| **Root** | The topmost node — has no parent |
| **Parent / Child** | A node's direct predecessor / successor |
| **Leaf** | A node with no children (degree = 0) |
| **Branch node** | Any non-leaf node |
| **Degree of a node** | Number of children |
| **Degree of the tree** | Maximum degree of any node |
| **Level** | Root is level 1; children of level i are level i+1 |
| **Height / Depth** | The maximum level in the tree |
| **Forest** | A set of disjoint trees; removing the root of a tree yields a forest of subtrees |
| **Subtree** | Any node and all of its descendants |

### Example Tree

```
          A          ← Level 1 (root), degree = 3
        / | \
       B  C  D       ← Level 2
      /|     |
     E  F    G       ← Level 3
```

- Degree of A = 3; Degree of B = 2; Degree of D = 1
- Leaves: E, F, C, G
- Height = 3

---

## Why Trees?

Trees model **hierarchical relationships** that naturally appear everywhere:

| Domain | Tree Representation |
|--------|-------------------|
| File system | Directories and files |
| HTML/XML DOM | Nested elements |
| Organisation chart | Manager → reports |
| Book/document outline | Chapters → sections → sub-sections |
| Arithmetic expression | Operators and operands |
| Decision tree / AI | Branching conditions |

---

## Binary Tree

### Definition

A **binary tree** is a tree where **every node has at most 2 children**, and the children are **ordered**: left child and right child (not interchangeable).

```go
type Node[T any] struct {
    Val         T
    Left, Right *Node[T]
}
```

### Special Forms

| Form | Definition | Characteristic |
|------|-----------|----------------|
| **Degenerate (skewed)** | Every non-leaf has only 1 child | Behaves like a linked list, height = n |
| **Full (perfect)** | Every node has exactly 0 or 2 children, all leaves at same level | 2ʰ − 1 nodes for height h |
| **Complete** | All levels filled except possibly the last, last level filled left-to-right | Used for Heap |

```
Full binary tree (height 3):
          1
        /   \
       2     3
      / \   / \
     4   5 6   7

Degenerate (left-skewed):
1
 \
  2
   \
    3
     \
      4
```

### Key Properties of Binary Trees

For a binary tree of height h:
- **Maximum nodes**: $2^h - 1$ (full tree)
- **Minimum nodes**: $h$ (degenerate/skewed tree)
- **Height of complete tree with n nodes**: $\lfloor \log_2 n \rfloor + 1$

---

## Representing a Binary Tree

### Option A — Array Representation (1-based index)

Store nodes level by level, left to right. For a node at index `i`:
- **Left child**: `2 * i`
- **Right child**: `2 * i + 1`
- **Parent**: `i / 2`

```
Tree:         Array T[1..7]:
     1         T[1]=1
   /   \       T[2]=2, T[3]=3
  2     3      T[4]=4, T[5]=5, T[6]=6, T[7]=7
 / \ / \
4   5 6  7
```

**Best for:** Complete / nearly complete trees (heap, segment tree). Wastes memory for sparse/skewed trees.

```go
type ArrayTree[T any] struct {
    T    []T
    Used []bool // marks which indices are occupied (for sparse trees)
}

func NewArrayTree[T any](cap int) *ArrayTree[T] {
    return &ArrayTree[T]{
        T:    make([]T, cap+1), // 1-based: index 0 unused
        Used: make([]bool, cap+1),
    }
}

func Left(i int) int   { return 2 * i }
func Right(i int) int  { return 2*i + 1 }
func Parent(i int) int { return i / 2 }
```

### Option B — Linked Representation

Each node is a struct with pointers to its left and right children. Access starts from the `root` pointer.

**Best for:** Sparse, skewed, or dynamically changing trees.

```go
type LinkedTree[T any] struct {
    Root *Node[T]
}

func (t *LinkedTree[T]) InsertLeft(p *Node[T], v T) *Node[T] {
    n := &Node[T]{Val: v}
    p.Left = n
    return n
}

func (t *LinkedTree[T]) InsertRight(p *Node[T], v T) *Node[T] {
    n := &Node[T]{Val: v}
    p.Right = n
    return n
}
```

### Comparison

| Criterion | Array (1-based) | Linked (pointer) |
|-----------|:---:|:---:|
| Random access by index | O(1) (formula) | Not supported |
| Memory for complete tree | Tight, no waste | 2 pointers per node overhead |
| Memory for sparse tree | Wastes empty slots | No wasted slots |
| Insert / delete subtree | Difficult (shift indices) | Easy (update pointers) |
| Level-order traversal | Natural (iterate indices) | Use a Queue (BFS) |
| Best for | Heap, Segment tree | BST, general trees |

---

## Binary Tree Traversals

**Traversal** = visiting every node exactly once in a systematic order.

Three recursive orderings based on when the **current node (N)** is visited relative to its **Left (L)** and **Right (R)** subtrees:

| Order | Pattern | Typical Use |
|-------|---------|-------------|
| **Preorder** | N → L → R | Serialize/copy tree; prefix expression |
| **Inorder** | L → N → R | BST gives sorted output; infix expression |
| **Postorder** | L → R → N | Delete tree; postfix expression |

```
Tree:
       A
      / \
     B   C
    / \   \
   D   E   F

Preorder  (NLR): A B D E C F
Inorder   (LNR): D B E A C F
Postorder (LRN): D E B F C A
```

### Recursive Implementation

```go
func Preorder[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    visit(r.Val)           // N
    Preorder(r.Left, visit)  // L
    Preorder(r.Right, visit) // R
}

func Inorder[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    Inorder(r.Left, visit)   // L
    visit(r.Val)             // N
    Inorder(r.Right, visit)  // R
}

func Postorder[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    Postorder(r.Left, visit)  // L
    Postorder(r.Right, visit) // R
    visit(r.Val)              // N
}
```

### Iterative Implementation (using explicit Stack/Queue)

When recursion depth is a concern (very deep trees), use an explicit stack.

```go
// Iterative Inorder (LNR) — most commonly tested in interviews
func InorderIter[T any](r *Node[T], visit func(T)) {
    var stack []*Node[T]
    for r != nil || len(stack) > 0 {
        // Go as far left as possible
        for r != nil {
            stack = append(stack, r)
            r = r.Left
        }
        // Pop and visit
        r = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        visit(r.Val)
        r = r.Right // move to right subtree
    }
}

// Iterative Preorder (NLR)
func PreorderIter[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    stack := []*Node[T]{r}
    for len(stack) > 0 {
        n := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        visit(n.Val)
        // Push Right first so Left is processed first
        if n.Right != nil { stack = append(stack, n.Right) }
        if n.Left != nil  { stack = append(stack, n.Left)  }
    }
}

// Level-order traversal (BFS) — uses a Queue
func LevelOrder[T any](r *Node[T], visit func(T)) {
    if r == nil { return }
    queue := []*Node[T]{r}
    for len(queue) > 0 {
        n := queue[0]
        queue = queue[1:]
        visit(n.Val)
        if n.Left != nil  { queue = append(queue, n.Left)  }
        if n.Right != nil { queue = append(queue, n.Right) }
    }
}
```

---

## K-ary Tree (General Tree)

### Definition

A **K-ary tree** is a tree where each node has **at most K children**, numbered 1 through K (ordered).

### Array Representation (0-based)

For a node at index `i` in a K-ary tree:
- **Child j** (1 ≤ j ≤ K): `i*K + j`
- **Parent** of node `x > 0`: `(x-1) / K`

```go
type KArrayTree[T any] struct {
    T    []T
    Used []bool
    K    int
}

func Child(i, j, K int) int { return i*K + j }  // j in [1..K]
func ParentK(x, K int) int  { if x == 0 { return -1 }; return (x-1)/K }
```

### Linked Representation

```go
type KNode[T any] struct {
    Val  T
    Kids []*KNode[T] // length K; nil entries mean absent children
}

func NewKNode[T any](k int, val T) *KNode[T] {
    return &KNode[T]{Val: val, Kids: make([]*KNode[T], k)}
}

// Preorder
func KPreorder[T any](u *KNode[T], visit func(T)) {
    if u == nil { return }
    visit(u.Val)
    for _, child := range u.Kids {
        KPreorder(child, visit)
    }
}

// Level-order (BFS)
func KLevelOrder[T any](root *KNode[T], visit func(T)) {
    if root == nil { return }
    queue := []*KNode[T]{root}
    for len(queue) > 0 {
        u := queue[0]; queue = queue[1:]
        visit(u.Val)
        for _, child := range u.Kids {
            if child != nil { queue = append(queue, child) }
        }
    }
}
```

---

## Pitfalls

- **Infinite loop in traversal**: always check `if r == nil { return }` — the base case.
- **Off-by-one in array tree**: use 1-based indexing consistently; index 0 is unused.
- **Forgetting to handle `nil` children** in insert/traverse operations.
- **Deep recursion on skewed trees**: height can reach n → stack depth = n → use iterative for production code on untrusted data.

---

## Memory Aid

> A tree is an **upside-down family tree** — the ancestor (root) is at the top, descendants at the bottom. Traversal is just deciding whether to visit parents first (preorder) or children first (postorder).

---

## What to Learn Next

- [07-sorting.md](07-sorting.md) — Heap Sort uses a binary tree stored as an array
- [08-search.md](08-search.md) — Binary Search Tree (BST) enables O(log n) lookup
