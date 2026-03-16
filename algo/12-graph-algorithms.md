# Thuật toán Đồ thị (Graph Algorithms)

> **Chủ đề**: Thuật toán | **Mức độ**: Trung bình → Nâng cao
> **Nguồn**: *Giải thuật và Lập trình* — Lê Minh Hoàng, Phần 4

---

## Tổng quan

**Đồ thị** là cấu trúc toán học mô hình hóa **quan hệ giữa các đối tượng**. Từ mạng xã hội, bản đồ giao thông, mạng máy tính, đến dependency graph trong phần mềm — đồ thị có mặt ở khắp nơi.

Phần 4 của sách trình bày 13 chủ đề:

| § | Chủ đề | Mức độ chi tiết |
|---|--------|:---:|
| 1 | Các khái niệm cơ bản | Tóm tắt |
| 2 | Biểu diễn đồ thị | Tóm tắt |
| **3** | **DFS và BFS** | **★★★ Chi tiết đầy đủ** |
| 4 | Tính liên thông | Trung bình |
| 5 | Đường đi Euler / Hamilton | Trung bình |
| 6 | Sắp xếp Tô-pô (Topological Sort) | Trung bình |
| 7 | Cây khung nhỏ nhất (MST) | Trung bình |
| **8** | **Đường đi ngắn nhất** | **★★★ Chi tiết đầy đủ** |
| 9 | Luồng cực đại (Max Flow) | Trung bình |
| 10 | Cặp ghép (Matching) | Tóm tắt |
| 11 | Bài toán trên đồ thị đặc biệt | Tóm tắt |
| 12 | Đồ thị phẳng | Tóm tắt |
| 13 | Bổ sung: Union-Find | Trung bình |

---

## Mục lục

1. [§1 — Các khái niệm cơ bản](#1-các-khái-niệm-cơ-bản)
2. [§2 — Biểu diễn đồ thị](#2-biểu-diễn-đồ-thị)
3. [§3 — DFS và BFS ★★★](#3-dfs-và-bfs)
4. [§4 — Tính liên thông](#4-tính-liên-thông)
5. [§5 — Đường đi Euler / Hamilton](#5-đường-đi-euler--hamilton)
6. [§6 — Sắp xếp Tô-pô](#6-sắp-xếp-tô-pô)
7. [§7 — Cây khung nhỏ nhất (MST)](#7-cây-khung-nhỏ-nhất-mst)
8. [§8 — Đường đi ngắn nhất ★★★](#8-đường-đi-ngắn-nhất)
9. [§9 — Luồng cực đại](#9-luồng-cực-đại)
10. [§10 — Cặp ghép](#10-cặp-ghép)
11. [§13 — Union-Find (Disjoint Set Union)](#13-union-find-disjoint-set-union)

---

## §1 — Các khái niệm cơ bản

### Định nghĩa

**Đồ thị** G = (V, E) gồm:
- **V** (Vertices): tập đỉnh.
- **E** (Edges): tập cạnh, mỗi cạnh nối hai đỉnh.

### Phân loại

| Loại | Đặc điểm | Ký hiệu cạnh |
|------|----------|:---:|
| **Vô hướng (Undirected)** | Cạnh không có chiều | {u, v} |
| **Có hướng (Directed)** | Cạnh có chiều | (u → v) |
| **Có trọng số (Weighted)** | Mỗi cạnh có giá trị chi phí/khoảng cách | w(u, v) |
| **Không trọng số** | Mọi cạnh "ngang bằng" | — |

### Thuật ngữ quan trọng

| Thuật ngữ | Định nghĩa |
|----------|------------|
| **Bậc (Degree)** | Số cạnh kề với đỉnh. Đồ thị có hướng: in-degree + out-degree |
| **Đường đi (Path)** | Dãy đỉnh v₁, v₂, …, vₖ sao cho (vᵢ, vᵢ₊₁) ∈ E |
| **Đường đi đơn** | Không qua đỉnh nào hai lần |
| **Chu trình (Cycle)** | Đường đi có v₁ = vₖ |
| **Liên thông (Connected)** | Mọi cặp đỉnh đều có đường đi nối nhau (vô hướng) |
| **Liên thông mạnh** | Mọi cặp đỉnh đều có đường đi theo cả hai chiều (có hướng) |
| **DAG** | Directed Acyclic Graph — đồ thị có hướng không chu trình |
| **Cây (Tree)** | Đồ thị liên thông, vô hướng, không chu trình, |V|−1 cạnh |
| **Đồ thị hai phía (Bipartite)** | V chia thành 2 tập, cạnh chỉ nối đỉnh khác tập |
| **Đồ thị đầy đủ** | Mọi cặp đỉnh đều có cạnh — C(n,2) cạnh |
| **Đồ thị thưa (Sparse)** | |E| ≈ O(|V|) |
| **Đồ thị dày (Dense)** | |E| ≈ O(|V|²) |

### Một số hệ thức

- Đồ thị vô hướng: $\sum_{v \in V} \deg(v) = 2|E|$
- Cây: $|E| = |V| - 1$
- Đồ thị đầy đủ n đỉnh: $|E| = \frac{n(n-1)}{2}$

---

## §2 — Biểu diễn đồ thị

### Ba cách biểu diễn

#### 2.1. Ma trận kề (Adjacency Matrix)

Mảng 2D `adj[u][v]` = trọng số cạnh (u, v), hoặc 0/false nếu không có cạnh.

```go
// Đồ thị n đỉnh, có trọng số
type GraphMatrix struct {
    n   int
    adj [][]int // adj[u][v] = weight, 0 = no edge
}

func NewGraphMatrix(n int) *GraphMatrix {
    adj := make([][]int, n)
    for i := range adj {
        adj[i] = make([]int, n)
    }
    return &GraphMatrix{n: n, adj: adj}
}

func (g *GraphMatrix) AddEdge(u, v, w int) {
    g.adj[u][v] = w
    g.adj[v][u] = w // bỏ dòng này nếu đồ thị có hướng
}
```

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Kiểm tra cạnh O(1) | Bộ nhớ O(V²) — lãng phí với đồ thị thưa |
| Đơn giản | Duyệt đỉnh kề O(V) |
| Tốt cho đồ thị dày | Không hiệu quả với V lớn, E nhỏ |

#### 2.2. Danh sách kề (Adjacency List) — **phổ biến nhất**

Mỗi đỉnh lưu danh sách các đỉnh kề (và trọng số nếu có).

```go
type Edge struct {
    To, Weight int
}

type GraphList struct {
    n   int
    adj [][]Edge
}

func NewGraphList(n int) *GraphList {
    return &GraphList{n: n, adj: make([][]Edge, n)}
}

func (g *GraphList) AddEdge(u, v, w int) {
    g.adj[u] = append(g.adj[u], Edge{v, w})
    g.adj[v] = append(g.adj[v], Edge{u, w}) // bỏ nếu có hướng
}
```

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Bộ nhớ O(V + E) | Kiểm tra cạnh O(degree) |
| Duyệt đỉnh kề O(degree) — hiệu quả | Phức tạp hơn matrix |
| Phù hợp đồ thị thưa (phần lớn bài thực tế) | |

#### 2.3. Danh sách cạnh (Edge List)

Lưu mảng tất cả cạnh. Dùng cho Kruskal, Bellman-Ford.

```go
type EdgeItem struct {
    From, To, Weight int
}

type GraphEdgeList struct {
    n     int
    edges []EdgeItem
}
```

### Khi nào chọn cách nào?

| Tiêu chí | Ma trận kề | Danh sách kề | Danh sách cạnh |
|----------|:---:|:---:|:---:|
| Đồ thị dày (E ≈ V²) | ✅ | ❌ | ❌ |
| Đồ thị thưa (E ≈ V) | ❌ | ✅ | ✅ |
| Kiểm tra cạnh (u,v) | O(1) ✅ | O(deg) | O(E) |
| Duyệt kề | O(V) | O(deg) ✅ | O(E) |
| Floyd-Warshall | ✅ | Chuyển sang matrix | ❌ |
| DFS/BFS | OK | ✅ | Cần chuyển đổi |
| Kruskal | ❌ | ❌ | ✅ |

> **Mặc định**: dùng **danh sách kề** cho hầu hết bài toán. Chỉ dùng ma trận kề khi đồ thị dày hoặc cần Floyd-Warshall.

---

## §3 — DFS và BFS ★★★

> **Đây là hai thuật toán nền tảng nhất của lý thuyết đồ thị.** Hầu hết mọi thuật toán đồ thị khác đều xây dựng trên DFS hoặc BFS.

### 3.1. DFS — Tìm kiếm theo chiều sâu (Depth-First Search)

#### Ý tưởng

Từ đỉnh nguồn, **đi sâu nhất có thể** trước khi quay lui. Giống đi trong mê cung: luôn rẽ trái, cụt thì quay lại.

```
Thăm đỉnh u → Đánh dấu u đã thăm → Với mỗi đỉnh kề v chưa thăm → DFS(v) → Quay lui
```

#### Minh họa

```
Đồ thị:
    0 --- 1 --- 3
    |     |
    2     4

DFS từ 0: 0 → 1 → 3 → (quay lui) → 4 → (quay lui) → (quay lui) → 2
Thứ tự: [0, 1, 3, 4, 2]
```

#### Cài đặt đệ quy (Go)

```go
func DFSRecursive(g *GraphList, start int) []int {
    visited := make([]bool, g.n)
    var order []int

    var dfs func(u int)
    dfs = func(u int) {
        visited[u] = true
        order = append(order, u)
        for _, e := range g.adj[u] {
            if !visited[e.To] {
                dfs(e.To)
            }
        }
    }

    dfs(start)
    return order
}
```

#### Cài đặt dùng Stack (lặp) — tránh stack overflow

```go
func DFSIterative(g *GraphList, start int) []int {
    visited := make([]bool, g.n)
    var order []int
    stack := []int{start}

    for len(stack) > 0 {
        u := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if visited[u] {
            continue
        }
        visited[u] = true
        order = append(order, u)

        // Đẩy đỉnh kề vào stack (ngược thứ tự để duyệt đúng chiều)
        for i := len(g.adj[u]) - 1; i >= 0; i-- {
            v := g.adj[u][i].To
            if !visited[v] {
                stack = append(stack, v)
            }
        }
    }
    return order
}
```

> **Chú ý**: DFS lặp và DFS đệ quy có thể cho thứ tự duyệt **khác nhau** do cách xử lý stack, nhưng cả hai đều đúng về mặt thuật toán.

#### DFS trên lưới (Grid DFS) — dạng rất phổ biến trên LeetCode

```go
// Đếm số hòn đảo (LeetCode 200)
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }
    m, n := len(grid), len(grid[0])
    count := 0

    var dfs func(i, j int)
    dfs = func(i, j int) {
        if i < 0 || i >= m || j < 0 || j >= n || grid[i][j] != '1' {
            return
        }
        grid[i][j] = '0' // đánh dấu đã thăm
        dfs(i+1, j)
        dfs(i-1, j)
        dfs(i, j+1)
        dfs(i, j-1)
    }

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' {
                count++
                dfs(i, j)
            }
        }
    }
    return count
}
```

#### DFS với thời gian vào/ra (Discovery & Finish Time)

Thông tin vào/ra quan trọng cho nhiều thuật toán nâng cao: phát hiện cầu, khớp, thành phần liên thông mạnh.

```go
type DFSInfo struct {
    visited      []bool
    discoveryTime []int
    finishTime   []int
    parent       []int
    timer        int
}

func NewDFSInfo(n int) *DFSInfo {
    parent := make([]int, n)
    for i := range parent {
        parent[i] = -1
    }
    return &DFSInfo{
        visited:       make([]bool, n),
        discoveryTime: make([]int, n),
        finishTime:    make([]int, n),
        parent:        parent,
    }
}

func (info *DFSInfo) DFS(g *GraphList, u int) {
    info.visited[u] = true
    info.timer++
    info.discoveryTime[u] = info.timer

    for _, e := range g.adj[u] {
        v := e.To
        if !info.visited[v] {
            info.parent[v] = u
            info.DFS(g, v)
        }
    }

    info.timer++
    info.finishTime[u] = info.timer
}
```

#### Phân loại cạnh bằng DFS (đồ thị có hướng)

| Loại cạnh | Điều kiện | Ý nghĩa |
|----------|----------|---------|
| **Tree edge** | v chưa thăm | Cạnh thuộc cây DFS |
| **Back edge** | v đã thăm, discovery[v] < discovery[u], chưa finish | **Chứng tỏ có chu trình** |
| **Forward edge** | v đã finish, discovery[u] < discovery[v] | Đi tới hậu duệ (qua đường tắt) |
| **Cross edge** | v đã finish, discovery[u] > discovery[v] | Đi tới nhánh khác |

```go
// Phát hiện chu trình trong đồ thị có hướng
func HasCycleDirected(g *GraphList) bool {
    n := g.n
    // 0: chưa thăm, 1: đang thăm (in stack), 2: đã xong
    color := make([]int, n)

    var dfs func(u int) bool
    dfs = func(u int) bool {
        color[u] = 1 // đang thăm
        for _, e := range g.adj[u] {
            v := e.To
            if color[v] == 1 {
                return true // back edge → chu trình!
            }
            if color[v] == 0 && dfs(v) {
                return true
            }
        }
        color[u] = 2 // đã xong
        return false
    }

    for u := 0; u < n; u++ {
        if color[u] == 0 && dfs(u) {
            return true
        }
    }
    return false
}
```

```go
// Phát hiện chu trình trong đồ thị vô hướng
func HasCycleUndirected(g *GraphList) bool {
    visited := make([]bool, g.n)

    var dfs func(u, parent int) bool
    dfs = func(u, parent int) bool {
        visited[u] = true
        for _, e := range g.adj[u] {
            v := e.To
            if !visited[v] {
                if dfs(v, u) {
                    return true
                }
            } else if v != parent {
                return true // gặp đỉnh đã thăm nhưng không phải cha → chu trình
            }
        }
        return false
    }

    for u := 0; u < g.n; u++ {
        if !visited[u] && dfs(u, -1) {
            return true
        }
    }
    return false
}
```

#### Độ phức tạp DFS

| | Danh sách kề | Ma trận kề |
|---|---|---|
| Thời gian | **O(V + E)** | O(V²) |
| Bộ nhớ | O(V) (visited + stack) | O(V) |

---

### 3.2. BFS — Tìm kiếm theo chiều rộng (Breadth-First Search)

#### Ý tưởng

Từ đỉnh nguồn, **thăm tất cả đỉnh cùng khoảng cách trước**, rồi mới đi xa hơn. Dùng **hàng đợi (Queue)**.

```
Queue = [start]
Khi queue không rỗng:
  u = dequeue
  Với mỗi đỉnh kề v chưa thăm:
    đánh dấu v, enqueue(v), dist[v] = dist[u] + 1
```

#### Minh họa

```
Đồ thị:
    0 --- 1 --- 3
    |     |
    2     4

BFS từ 0:
  Mức 0: [0]
  Mức 1: [1, 2]
  Mức 2: [3, 4]
Thứ tự: [0, 1, 2, 3, 4]
```

#### Cài đặt cơ bản (Go)

```go
func BFS(g *GraphList, start int) ([]int, []int) {
    visited := make([]bool, g.n)
    dist := make([]int, g.n)
    for i := range dist {
        dist[i] = -1
    }
    parent := make([]int, g.n)
    for i := range parent {
        parent[i] = -1
    }

    var order []int
    queue := []int{start}
    visited[start] = true
    dist[start] = 0

    for len(queue) > 0 {
        u := queue[0]
        queue = queue[1:]
        order = append(order, u)

        for _, e := range g.adj[u] {
            v := e.To
            if !visited[v] {
                visited[v] = true
                dist[v] = dist[u] + 1
                parent[v] = u
                queue = append(queue, v)
            }
        }
    }
    return order, dist
}
```

#### Truy vết đường đi ngắn nhất (BFS)

```go
func BFSShortestPath(g *GraphList, start, end int) []int {
    if start == end {
        return []int{start}
    }
    visited := make([]bool, g.n)
    parent := make([]int, g.n)
    for i := range parent {
        parent[i] = -1
    }

    queue := []int{start}
    visited[start] = true

    for len(queue) > 0 {
        u := queue[0]
        queue = queue[1:]

        for _, e := range g.adj[u] {
            v := e.To
            if !visited[v] {
                visited[v] = true
                parent[v] = u
                if v == end {
                    // Truy vết ngược
                    var path []int
                    for cur := end; cur != -1; cur = parent[cur] {
                        path = append(path, cur)
                    }
                    // Đảo ngược
                    for i, j := 0, len(path)-1; i < j; i, j = i+1, j-1 {
                        path[i], path[j] = path[j], path[i]
                    }
                    return path
                }
                queue = append(queue, v)
            }
        }
    }
    return nil // không có đường đi
}
```

#### BFS trên lưới — Tìm đường đi ngắn nhất

```go
// LeetCode 1091 — Shortest Path in Binary Matrix
func shortestPathBinaryMatrix(grid [][]int) int {
    n := len(grid)
    if grid[0][0] == 1 || grid[n-1][n-1] == 1 {
        return -1
    }
    if n == 1 {
        return 1
    }

    dirs := [8][2]int{{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}}
    queue := [][2]int{{0, 0}}
    grid[0][0] = 1 // đánh dấu đã thăm
    steps := 1

    for len(queue) > 0 {
        steps++
        size := len(queue)
        for k := 0; k < size; k++ {
            cell := queue[0]
            queue = queue[1:]
            for _, d := range dirs {
                ni, nj := cell[0]+d[0], cell[1]+d[1]
                if ni >= 0 && ni < n && nj >= 0 && nj < n && grid[ni][nj] == 0 {
                    if ni == n-1 && nj == n-1 {
                        return steps
                    }
                    grid[ni][nj] = 1
                    queue = append(queue, [2]int{ni, nj})
                }
            }
        }
    }
    return -1
}
```

#### BFS đa nguồn (Multi-source BFS)

Khởi tạo queue với **tất cả đỉnh nguồn** cùng lúc. Tính khoảng cách từ **tập nguồn** đến mọi đỉnh.

```go
// LeetCode 542 — 01 Matrix: khoảng cách mỗi ô đến 0 gần nhất
func updateMatrix(mat [][]int) [][]int {
    m, n := len(mat), len(mat[0])
    dist := make([][]int, m)
    queue := [][2]int{}

    for i := range dist {
        dist[i] = make([]int, n)
        for j := range dist[i] {
            if mat[i][j] == 0 {
                dist[i][j] = 0
                queue = append(queue, [2]int{i, j}) // đa nguồn
            } else {
                dist[i][j] = m*n + 1 // sentinel
            }
        }
    }

    dirs := [4][2]int{{-1,0},{1,0},{0,-1},{0,1}}
    for len(queue) > 0 {
        cell := queue[0]
        queue = queue[1:]
        for _, d := range dirs {
            ni, nj := cell[0]+d[0], cell[1]+d[1]
            if ni >= 0 && ni < m && nj >= 0 && nj < n {
                if dist[ni][nj] > dist[cell[0]][cell[1]]+1 {
                    dist[ni][nj] = dist[cell[0]][cell[1]] + 1
                    queue = append(queue, [2]int{ni, nj})
                }
            }
        }
    }
    return dist
}
```

#### 0-1 BFS (Deque BFS) — cạnh trọng số 0 hoặc 1

Khi cạnh có trọng số chỉ là 0 hoặc 1, dùng **deque** thay vì priority queue: cạnh trọng số 0 → đẩy vào **đầu** deque, cạnh trọng số 1 → đẩy vào **cuối**.

```go
func BFS01(g *GraphList, start int) []int {
    dist := make([]int, g.n)
    for i := range dist {
        dist[i] = 1<<63 - 1
    }
    dist[start] = 0

    // Deque bằng doubly-linked list hoặc slice đơn giản
    deque := []int{start}

    for len(deque) > 0 {
        u := deque[0]
        deque = deque[1:]

        for _, e := range g.adj[u] {
            v, w := e.To, e.Weight
            if dist[u]+w < dist[v] {
                dist[v] = dist[u] + w
                if w == 0 {
                    deque = append([]int{v}, deque...) // đầu deque
                } else {
                    deque = append(deque, v) // cuối deque
                }
            }
        }
    }
    return dist
}
```

Phức tạp: **O(V + E)** — nhanh hơn Dijkstra cho trường hợp đặc biệt này.

#### Độ phức tạp BFS

| | Danh sách kề | Ma trận kề |
|---|---|---|
| Thời gian | **O(V + E)** | O(V²) |
| Bộ nhớ | O(V) (visited + queue) | O(V) |

---

### 3.3. So sánh DFS vs BFS

| Tiêu chí | DFS | BFS |
|---------|-----|-----|
| Cấu trúc dữ liệu | **Stack** (hoặc đệ quy) | **Queue** |
| Chiến lược | Đi sâu trước | Đi rộng trước |
| Đường đi ngắn nhất (không trọng số) | ❌ Không đảm bảo | ✅ **Đảm bảo** |
| Phát hiện chu trình | ✅ (back edge) | ✅ |
| Sắp xếp tô-pô | ✅ (finish time ngược) | ✅ (Kahn's — đếm in-degree) |
| Thành phần liên thông | ✅ | ✅ |
| Tìm cầu / khớp | ✅ (Tarjan) | ❌ |
| Thành phần liên thông mạnh | ✅ (Tarjan / Kosaraju) | ❌ |
| Bộ nhớ worst case | O(V) — stack sâu | O(V) — queue rộng |
| Ứng dụng phổ biến nhất | Tô-pô sort, SCC, backtracking | Đường đi ngắn nhất, level by level |

### 3.4. Kiểm tra đồ thị hai phía (Bipartite)

Đồ thị hai phía ↔ **tô được 2 màu** sao cho không có 2 đỉnh kề cùng màu.

```go
// BFS kiểm tra bipartite — LeetCode 785
func isBipartite(graph [][]int) bool {
    n := len(graph)
    color := make([]int, n) // 0: chưa tô, 1/-1: hai màu

    for start := 0; start < n; start++ {
        if color[start] != 0 {
            continue
        }
        color[start] = 1
        queue := []int{start}

        for len(queue) > 0 {
            u := queue[0]
            queue = queue[1:]
            for _, v := range graph[u] {
                if color[v] == 0 {
                    color[v] = -color[u]
                    queue = append(queue, v)
                } else if color[v] == color[u] {
                    return false // hai đỉnh kề cùng màu
                }
            }
        }
    }
    return true
}
```

### 3.5. Lỗi thường gặp — DFS/BFS

- **Quên đánh dấu visited TRƯỚC khi enqueue (BFS)** → cùng đỉnh enqueue nhiều lần → TLE hoặc sai.
- **DFS trên đồ thị lớn dùng đệ quy** → stack overflow. Chuyển sang DFS lặp với stack tường minh.
- **BFS trên đồ thị có trọng số** → kết quả SAI. BFS chỉ đúng khi mọi cạnh trọng số bằng nhau.
- **Quên xử lý đồ thị không liên thông** → chỉ duyệt từ 1 đỉnh, bỏ sót component khác. Luôn lặp qua mọi đỉnh.
- **Kiểm tra chu trình vô hướng**: so `v != parent` không đủ nếu có đa cạnh (multi-edge). Cần so sánh chỉ số cạnh.
- **BFS level-by-level**: quên lưu `size := len(queue)` trước vòng lặp → các mức bị trộn lẫn.

---

## §4 — Tính liên thông

### Thành phần liên thông (Undirected)

Chạy DFS/BFS từ từng đỉnh chưa thăm. Mỗi lần chạy = một thành phần.

```go
func ConnectedComponents(g *GraphList) [][]int {
    visited := make([]bool, g.n)
    var components [][]int

    for u := 0; u < g.n; u++ {
        if !visited[u] {
            var comp []int
            // BFS
            queue := []int{u}
            visited[u] = true
            for len(queue) > 0 {
                v := queue[0]
                queue = queue[1:]
                comp = append(comp, v)
                for _, e := range g.adj[v] {
                    if !visited[e.To] {
                        visited[e.To] = true
                        queue = append(queue, e.To)
                    }
                }
            }
            components = append(components, comp)
        }
    }
    return components
}
```

### Cầu và Khớp (Bridges and Articulation Points) — Tarjan

**Cầu**: cạnh mà xóa nó làm đồ thị mất liên thông.
**Khớp**: đỉnh mà xóa nó (và các cạnh kề) làm đồ thị mất liên thông.

Dùng DFS với mảng `disc[]` (thời gian phát hiện) và `low[]` (đỉnh thấp nhất đạt được qua back edge).

```go
func FindBridges(g *GraphList) [][2]int {
    n := g.n
    disc := make([]int, n)
    low := make([]int, n)
    visited := make([]bool, n)
    timer := 0
    var bridges [][2]int

    var dfs func(u, parent int)
    dfs = func(u, parent int) {
        visited[u] = true
        timer++
        disc[u] = timer
        low[u] = timer

        for _, e := range g.adj[u] {
            v := e.To
            if !visited[v] {
                dfs(v, u)
                if low[v] < low[u] {
                    low[u] = low[v]
                }
                // v không thể đi ngược lên trên u → (u,v) là cầu
                if low[v] > disc[u] {
                    bridges = append(bridges, [2]int{u, v})
                }
            } else if v != parent {
                if disc[v] < low[u] {
                    low[u] = disc[v]
                }
            }
        }
    }

    for u := 0; u < n; u++ {
        if !visited[u] {
            dfs(u, -1)
        }
    }
    return bridges
}
```

### Thành phần liên thông mạnh (SCC) — Kosaraju

Đồ thị **có hướng**: u và v thuộc cùng SCC ↔ có đường u→v VÀ v→u.

```go
func KosarajuSCC(g *GraphList) [][]int {
    n := g.n

    // Bước 1: DFS trên đồ thị gốc, ghi finish order
    visited := make([]bool, n)
    var finishOrder []int
    var dfs1 func(u int)
    dfs1 = func(u int) {
        visited[u] = true
        for _, e := range g.adj[u] {
            if !visited[e.To] {
                dfs1(e.To)
            }
        }
        finishOrder = append(finishOrder, u)
    }
    for u := 0; u < n; u++ {
        if !visited[u] {
            dfs1(u)
        }
    }

    // Bước 2: Xây đồ thị ngược (transpose)
    rev := NewGraphList(n)
    for u := 0; u < n; u++ {
        for _, e := range g.adj[u] {
            rev.adj[e.To] = append(rev.adj[e.To], Edge{u, e.Weight})
        }
    }

    // Bước 3: DFS trên đồ thị ngược theo finish order ngược
    visited = make([]bool, n)
    var sccs [][]int
    var dfs2 func(u int, comp *[]int)
    dfs2 = func(u int, comp *[]int) {
        visited[u] = true
        *comp = append(*comp, u)
        for _, e := range rev.adj[u] {
            if !visited[e.To] {
                dfs2(e.To, comp)
            }
        }
    }
    for i := len(finishOrder) - 1; i >= 0; i-- {
        u := finishOrder[i]
        if !visited[u] {
            var comp []int
            dfs2(u, &comp)
            sccs = append(sccs, comp)
        }
    }
    return sccs
}
```

---

## §5 — Đường đi Euler / Hamilton

### Đường đi / Chu trình Euler

- **Đường đi Euler**: đi qua **mỗi cạnh đúng 1 lần**.
- **Chu trình Euler**: đường đi Euler kết thúc tại đỉnh xuất phát.

**Điều kiện tồn tại** (đồ thị vô hướng liên thông):
- Chu trình Euler ↔ mọi đỉnh có **bậc chẵn**.
- Đường đi Euler ↔ có **đúng 2 đỉnh bậc lẻ** (đó là điểm đầu và cuối).

```go
// Thuật toán Hierholzer — tìm chu trình Euler
// Giả sử đồ thị thỏa điều kiện tồn tại
func EulerCircuit(n int, edges [][2]int) []int {
    adj := make([][]struct{ to, idx int }, n)
    used := make([]bool, len(edges))

    for i, e := range edges {
        adj[e[0]] = append(adj[e[0]], struct{ to, idx int }{e[1], i})
        adj[e[1]] = append(adj[e[1]], struct{ to, idx int }{e[0], i})
    }

    ptr := make([]int, n) // con trỏ duyệt cho mỗi đỉnh
    var circuit []int
    stack := []int{0}

    for len(stack) > 0 {
        u := stack[len(stack)-1]
        found := false
        for ptr[u] < len(adj[u]) {
            e := adj[u][ptr[u]]
            ptr[u]++
            if !used[e.idx] {
                used[e.idx] = true
                stack = append(stack, e.to)
                found = true
                break
            }
        }
        if !found {
            circuit = append(circuit, u)
            stack = stack[:len(stack)-1]
        }
    }
    return circuit
}
```

### Đường đi Hamilton

- Đi qua **mỗi đỉnh đúng 1 lần**.
- **Không có điều kiện cần và đủ đơn giản** — bài toán NP-complete.
- Giải bằng quay lui hoặc bitmask DP: `dp[mask][i]` = có đường đi thăm tập đỉnh `mask` kết thúc tại `i`.

---

## §6 — Sắp xếp Tô-pô

### Ý tưởng

Sắp xếp đỉnh của **DAG** sao cho với mỗi cạnh u→v, u đứng trước v.

### Cách 1: DFS + Finish order ngược

```go
func TopologicalSortDFS(g *GraphList) []int {
    visited := make([]bool, g.n)
    var result []int

    var dfs func(u int)
    dfs = func(u int) {
        visited[u] = true
        for _, e := range g.adj[u] {
            if !visited[e.To] {
                dfs(e.To)
            }
        }
        result = append(result, u)
    }

    for u := 0; u < g.n; u++ {
        if !visited[u] {
            dfs(u)
        }
    }

    // Đảo ngược
    for i, j := 0, len(result)-1; i < j; i, j = i+1, j-1 {
        result[i], result[j] = result[j], result[i]
    }
    return result
}
```

### Cách 2: BFS dùng In-degree (Thuật toán Kahn)

```go
func TopologicalSortBFS(g *GraphList) ([]int, bool) {
    inDeg := make([]int, g.n)
    for u := 0; u < g.n; u++ {
        for _, e := range g.adj[u] {
            inDeg[e.To]++
        }
    }

    var queue []int
    for u := 0; u < g.n; u++ {
        if inDeg[u] == 0 {
            queue = append(queue, u)
        }
    }

    var order []int
    for len(queue) > 0 {
        u := queue[0]
        queue = queue[1:]
        order = append(order, u)

        for _, e := range g.adj[u] {
            inDeg[e.To]--
            if inDeg[e.To] == 0 {
                queue = append(queue, e.To)
            }
        }
    }

    if len(order) != g.n {
        return nil, false // có chu trình → không topo sort được
    }
    return order, true
}
```

> **Kahn's cũng phát hiện chu trình**: nếu `len(order) < n` → có chu trình.

---

## §7 — Cây khung nhỏ nhất (MST)

### Bài toán

Cho đồ thị **vô hướng, liên thông, có trọng số**. Tìm tập cạnh tạo thành **cây** (n−1 cạnh, liên thông, không chu trình) có **tổng trọng số nhỏ nhất**.

### Kruskal — O(E log E)

Sắp cạnh theo trọng số tăng dần. Duyệt từng cạnh, thêm vào MST nếu không tạo chu trình (kiểm tra bằng **Union-Find**).

```go
func Kruskal(n int, edges []EdgeItem) (int, []EdgeItem) {
    // Sắp xếp cạnh theo trọng số
    sort.Slice(edges, func(i, j int) bool {
        return edges[i].Weight < edges[j].Weight
    })

    uf := NewUnionFind(n)
    totalWeight := 0
    var mst []EdgeItem

    for _, e := range edges {
        if uf.Find(e.From) != uf.Find(e.To) {
            uf.Union(e.From, e.To)
            totalWeight += e.Weight
            mst = append(mst, e)
            if len(mst) == n-1 {
                break
            }
        }
    }
    return totalWeight, mst
}
```

### Prim — O(E log V) với Priority Queue

Bắt đầu từ 1 đỉnh, liên tục thêm cạnh rẻ nhất nối **đỉnh đã chọn** với **đỉnh chưa chọn**.

```go
import "container/heap"

type PrimEdge struct {
    to, weight int
}
type PrimHeap []PrimEdge

func (h PrimHeap) Len() int           { return len(h) }
func (h PrimHeap) Less(i, j int) bool { return h[i].weight < h[j].weight }
func (h PrimHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PrimHeap) Push(x any)        { *h = append(*h, x.(PrimEdge)) }
func (h *PrimHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func Prim(g *GraphList) int {
    inMST := make([]bool, g.n)
    h := &PrimHeap{{0, 0}} // bắt đầu từ đỉnh 0
    heap.Init(h)
    totalWeight := 0
    count := 0

    for h.Len() > 0 && count < g.n {
        e := heap.Pop(h).(PrimEdge)
        if inMST[e.to] {
            continue
        }
        inMST[e.to] = true
        totalWeight += e.weight
        count++

        for _, ne := range g.adj[e.to] {
            if !inMST[ne.To] {
                heap.Push(h, PrimEdge{ne.To, ne.Weight})
            }
        }
    }
    return totalWeight
}
```

### So sánh Kruskal vs Prim

| | Kruskal | Prim |
|---|---|---|
| Phức tạp | O(E log E) | O(E log V) |
| Cấu trúc dữ liệu | Union-Find | Priority Queue |
| Phù hợp | Đồ thị **thưa** | Đồ thị **dày** |
| Biểu diễn | Danh sách cạnh | Danh sách kề |

---

## §8 — Đường đi ngắn nhất ★★★

> **Đây là nhóm thuật toán quan trọng nhất trong lý thuyết đồ thị.** Ba thuật toán kinh điển: Dijkstra, Bellman-Ford, Floyd-Warshall.

### Tổng quan

| Thuật toán | Nguồn | Trọng số âm | Phức tạp | Phát hiện chu trình âm |
|-----------|-------|:---:|----------|:---:|
| **BFS** | Đơn nguồn | ❌ | O(V + E) | ❌ |
| **Dijkstra** | Đơn nguồn | ❌ | O((V+E) log V) | ❌ |
| **Bellman-Ford** | Đơn nguồn | ✅ | O(V × E) | ✅ |
| **SPFA** | Đơn nguồn | ✅ | O(V × E) worst | ✅ |
| **Floyd-Warshall** | Mọi cặp | ✅ | O(V³) | ✅ |
| **0-1 BFS** | Đơn nguồn | ❌ (0/1) | O(V + E) | ❌ |

> BFS chỉ đúng khi **mọi cạnh trọng số bằng nhau** (hoặc không trọng số).

---

### 8.1. Dijkstra ★★★

#### Ý tưởng

Duy trì mảng `dist[v]` = khoảng cách ngắn nhất **hiện biết** từ nguồn đến v. Luôn chọn đỉnh **chưa xử lý** có `dist` nhỏ nhất, cập nhật các đỉnh kề (**relaxation**).

Đây là thuật toán **tham lam**: xử lý đỉnh gần nhất trước, vì với trọng số **không âm**, đỉnh đó đã có khoảng cách chính xác.

#### Thuật toán chi tiết

```
1. dist[source] = 0, dist[v] = ∞ ∀v ≠ source
2. Priority queue PQ = {(0, source)}
3. Khi PQ không rỗng:
   a. (d, u) = extract-min từ PQ
   b. Nếu d > dist[u] → bỏ qua (phiên bản cũ)
   c. Với mỗi cạnh (u, v, w):
      Nếu dist[u] + w < dist[v]:
        dist[v] = dist[u] + w
        parent[v] = u
        push (dist[v], v) vào PQ
```

#### Ví dụ truy vết

```
Đồ thị:
    0 --1-- 1 --3-- 3
    |       |       |
    4       2       1
    |       |       |
    2 --5-- 4 ------+

Dijkstra từ 0:

Bước  PQ extract   dist cập nhật
1     (0, 0)       dist[1]=1, dist[2]=4
2     (1, 1)       dist[3]=4, dist[4]=3
3     (3, 4)       dist[3]=min(4,4)=4 (không cải thiện)
4     (4, 2)       (không cải thiện gì)
5     (4, 3)       (xong)

Kết quả: dist = [0, 1, 4, 4, 3]
Đường: 0→3: 0 → 1 → 3 (cost 4)
       0→4: 0 → 1 → 4 (cost 3)
```

#### Cài đặt với Min-Heap (Go)

```go
import "container/heap"

type DijkItem struct {
    node, dist int
}
type DijkHeap []DijkItem

func (h DijkHeap) Len() int           { return len(h) }
func (h DijkHeap) Less(i, j int) bool { return h[i].dist < h[j].dist }
func (h DijkHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *DijkHeap) Push(x any)        { *h = append(*h, x.(DijkItem)) }
func (h *DijkHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func Dijkstra(g *GraphList, source int) ([]int, []int) {
    n := g.n
    const INF = 1<<63 - 1
    dist := make([]int, n)
    parent := make([]int, n)
    for i := range dist {
        dist[i] = INF
        parent[i] = -1
    }
    dist[source] = 0

    h := &DijkHeap{{source, 0}}
    heap.Init(h)

    for h.Len() > 0 {
        item := heap.Pop(h).(DijkItem)
        u, d := item.node, item.dist

        if d > dist[u] {
            continue // phiên bản cũ, bỏ qua
        }

        for _, e := range g.adj[u] {
            v, w := e.To, e.Weight
            if dist[u]+w < dist[v] {
                dist[v] = dist[u] + w
                parent[v] = u
                heap.Push(h, DijkItem{v, dist[v]})
            }
        }
    }
    return dist, parent
}

// Truy vết đường đi
func ReconstructPath(parent []int, target int) []int {
    var path []int
    for v := target; v != -1; v = parent[v] {
        path = append(path, v)
    }
    for i, j := 0, len(path)-1; i < j; i, j = i+1, j-1 {
        path[i], path[j] = path[j], path[i]
    }
    return path
}
```

#### LeetCode — Network Delay Time

```go
// LeetCode 743 — Network Delay Time
func networkDelayTime(times [][]int, n int, k int) int {
    g := NewGraphList(n + 1) // 1-indexed
    for _, t := range times {
        g.adj[t[0]] = append(g.adj[t[0]], Edge{t[1], t[2]})
    }

    dist, _ := Dijkstra(g, k)

    maxDist := 0
    for v := 1; v <= n; v++ {
        if dist[v] == 1<<63-1 {
            return -1 // không đến được
        }
        if dist[v] > maxDist {
            maxDist = dist[v]
        }
    }
    return maxDist
}
```

#### Dijkstra đơn giản — không cần heap (O(V²))

Phù hợp cho đồ thị **dày** (E ≈ V²) hoặc khi không muốn implement heap.

```go
func DijkstraSimple(g *GraphList, source int) []int {
    n := g.n
    const INF = 1<<63 - 1
    dist := make([]int, n)
    visited := make([]bool, n)
    for i := range dist {
        dist[i] = INF
    }
    dist[source] = 0

    for iter := 0; iter < n; iter++ {
        // Tìm đỉnh chưa xử lý có dist nhỏ nhất
        u := -1
        for v := 0; v < n; v++ {
            if !visited[v] && (u == -1 || dist[v] < dist[u]) {
                u = v
            }
        }
        if u == -1 || dist[u] == INF {
            break
        }
        visited[u] = true

        // Relaxation
        for _, e := range g.adj[u] {
            v, w := e.To, e.Weight
            if dist[u]+w < dist[v] {
                dist[v] = dist[u] + w
            }
        }
    }
    return dist
}
```

#### Tại sao Dijkstra KHÔNG hoạt động với trọng số âm?

```
A --1--> B --(-5)--> C
A --2--> C

Dijkstra xử lý A, thấy dist[C] = 2, dist[B] = 1.
Xử lý B, thấy dist[C] = 1 + (-5) = -4 < 2.
Nhưng C đã bị xử lý (đánh dấu visited) → KHÔNG cập nhật lại.
Kết quả sai: dist[C] = 2 thay vì -4.
```

> Nếu có trọng số âm → dùng **Bellman-Ford**.

#### Độ phức tạp Dijkstra

| Biến thể | Phức tạp |
|----------|---------|
| Mảng đơn giản (O(V²)) | O(V²) |
| Binary Heap | O((V + E) log V) |
| Fibonacci Heap | O(V log V + E) — lý thuyết |

> **Thực tế**: Binary Heap là lựa chọn tốt nhất cho hầu hết bài toán.

---

### 8.2. Bellman-Ford ★★★

#### Ý tưởng

Lặp V−1 lần, mỗi lần duyệt **tất cả cạnh** và thực hiện relaxation. Sau V−1 lần, mọi đường đi ngắn nhất (nếu tồn tại) đã được tìm.

**Tại sao V−1 lần?** Đường đi ngắn nhất (không chu trình âm) có tối đa V−1 cạnh. Mỗi lần lặp, ít nhất 1 cạnh mới trên đường đi ngắn nhất được xác định đúng.

#### Thuật toán chi tiết

```
1. dist[source] = 0, dist[v] = ∞ ∀v ≠ source
2. Lặp V-1 lần:
   Với MỌI cạnh (u, v, w):
     Nếu dist[u] + w < dist[v]:
       dist[v] = dist[u] + w
       parent[v] = u
3. Kiểm tra chu trình âm:
   Duyệt tất cả cạnh một lần nữa.
   Nếu còn relaxation được → CÓ chu trình âm.
```

#### Cài đặt (Go)

```go
func BellmanFord(n int, edges []EdgeItem, source int) ([]int, []int, bool) {
    const INF = 1<<63 - 1
    dist := make([]int, n)
    parent := make([]int, n)
    for i := range dist {
        dist[i] = INF
        parent[i] = -1
    }
    dist[source] = 0

    // V-1 lần relaxation
    for iter := 0; iter < n-1; iter++ {
        updated := false
        for _, e := range edges {
            if dist[e.From] != INF && dist[e.From]+e.Weight < dist[e.To] {
                dist[e.To] = dist[e.From] + e.Weight
                parent[e.To] = e.From
                updated = true
            }
        }
        if !updated {
            break // tối ưu sớm — không có cải thiện
        }
    }

    // Kiểm tra chu trình âm
    hasNegCycle := false
    for _, e := range edges {
        if dist[e.From] != INF && dist[e.From]+e.Weight < dist[e.To] {
            hasNegCycle = true
            break
        }
    }

    return dist, parent, hasNegCycle
}
```

#### Ví dụ truy vết

```
Đồ thị (có hướng):
  0 --6--> 1
  0 --7--> 2
  1 --5--> 2
  1 --(-4)--> 3
  2 --(-2)--> 1
  3 --7--> 2

Edges: (0,1,6), (0,2,7), (1,2,5), (1,3,-4), (2,1,-2), (3,2,7)

Lần 1: dist = [0, 6, 7, ∞, ...] → relaxation → dist = [0, 6, 7, 2]
Lần 2: cạnh (2,1,-2): dist[1] = min(6, 7+(-2)) = 5
        cạnh (1,3,-4): dist[3] = min(2, 5+(-4)) = 1
Lần 3: cạnh (2,1,-2): dist[1] = min(5, 7+(-2)) = 5 (không đổi)
        → ổn định

Kết quả: dist = [0, 5, 7, 1]
```

#### SPFA — Tối ưu Bellman-Ford bằng Queue

Chỉ relaxation từ đỉnh **vừa được cập nhật**. Trung bình nhanh hơn Bellman-Ford, nhưng worst-case vẫn O(VE).

```go
func SPFA(g *GraphList, source int) ([]int, bool) {
    n := g.n
    const INF = 1<<63 - 1
    dist := make([]int, n)
    inQueue := make([]bool, n)
    count := make([]int, n) // số lần vào queue
    for i := range dist {
        dist[i] = INF
    }
    dist[source] = 0
    inQueue[source] = true
    count[source] = 1
    queue := []int{source}

    for len(queue) > 0 {
        u := queue[0]
        queue = queue[1:]
        inQueue[u] = false

        for _, e := range g.adj[u] {
            v, w := e.To, e.Weight
            if dist[u]+w < dist[v] {
                dist[v] = dist[u] + w
                if !inQueue[v] {
                    inQueue[v] = true
                    count[v]++
                    if count[v] >= n {
                        return nil, true // chu trình âm
                    }
                    queue = append(queue, v)
                }
            }
        }
    }
    return dist, false
}
```

#### So sánh Dijkstra vs Bellman-Ford

| | Dijkstra | Bellman-Ford |
|---|---|---|
| Phức tạp | O((V+E) log V) | O(V × E) |
| Trọng số âm | ❌ **Không đúng** | ✅ Đúng |
| Chu trình âm | Không phát hiện | ✅ Phát hiện |
| Khi nào dùng | Trọng số ≥ 0 (phần lớn bài) | Có trọng số âm, hoặc cần phát hiện chu trình âm |
| Thực tế | Nhanh hơn nhiều | Chậm hơn |

---

### 8.3. Floyd-Warshall — Đường đi ngắn nhất mọi cặp ★★★

#### Ý tưởng

Tính đường đi ngắn nhất giữa **mọi cặp đỉnh** cùng lúc. Ý tưởng: thử từng đỉnh `k` làm "trung gian" — nếu đi qua k ngắn hơn, cập nhật.

$$dist[i][j] = \min(dist[i][j], \; dist[i][k] + dist[k][j])$$

Duyệt **k ở vòng ngoài cùng**.

#### Thuật toán chi tiết

```
1. dist[i][j] = weight(i,j) nếu có cạnh, ∞ nếu không, 0 nếu i=j
2. Với k = 0 → V-1:           ← VÒNG NGOÀI là k
     Với i = 0 → V-1:
       Với j = 0 → V-1:
         dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])
3. Nếu dist[i][i] < 0 → có chu trình âm qua i
```

#### Cài đặt (Go)

```go
func FloydWarshall(n int, edges []EdgeItem) ([][]int, [][]int) {
    const INF = 1<<62 - 1 // dùng 1<<62 để tránh tràn khi cộng
    dist := make([][]int, n)
    next := make([][]int, n) // để truy vết đường đi

    for i := range dist {
        dist[i] = make([]int, n)
        next[i] = make([]int, n)
        for j := range dist[i] {
            if i == j {
                dist[i][j] = 0
            } else {
                dist[i][j] = INF
            }
            next[i][j] = -1
        }
    }

    for _, e := range edges {
        if e.Weight < dist[e.From][e.To] {
            dist[e.From][e.To] = e.Weight
            next[e.From][e.To] = e.To
        }
    }

    // Floyd-Warshall — K Ở VÒNG NGOÀI CÙNG
    for k := 0; k < n; k++ {
        for i := 0; i < n; i++ {
            for j := 0; j < n; j++ {
                if dist[i][k] != INF && dist[k][j] != INF {
                    if dist[i][k]+dist[k][j] < dist[i][j] {
                        dist[i][j] = dist[i][k] + dist[k][j]
                        next[i][j] = next[i][k] // đường đi qua k
                    }
                }
            }
        }
    }
    return dist, next
}

// Truy vết đường đi từ u đến v
func FloydPath(next [][]int, u, v int) []int {
    if next[u][v] == -1 {
        return nil // không có đường
    }
    path := []int{u}
    for u != v {
        u = next[u][v]
        path = append(path, u)
    }
    return path
}
```

#### Ví dụ truy vết

```
4 đỉnh, cạnh:
  0→1: 3,  0→3: 7
  1→0: 8,  1→2: 2
  2→0: 5,  2→3: 1
  3→0: 2

Ma trận ban đầu:
      0    1    2    3
  0 [ 0    3    ∞    7  ]
  1 [ 8    0    2    ∞  ]
  2 [ 5    ∞    0    1  ]
  3 [ 2    ∞    ∞    0  ]

Sau Floyd-Warshall:
      0    1    2    3
  0 [ 0    3    5    6  ]
  1 [ 5    0    2    3  ]
  2 [ 3    6    0    1  ]
  3 [ 2    5    7    0  ]

Đường 1→3: 1→2→3, cost = 3
Đường 3→1: 3→0→1, cost = 5
```

#### Phát hiện chu trình âm

```go
func HasNegCycle(dist [][]int, n int) bool {
    for i := 0; i < n; i++ {
        if dist[i][i] < 0 {
            return true
        }
    }
    return false
}
```

#### Khi nào dùng Floyd-Warshall?

| Tình huống | Dùng? |
|-----------|:---:|
| Cần khoảng cách **mọi cặp** | ✅ |
| V nhỏ (≤ 400–500) | ✅ |
| Đồ thị **dày** | ✅ |
| V lớn, chỉ cần đơn nguồn | ❌ → Dijkstra |
| Có trọng số âm, cần mọi cặp | ✅ |

---

### 8.4. So sánh tổng hợp

| | BFS | Dijkstra | Bellman-Ford | Floyd |
|---|---|---|---|---|
| Loại | Đơn nguồn | Đơn nguồn | Đơn nguồn | Mọi cặp |
| Trọng số | Bằng nhau | ≥ 0 | Bất kỳ | Bất kỳ |
| Phức tạp | O(V+E) | O((V+E)log V) | O(VE) | O(V³) |
| Chu trình âm | — | — | Phát hiện | Phát hiện |
| Cấu trúc DL | Queue | Priority Queue | Mảng cạnh | Ma trận |

#### Chọn thuật toán nào?

```
Đường đi ngắn nhất?
├── Tất cả cạnh trọng số bằng nhau?
│   └── BFS
├── Cần mọi cặp + V ≤ 500?
│   └── Floyd-Warshall
├── Có trọng số âm?
│   ├── Đơn nguồn → Bellman-Ford / SPFA
│   └── Mọi cặp → Floyd-Warshall
└── Trọng số ≥ 0, đơn nguồn?
    └── Dijkstra ★ (lựa chọn mặc định)
```

### 8.5. Lỗi thường gặp — Đường đi ngắn nhất

- **Dùng Dijkstra với trọng số âm** → kết quả sai. Luôn kiểm tra.
- **Floyd: k ở vòng TRONG thay vì NGOÀI** → sai hoàn toàn. `k` PHẢI ở vòng ngoài cùng.
- **Tràn số**: `dist[u] + w` có thể tràn nếu dist[u] = INT_MAX. Kiểm tra `dist[u] != INF` trước.
- **Dijkstra: không bỏ qua phiên bản cũ** → xử lý đỉnh nhiều lần → TLE.
- **Bellman-Ford: quên tối ưu sớm** → luôn chạy V-1 lần dù đã hội tụ.
- **BFS cho đồ thị có trọng số** → kết quả sai. BFS chỉ đúng khi trọng số đồng nhất.

---

## §9 — Luồng cực đại (Max Flow)

### Bài toán

Cho mạng (đồ thị có hướng, có trọng số = **dung lượng**), nguồn `s`, đích `t`. Tìm **lượng lớn nhất** có thể chuyển từ s đến t, mỗi cạnh không vượt quá dung lượng.

### Ford-Fulkerson (ý tưởng)

```
Khi còn đường tăng luồng (augmenting path) từ s đến t trong đồ thị thặng dư:
  Tìm đường tăng (BFS → Edmonds-Karp)
  Tăng luồng dọc đường bằng bottleneck (cạnh nhỏ nhất)
  Cập nhật đồ thị thặng dư (giảm cạnh xuôi, tăng cạnh ngược)
```

### Edmonds-Karp — O(VE²) — BFS tìm đường tăng

```go
func EdmondsKarp(capacity [][]int, source, sink int) int {
    n := len(capacity)
    residual := make([][]int, n)
    for i := range residual {
        residual[i] = make([]int, n)
        copy(residual[i], capacity[i])
    }

    maxFlow := 0

    for {
        // BFS tìm đường tăng
        parent := make([]int, n)
        for i := range parent { parent[i] = -1 }
        parent[source] = source
        queue := []int{source}

        for len(queue) > 0 && parent[sink] == -1 {
            u := queue[0]
            queue = queue[1:]
            for v := 0; v < n; v++ {
                if parent[v] == -1 && residual[u][v] > 0 {
                    parent[v] = u
                    queue = append(queue, v)
                }
            }
        }

        if parent[sink] == -1 {
            break // không còn đường tăng
        }

        // Tìm bottleneck
        pathFlow := 1<<63 - 1
        for v := sink; v != source; v = parent[v] {
            u := parent[v]
            if residual[u][v] < pathFlow {
                pathFlow = residual[u][v]
            }
        }

        // Cập nhật đồ thị thặng dư
        for v := sink; v != source; v = parent[v] {
            u := parent[v]
            residual[u][v] -= pathFlow
            residual[v][u] += pathFlow
        }
        maxFlow += pathFlow
    }
    return maxFlow
}
```

### Định lý Max-Flow Min-Cut

$$\text{Luồng cực đại} = \text{Lát cắt cực tiểu}$$

Lát cắt (S, T) với S chứa source, T chứa sink. Tổng dung lượng cạnh từ S sang T = luồng cực đại.

---

## §10 — Cặp ghép (Matching)

### Bài toán cặp ghép cực đại trong đồ thị hai phía

Tìm số cặp đỉnh lớn nhất sao cho mỗi đỉnh chỉ thuộc tối đa 1 cặp.

### Hungarian Algorithm / Hopcroft-Karp

Bản đơn giản dùng DFS tìm đường mở (augmenting path):

```go
// Cặp ghép cực đại — đồ thị hai phía
// left: 0..n-1, right: 0..m-1
func MaxMatching(n, m int, adj [][]int) int {
    matchR := make([]int, m)
    for i := range matchR { matchR[i] = -1 }

    var dfs func(u int, visited []bool) bool
    dfs = func(u int, visited []bool) bool {
        for _, v := range adj[u] {
            if !visited[v] {
                visited[v] = true
                if matchR[v] == -1 || dfs(matchR[v], visited) {
                    matchR[v] = u
                    return true
                }
            }
        }
        return false
    }

    result := 0
    for u := 0; u < n; u++ {
        visited := make([]bool, m)
        if dfs(u, visited) {
            result++
        }
    }
    return result
}
```

---

## §13 — Union-Find (Disjoint Set Union)

### Ý tưởng

Quản lý tập hợp các nhóm phần tử. Hỗ trợ hai phép toán:
- **Find(x)**: tìm đại diện (root) của nhóm chứa x.
- **Union(x, y)**: gộp hai nhóm chứa x và y.

Tối ưu: **path compression** + **union by rank** → gần O(1) mỗi thao tác (amortized O(α(n))).

```go
type UnionFind struct {
    parent, rank []int
}

func NewUnionFind(n int) *UnionFind {
    parent := make([]int, n)
    rank := make([]int, n)
    for i := range parent {
        parent[i] = i
    }
    return &UnionFind{parent, rank}
}

func (uf *UnionFind) Find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.Find(uf.parent[x]) // path compression
    }
    return uf.parent[x]
}

func (uf *UnionFind) Union(x, y int) bool {
    rx, ry := uf.Find(x), uf.Find(y)
    if rx == ry {
        return false // cùng nhóm rồi
    }
    // Union by rank
    if uf.rank[rx] < uf.rank[ry] {
        rx, ry = ry, rx
    }
    uf.parent[ry] = rx
    if uf.rank[rx] == uf.rank[ry] {
        uf.rank[rx]++
    }
    return true
}

func (uf *UnionFind) Connected(x, y int) bool {
    return uf.Find(x) == uf.Find(y)
}
```

### Ứng dụng Union-Find

| Bài toán | Cách dùng |
|---------|----------|
| Đếm thành phần liên thông | Union mỗi cạnh, đếm root phân biệt |
| Kruskal MST | Kiểm tra chu trình khi thêm cạnh |
| Phát hiện chu trình (vô hướng) | Nếu Find(u) == Find(v) trước Union → chu trình |
| Kết nối động | Union online, trả lời truy vấn kết nối |

---

## Bài tập (LeetCode)

### DFS / BFS cơ bản

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Number of Islands | DFS/BFS lưới | [200](https://leetcode.com/problems/number-of-islands/) |
| Max Area of Island | DFS lưới | [695](https://leetcode.com/problems/max-area-of-island/) |
| Clone Graph | DFS/BFS + HashMap | [133](https://leetcode.com/problems/clone-graph/) |
| Pacific Atlantic Water Flow | DFS/BFS đa nguồn | [417](https://leetcode.com/problems/pacific-atlantic-water-flow/) |
| Surrounded Regions | DFS/BFS từ biên | [130](https://leetcode.com/problems/surrounded-regions/) |
| Rotting Oranges | BFS đa nguồn | [994](https://leetcode.com/problems/rotting-oranges/) |
| 01 Matrix | BFS đa nguồn | [542](https://leetcode.com/problems/01-matrix/) |
| Shortest Path in Binary Matrix | BFS lưới | [1091](https://leetcode.com/problems/shortest-path-in-binary-matrix/) |
| Word Ladder | BFS | [127](https://leetcode.com/problems/word-ladder/) |
| Open the Lock | BFS | [752](https://leetcode.com/problems/open-the-lock/) |

### Tô-pô & Phát hiện chu trình

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Course Schedule | Tô-pô / phát hiện chu trình | [207](https://leetcode.com/problems/course-schedule/) |
| Course Schedule II | Tô-pô sort | [210](https://leetcode.com/problems/course-schedule-ii/) |
| Alien Dictionary | Tô-pô sort | [269](https://leetcode.com/problems/alien-dictionary/) |
| Is Graph Bipartite? | BFS/DFS tô 2 màu | [785](https://leetcode.com/problems/is-graph-bipartite/) |

### Đường đi ngắn nhất

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Network Delay Time | Dijkstra | [743](https://leetcode.com/problems/network-delay-time/) |
| Path with Minimum Effort | Dijkstra trên lưới | [1631](https://leetcode.com/problems/path-with-minimum-effort/) |
| Cheapest Flights Within K Stops | Bellman-Ford / BFS biến thể | [787](https://leetcode.com/problems/cheapest-flights-within-k-stops/) |
| Swim in Rising Water | Dijkstra / Binary Search + BFS | [778](https://leetcode.com/problems/swim-in-rising-water/) |
| Find the City | Floyd-Warshall | [1334](https://leetcode.com/problems/find-the-city-with-the-smallest-number-of-neighbors-at-a-threshold-distance/) |
| Shortest Path Visiting All Nodes | BFS + Bitmask | [847](https://leetcode.com/problems/shortest-path-visiting-all-nodes/) |

### MST & Union-Find

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Min Cost to Connect All Points | Kruskal/Prim | [1584](https://leetcode.com/problems/min-cost-to-connect-all-points/) |
| Number of Connected Components | Union-Find | [323](https://leetcode.com/problems/number-of-connected-components-in-an-undirected-graph/) |
| Redundant Connection | Union-Find (tìm cạnh tạo chu trình) | [684](https://leetcode.com/problems/redundant-connection/) |
| Accounts Merge | Union-Find | [721](https://leetcode.com/problems/accounts-merge/) |
| Number of Provinces | DFS/Union-Find | [547](https://leetcode.com/problems/number-of-provinces/) |

### Nâng cao

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Critical Connections | Tarjan (cầu) | [1192](https://leetcode.com/problems/critical-connections-in-a-network/) |
| Reconstruct Itinerary | Euler path (Hierholzer) | [332](https://leetcode.com/problems/reconstruct-itinerary/) |
| All Paths Source→Target | DFS/Backtracking trên DAG | [797](https://leetcode.com/problems/all-paths-from-source-to-target/) |

---

## Pattern liên quan

- [05-stack-queue.md](05-stack-queue.md) — Stack cho DFS, Queue cho BFS.
- [06-tree.md](06-tree.md) — Cây là trường hợp đặc biệt của đồ thị.
- [09-enumeration.md](09-enumeration.md) — Quay lui trên đồ thị (Hamilton, tô màu).
- [11-dynamic-programming.md](11-dynamic-programming.md) — DP trên DAG, bitmask DP (TSP).
- **Tham lam**: Dijkstra, Prim, Kruskal đều là thuật toán tham lam.

---

## Ghi nhớ nhanh

### Biểu diễn
- **Mặc định**: Danh sách kề. Ma trận kề chỉ khi cần Floyd hoặc đồ thị dày.

### DFS (§3)
- Stack / đệ quy. O(V + E).
- Ứng dụng: tô-pô, SCC, cầu/khớp, phát hiện chu trình, backtracking.
- Back edge → chu trình.
- **Luôn** kiểm tra đồ thị không liên thông (lặp qua mọi đỉnh).

### BFS (§3)
- Queue. O(V + E).
- **Đường đi ngắn nhất không trọng số**.
- BFS đa nguồn: khởi tạo queue với tất cả nguồn.
- 0-1 BFS: deque, cạnh 0 → đầu, cạnh 1 → cuối.
- **Đánh dấu visited TRƯỚC khi enqueue**, không phải khi dequeue.

### Đường đi ngắn nhất (§8)
- **Dijkstra**: trọng số ≥ 0, O((V+E) log V). Priority queue. Đầu tiên nghĩ đến.
- **Bellman-Ford**: trọng số bất kỳ, phát hiện chu trình âm, O(VE).
- **Floyd**: mọi cặp, O(V³), V ≤ 500. **k vòng NGOÀI**.
- Tránh tràn số: kiểm tra `dist[u] != INF` trước khi cộng.

### MST
- **Kruskal**: sắp cạnh + Union-Find. Đồ thị thưa.
- **Prim**: Priority queue. Đồ thị dày.

### Union-Find
- Path compression + union by rank → O(α(n)) ≈ O(1).
- Dùng cho: liên thông, Kruskal, phát hiện chu trình vô hướng.

---

## Học tiếp

- [09-enumeration.md](09-enumeration.md) — Phần 1: Bài toán liệt kê
- [10-data-structures-algorithms.md](10-data-structures-algorithms.md) — Phần 2: CTDL & Giải thuật
- [11-dynamic-programming.md](11-dynamic-programming.md) — Phần 3: Quy hoạch động
