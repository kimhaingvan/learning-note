# B-Tree and B+Tree in Database Systems

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> How B-tree and B+tree data structures power database indexes — with insertion, splitting, and search mechanics explained step by step.

---

## Table of Contents

1. [One-Line Summary](#1-one-line-summary)
2. [What Is a B-Tree?](#2-what-is-a-b-tree)
3. [B-Tree Limitations](#3-b-tree-limitations)
4. [What Is a B+Tree?](#4-what-is-a-btree)
5. [Why Use B-Tree?](#5-why-use-b-tree)
6. [Is It "Big" If All Data Is in Leaf Nodes?](#6-is-it-big-if-all-data-is-in-leaf-nodes)
7. [How a B-Tree Is Built (Insertion + Splitting)](#7-how-a-b-tree-is-built-insertion--splitting)
8. [How a B+Tree Is Built](#8-how-a-btree-is-built)
9. [Searching a B-Tree](#9-searching-a-b-tree)

---

## 1. One-Line Summary

> B-tree is PostgreSQL's default "general-purpose" index: best for equality, ranges, ordering, and joins — used everywhere from primary keys to pagination and time filtering.

---

## 2. What Is a B-Tree?

**B-tree** is PostgreSQL's **default index type**. It organizes data in a balanced tree structure where every path from root to leaf has the same length.

### Structure

A PostgreSQL B-tree index is a tree of **pages/nodes** (each page = 8 KB):

```
              ┌──────────────┐
              │  Root Page   │
              └──────┬───────┘
         ┌───────────┼───────────┐
         ▼           ▼           ▼
  ┌────────────┐ ┌────────────┐ ┌────────────┐
  │ Internal   │ │ Internal   │ │ Internal   │
  │ Page       │ │ Page       │ │ Page       │
  └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
    ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
    ▼         ▼     ▼         ▼     ▼         ▼
 ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
 │ Leaf │ │ Leaf │ │ Leaf │ │ Leaf │ │ Leaf │ │ Leaf │
 └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
```

### Key Concepts

- Each element has a **key** and a **value**:
  - **Key**: The indexed column value (or composite key).
  - **Value (TID)**: Pointer to the heap row = `(block_number, offset)`.
  - Example: `email="alice@example.com"` → `TID(1234, 17)`

- Each page contains:
  - A **page header** (metadata)
  - Many **index tuples** (entries)
  - A **special space** area for B-tree-specific info (sibling links, flags)

- **Each node in a B-tree is a disk page** in memory.

### Node Capacity

A node can store up to **d keys** (not only 2):

- Max keys per node: `L = d`
- Overflow size (before split): `d + 1`
- On split:
  - Left child gets: `ceil((d+1) / 2)` keys
  - Right child gets: `floor((d+1) / 2)` keys

### Important Distinction

In a B-tree, **keys and values exist in ALL nodes** — both internal nodes and leaf nodes store actual data pointers.

---

## 3. B-Tree Limitations

| Limitation | Impact |
|-----------|--------|
| All nodes store both key **and** value | Higher space usage → higher memory cost |
| Internal nodes are larger | Require more I/O to traverse, slowing lookups |
| Range queries are slow | Must traverse back up and down the tree (random access pattern) |

> **B+Tree solves all of these problems.**

---

## 4. What Is a B+Tree?

A **B+tree** is a variation of the B-tree where:

- **Satellite data (TIDs/values) is stored ONLY in leaf nodes**.
- **Internal nodes store keys for routing only** — they don't point to heap rows.
- **Leaf nodes are linked together** in a doubly-linked list.

```
Internal nodes: [keys only, for navigation]
              ┌──────────────┐
              │  Root: [20]  │    ← keys only, no TIDs
              └──────┬───────┘
         ┌───────────┴───────────┐
         ▼                       ▼
  ┌────────────┐          ┌────────────┐
  │ Internal   │          │ Internal   │    ← keys only
  │ [5, 10]    │          │ [30, 40]   │
  └──────┬─────┘          └──────┬─────┘
         │                       │
Leaf nodes: [keys + TIDs, linked together]
  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
  │1,2,3 │──▶│5,7,9 │──▶│20,25 │──▶│30,35 │
  │+TIDs │   │+TIDs │   │+TIDs │   │+TIDs │
  └──────┘   └──────┘   └──────┘   └──────┘
```

### Key Differences from B-Tree

| Feature | B-Tree | B+Tree |
|---------|--------|--------|
| Where data pointers live | All nodes | Leaf nodes only |
| Internal node size | Larger (key + value) | Smaller (key only) |
| Leaf-to-leaf links | No | Yes (linked list) |
| Range query efficiency | Poor (random access) | Excellent (follow links) |

### Searching in a B+Tree

- You **must always traverse to a leaf** node to retrieve data.
- Internal nodes only guide you left or right.

### Range Queries

This is where B+tree excels:

1. Find the leaf containing the **start key**.
2. Follow the **leaf links** until the **end key**.
3. No need to go back up the tree.

---

## 5. Why Use B-Tree?

B-tree (specifically B+tree, which PostgreSQL actually uses) is the "best first choice" index because it supports the most common query patterns:

| Pattern | How B-tree Helps |
|---------|-----------------|
| `WHERE id = ...` | Fast equality lookup — O(log n) |
| `WHERE created_at BETWEEN ...` | Fast range query via leaf links |
| `ORDER BY column` | If index order matches, PostgreSQL skips sorting entirely |
| `LIKE 'abc%'` | Prefix match uses B-tree efficiently |
| `JOIN ON indexed_column` | B-tree on join keys makes joins much faster |

---

## 6. Is It "Big" If All Data Is in Leaf Nodes?

No. In practice, B+tree is actually **faster** despite having data only in leaves:

### Why B+Tree Is Fast in Practice

- **Shallower tree**: Internal nodes store **only keys** → higher fanout per page → fewer tree levels → **fewer page reads**.
- **Range scans are efficient**: Leaves are linked → `BETWEEN`, `ORDER BY`, pagination become "walk forward" operations.
- **Better caching**: The top levels (root + a few internal pages) are small and tend to **stay hot in the buffer pool** (memory).

> **Plain English**: A B+tree with 1 billion rows might only be 3-4 levels deep. The root and internal pages are so frequently accessed that they're almost always cached in memory — so most lookups only need 1 actual disk I/O (to read the leaf page).

---

## 7. How a B-Tree Is Built (Insertion + Splitting)

### Insertion Process

1. Insert keys into a node in **sorted order** until it reaches capacity.
2. If inserting would exceed max keys → **split**:
   - Choose the **middle key**.
   - **Promote** the middle key to the parent.
   - Left keys form the left child; right keys form the right child.
3. If the parent is also full, split it upward (possibly creating a new root).

---

## 8. How a B+Tree Is Built

### Rule 1: Leaf Node Overflow (Split)

When a **leaf node** overflows:

1. **Split the leaf into two leaves** (left + right).
   - Left leaf gets: `ceil((L+1) / 2)` keys
   - Right leaf gets: `floor((L+1) / 2)` keys
2. **Copy-up** the first key of the new right leaf into the parent.
   - **Copy-up** means: the key appears in the parent **AND** remains in the leaf.
   - (Because leaf nodes must keep all data.)

### Rule 2: Internal Node Overflow (Split)

When an **internal node** overflows:

1. **Split** it into two internal nodes.
2. **Promote** a separator key to the parent (standard B-tree-style push-up).
3. This is **NOT** a copy-up — the promoted key is **removed** from the child.
   - (Because internal nodes don't store data, the key only needs to exist once for routing.)

### Complete Insertion Algorithm

1. **Find the target leaf** (using internal router keys).
2. **Insert** key/value in sorted order into the leaf.
3. If the leaf overflows → **split leaf** + **copy-up** first key of right leaf to parent.
4. If the parent overflows → **split internal node** + **promote** (not copy-up) separator key.
5. If the root overflows → a **new root** is created (tree grows one level taller).

### Visual Example

```
Insert sequence: 5, 10, 15, 20, 25 (max 3 keys per node)

Step 1: [5]
Step 2: [5, 10]
Step 3: [5, 10, 15]     ← full!
Step 4: Insert 20 → overflow → split:

          [15]                 ← copy-up
         /    \
    [5,10]   [15,20]          ← leaf nodes (linked)

Step 5: Insert 25:

          [15]
         /    \
    [5,10]   [15,20,25]      ← right leaf full!

Next insert would trigger another leaf split...
```

---

## 9. Searching a B-Tree

### How Search Works

1. Start at the **root node**.
2. In each node: scan keys until you find the first key **≥** target.
3. Either **match** it (found!) or **descend** into the appropriate child pointer.
4. Repeat until you reach a leaf (in B+tree) or find the key (in B-tree).

### Complexity

| Operation | Time Complexity |
|-----------|----------------|
| Search | **O(log n)** |
| Insert | **O(log n)** |
| Delete | **O(log n)** |

> All operations are logarithmic because the tree is always **balanced** — every path from root to leaf has the same length.

---

## What to Learn Next

1. **B-tree page structure in PostgreSQL** — How `pageinspect` extension reveals index internals.
2. **Index bloat** — How dead tuples in indexes waste space and slow queries.
3. **REINDEX** — When and how to rebuild bloated indexes.
4. **GIN and GiST trees** — Alternative index structures for full-text search and geometric data.
5. **Hash indexes** — When they outperform B-tree (exact match only, no range).
