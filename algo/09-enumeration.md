# Bài Toán Liệt Kê

> **Chủ đề**: Thuật toán | **Mức độ**: Trung bình → Nâng cao
> **Nguồn**: *Giải thuật và Lập trình* — Lê Minh Hoàng, Phần 1

---

## Tổng quan

**Bài toán liệt kê** là quá trình **duyệt có hệ thống tất cả các cấu hình** thỏa mãn một tập ràng buộc cho trước. Thay vì tìm *một* đáp án, ta sinh ra *mọi* đáp án hợp lệ.

Đây là kỹ năng nền tảng đằng sau brute-force, quay lui (backtracking) và nhánh cận (branch & bound) — đồng thời giúp hiểu tại sao các bài toán tối ưu lại khó (bùng nổ tổ hợp).

---

## Tại sao cần học

- Nhiều bài toán thực tế **không có thuật toán hiệu quả** — cách tốt nhất là xét toàn bộ ứng viên rồi chọn tốt nhất (hoặc tất cả hợp lệ).
- Thuật toán liệt kê là **bước đầu tiên trong giải quyết vấn đề**: bắt đầu bằng brute-force, sau đó tối ưu.
- Hiểu liệt kê giúp bạn suy luận về **kích thước bài toán**, **không gian trạng thái** và **cây tìm kiếm** — khái niệm cốt lõi trong competitive programming, tìm kiếm AI, thỏa mãn ràng buộc và tối ưu hóa.

---

## Mục lục

1. [Nhắc lại kiến thức Đại số Tổ hợp](#1-nhắc-lại-kiến-thức-đại-số-tổ-hợp)
2. [Phương pháp Sinh (Generation)](#2-phương-pháp-sinh-generation)
3. [Thuật toán Quay lui (Backtracking)](#3-thuật-toán-quay-lui-backtracking)
4. [Kỹ thuật Nhánh cận (Branch and Bound)](#4-kỹ-thuật-nhánh-cận-branch-and-bound)

---

## 1. Nhắc lại kiến thức Đại số Tổ hợp

### Định nghĩa / Ý tưởng

- **Định nghĩa**: Đại số tổ hợp là nhánh toán học nghiên cứu việc đếm, sắp xếp và lựa chọn các đối tượng rời rạc.
- **Ý tưởng**: Trước khi liệt kê cấu hình, ta cần biết **có bao nhiêu** cấu hình và chúng có **cấu trúc gì**.
- **Mục đích**: Cung cấp từ vựng (hoán vị, chỉnh hợp, tổ hợp) và công thức để ước lượng kích thước không gian tìm kiếm.

### Các khái niệm chính

#### 1.1. Chỉnh hợp lặp

Chọn **k** phần tử từ tập **n** phần tử, **có thứ tự**, **cho phép lặp**.

$$A'(n, k) = n^k$$

> **Ví dụ**: Tất cả chuỗi nhị phân độ dài 3 → chọn từ {0, 1}, k=3 → 2³ = 8 chuỗi:
> `000, 001, 010, 011, 100, 101, 110, 111`

**Thực tế**: Sinh mật khẩu (mỗi ký tự chọn từ bảng chữ cái n ký hiệu, độ dài k).

#### 1.2. Chỉnh hợp không lặp

Chọn **k** phần tử từ **n**, **có thứ tự**, **không lặp**.

$$A(n, k) = \frac{n!}{(n-k)!} = n \times (n-1) \times \cdots \times (n-k+1)$$

> **Ví dụ**: Xếp 3 người từ 5 vào vị trí Vàng/Bạc/Đồng → 5 × 4 × 3 = 60 cách.

**Thực tế**: Phân công công việc (giao k nhiệm vụ khác nhau cho k trong n người).

#### 1.3. Hoán vị

Trường hợp đặc biệt khi **k = n** — sắp xếp **toàn bộ n phần tử** theo một thứ tự nào đó.

$$P(n) = n!$$

> **Ví dụ**: Tất cả thứ tự của {A, B, C} → 3! = 6:
> `ABC, ACB, BAC, BCA, CAB, CBA`

**Thực tế**: Lập lịch (tất cả thứ tự có thể của n công việc).

#### 1.4. Tổ hợp

Chọn **k** phần tử từ **n**, **không phân biệt thứ tự**, **không lặp**.

$$C(n, k) = \binom{n}{k} = \frac{n!}{k!(n-k)!}$$

> **Ví dụ**: Chọn 2 người từ {A, B, C, D} → C(4,2) = 6:
> `{A,B}, {A,C}, {A,D}, {B,C}, {B,D}, {C,D}`

**Thực tế**: Chọn ban cố vấn, chọn tập con tính năng cần bật.

### Kích thước không gian tìm kiếm — Tại sao quan trọng

| Cấu hình | Công thức | n=10, k=5 | n=20, k=10 |
|----------|---------|-----------|------------|
| Chỉnh hợp lặp | nᵏ | 100,000 | ~10¹³ |
| Chỉnh hợp không lặp | n!/(n-k)! | 30,240 | ~6.7×10¹² |
| Hoán vị | n! | 3,628,800 | ~2.4×10¹⁸ |
| Tổ hợp | C(n,k) | 252 | 184,756 |
| Tập con | 2ⁿ | 1,024 | 1,048,576 |

> **Nhận xét quan trọng**: Không gian tìm kiếm tăng **theo hàm mũ hoặc giai thừa**. Ngay cả với n khiêm tốn (20–30), brute-force trở nên bất khả thi nếu không cắt tỉa.

### Ghi nhớ nhanh

- **Có thứ tự** → chỉnh hợp / hoán vị.
- **Không thứ tự** → tổ hợp.
- **Cho phép lặp** → mỗi vị trí nhân với n lựa chọn.
- C(n,k) = C(n, n−k) — chọn phần tử lấy = chọn phần tử bỏ.
- Kích thước không gian tìm kiếm quyết định brute-force có khả thi hay cần tối ưu/cắt tỉa.

---

## 2. Phương pháp Sinh (Generation)

### Định nghĩa / Ý tưởng

- **Định nghĩa**: Kỹ thuật sinh tất cả cấu hình thuộc một kiểu nào đó bằng cách xác định một **thứ tự toàn phần** (ví dụ: thứ tự từ điển) và hàm **next()** biến đổi cấu hình hiện tại thành cấu hình kế tiếp.
- **Ý tưởng**: Bắt đầu từ cấu hình **đầu tiên** (nhỏ nhất). Lặp lại `next()` cho đến khi gặp cấu hình **cuối cùng** (lớn nhất). Mỗi cấu hình hợp lệ được duyệt đúng một lần.
- **Mục đích**: Khi cần liệt kê tất cả phần tử thuộc một kiểu **đầy đủ và không trùng lặp**, và phép biến đổi "kế tiếp" được định nghĩa rõ ràng.

### Khi nào sử dụng

- Cần **tất cả cấu hình** của một kiểu cụ thể (tất cả chuỗi nhị phân, tập con, hoán vị).
- Tập cấu hình có **thứ tự tự nhiên** (thứ tự từ điển).
- Kích thước bài toán đủ nhỏ để liệt kê toàn bộ.
- Muốn cách tiếp cận **lặp (không đệ quy)**.

### Cách tiếp cận chính

```
1. Sinh cấu hình ĐẦU TIÊN
2. LẶP:
   a. Xuất cấu hình hiện tại
   b. Nếu là cấu hình CUỐI CÙNG → dừng
   c. Sinh cấu hình KẾ TIẾP từ cấu hình hiện tại
```

Hoàn toàn lặp — không đệ quy, không stack. Thách thức là định nghĩa `next()` đúng cho từng kiểu cấu hình.

---

### 2.1. Sinh tất cả dãy nhị phân độ dài n

**Bài toán**: Liệt kê tất cả chuỗi gồm 0 và 1 có độ dài n.

**Thứ tự**: Từ điển (coi như số nhị phân n chữ số: 000…0 → 111…1).

**Quy tắc next()**: Tìm bit `0` bên phải nhất, đặt thành `1`, đặt tất cả bit bên phải nó thành `0`.

```
n=3: 000 → 001 → 010 → 011 → 100 → 101 → 110 → 111
```

#### Cài đặt (Go)

```go
// Sinh tất cả dãy nhị phân độ dài n
func GenBinaryStrings(n int) [][]int {
    var result [][]int
    b := make([]int, n) // bắt đầu: 000...0

    for {
        // Ghi lại cấu hình hiện tại
        cur := make([]int, n)
        copy(cur, b)
        result = append(result, cur)

        // Tìm bit 0 bên phải nhất (vị trí có thể tăng)
        i := n - 1
        for i >= 0 && b[i] == 1 {
            i--
        }
        if i < 0 {
            break // b == 111...1 → cấu hình cuối cùng
        }
        b[i] = 1
        for j := i + 1; j < n; j++ {
            b[j] = 0 // đặt lại tất cả bên phải
        }
    }
    return result
}
```

**Độ phức tạp**: O(n × 2ⁿ) thời gian, O(n) bộ nhớ (không tính output).

---

### 2.2. Liệt kê các tập con k phần tử của {1, 2, …, n}

**Bài toán**: Liệt kê tất cả cách chọn k phần tử từ {1, 2, …, n}. Biểu diễn mỗi tập con dạng bộ sắp tăng `(c₁, c₂, …, cₖ)` với `c₁ < c₂ < … < cₖ`.

**Cấu hình đầu**: `(1, 2, 3, …, k)`  
**Cấu hình cuối**: `(n−k+1, n−k+2, …, n)`

**Quy tắc next()**: Tìm vị trí `i` bên phải nhất sao cho `cᵢ` còn tăng được (tức `cᵢ < n − k + i`). Tăng `cᵢ`, sau đó đặt `cᵢ₊₁ = cᵢ + 1`, `cᵢ₊₂ = cᵢ + 2`, v.v.

```
n=5, k=3:
(1,2,3) → (1,2,4) → (1,2,5) → (1,3,4) → (1,3,5) → (1,4,5) →
(2,3,4) → (2,3,5) → (2,4,5) → (3,4,5)
```

#### Cài đặt (Go)

```go
// Sinh tất cả tập con k phần tử của {1..n} theo thứ tự từ điển
func GenCombinations(n, k int) [][]int {
    var result [][]int
    c := make([]int, k)
    for i := range c {
        c[i] = i + 1 // đầu tiên: (1, 2, ..., k)
    }

    for {
        cur := make([]int, k)
        copy(cur, c)
        result = append(result, cur)

        // Tìm vị trí bên phải nhất có thể tăng
        i := k - 1
        for i >= 0 && c[i] == n-k+i+1 {
            i--
        }
        if i < 0 {
            break // đã đến tổ hợp cuối cùng
        }
        c[i]++
        for j := i + 1; j < k; j++ {
            c[j] = c[j-1] + 1
        }
    }
    return result
}
```

**Độ phức tạp**: O(k × C(n,k)) thời gian, O(k) bộ nhớ.

---

### 2.3. Liệt kê các hoán vị của {1, 2, …, n}

**Bài toán**: Liệt kê toàn bộ n! thứ tự.

**Cấu hình đầu**: `(1, 2, 3, …, n)` — thứ tự tăng.  
**Cấu hình cuối**: `(n, n−1, …, 2, 1)` — thứ tự giảm.

**Quy tắc next()** (tìm hoán vị kế tiếp theo thứ tự từ điển):

```
1. Tìm chỉ số i lớn nhất sao cho p[i] < p[i+1]
   (điểm tăng cuối cùng bên phải — "dip point")
2. Tìm chỉ số j lớn nhất sao cho p[j] > p[i]
   (phần tử nhỏ nhất bên phải i mà lớn hơn p[i])
3. Hoán đổi p[i] và p[j]
4. Đảo ngược (reverse) phần hậu tố từ p[i+1] đến p[n-1]
```

```
(1,2,3) → (1,3,2) → (2,1,3) → (2,3,1) → (3,1,2) → (3,2,1)
```

#### Cài đặt (Go)

```go
// NextPermutation trả về false khi đã ở hoán vị cuối cùng
func NextPermutation(p []int) bool {
    n := len(p)
    // Bước 1: tìm điểm tăng bên phải nhất
    i := n - 2
    for i >= 0 && p[i] >= p[i+1] {
        i--
    }
    if i < 0 {
        return false // thứ tự giảm → hoán vị cuối cùng
    }
    // Bước 2: tìm phần tử bên phải nhất > p[i]
    j := n - 1
    for p[j] <= p[i] {
        j--
    }
    // Bước 3: hoán đổi
    p[i], p[j] = p[j], p[i]
    // Bước 4: đảo ngược hậu tố
    lo, hi := i+1, n-1
    for lo < hi {
        p[lo], p[hi] = p[hi], p[lo]
        lo++
        hi--
    }
    return true
}

// Sinh tất cả hoán vị của {1..n}
func GenPermutations(n int) [][]int {
    var result [][]int
    p := make([]int, n)
    for i := range p {
        p[i] = i + 1
    }
    for {
        cur := make([]int, n)
        copy(cur, p)
        result = append(result, cur)
        if !NextPermutation(p) {
            break
        }
    }
    return result
}
```

**Độ phức tạp**: O(n × n!) tổng thời gian, O(n) bộ nhớ.

### Vấn đề chung của phương pháp sinh

- Chỉ hoạt động khi tồn tại **thứ tự toàn phần** và `next()` tính được hiệu quả.
- Luôn duyệt **tất cả** cấu hình — không thể dừng sớm hay cắt tỉa.
- Phù hợp nhất cho bài toán kích thước nhỏ hoặc khi thực sự cần mọi cấu hình.

### Lỗi thường gặp

- Sai lệch chỉ số (off-by-one) khi tìm "vị trí bên phải nhất có thể tăng" — chỉ số 0-based và 1-based khác nhau.
- Quên **đảo ngược hậu tố** trong next permutation (chỉ swap là chưa đủ).
- Không nhận ra cấu hình **cuối cùng** → lặp vô hạn.
- Sinh trùng lặp khi đầu vào có phần tử lặp (cần logic next-permutation sửa đổi).

---

## 3. Thuật toán Quay lui (Backtracking)

### Định nghĩa / Ý tưởng

- **Định nghĩa**: Chiến lược thử-và-sai có hệ thống, xây dựng lời giải **từng thành phần một**, bỏ qua ("quay lui") lời giải bộ phận ngay khi phát hiện nó không thể dẫn đến lời giải hợp lệ/đầy đủ.
- **Ý tưởng**: Duyệt **cây tìm kiếm** theo chiều sâu (DFS). Tại mỗi nút, thử tất cả lựa chọn hợp lệ cho thành phần tiếp theo. Nếu gặp ngõ cụt, hoàn tác lựa chọn cuối và thử phương án khác.
- **Mục đích**: Liệt kê tất cả lời giải (hoặc tìm một lời giải) cho bài toán **thỏa mãn ràng buộc** và **tổ hợp** mà không cần sinh mọi cấu hình khả dĩ — cắt sớm nhánh vi phạm ràng buộc.

### Khi nào sử dụng

- Bài toán yêu cầu **sinh tất cả cấu hình hợp lệ** thỏa ràng buộc.
- Bài toán yêu cầu **một cấu hình hợp lệ** (dừng khi tìm được).
- Tìm kiếm tổ hợp: tập con, hoán vị, chỉnh hợp, phân hoạch.
- Giải puzzle: N-Queens, Sudoku, ô chữ.
- Bài toán đồ thị: tìm tất cả đường đi, đường Hamilton.

### Cách tiếp cận chính — Mẫu Quay lui

Lời giải được xây dựng dạng dãy `(x₁, x₂, …, xₙ)`. Tại bước `i`, thử tất cả ứng viên cho `xᵢ`:

```
procedure Try(i):
    for mỗi ứng viên v trong miền(xᵢ):
        if hợp_lệ(x₁, ..., x_{i-1}, v):   ← kiểm tra ràng buộc (cắt tỉa)
            x[i] = v
            if i == n:                       ← lời giải đầy đủ
                ghiNhậnKếtQuả()
            else:
                Try(i + 1)                   ← đệ quy sâu hơn
            hoàn_tác(x[i])                   ← quay lui (khôi phục trạng thái)
```

Đây thực chất là **DFS trên cây tìm kiếm** trong đó:
- Mỗi **mức** tương ứng với một thành phần của lời giải.
- Mỗi **nhánh** tại mức i tương ứng với một giá trị ứng viên cho xᵢ.
- **Cắt tỉa** xảy ra tại `hợp_lệ()` — bỏ qua nhánh không thể dẫn đến lời giải hợp lệ.

### Độ phức tạp

| Bài toán | Không gian tìm kiếm | Có cắt tỉa |
|---------|-------------|-------------|
| Dãy nhị phân độ dài n | 2ⁿ | 2ⁿ (không cắt tỉa được) |
| Tập con k phần tử | C(n,k) | C(n,k) |
| Hoán vị {1..n} | n! | n! (cắt tỉa nhẹ) |
| N-Queens | nⁿ (naïve) | Thực tế ít hơn nhiều (~O(n!) cận trên) |
| Sudoku | 9⁸¹ (naïve) | Cắt tỉa mạnh → khả thi |

> Quay lui KHÔNG thay đổi lớp độ phức tạp worst-case, nhưng **trên thực tế** cắt tỉa có thể giảm không gian duyệt hàng bậc.

---

### 3.1. Dãy nhị phân độ dài n (bản quay lui)

```go
func genBinary(n int, cur []int, result *[][]int) {
    if len(cur) == n {
        tmp := make([]int, n)
        copy(tmp, cur)
        *result = append(*result, tmp)
        return
    }
    for v := 0; v <= 1; v++ {
        cur = append(cur, v)
        genBinary(n, cur, result)
        cur = cur[:len(cur)-1] // quay lui
    }
}
```

---

### 3.2. Tập con k phần tử (bản quay lui)

```go
func genSubsets(n, k, start int, cur []int, result *[][]int) {
    if len(cur) == k {
        tmp := make([]int, k)
        copy(tmp, cur)
        *result = append(*result, tmp)
        return
    }
    // Cắt tỉa: số phần tử còn lại phải đủ để điền các vị trí trống
    remaining := n - start + 1
    needed := k - len(cur)
    if remaining < needed {
        return // cắt tỉa — không thể điền đủ
    }
    for v := start; v <= n; v++ {
        cur = append(cur, v)
        genSubsets(n, k, v+1, cur, result)
        cur = cur[:len(cur)-1] // quay lui
    }
}
```

---

### 3.3. Chỉnh hợp không lặp chập k của n

```go
func genArrangements(n, k int, cur []int, used []bool, result *[][]int) {
    if len(cur) == k {
        tmp := make([]int, k)
        copy(tmp, cur)
        *result = append(*result, tmp)
        return
    }
    for v := 1; v <= n; v++ {
        if !used[v] {
            used[v] = true
            cur = append(cur, v)
            genArrangements(n, k, cur, used, result)
            cur = cur[:len(cur)-1]
            used[v] = false // quay lui
        }
    }
}
```

---

### 3.4. Bài toán phân tích số

**Bài toán**: Liệt kê tất cả cách biểu diễn số nguyên dương `n` thành tổng các số nguyên dương (không phân biệt thứ tự — partitions).

> n = 5: `5 = 4+1 = 3+2 = 3+1+1 = 2+2+1 = 2+1+1+1 = 1+1+1+1+1` → 7 cách phân tích.

```go
func partitions(n, maxVal int, cur []int, result *[][]int) {
    if n == 0 {
        tmp := make([]int, len(cur))
        copy(tmp, cur)
        *result = append(*result, tmp)
        return
    }
    for v := min(n, maxVal); v >= 1; v-- {
        cur = append(cur, v)
        partitions(n-v, v, cur, result) // phần tiếp ≤ phần hiện tại (tránh trùng)
        cur = cur[:len(cur)-1]
    }
}

func min(a, b int) int {
    if a < b { return a }
    return b
}
```

---

### 3.5. Bài toán xếp hậu (N-Queens)

**Bài toán**: Đặt n quân hậu lên bàn cờ n×n sao cho không có hai quân hậu nào đe dọa nhau (không chung hàng, cột, hoặc đường chéo).

Đây là **bài toán quay lui kinh điển**. Đặt mỗi quân hậu trên mỗi hàng. Tại hàng `i`, thử từng cột `j` và kiểm tra:
- Cột `j` chưa dùng.
- Đường chéo chính `(i−j)` chưa dùng.
- Đường chéo phụ `(i+j)` chưa dùng.

```go
func solveNQueens(n int) [][]int {
    var solutions [][]int
    col := make([]int, n)         // col[hàng] = cột đặt quân hậu tại hàng đó
    usedCol := make([]bool, n)
    usedDiag := make([]bool, 2*n) // hàng - cột + n
    usedAnti := make([]bool, 2*n) // hàng + cột

    var solve func(row int)
    solve = func(row int) {
        if row == n {
            tmp := make([]int, n)
            copy(tmp, col)
            solutions = append(solutions, tmp)
            return
        }
        for c := 0; c < n; c++ {
            d, a := row-c+n, row+c
            if !usedCol[c] && !usedDiag[d] && !usedAnti[a] {
                col[row] = c
                usedCol[c], usedDiag[d], usedAnti[a] = true, true, true
                solve(row + 1)
                usedCol[c], usedDiag[d], usedAnti[a] = false, false, false
            }
        }
    }
    solve(0)
    return solutions
}
```

| n | Số lời giải | Số nút duyệt (xấp xỉ) |
|---|----------|-------------------------|
| 4 | 2 | ~60 |
| 8 | 92 | ~15,000 |
| 12 | 14,200 | ~1.7M |
| 14 | 365,596 | ~39M |

### Khái niệm tổng quát & Vấn đề của Quay lui

**Cây tìm kiếm (Search Tree)**:
Mọi thuật toán quay lui đều ngầm duyệt một **cây lời giải bộ phận**. Mỗi mức đại diện cho một biến quyết định. Mỗi nhánh đại diện cho một giá trị ứng viên. Nút lá hoặc là lời giải đầy đủ hoặc là ngõ cụt.

```
                  start
               /    |    \
            x₁=1  x₁=2  x₁=3
           / \     / \     / \
        x₂=..  x₂=.. x₂=..
        ...
```

**Cắt tỉa (pruning)** là yếu tố quyết định hiệu quả:
- **Cắt tỉa tính khả thi**: bỏ nhánh khi ràng buộc đã bị vi phạm.
- **Cắt tỉa đối xứng**: tránh duyệt cấu hình tương đương với cấu hình đã xét.
- **Cắt tỉa cận**: trong bài toán tối ưu, bỏ nhánh không thể cải thiện lời giải tốt nhất hiện tại (dẫn đến Nhánh cận — §4).

**Quản lý trạng thái** rất quan trọng:
- Phải **hoàn tác** mọi thay đổi khi quay lui. Quên khôi phục trạng thái là lỗi #1.
- Mẫu thường gặp: đánh dấu/bỏ đánh dấu mảng `used[]`, hoặc append/pop trên slice `cur`.

### Lỗi thường gặp

- **Quên hoàn tác trạng thái** sau lời gọi đệ quy → trạng thái hỏng, kết quả sai.
- **Không cắt tỉa đủ** → duyệt nhánh chết đến tận lá → TLE (vượt thời gian).
- **Sinh trùng lặp** khi đầu vào có phần tử lặp → sắp xếp đầu vào + bỏ qua phần tử bằng nhau cùng mức.
- **Sai chỉ số** (off-by-one) tại start index khi sinh tập con.
- **Stack đệ quy quá sâu** → với n rất lớn, cân nhắc quay lui lặp (iterative) với stack tường minh.

---

## 4. Kỹ thuật Nhánh cận (Branch and Bound)

### Định nghĩa / Ý tưởng

- **Định nghĩa**: Chiến lược quay lui cải tiến cho **bài toán tối ưu**. Tại mỗi nút trong cây tìm kiếm, tính **cận** (cận dưới cho bài toán cực tiểu / cận trên cho bài toán cực đại) của lời giải tốt nhất có thể đạt được từ nút đó. Nếu cận **tệ hơn** lời giải tốt nhất hiện biết → **cắt** toàn bộ cây con.
- **Ý tưởng**: Quay lui + **ước lượng lạc quan** (hàm cận). Nếu ngay cả kịch bản tốt nhất từ lời giải bộ phận cũng không thắng được "nhà vô địch" hiện tại, dừng ngay.
- **Mục đích**: Giải bài toán tối ưu tổ hợp (TSP, bài phân công, lập lịch) khi brute-force quá chậm nhưng không có thuật toán đa thức.

### Khi nào sử dụng

- Cần lời giải **tối ưu** (min hoặc max), không chỉ lời giải hợp lệ bất kỳ.
- Không gian lời giải quá lớn để duyệt toàn bộ.
- Có thể tính **cận chặt** một cách hiệu quả (cận càng chặt, cắt tỉa càng mạnh).
- Bài toán NP-hard và cần lời giải **chính xác** cho kích thước đầu vào vừa phải.

### Cách tiếp cận chính

```
procedure NhánhCận(i, lờiGiảiBộPhận):
    if i == n:                                       // lời giải đầy đủ
        if chiPhí(lờiGiảiBộPhận) < chiPhíTốtNhất:
            chiPhíTốtNhất = chiPhí(lờiGiảiBộPhận)
            lờiGiảiTốtNhất = lờiGiảiBộPhận
        return

    for mỗi ứng viên v trong miền(xᵢ):
        if hợp_lệ(lờiGiảiBộPhận, v):
            lờiGiảiBộPhận[i] = v
            cận = cậnDưới(lờiGiảiBộPhận, i)          ← BƯỚC THEN CHỐT
            if cận < chiPhíTốtNhất:                   ← có đáng duyệt?
                NhánhCận(i + 1, lờiGiảiBộPhận)
            hoàn_tác(lờiGiảiBộPhận[i])
```

**Khác biệt then chốt so với quay lui**: hàm `cậnDưới()`. Đây là "cận" — **ước lượng lạc quan** của lời giải tốt nhất có thể đạt được từ trạng thái hiện tại. Nếu ước lượng lạc quan này còn tệ hơn lời giải hiện có → cắt.

### Bùng nổ tổ hợp

| n | Hoán vị (n!) | Tập con (2ⁿ) |
|---|-------------------|--------------|
| 10 | 3,628,800 | 1,024 |
| 15 | 1.3 × 10¹² | 32,768 |
| 20 | 2.4 × 10¹⁸ | 1,048,576 |
| 25 | 1.5 × 10²⁵ | 33,554,432 |

> Không cắt tỉa, ngay cả bài toán O(n!) với n = 15 đã bất khả thi. Nhánh cận giúp tìm lời giải chính xác khả thi cho n lên đến 20–30 trong nhiều bài toán.

---

### 4.1. Bài toán người du lịch (TSP)

**Bài toán**: Cho n thành phố và khoảng cách giữa mọi cặp, tìm hành trình ngắn nhất ghé thăm mỗi thành phố đúng một lần rồi quay về điểm xuất phát.

- Brute-force: (n−1)! hành trình.
- Nhánh cận: cắt hành trình có chi phí bộ phận đã vượt lời giải tốt nhất.

**Ý tưởng cận dưới**: chi phí đường đi hiện tại + tổng cạnh đi ra nhỏ nhất của mỗi thành phố chưa thăm.

```go
func tspBranchBound(dist [][]int, n int) int {
    visited := make([]bool, n)
    visited[0] = true
    bestCost := 1<<63 - 1

    // Cận dưới: chi phí hiện tại + cạnh đi ra nhỏ nhất của các nút chưa thăm
    lowerBound := func(curCost int) int {
        lb := curCost
        for i := 0; i < n; i++ {
            if !visited[i] {
                minEdge := 1<<63 - 1
                for j := 0; j < n; j++ {
                    if i != j && dist[i][j] < minEdge {
                        minEdge = dist[i][j]
                    }
                }
                lb += minEdge
            }
        }
        return lb
    }

    var solve func(city, count, cost int)
    solve = func(city, count, cost int) {
        if count == n {
            total := cost + dist[city][0] // quay về điểm xuất phát
            if total < bestCost {
                bestCost = total
            }
            return
        }
        for next := 0; next < n; next++ {
            if !visited[next] {
                newCost := cost + dist[city][next]
                // Nhánh cận: cắt nếu cận dưới >= bestCost
                visited[next] = true
                if lowerBound(newCost) < bestCost {
                    solve(next, count+1, newCost)
                }
                visited[next] = false
            }
        }
    }

    solve(0, 1, 0)
    return bestCost
}
```

---

### 4.2. Dãy ABC

**Bài toán**: Tìm dãy độ dài n sử dụng ký tự {A, B, C} (ánh xạ {1, 2, 3}) sao cho tối thiểu hóa hàm chi phí, thỏa mãn ràng buộc kề.

Bài này minh họa cách nhánh cận áp dụng cho **tối ưu dãy** — mỗi vị trí là biến quyết định với miền {1, 2, 3}, hàm cận ước lượng chi phí còn lại nhỏ nhất.

```go
// Khung nhánh cận tổng quát cho tối ưu dãy
func sequenceBnB(n int, cost func([]int) int, bound func([]int, int) int) []int {
    best := make([]int, n)
    bestCost := 1<<63 - 1
    cur := make([]int, 0, n)

    var solve func(depth int)
    solve = func(depth int) {
        if depth == n {
            c := cost(cur)
            if c < bestCost {
                bestCost = c
                copy(best, cur)
            }
            return
        }
        for v := 1; v <= 3; v++ {
            cur = append(cur, v)
            if bound(cur, depth) < bestCost {
                solve(depth + 1)
            }
            cur = cur[:len(cur)-1]
        }
    }

    solve(0)
    return best
}
```

### Khái niệm tổng quát & Vấn đề của Nhánh cận

**Chất lượng hàm cận quyết định tất cả**:
- **Cận tầm thường** (ví dụ: 0 cho bài cực tiểu) không bao giờ cắt → thoái hóa thành quay lui.
- **Cận chặt** cắt tỉa mạnh → giảm đáng kể số nút duyệt.
- Việc tính cận phải **nhanh** — nếu tốn kém bằng giải bài toán con, ta không được lợi gì.

**Cận trên ban đầu**:
- Dùng **heuristic tham lam** để có `bestCost` tốt ngay từ đầu. Giúp cắt tỉa sớm ngay từ các mức đầu tiên.
- Ví dụ trong TSP: dùng heuristic "láng giềng gần nhất" (nearest-neighbour) làm chi phí hành trình khởi tạo.

**Thứ tự duyệt có ý nghĩa**:
- DFS (tiêu chuẩn): bộ nhớ thấp, nhưng có thể duyệt nhánh xấu trước.
- Best-First Search: mở rộng nút có cận tốt nhất trước — thường tìm được lời giải tối ưu sớm hơn.
- Sách tập trung vào nhánh cận dạng DFS.

### Độ phức tạp

| Khía cạnh | Chi tiết |
|--------|--------|
| Worst case | Bằng brute-force (cắt tỉa không giúp được) |
| Average case | Nhanh hơn brute-force hàng bậc |
| Bộ nhớ (DFS) | O(n) — độ sâu cây tìm kiếm |
| Bộ nhớ (Best-first) | O(2ⁿ) — nhiều nút mở |

### Lỗi thường gặp

- **Cận quá lỏng** → hầu như không cắt → thực chất là brute-force với overhead.
- **Tính cận quá tốn kém** → chi phí tính > lợi ích cắt tỉa.
- **Quên cập nhật bestCost** khi tìm được lời giải đầy đủ.
- **Không dùng lời giải tham lam ban đầu** → bestCost bắt đầu = ∞, các mức đầu không được cắt.
- **Không khôi phục trạng thái** khi quay lui (giống lỗi quay lui).

---

## So sánh ba phương pháp liệt kê

| Khía cạnh | Phương pháp Sinh | Quay lui | Nhánh cận |
|--------|-----------|-------------|----------------|
| Chiến lược | Lặp next() | DFS đệ quy | DFS đệ quy + hàm cận |
| Cắt tỉa | Không | Chỉ tính khả thi | Tính khả thi + cận tối ưu |
| Loại bài toán | Liệt kê tất cả | Liệt kê hợp lệ | Tìm cấu hình tối ưu |
| Bộ nhớ | O(cấu hình hiện tại) | O(độ sâu) call stack | O(độ sâu) call stack |
| Xử lý ràng buộc | Không | Có | Có + tối ưu hóa |
| Ví dụ điển hình | Tập con, hoán vị | N-Queens, Sudoku | TSP, phân công, lập lịch |

---

## Bài tập (LeetCode)

### Tổ hợp + Phương pháp Sinh

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Next Permutation | Sinh (next()) | [31](https://leetcode.com/problems/next-permutation/) |
| Permutations | Quay lui | [46](https://leetcode.com/problems/permutations/) |
| Permutations II (có lặp) | Quay lui + bỏ qua trùng | [47](https://leetcode.com/problems/permutations-ii/) |
| Subsets | Quay lui | [78](https://leetcode.com/problems/subsets/) |
| Subsets II (có lặp) | Quay lui + bỏ qua trùng | [90](https://leetcode.com/problems/subsets-ii/) |
| Combinations | Quay lui | [77](https://leetcode.com/problems/combinations/) |
| Combination Sum | Quay lui + cắt tỉa | [39](https://leetcode.com/problems/combination-sum/) |
| Combination Sum II | Quay lui + bỏ qua trùng | [40](https://leetcode.com/problems/combination-sum-ii/) |

### Quay lui — Bài toán kinh điển

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| N-Queens | Quay lui + ràng buộc | [51](https://leetcode.com/problems/n-queens/) |
| N-Queens II (đếm) | Quay lui | [52](https://leetcode.com/problems/n-queens-ii/) |
| Sudoku Solver | Quay lui + cắt tỉa mạnh | [37](https://leetcode.com/problems/sudoku-solver/) |
| Generate Parentheses | Quay lui + ràng buộc cân bằng | [22](https://leetcode.com/problems/generate-parentheses/) |
| Letter Combinations of Phone | Quay lui | [17](https://leetcode.com/problems/letter-combinations-of-a-phone-number/) |
| Palindrome Partitioning | Quay lui | [131](https://leetcode.com/problems/palindrome-partitioning/) |
| Word Search | Quay lui trên lưới | [79](https://leetcode.com/problems/word-search/) |
| Restore IP Addresses | Quay lui + ràng buộc | [93](https://leetcode.com/problems/restore-ip-addresses/) |
| Partition to K Equal Sum Subsets | Quay lui + cắt tỉa | [698](https://leetcode.com/problems/partition-to-k-equal-sum-subsets/) |

### Quay lui — Đường đi & Đồ thị

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| All Paths From Source to Target | Quay lui trên DAG | [797](https://leetcode.com/problems/all-paths-from-source-to-target/) |
| Permutation Sequence (thứ k) | Toán + sinh | [60](https://leetcode.com/problems/permutation-sequence/) |

### Nhánh cận / Tối ưu

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Partition Equal Subset Sum | DP (nhánh cận quá chậm) | [416](https://leetcode.com/problems/partition-equal-subset-sum/) |
| Travelling Salesman (Bitmask DP) | DP hiệu quả hơn nhánh cận | [943](https://leetcode.com/problems/find-the-shortest-superstring/) |
| Stickers to Spell Word | Quay lui + memo | [691](https://leetcode.com/problems/stickers-to-spell-word/) |

> **Lưu ý**: Nhánh cận thuần hiếm khi xuất hiện trên LeetCode — hầu hết bài tối ưu được giải bằng DP hoặc tham lam. Nhánh cận phổ biến hơn trong thi lập trình và vận trù học thực tế.

---

## Mẫu (pattern) liên quan

- **DFS / BFS**: Quay lui chính LÀ tìm kiếm theo chiều sâu trên cây tìm kiếm ngầm.
- **Quy hoạch động (DP)**: Khi bài toán con chồng lấn, ghi nhớ (memo) thay vì duyệt lại → DP thay thế quay lui naïve.
- **Tham lam**: Khi lựa chọn tối ưu cục bộ dẫn đến tối ưu toàn cục → tham lam thay thế liệt kê toàn bộ.
- **Bitmask**: Biểu diễn tập con bằng bitmask cho trạng thái gọn và phép toán tập hợp nhanh (dùng nhiều trong DP trên tập con).
- **Lan truyền ràng buộc (Constraint Propagation)**: Cắt tỉa nâng cao (ví dụ: Arc Consistency trong Sudoku) — thu hẹp miền trước khi phân nhánh.

---

## Ghi nhớ nhanh

### Tổ hợp
- A'(n,k) = nᵏ (có lặp) | A(n,k) = n!/(n−k)! (không lặp) | P(n) = n! | C(n,k) = n!/[k!(n−k)!]

### Phương pháp Sinh
- Xác định **đầu tiên**, **cuối cùng**, và **next()**. Lặp. Không đệ quy.
- Next Permutation: tìm điểm tăng bên phải nhất → swap với phần tử lớn hơn nhỏ nhất → đảo hậu tố.

### Quay lui
- DFS trên cây tìm kiếm. Thử → Đệ quy → Hoàn tác.
- **Luôn khôi phục trạng thái** sau lời gọi đệ quy.
- Cắt tỉa sớm bằng kiểm tra ràng buộc trước khi đệ quy.
- Tránh trùng khi đầu vào có lặp: sắp xếp + bỏ qua phần tử bằng nhau cùng mức.

### Nhánh cận
- Quay lui + hàm cận dưới/trên.
- Nếu cận ≥ bestCost → cắt toàn bộ cây con.
- Chất lượng hàm cận = chất lượng cắt tỉa.
- Dùng heuristic tham lam cho bestCost ban đầu.

### Mẫu tổng quát

```go
func backtrack(state State, depth int) {
    if isComplete(state) {
        process(state)
        return
    }
    for _, candidate := range getCandidates(state, depth) {
        if isValid(state, candidate) {               // cắt tỉa tính khả thi
            applyChoice(state, candidate)
            // Nhánh cận: if lowerBound(state) < bestCost {
            backtrack(state, depth+1)
            // }
            undoChoice(state, candidate)              // LUÔN hoàn tác
        }
    }
}
```

---

## Học tiếp

- [03-recursion.md](03-recursion.md) — Đệ quy chi tiết — cơ chế nền tảng của quay lui
- [08-search.md](08-search.md) — Chiến lược tìm kiếm BFS/DFS là nền tảng của liệt kê
- Phần 3 (Quy Hoạch Động) — Khi bài toán con chồng lấn khiến quay lui kém hiệu quả → chuyển sang DP
