# Cấu trúc dữ liệu và Giải thuật

> **Chủ đề**: Cấu trúc dữ liệu & Giải thuật | **Mức độ**: Cơ bản → Trung bình
> **Nguồn**: *Giải thuật và Lập trình* — Lê Minh Hoàng, Phần 2

---

## Tổng quan

Phần 2 của sách trình bày **nền tảng cốt lõi** của khoa học máy tính: cách tổ chức dữ liệu hiệu quả và các giải thuật cơ bản để thao tác trên dữ liệu đó.

Chín mục chính:

| § | Chủ đề | File tham khảo chi tiết |
|---|--------|------------------------|
| 1 | Phương pháp giải bài toán | [01-problem-solving-steps.md](01-problem-solving-steps.md) |
| 2 | Phân tích thời gian thực hiện | [02-algorithm-complexity.md](02-algorithm-complexity.md) |
| 3 | Đệ quy | [03-recursion.md](03-recursion.md) |
| 4 | Danh sách (List) | [04-list-data-structures.md](04-list-data-structures.md) |
| 5 | Ngăn xếp và Hàng đợi | [05-stack-queue.md](05-stack-queue.md) |
| 6 | Cây (Tree) | [06-tree.md](06-tree.md) |
| 7 | Biểu thức — Ký pháp & Tính toán | *(mục này)* |
| 8 | Sắp xếp (Sorting) | [07-sorting.md](07-sorting.md) |
| 9 | Tìm kiếm (Searching) | [08-search.md](08-search.md) |

> Các file 01–08 đã trình bày chi tiết (bằng tiếng Anh) cho từng chủ đề. File này tổng hợp bằng **tiếng Việt** theo cấu trúc của sách, bổ sung **§7 Biểu thức** — chủ đề chưa có file riêng.

---

## Mục lục

1. [§1 — Phương pháp giải bài toán trên máy tính](#1-phương-pháp-giải-bài-toán-trên-máy-tính)
2. [§2 — Phân tích thời gian thực hiện giải thuật](#2-phân-tích-thời-gian-thực-hiện-giải-thuật)
3. [§3 — Đệ quy và giải thuật đệ quy](#3-đệ-quy-và-giải-thuật-đệ-quy)
4. [§4 — Cấu trúc dữ liệu danh sách](#4-cấu-trúc-dữ-liệu-danh-sách)
5. [§5 — Ngăn xếp và Hàng đợi](#5-ngăn-xếp-và-hàng-đợi)
6. [§6 — Cây](#6-cây)
7. [§7 — Biểu thức: Ký pháp và Tính toán](#7-biểu-thức-ký-pháp-và-tính-toán)
8. [§8 — Sắp xếp](#8-sắp-xếp)
9. [§9 — Tìm kiếm](#9-tìm-kiếm)

---

## §1 — Phương pháp giải bài toán trên máy tính

### Ý tưởng chủ đạo

Giải một bài toán tin học là quá trình đi từ **đề bài** đến **chương trình chạy đúng**. Sách đề xuất **6 bước** có hệ thống:

```
Đề bài → Hiểu → Cấu trúc dữ liệu → Giải thuật → Mã hóa → Kiểm thử → Tối ưu
```

### 6 bước chi tiết

| Bước | Nội dung | Câu hỏi cần trả lời |
|------|---------|---------------------|
| 1. Xác định bài toán | Đọc kỹ, xác định Input / Output / Quan hệ | Input là gì? Output là gì? Ràng buộc? |
| 2. Chọn cấu trúc dữ liệu | Biểu diễn dữ liệu phù hợp nhất | Mảng? Danh sách liên kết? Cây? Hash? |
| 3. Tìm giải thuật | Xác định quy trình xử lý từng bước | Brute-force? Chia để trị? Tham lam? DP? |
| 4. Mã hóa | Viết code chính xác, rõ ràng | Code có phản ánh đúng giải thuật? |
| 5. Kiểm thử | Thử với test thường, biên, đặc biệt | Edge case? Empty input? Max input? |
| 6. Tối ưu | Phân tích độ phức tạp, cải tiến | Có cách giảm O(n²) → O(n log n)? |

### Mô hình bài toán

Mọi bài toán đều có dạng:

$$\text{Input} \xrightarrow{\text{Xử lý}} \text{Output}$$

- **Bài toán chính xác**: yêu cầu đáp án đúng tuyệt đối.
- **Bài toán xấp xỉ**: chấp nhận đáp án đủ tốt nếu giải chính xác quá tốn kém.

### Biểu diễn giải thuật

| Cách | Ưu điểm | Nhược điểm |
|------|---------|-----------|
| Ngôn ngữ tự nhiên | Dễ hiểu | Mơ hồ |
| Sơ đồ khối (flowchart) | Trực quan | Phức tạp với bài lớn |
| Mã giả (pseudocode) | Chính xác, không phụ thuộc ngôn ngữ | Cần quy ước |
| Code thực | Chạy được | Phụ thuộc ngôn ngữ |

> **Chi tiết**: xem [01-problem-solving-steps.md](01-problem-solving-steps.md)

---

## §2 — Phân tích thời gian thực hiện giải thuật

### Ý tưởng chủ đạo

Đánh giá giải thuật **không phụ thuộc phần cứng** bằng cách đo **số phép toán** theo kích thước đầu vào `n`. Dùng **ký hiệu Big O** để biểu diễn bậc tăng trưởng.

### Các lớp độ phức tạp chính

| Lớp | Big O | Ví dụ | Giới hạn thực tế |
|-----|-------|-------|------------------|
| Hằng số | O(1) | Truy cập mảng theo chỉ số | Không giới hạn |
| Logarit | O(log n) | Tìm kiếm nhị phân | n ~ 10¹⁸ |
| Tuyến tính | O(n) | Duyệt mảng | n ~ 10⁸ |
| n log n | O(n log n) | Merge sort, Quick sort | n ~ 10⁶–10⁷ |
| Bình phương | O(n²) | Hai vòng lặp lồng nhau | n ~ 10⁴ |
| Lập phương | O(n³) | Nhân ma trận naïve | n ~ 500 |
| Mũ | O(2ⁿ) | Duyệt tập con (brute-force) | n ~ 25 |
| Giai thừa | O(n!) | Duyệt hoán vị | n ~ 12 |

> "Giới hạn thực tế" ước lượng giá trị n lớn nhất để chương trình chạy trong ~1 giây.

### Quy tắc xác định độ phức tạp

| Mẫu code | Độ phức tạp |
|----------|------------|
| Câu lệnh đơn (không vòng lặp) | O(1) |
| Một vòng lặp chạy n lần | O(n) |
| Hai vòng lặp lồng nhau, mỗi vòng n | O(n²) |
| Vòng lặp chia đôi/nhân đôi biến | O(log n) |
| Đệ quy T(n) = 2T(n/2) + O(n) | O(n log n) — Master Theorem |
| Phép nối tiếp O(f) rồi O(g) | O(max(f, g)) |
| Phép lồng O(f) chứa O(g) | O(f × g) |

### Master Theorem (rút gọn)

Cho đệ quy: $T(n) = aT(n/b) + O(n^d)$

| Điều kiện | Kết quả |
|-----------|---------|
| $d > \log_b a$ | $T(n) = O(n^d)$ |
| $d = \log_b a$ | $T(n) = O(n^d \log n)$ |
| $d < \log_b a$ | $T(n) = O(n^{\log_b a})$ |

> **Chi tiết**: xem [02-algorithm-complexity.md](02-algorithm-complexity.md)

---

## §3 — Đệ quy và giải thuật đệ quy

### Ý tưởng chủ đạo

**Đệ quy** là kỹ thuật trong đó hàm **gọi chính nó** để giải bài toán con nhỏ hơn cùng loại, rồi tổng hợp kết quả.

$$P(n) = \text{tổ hợp}\big(P'(n'), \text{xử lý bổ sung}\big), \quad n' < n$$

### Cấu trúc bắt buộc

| Thành phần | Vai trò | Thiếu thì sao? |
|-----------|---------|----------------|
| **Cơ sở (base case)** | Trường hợp nhỏ nhất, giải trực tiếp | Đệ quy vô hạn → stack overflow |
| **Đệ quy (recursive case)** | Thu nhỏ bài toán, gọi đệ quy | Không giải được bài toán |

### Mẫu tổng quát (Go)

```go
func solve(problem Problem) Solution {
    if isBaseCase(problem) {
        return directSolution(problem)
    }
    smaller := reduce(problem)
    subSol := solve(smaller)
    return combine(subSol)
}
```

### Các ví dụ kinh điển

| Bài toán | Công thức đệ quy | Base case |
|---------|-------------------|-----------|
| Giai thừa | f(n) = n × f(n−1) | f(0) = 1 |
| Fibonacci | f(n) = f(n−1) + f(n−2) | f(0)=0, f(1)=1 |
| Tháp Hà Nội | T(n) = 2T(n−1) + 1 | T(1) = 1 |
| Tìm kiếm nhị phân | search(lo, hi) → search(lo, mid) hoặc search(mid, hi) | lo > hi → không tìm thấy |
| Chia để trị (Merge sort) | sort(a) = merge(sort(trái), sort(phải)) | len ≤ 1 |

### Đệ quy vs Lặp

| Tiêu chí | Đệ quy | Lặp |
|---------|--------|-----|
| Tính tự nhiên | Bài toán tự tương tự (cây, chia để trị) | Bài toán tuyến tính |
| Bộ nhớ | O(n) call stack | O(1) (thường) |
| Nguy cơ | Stack overflow với n lớn | Vòng lặp vô hạn |
| Tối ưu | Đệ quy đuôi (tail recursion) → compiler tối ưu | Không cần |

### Cạm bẫy phổ biến

- **Tính lại bài toán con**: Fibonacci naïve → O(2ⁿ). Giải pháp: ghi nhớ (memoization) → O(n).
- **Quên base case**: Stack overflow.
- **Base case sai**: Kết quả sai hoặc vòng lặp vô hạn.
- **Không thu nhỏ bài toán**: Mỗi lời gọi đệ quy PHẢI giảm kích thước.

> **Chi tiết**: xem [03-recursion.md](03-recursion.md)

---

## §4 — Cấu trúc dữ liệu danh sách

### Ý tưởng chủ đạo

**Danh sách** là tập hợp có thứ tự các phần tử cùng kiểu, cho phép truy cập, chèn, xóa. Tùy theo mô hình truy cập, ta chọn cách cài đặt khác nhau.

### Bốn cách cài đặt

| Cấu trúc | Truy cập ngẫu nhiên | Chèn/Xóa đầu | Chèn/Xóa giữa | Bộ nhớ |
|----------|:---:|:---:|:---:|--------|
| **Mảng (Array list)** | O(1) ✅ | O(n) ❌ | O(n) ❌ | Liên tục, cache-friendly |
| **DSLK đơn (Singly linked)** | O(n) ❌ | O(1) ✅ | O(1) nếu có con trỏ ✅ | Overhead mỗi nút |
| **DSLK đôi (Doubly linked)** | O(n) ❌ | O(1) ✅ | O(1) nếu có con trỏ ✅ | 2× con trỏ/nút |
| **DSLK vòng (Circular)** | O(n) ❌ | O(1) ✅ | O(1) nếu có con trỏ ✅ | Tùy loại |

### Mảng (Array List)

Lưu phần tử **liên tục trong bộ nhớ**. Truy cập O(1) qua chỉ số. Chèn/xóa ở vị trí bất kỳ phải dịch chuyển O(n) phần tử.

```go
type ArrayList struct {
    data []int
}

func (l *ArrayList) Get(i int) int   { return l.data[i] }       // O(1)
func (l *ArrayList) Append(x int)    { l.data = append(l.data, x) } // O(1) amortized
func (l *ArrayList) Insert(k, x int) {                           // O(n)
    l.data = append(l.data, 0)
    copy(l.data[k+1:], l.data[k:])
    l.data[k] = x
}
```

### Danh sách liên kết đơn (Singly Linked List)

Mỗi nút chứa dữ liệu và con trỏ đến nút tiếp theo. Chèn/xóa O(1) nếu có con trỏ đến nút trước.

```go
type Node struct {
    Val  int
    Next *Node
}

type SLinkedList struct {
    Head *Node
}

func (l *SLinkedList) InsertFront(val int) { // O(1)
    l.Head = &Node{Val: val, Next: l.Head}
}
```

### Danh sách liên kết đôi (Doubly Linked List)

Mỗi nút có **hai con trỏ**: `Prev` và `Next`. Duyệt hai chiều. Chèn/xóa O(1) khi có con trỏ đến nút.

### Danh sách liên kết vòng (Circular Linked List)

Nút cuối trỏ về nút đầu (thay vì `nil`). Phù hợp cho bài toán lặp vòng (vd: bài Josephus).

### Khi nào chọn cấu trúc nào?

| Tình huống | Chọn |
|-----------|------|
| Truy cập theo index thường xuyên | Mảng |
| Chèn/xóa đầu danh sách liên tục | DSLK đơn |
| Chèn/xóa hai đầu, duyệt hai chiều | DSLK đôi |
| Xử lý nhiệm vụ vòng tròn | DSLK vòng |
| Collection nhỏ, đơn giản | Slice (Go) |

> **Chi tiết**: xem [04-list-data-structures.md](04-list-data-structures.md)

---

## §5 — Ngăn xếp và Hàng đợi

### Ý tưởng chủ đạo

Ngăn xếp và Hàng đợi là **danh sách truy cập hạn chế** — giới hạn vị trí chèn/xóa để đảm bảo thứ tự xử lý mong muốn.

### Ngăn xếp (Stack — LIFO)

**Last In, First Out**: phần tử vào sau được lấy ra trước.

| Phép toán | Mô tả | Phức tạp |
|----------|-------|---------|
| `Push(v)` | Đẩy phần tử lên đỉnh | O(1) |
| `Pop()` | Lấy phần tử ở đỉnh ra | O(1) |
| `Peek()` | Xem phần tử đỉnh (không lấy ra) | O(1) |

```go
type Stack[T any] struct{ a []T }

func (s *Stack[T]) Push(v T)       { s.a = append(s.a, v) }
func (s *Stack[T]) Pop() (T, bool) {
    if len(s.a) == 0 { var z T; return z, false }
    v := s.a[len(s.a)-1]; s.a = s.a[:len(s.a)-1]; return v, true
}
```

**Ứng dụng chính**: quản lý lời gọi hàm (call stack), quay lui, tính giá trị biểu thức, kiểm tra dấu ngoặc, duyệt DFS, undo/redo.

### Hàng đợi (Queue — FIFO)

**First In, First Out**: phần tử vào trước được lấy ra trước.

| Phép toán | Mô tả | Phức tạp |
|----------|-------|---------|
| `Enqueue(v)` | Thêm phần tử vào cuối | O(1) |
| `Dequeue()` | Lấy phần tử ở đầu ra | O(1) |

```go
type Queue[T any] struct{ a []T }

func (q *Queue[T]) Enqueue(v T)       { q.a = append(q.a, v) }
func (q *Queue[T]) Dequeue() (T, bool) {
    if len(q.a) == 0 { var z T; return z, false }
    v := q.a[0]; q.a = q.a[1:]; return v, true
}
```

**Ứng dụng chính**: duyệt BFS, lập lịch (task scheduling), hàng đợi tin nhắn, xử lý theo thứ tự.

### Hàng đợi ưu tiên (Priority Queue)

Phần tử có **độ ưu tiên cao nhất** được lấy ra trước (không phụ thuộc thứ tự vào). Cài đặt hiệu quả bằng **Heap**.

| Phép toán | Mảng | DSLK sắp thứ tự | Heap |
|----------|------|----------------|------|
| Insert | O(1) | O(n) | O(log n) |
| Extract-Min/Max | O(n) | O(1) | O(log n) |

> **Chi tiết**: xem [05-stack-queue.md](05-stack-queue.md)

---

## §6 — Cây

### Ý tưởng chủ đạo

**Cây** là cấu trúc dữ liệu phân cấp gồm các nút nối bằng cạnh, có một nút gốc duy nhất. *Định nghĩa đệ quy*: cây là một nút gốc nối với các cây con rời nhau.

### Thuật ngữ cơ bản

| Thuật ngữ | Định nghĩa |
|----------|------------|
| **Gốc (Root)** | Nút cao nhất, không có cha |
| **Cha / Con** | Nút liền trước / liền sau |
| **Lá (Leaf)** | Nút không có con |
| **Bậc (Degree)** | Số con của một nút |
| **Chiều cao (Height)** | Mức lớn nhất trong cây |
| **Cây con (Subtree)** | Một nút và toàn bộ hậu duệ |

### Cây nhị phân (Binary Tree)

Mỗi nút có **tối đa 2 con**: trái và phải (có phân biệt thứ tự).

```go
type TreeNode struct {
    Val         int
    Left, Right *TreeNode
}
```

| Dạng đặc biệt | Đặc điểm |
|---------------|----------|
| **Suy biến** | Mỗi nút chỉ có 1 con → giống danh sách liên kết |
| **Đầy đủ (Full/Perfect)** | Mọi nút có 0 hoặc 2 con, lá cùng mức → 2ʰ − 1 nút |
| **Hoàn chỉnh (Complete)** | Mọi mức đầy trừ mức cuối (lấp từ trái) → dùng cho Heap |

### Duyệt cây (Tree Traversal)

| Kiểu duyệt | Thứ tự | Ứng dụng |
|------------|--------|----------|
| **Preorder** (NLR) | Gốc → Trái → Phải | Sao chép cây, biểu thức tiền tố |
| **Inorder** (LNR) | Trái → Gốc → Phải | BST cho thứ tự sắp xếp, biểu thức trung tố |
| **Postorder** (LRN) | Trái → Phải → Gốc | Xóa cây, tính kích thước, biểu thức hậu tố |
| **Level-order** (BFS) | Từng mức, trái→phải | In theo mức, tìm đường ngắn nhất |

```go
func inorder(node *TreeNode, result *[]int) {
    if node == nil { return }
    inorder(node.Left, result)
    *result = append(*result, node.Val)
    inorder(node.Right, result)
}
```

### Cây tìm kiếm nhị phân (BST)

Với mỗi nút: mọi giá trị ở cây con trái < giá trị nút < mọi giá trị ở cây con phải.
- Tìm kiếm, chèn, xóa: **O(h)**, trong đó h là chiều cao.
- Trường hợp tốt nhất (cây cân bằng): h = O(log n).
- Trường hợp xấu nhất (suy biến): h = O(n).

### Cây cân bằng (AVL)

Tự cân bằng sau mỗi phép chèn/xóa bằng các phép **quay** (rotation). Đảm bảo h = O(log n) luôn.

> **Chi tiết**: xem [06-tree.md](06-tree.md)

---

## §7 — Biểu thức: Ký pháp và Tính toán

> **Đây là nội dung chính riêng biệt của chương này** — ứng dụng quan trọng của Stack và Tree trong xử lý biểu thức toán học.

### 7.1. Tổng quan

Biểu thức toán học như `3 + 4 × 2` quen thuộc với con người nhưng **không dễ cho máy tính** vì:
- Có **thứ tự ưu tiên** toán tử (× trước +).
- Có **dấu ngoặc** thay đổi thứ tự.
- Máy tính cần biết **khi nào** thực hiện phép toán nào.

Ba cách viết biểu thức giải quyết vấn đề này:

| Ký pháp | Tên gọi | Ví dụ (3 + 4 × 2) | Đặc điểm |
|---------|---------|-------------------|----------|
| **Trung tố (Infix)** | Thông thường | `3 + 4 * 2` | Con người đọc; cần ưu tiên & ngoặc |
| **Hậu tố (Postfix)** | RPN (Reverse Polish) | `3 4 2 * +` | Không cần ngoặc; dễ tính bằng stack |
| **Tiền tố (Prefix)** | Polish Notation | `+ 3 * 4 2` | Không cần ngoặc; dễ cho đệ quy |

### 7.2. Ký pháp Trung tố (Infix)

Toán tử đứng **giữa** hai toán hạng: `A op B`.

```
(3 + 4) × 2 = 14
 3 + 4  × 2 = 11    ← ưu tiên × > +
```

**Ưu điểm**: tự nhiên với con người.
**Nhược điểm**: cần bảng ưu tiên và dấu ngoặc → phức tạp cho máy tính.

#### Bảng ưu tiên toán tử

| Ưu tiên | Toán tử | Kết hợp |
|---------|---------|---------|
| Cao nhất | `()` (ngoặc) | — |
| 2 | `^` (lũy thừa) | Phải → Trái |
| 3 | `*`, `/`, `%` | Trái → Phải |
| Thấp nhất | `+`, `-` | Trái → Phải |

### 7.3. Ký pháp Hậu tố (Postfix / RPN)

Toán tử đứng **sau** hai toán hạng: `A B op`.

```
Infix:    (3 + 4) × 2
Postfix:  3 4 + 2 *
```

**Ưu điểm**:
- **Không cần ngoặc** — thứ tự toán tử xác định bởi vị trí.
- **Tính giá trị cực kỳ đơn giản** bằng một Stack.

#### Thuật toán tính giá trị biểu thức hậu tố

```
Duyệt từ trái sang phải:
  - Gặp SỐ → đẩy vào Stack
  - Gặp TOÁN TỬ → lấy 2 số từ Stack, tính, đẩy kết quả vào Stack
Kết quả cuối cùng = phần tử duy nhất còn trong Stack
```

**Ví dụ truy vết**: `3 4 2 * +`

| Token | Hành động | Stack |
|-------|----------|-------|
| 3 | push | [3] |
| 4 | push | [3, 4] |
| 2 | push | [3, 4, 2] |
| * | pop 2,4 → push 8 | [3, 8] |
| + | pop 8,3 → push 11 | [11] |

Kết quả: **11** ✓

#### Cài đặt (Go)

```go
import (
    "strconv"
    "strings"
)

func EvalPostfix(expr string) int {
    tokens := strings.Fields(expr)
    stack := make([]int, 0, len(tokens))

    for _, tok := range tokens {
        switch tok {
        case "+", "-", "*", "/":
            b := stack[len(stack)-1]
            a := stack[len(stack)-2]
            stack = stack[:len(stack)-2]
            var res int
            switch tok {
            case "+": res = a + b
            case "-": res = a - b
            case "*": res = a * b
            case "/": res = a / b
            }
            stack = append(stack, res)
        default:
            num, _ := strconv.Atoi(tok)
            stack = append(stack, num)
        }
    }
    return stack[0]
}
```

### 7.4. Ký pháp Tiền tố (Prefix / Polish Notation)

Toán tử đứng **trước** hai toán hạng: `op A B`.

```
Infix:   (3 + 4) × 2
Prefix:  * + 3 4 2
```

#### Thuật toán tính giá trị biểu thức tiền tố

```
Duyệt từ PHẢI sang TRÁI:
  - Gặp SỐ → đẩy vào Stack
  - Gặp TOÁN TỬ → lấy 2 số từ Stack, tính, đẩy kết quả
  (Chú ý: a = pop đầu, b = pop thứ hai → tính a op b)
```

Hoặc dùng **đệ quy**: gặp toán tử → đệ quy tính toán hạng trái, rồi toán hạng phải, rồi thực hiện phép toán.

```go
func EvalPrefix(expr string) int {
    tokens := strings.Fields(expr)
    stack := make([]int, 0, len(tokens))

    // Duyệt từ phải sang trái
    for i := len(tokens) - 1; i >= 0; i-- {
        tok := tokens[i]
        switch tok {
        case "+", "-", "*", "/":
            a := stack[len(stack)-1]
            b := stack[len(stack)-2]
            stack = stack[:len(stack)-2]
            var res int
            switch tok {
            case "+": res = a + b
            case "-": res = a - b
            case "*": res = a * b
            case "/": res = a / b
            }
            stack = append(stack, res)
        default:
            num, _ := strconv.Atoi(tok)
            stack = append(stack, num)
        }
    }
    return stack[0]
}
```

### 7.5. Chuyển Infix → Postfix (Thuật toán Shunting-Yard)

Thuật toán của Edsger Dijkstra, sử dụng **một Stack toán tử** và **một hàng đợi đầu ra**.

#### Quy tắc

```
Duyệt từng token của biểu thức infix:

1. SỐ → đưa thẳng vào output

2. TOÁN TỬ (op):
   - Khi stack không rỗng VÀ đỉnh stack là toán tử
     có ưu tiên ≥ op (nếu op kết hợp trái)
     hoặc ưu tiên > op (nếu op kết hợp phải):
       → pop đỉnh stack vào output
   - Push op vào stack

3. DẤU '(' → push vào stack

4. DẤU ')':
   - Pop stack vào output cho đến khi gặp '('
   - Bỏ '(' (không đưa vào output)

5. KẾT THÚC: pop hết stack vào output
```

#### Ví dụ truy vết: `3 + 4 * 2 - 1`

| Token | Output | Stack (đỉnh→) | Ghi chú |
|-------|--------|---------------|---------|
| 3 | `3` | | số → output |
| + | `3` | `+` | push |
| 4 | `3 4` | `+` | số → output |
| * | `3 4` | `+ *` | ưu tiên * > + → push |
| 2 | `3 4 2` | `+ *` | số → output |
| - | `3 4 2 * +` | `-` | ưu tiên - ≤ * → pop *; - ≤ + → pop +; push - |
| 1 | `3 4 2 * + 1` | `-` | số → output |
| (hết) | `3 4 2 * + 1 -` | | pop hết |

Kết quả Postfix: **`3 4 2 * + 1 -`** → tính: 3 + 8 − 1 = 10 ✓

#### Ví dụ có ngoặc: `(3 + 4) * 2`

| Token | Output | Stack | Ghi chú |
|-------|--------|-------|---------|
| ( | | `(` | push |
| 3 | `3` | `(` | số |
| + | `3` | `( +` | push (bên trong ngoặc) |
| 4 | `3 4` | `( +` | số |
| ) | `3 4 +` | | pop đến '(' |
| * | `3 4 +` | `*` | push |
| 2 | `3 4 + 2` | `*` | số |
| (hết) | `3 4 + 2 *` | | pop hết |

Kết quả Postfix: **`3 4 + 2 *`** → tính: 7 × 2 = 14 ✓

#### Cài đặt (Go)

```go
func precedence(op string) int {
    switch op {
    case "+", "-": return 1
    case "*", "/": return 2
    case "^":      return 3
    }
    return 0
}

func isRightAssoc(op string) bool {
    return op == "^"
}

func isOperator(tok string) bool {
    return tok == "+" || tok == "-" || tok == "*" || tok == "/" || tok == "^"
}

func InfixToPostfix(expr string) string {
    tokens := strings.Fields(expr)
    var output []string
    var stack []string

    for _, tok := range tokens {
        switch {
        case tok == "(":
            stack = append(stack, tok)

        case tok == ")":
            for len(stack) > 0 && stack[len(stack)-1] != "(" {
                output = append(output, stack[len(stack)-1])
                stack = stack[:len(stack)-1]
            }
            stack = stack[:len(stack)-1] // bỏ '('

        case isOperator(tok):
            for len(stack) > 0 && isOperator(stack[len(stack)-1]) {
                top := stack[len(stack)-1]
                if (!isRightAssoc(tok) && precedence(tok) <= precedence(top)) ||
                    (isRightAssoc(tok) && precedence(tok) < precedence(top)) {
                    output = append(output, top)
                    stack = stack[:len(stack)-1]
                } else {
                    break
                }
            }
            stack = append(stack, tok)

        default: // số
            output = append(output, tok)
        }
    }

    // Pop phần còn lại
    for len(stack) > 0 {
        output = append(output, stack[len(stack)-1])
        stack = stack[:len(stack)-1]
    }

    return strings.Join(output, " ")
}
```

### 7.6. Chuyển Infix → Prefix

**Phương pháp**: đảo ngược biểu thức infix, hoán đổi `(` ↔ `)`, áp dụng Shunting-Yard cho postfix, rồi đảo ngược kết quả.

```go
func InfixToPrefix(expr string) string {
    tokens := strings.Fields(expr)

    // Bước 1: Đảo ngược, hoán đổi ngoặc
    reversed := make([]string, len(tokens))
    for i, tok := range tokens {
        j := len(tokens) - 1 - i
        switch tok {
        case "(": reversed[j] = ")"
        case ")": reversed[j] = "("
        default:  reversed[j] = tok
        }
    }

    // Bước 2: Shunting-Yard (dùng InfixToPostfix)
    postfix := InfixToPostfix(strings.Join(reversed, " "))

    // Bước 3: Đảo ngược kết quả
    parts := strings.Fields(postfix)
    for i, j := 0, len(parts)-1; i < j; i, j = i+1, j-1 {
        parts[i], parts[j] = parts[j], parts[i]
    }
    return strings.Join(parts, " ")
}
```

### 7.7. Cây biểu thức (Expression Tree)

**Cây biểu thức** là cây nhị phân trong đó:
- **Nút lá** = toán hạng (số hoặc biến).
- **Nút trong** = toán tử.
- **Cây con trái** = toán hạng trái, **cây con phải** = toán hạng phải.

```
       *
      / \
     +   2
    / \
   3   4

Infix:    (3 + 4) * 2
Prefix:   * + 3 4 2     ← Duyệt Preorder (NLR)
Postfix:  3 4 + 2 *     ← Duyệt Postorder (LRN)
Infix:    3 + 4 * 2     ← Duyệt Inorder (LNR) — cần thêm ngoặc
```

> **Insight quan trọng**: Ba cách duyệt cây (Preorder, Inorder, Postorder) cho ra đúng ba ký pháp biểu thức (Prefix, Infix, Postfix).

#### Xây dựng cây biểu thức từ Postfix

```
Duyệt biểu thức postfix từ trái sang phải:
  - Gặp SỐ → tạo nút lá, đẩy vào stack
  - Gặp TOÁN TỬ → pop 2 nút, tạo nút toán tử
    (nút pop đầu = con phải, pop sau = con trái), đẩy nút mới vào stack
Kết quả: nút duy nhất còn trong stack = gốc cây biểu thức
```

```go
type ExprNode struct {
    Val         string
    Left, Right *ExprNode
}

func BuildExprTree(postfix string) *ExprNode {
    tokens := strings.Fields(postfix)
    stack := make([]*ExprNode, 0, len(tokens))

    for _, tok := range tokens {
        if isOperator(tok) {
            right := stack[len(stack)-1]
            left := stack[len(stack)-2]
            stack = stack[:len(stack)-2]
            node := &ExprNode{Val: tok, Left: left, Right: right}
            stack = append(stack, node)
        } else {
            stack = append(stack, &ExprNode{Val: tok})
        }
    }
    return stack[0]
}

// Tính giá trị cây biểu thức bằng đệ quy (postorder)
func EvalTree(node *ExprNode) int {
    if node.Left == nil && node.Right == nil {
        num, _ := strconv.Atoi(node.Val)
        return num
    }
    left := EvalTree(node.Left)
    right := EvalTree(node.Right)
    switch node.Val {
    case "+": return left + right
    case "-": return left - right
    case "*": return left * right
    case "/": return left / right
    }
    return 0
}
```

### 7.8. Tổng hợp chuyển đổi giữa các ký pháp

```
                  Infix
                 ↗     ↖
   Shunting-Yard         Inorder
               ↓           ↑
             Postfix ←→ Expression Tree ←→ Prefix
          (Postorder)                    (Preorder)
```

| Từ | Sang | Phương pháp |
|----|------|------------|
| Infix → Postfix | | Shunting-Yard |
| Infix → Prefix | | Đảo + Shunting-Yard + Đảo |
| Postfix → Giá trị | | Stack: số→push, toán tử→pop 2, tính, push |
| Prefix → Giá trị | | Duyệt phải→trái + Stack |
| Postfix → Cây | | Stack: số→nút lá, toán tử→nút trong nối 2 con |
| Cây → Infix | | Inorder + thêm ngoặc |
| Cây → Postfix | | Postorder traversal |
| Cây → Prefix | | Preorder traversal |

### 7.9. Lỗi thường gặp

- **Nhầm thứ tự toán hạng**: Với phép `-` và `/`, thứ tự quan trọng. Postfix `a b -` = a − b (KHÔNG phải b − a). Pop b trước, pop a sau → tính a − b.
- **Quên tính kết hợp**: `2 ^ 3 ^ 4` = `2 ^ (3 ^ 4)` (phải→trái), KHÔNG phải `(2 ^ 3) ^ 4`.
- **Quên xử lý ngoặc** trong Shunting-Yard.
- **Lẫn Infix với Postfix** khi viết biểu thức cho stack calculator.
- **Không validate input**: biểu thức sai cú pháp → stack rỗng khi pop → panic.

### 7.10. Bài tập (LeetCode)

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Evaluate RPN | Stack hậu tố | [150](https://leetcode.com/problems/evaluate-reverse-polish-notation/) |
| Basic Calculator | Shunting-Yard / Stack | [224](https://leetcode.com/problems/basic-calculator/) |
| Basic Calculator II | Stack + ưu tiên | [227](https://leetcode.com/problems/basic-calculator-ii/) |
| Basic Calculator III | Đầy đủ ngoặc + 4 phép | [772](https://leetcode.com/problems/basic-calculator-iii/) |
| Expression Add Operators | Quay lui + tính biểu thức | [282](https://leetcode.com/problems/expression-add-operators/) |
| Different Ways to Add Parentheses | Chia để trị trên biểu thức | [241](https://leetcode.com/problems/different-ways-to-add-parentheses/) |
| Design an Expression Tree | Xây cây biểu thức | [1628](https://leetcode.com/problems/design-an-expression-tree-with-evaluate-function/) |
| Parse Lisp Expression | Prefix-style parsing | [736](https://leetcode.com/problems/parse-lisp-expression/) |

---

## §8 — Sắp xếp

### Ý tưởng chủ đạo

**Sắp xếp** là sắp lại các phần tử theo một thứ tự xác định (tăng, giảm, từ điển…). Đây là bài toán cơ bản nhất, xuất hiện ở mọi nơi — từ hiển thị dữ liệu đến tiền xử lý cho tìm kiếm nhị phân.

### So sánh 6 thuật toán chính

| Thuật toán | Tốt nhất | Trung bình | Xấu nhất | Bộ nhớ | Ổn định? | Ghi chú |
|-----------|---------|----------|---------|--------|---------|---------|
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | ❌ | Ít swap nhất |
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ | Đơn giản; cờ dừng sớm |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ | Nhanh trên dữ liệu gần sắp |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ | Luôn O(n log n) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ | Nhanh nhất thực tế |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | Ổn định; tốt cho external sort |

### Ý tưởng mỗi thuật toán (tóm tắt)

**Selection Sort**: mỗi vòng tìm phần tử nhỏ nhất, đưa về đúng vị trí.

**Bubble Sort**: liên tục đổi chỗ hai phần tử kề không đúng thứ tự. Phần tử lớn "nổi" lên cuối.

**Insertion Sort**: lần lượt chèn từng phần tử vào đúng vị trí trong phần đã sắp. Như xếp bài trên tay.

**Heap Sort**: xây Max-Heap, lần lượt lấy phần tử lớn nhất (gốc heap) ra cuối mảng. Kết hợp cây nhị phân + sắp xếp.

**Quick Sort**: chọn **pivot**, phân hoạch mảng thành 2 phần (≤ pivot, > pivot), đệ quy. Chia để trị.

```go
func quickSort(a []int, lo, hi int) {
    if lo >= hi { return }
    pivot := a[hi]
    i := lo
    for j := lo; j < hi; j++ {
        if a[j] <= pivot {
            a[i], a[j] = a[j], a[i]
            i++
        }
    }
    a[i], a[hi] = a[hi], a[i]
    quickSort(a, lo, i-1)
    quickSort(a, i+1, hi)
}
```

**Merge Sort**: chia đôi mảng, đệ quy sắp xếp hai nửa, trộn hai nửa đã sắp. Luôn O(n log n), ổn định.

```go
func mergeSort(a []int) []int {
    if len(a) <= 1 { return a }
    mid := len(a) / 2
    left := mergeSort(a[:mid])
    right := mergeSort(a[mid:])
    return merge(left, right)
}

func merge(a, b []int) []int {
    result := make([]int, 0, len(a)+len(b))
    i, j := 0, 0
    for i < len(a) && j < len(b) {
        if a[i] <= b[j] {
            result = append(result, a[i]); i++
        } else {
            result = append(result, b[j]); j++
        }
    }
    result = append(result, a[i:]...)
    result = append(result, b[j:]...)
    return result
}
```

### Khi nào chọn thuật toán nào?

| Tình huống | Chọn |
|-----------|------|
| Dữ liệu nhỏ (n < 50) | Insertion Sort |
| Cần ổn định, bộ nhớ không giới hạn | Merge Sort |
| Cần in-place, không cần ổn định | Quick Sort |
| Cần worst-case O(n log n) đảm bảo | Heap Sort |
| Dữ liệu gần sắp xếp rồi | Insertion Sort |
| External sort (dữ liệu trên đĩa) | Merge Sort |

### Giới hạn lý thuyết

Mọi thuật toán sắp xếp dựa trên **so sánh** đều có cận dưới **Ω(n log n)** — không thể làm tốt hơn trong trường hợp xấu nhất.

Các thuật toán **không dựa trên so sánh** (Counting Sort, Radix Sort, Bucket Sort) có thể đạt O(n) nhưng cần điều kiện đặc biệt về miền giá trị.

> **Chi tiết**: xem [07-sorting.md](07-sorting.md)

---

## §9 — Tìm kiếm

### Ý tưởng chủ đạo

**Tìm kiếm** là quá trình tìm một phần tử (hoặc vị trí) trong một tập dữ liệu. Hiệu quả phụ thuộc vào **tính chất của dữ liệu** (đã sắp? tĩnh? phân bố đều?).

### Các mô hình tìm kiếm

| Mô hình | Mục tiêu |
|---------|---------|
| Tìm chính xác | Tìm vị trí `a[i] == x` |
| Cận dưới (Lower bound) | Vị trí đầu tiên `a[i] ≥ x` |
| Cận trên (Upper bound) | Vị trí đầu tiên `a[i] > x` |
| Tìm theo khoảng | Tất cả phần tử trong `[lo, hi]` |

### So sánh các thuật toán tìm kiếm

| Thuật toán | Phức tạp | Yêu cầu | Khi nào dùng |
|-----------|---------|---------|-------------|
| **Tìm tuyến tính** | O(n) | Không | Dữ liệu chưa sắp, nhỏ |
| **Tìm nhị phân** | O(log n) | Đã sắp | Dữ liệu tĩnh, đã sắp |
| **Tìm nội suy** | O(log log n) TB | Đã sắp, phân bố đều | Dữ liệu số, phân bố đều |
| **Cây BST** | O(log n) TB | Cây cân bằng | Dữ liệu động (chèn/xóa thường xuyên) |
| **Bảng băm** | O(1) TB | Hàm băm tốt | Tìm chính xác, không cần thứ tự |

### Tìm tuyến tính (Linear Search)

Duyệt lần lượt từ đầu đến cuối. Đơn giản nhất, hoạt động trên mọi dữ liệu.

```go
func LinearSearch(a []int, x int) int {
    for i, v := range a {
        if v == x { return i }
    }
    return -1
}
```

**Cải tiến — Lính canh (Sentinel)**: thêm x vào cuối mảng → không cần kiểm tra biên trong vòng lặp.

### Tìm nhị phân (Binary Search)

**Yêu cầu**: mảng ĐÃ SẮP XẾP. Chia đôi khoảng tìm kiếm mỗi bước.

```go
func BinarySearch(a []int, x int) int {
    lo, hi := 0, len(a)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2 // tránh tràn số
        if a[mid] == x {
            return mid
        } else if a[mid] < x {
            lo = mid + 1
        } else {
            hi = mid - 1
        }
    }
    return -1
}
```

**Cận dưới (Lower Bound)** — vị trí đầu tiên ≥ x:

```go
func LowerBound(a []int, x int) int {
    lo, hi := 0, len(a)
    for lo < hi {
        mid := lo + (hi-lo)/2
        if a[mid] < x {
            lo = mid + 1
        } else {
            hi = mid
        }
    }
    return lo
}
```

### Tìm nội suy (Interpolation Search)

Thay vì luôn chọn giữa (mid), **ước lượng** vị trí dựa trên giá trị:

$$\text{pos} = lo + \frac{(x - a[lo]) \times (hi - lo)}{a[hi] - a[lo]}$$

- Trung bình O(log log n) nếu phân bố đều.
- Xấu nhất O(n) nếu phân bố lệch.

### Cây tìm kiếm nhị phân (BST)

Kết hợp tìm kiếm nhanh + chèn/xóa hiệu quả. Xem §6 — Cây.

### Lỗi thường gặp

- **Dùng binary search trên mảng chưa sắp** → kết quả sai.
- **Tràn số** khi tính `mid = (lo + hi) / 2` với lo, hi lớn → dùng `lo + (hi - lo) / 2`.
- **Vòng lặp vô hạn** do cập nhật `lo`/`hi` sai (off-by-one).
- **Lẫn lower_bound với upper_bound**: `<` vs `<=` tại điều kiện.

> **Chi tiết**: xem [08-search.md](08-search.md)

---

## Bài tập tổng hợp (LeetCode)

### Cấu trúc dữ liệu cơ bản

| Bài tập | Chủ đề | Link |
|---------|--------|------|
| Reverse Linked List | DSLK | [206](https://leetcode.com/problems/reverse-linked-list/) |
| Merge Two Sorted Lists | DSLK | [21](https://leetcode.com/problems/merge-two-sorted-lists/) |
| Valid Parentheses | Stack | [20](https://leetcode.com/problems/valid-parentheses/) |
| Min Stack | Stack | [155](https://leetcode.com/problems/min-stack/) |
| Implement Queue using Stacks | Hàng đợi | [232](https://leetcode.com/problems/implement-queue-using-stacks/) |
| Implement Stack using Queues | Ngăn xếp | [225](https://leetcode.com/problems/implement-stack-using-queues/) |
| LRU Cache | Hash + DSLK đôi | [146](https://leetcode.com/problems/lru-cache/) |

### Cây

| Bài tập | Chủ đề | Link |
|---------|--------|------|
| Inorder Traversal | Duyệt cây | [94](https://leetcode.com/problems/binary-tree-inorder-traversal/) |
| Level Order Traversal | BFS | [102](https://leetcode.com/problems/binary-tree-level-order-traversal/) |
| Maximum Depth | Đệ quy / DFS | [104](https://leetcode.com/problems/maximum-depth-of-binary-tree/) |
| Validate BST | BST + Inorder | [98](https://leetcode.com/problems/validate-binary-search-tree/) |
| Lowest Common Ancestor (BST) | BST | [235](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/) |
| Balanced Binary Tree | Đệ quy | [110](https://leetcode.com/problems/balanced-binary-tree/) |

### Sắp xếp & Tìm kiếm

| Bài tập | Chủ đề | Link |
|---------|--------|------|
| Sort an Array | Merge/Quick Sort | [912](https://leetcode.com/problems/sort-an-array/) |
| Merge Intervals | Sắp xếp + gộp | [56](https://leetcode.com/problems/merge-intervals/) |
| Kth Largest Element | Quick Select / Heap | [215](https://leetcode.com/problems/kth-largest-element-in-an-array/) |
| Binary Search | Tìm nhị phân | [704](https://leetcode.com/problems/binary-search/) |
| Search in Rotated Array | Binary Search biến thể | [33](https://leetcode.com/problems/search-in-rotated-sorted-array/) |
| Find First and Last Position | Lower/Upper bound | [34](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/) |

### Biểu thức & Ký pháp

| Bài tập | Chủ đề | Link |
|---------|--------|------|
| Evaluate RPN | Stack hậu tố | [150](https://leetcode.com/problems/evaluate-reverse-polish-notation/) |
| Basic Calculator | Infix evaluation | [224](https://leetcode.com/problems/basic-calculator/) |
| Basic Calculator II | Stack + ưu tiên | [227](https://leetcode.com/problems/basic-calculator-ii/) |
| Different Ways to Add Parens | Chia để trị | [241](https://leetcode.com/problems/different-ways-to-add-parentheses/) |

---

## Pattern liên quan

- [09-enumeration.md](09-enumeration.md) — Bài toán liệt kê (Phần 1 sách) — sử dụng quay lui, đệ quy, stack.
- **Quy hoạch động (Phần 3)**: khi bài toán con chồng lấn → ghi nhớ thay vì tính lại.
- **Đồ thị (Phần 4)**: mở rộng cây thành cấu trúc tổng quát hơn; dùng stack (DFS) và queue (BFS).
- **Bitmask**: biểu diễn tập con bằng bit — kết hợp với sắp xếp, tìm kiếm.

---

## Ghi nhớ nhanh

### Cấu trúc dữ liệu
- **Mảng**: O(1) truy cập, O(n) chèn/xóa giữa.
- **DSLK**: O(1) chèn/xóa (có con trỏ), O(n) truy cập.
- **Stack (LIFO)**: Push/Pop O(1). Dùng cho: biểu thức, quay lui, DFS, undo.
- **Queue (FIFO)**: Enqueue/Dequeue O(1). Dùng cho: BFS, lập lịch.
- **Cây nhị phân**: duyệt Preorder/Inorder/Postorder ↔ Prefix/Infix/Postfix.
- **BST**: tìm/chèn/xóa O(log n) nếu cân bằng.

### Biểu thức (§7)
- **Infix → Postfix**: Shunting-Yard (stack toán tử + output).
- **Tính Postfix**: stack — số→push, toán tử→pop 2, tính, push.
- **Cây biểu thức**: Preorder=Prefix, Inorder=Infix, Postorder=Postfix.
- **Thứ tự pop**: phép `-`, `/` → a (pop sau) op b (pop trước).

### Sắp xếp
- O(n²): Selection, Bubble, Insertion — cho n nhỏ, dữ liệu gần sắp.
- O(n log n): Quick (thực tế nhanh nhất), Merge (ổn định), Heap (worst-case đảm bảo).
- Cận dưới so sánh: **Ω(n log n)**.

### Tìm kiếm
- Chưa sắp → Linear O(n).
- Đã sắp → Binary O(log n). Luôn dùng `lo + (hi - lo) / 2`.
- Dữ liệu động → BST hoặc Hash.

---

## Học tiếp

- [09-enumeration.md](09-enumeration.md) — Phần 1: Bài toán liệt kê
- Phần 3: Quy hoạch động — khi đệ quy có bài toán con chồng lấn
- Phần 4: Thuật toán đồ thị — mở rộng cây thành đồ thị tổng quát
