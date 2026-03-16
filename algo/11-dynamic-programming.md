# Quy hoạch động (Dynamic Programming)

> **Chủ đề**: Thuật toán | **Mức độ**: Trung bình → Nâng cao
> **Nguồn**: *Giải thuật và Lập trình* — Lê Minh Hoàng, Phần 3

---

## Tổng quan

**Quy hoạch động (QHĐ)** là phương pháp giải bài toán bằng cách **chia thành các bài toán con chồng lấn**, giải mỗi bài toán con **đúng một lần**, lưu kết quả, và tổng hợp thành lời giải bài toán gốc.

QHĐ biến thuật toán **đệ quy mũ** thành thuật toán **đa thức** — là một trong những kỹ thuật mạnh nhất trong lập trình thi đấu và phỏng vấn.

```
Đệ quy naïve: tính lại bài toán con → O(2ⁿ)
QHĐ: lưu kết quả bài toán con    → O(n), O(n²), O(n³)
```

---

## Tại sao cần học

- Hàng trăm bài toán trên LeetCode, Codeforces, ICPC yêu cầu QHĐ.
- QHĐ là **bước tiến hóa tự nhiên** từ đệ quy (Phần 2, §3) và quay lui (Phần 1, §3).
- Nhiều bài toán thực tế (lộ trình, tối ưu tài nguyên, xử lý chuỗi, sinh học tính toán) có lời giải hiệu quả nhờ QHĐ.
- Nắm vững QHĐ giúp nhận diện **cấu trúc bài toán con tối ưu** — kỹ năng phân tích quan trọng nhất.

---

## Mục lục

1. [§1 — Công thức truy hồi](#1-công-thức-truy-hồi)
2. [§2 — Phương pháp Quy hoạch động](#2-phương-pháp-quy-hoạch-động)
3. [§3 — Các bài toán QHĐ kinh điển](#3-các-bài-toán-qhđ-kinh-điển)

---

## §1 — Công thức truy hồi

### Định nghĩa / Ý tưởng

**Công thức truy hồi** (recurrence relation) là công thức biểu diễn giá trị tại bước `n` qua các giá trị tại bước nhỏ hơn, kèm **điều kiện ban đầu** (base case).

$$a_n = f(a_{n-1}, a_{n-2}, \ldots, a_{n-k})$$

Đây là **nền tảng toán học** của QHĐ — mỗi bài QHĐ đều bắt đầu bằng việc thiết lập công thức truy hồi.

### Ví dụ kinh điển

#### 1.1. Dãy Fibonacci

$$F(n) = F(n-1) + F(n-2), \quad F(0) = 0, \; F(1) = 1$$

```
F: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, ...
```

#### Đệ quy naïve — O(2ⁿ)

```go
func FibNaive(n int) int {
    if n <= 1 {
        return n
    }
    return FibNaive(n-1) + FibNaive(n-2)
}
```

**Vấn đề**: Cây đệ quy có **bài toán con chồng lấn** — cùng một F(k) bị tính lại nhiều lần.

```
                    F(5)
                   /    \
                F(4)    F(3)        ← F(3) tính 2 lần
               /   \    /   \
            F(3)  F(2) F(2) F(1)   ← F(2) tính 3 lần
           /  \
        F(2)  F(1)
```

Số lời gọi: xấp xỉ $2^n$ → **cực kỳ chậm** cho n lớn.

#### Ghi nhớ (Top-down Memoization) — O(n)

```go
func FibMemo(n int, memo map[int]int) int {
    if n <= 1 {
        return n
    }
    if v, ok := memo[n]; ok {
        return v // đã tính rồi, trả về ngay
    }
    memo[n] = FibMemo(n-1, memo) + FibMemo(n-2, memo)
    return memo[n]
}

// Sử dụng:
// memo := make(map[int]int)
// result := FibMemo(50, memo) // chạy tức thì
```

#### Bảng phương (Bottom-up Tabulation) — O(n)

```go
func FibTab(n int) int {
    if n <= 1 {
        return n
    }
    dp := make([]int, n+1)
    dp[0], dp[1] = 0, 1
    for i := 2; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }
    return dp[n]
}
```

#### Tối ưu bộ nhớ — O(1) không gian

```go
func FibOpt(n int) int {
    if n <= 1 {
        return n
    }
    a, b := 0, 1
    for i := 2; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}
```

> **Bài học**: Từ O(2ⁿ) thời gian → O(n) thời gian, O(1) bộ nhớ — chỉ bằng cách **nhận ra bài toán con chồng lấn** và **lưu kết quả**.

---

#### 1.2. Tổ hợp — Tam giác Pascal

$$C(n, k) = C(n-1, k-1) + C(n-1, k)$$

Base case: $C(n, 0) = C(n, n) = 1$

```
          1
         1 1
        1 2 1
       1 3 3 1
      1 4 6 4 1
```

```go
func Comb(n, k int) int {
    // Bảng 2D (bottom-up)
    dp := make([][]int, n+1)
    for i := range dp {
        dp[i] = make([]int, k+1)
        dp[i][0] = 1
        if i <= k {
            dp[i][i] = 1
        }
    }
    for i := 2; i <= n; i++ {
        for j := 1; j < i && j <= k; j++ {
            dp[i][j] = dp[i-1][j-1] + dp[i-1][j]
        }
    }
    return dp[n][k]
}
```

Tối ưu bộ nhớ — chỉ cần 1 hàng:

```go
func CombOpt(n, k int) int {
    dp := make([]int, k+1)
    dp[0] = 1
    for i := 1; i <= n; i++ {
        // Duyệt NGƯỢC để không ghi đè giá trị hàng cũ
        for j := min(i, k); j >= 1; j-- {
            dp[j] += dp[j-1]
        }
    }
    return dp[k]
}
```

---

#### 1.3. Dãy Catalan

$$C_n = \sum_{i=0}^{n-1} C_i \cdot C_{n-1-i} = \frac{1}{n+1}\binom{2n}{n}$$

Base case: $C_0 = 1$

**Ứng dụng**: số cây nhị phân có n nút, số cách đặt ngoặc, số đường đi lưới không vượt đường chéo.

```go
func Catalan(n int) int {
    dp := make([]int, n+1)
    dp[0] = 1
    for i := 1; i <= n; i++ {
        for j := 0; j < i; j++ {
            dp[i] += dp[j] * dp[i-1-j]
        }
    }
    return dp[n]
}
```

---

#### 1.4. Hệ số Stirling loại 2

Số cách phân hoạch tập n phần tử thành đúng k tập con không rỗng:

$$S(n, k) = k \cdot S(n-1, k) + S(n-1, k-1)$$

Base case: $S(0, 0) = 1$, $S(n, 0) = 0$ với n > 0.

```go
func Stirling2(n, k int) int {
    dp := make([][]int, n+1)
    for i := range dp {
        dp[i] = make([]int, k+1)
    }
    dp[0][0] = 1
    for i := 1; i <= n; i++ {
        for j := 1; j <= min(i, k); j++ {
            dp[i][j] = j*dp[i-1][j] + dp[i-1][j-1]
        }
    }
    return dp[n][k]
}
```

---

### Phương pháp giải công thức truy hồi

| Phương pháp | Áp dụng khi | Ví dụ |
|-------------|------------|-------|
| **Thế trực tiếp** | Dễ nhận dạng pattern | Fibonacci → tỉ số vàng |
| **Phương trình đặc trưng** | Truy hồi tuyến tính hệ số hằng | $a_n = 5a_{n-1} - 6a_{n-2}$ → $r² - 5r + 6 = 0$ |
| **Hàm sinh (Generating functions)** | Truy hồi phức tạp | Catalan |
| **Master Theorem** | Dạng chia để trị T(n) = aT(n/b) + f(n) | Merge sort |

### Ghi nhớ nhanh — §1

- Mọi bài QHĐ đều bắt đầu bằng **công thức truy hồi** + **base case**.
- Truy hồi chỉ là **mô tả toán học** — QHĐ là **cách tính hiệu quả**.
- Kiểm tra công thức bằng **thế vài giá trị nhỏ** trước khi cài đặt.

---

## §2 — Phương pháp Quy hoạch động

### Định nghĩa / Ý tưởng

**Quy hoạch động** là kỹ thuật thiết kế giải thuật dựa trên hai tính chất:

1. **Bài toán con tối ưu (Optimal Substructure)**: Lời giải tối ưu của bài toán chứa lời giải tối ưu của các bài toán con.
2. **Bài toán con chồng lấn (Overlapping Subproblems)**: Cùng một bài toán con được giải nhiều lần trong quá trình đệ quy.

> Nếu chỉ có (1) mà không có (2) → dùng **chia để trị** (Merge sort).
> Nếu có cả (1) và (2) → dùng **QHĐ**.

### Hai cách tiếp cận

| | Top-down (Ghi nhớ) | Bottom-up (Bảng phương) |
|---|---------------------|------------------------|
| **Cách hoạt động** | Đệ quy + cache kết quả đã tính | Điền bảng từ base case lên |
| **Thứ tự tính** | Chỉ tính bài toán con **cần dùng** | Tính **tất cả** bài toán con theo thứ tự |
| **Cài đặt** | Dễ — giữ cấu trúc đệ quy, thêm memo | Cần xác định **thứ tự điền bảng** |
| **Bộ nhớ** | Có thể tiết kiệm nếu ít bài toán con thực sự dùng | Dễ tối ưu (cuộn mảng — rolling array) |
| **Stack** | Có thể stack overflow nếu quá sâu | Không dùng stack đệ quy |
| **Khi nào chọn** | Bài toán con thưa, khó xác định thứ tự | Mọi bài toán con đều cần, muốn tối ưu bộ nhớ |

### Quy trình 5 bước giải bài QHĐ

```
Bước 1: Xác định TRẠNG THÁI (state)
        → "Bài toán con cần giải là gì?"
        → Định nghĩa dp[i], dp[i][j], dp[mask], ...

Bước 2: Thiết lập CÔNG THỨC CHUYỂN TRẠNG THÁI (transition)
        → "dp[i] được tính từ dp[?] như thế nào?"
        → Đây là công thức truy hồi.

Bước 3: Xác định BASE CASE
        → "Trạng thái nhỏ nhất / đơn giản nhất?"
        → dp[0] = ?, dp[0][0] = ?, ...

Bước 4: Xác định THỨ TỰ TÍNH
        → "Tính dp[i] trước dp[j] nếu dp[i] phụ thuộc dp[j]"
        → Thường: chỉ số nhỏ trước, chỉ số lớn sau.

Bước 5: Xác định ĐÁP ÁN
        → "Kết quả cuối nằm ở đâu trong bảng?"
        → dp[n]? max(dp[...])? dp[n][W]?
```

### Ví dụ minh họa — Bài toán leo cầu thang

**Đề bài**: Bạn đang ở bậc 0, cần lên bậc `n`. Mỗi bước có thể bước 1 hoặc 2 bậc. Hỏi có bao nhiêu cách?

**Áp dụng 5 bước**:

| Bước | Nội dung |
|------|---------|
| 1. Trạng thái | `dp[i]` = số cách đến bậc thứ `i` |
| 2. Chuyển trạng thái | `dp[i] = dp[i-1] + dp[i-2]` (bước 1 từ bậc i−1, hoặc bước 2 từ bậc i−2) |
| 3. Base case | `dp[0] = 1` (1 cách: đứng yên), `dp[1] = 1` (1 cách: bước 1) |
| 4. Thứ tự | i = 2, 3, ..., n |
| 5. Đáp án | `dp[n]` |

```go
// LeetCode 70 — Climbing Stairs
func climbStairs(n int) int {
    if n <= 1 {
        return 1
    }
    dp := make([]int, n+1)
    dp[0], dp[1] = 1, 1
    for i := 2; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }
    return dp[n]
}
```

Tối ưu — O(1) bộ nhớ:

```go
func climbStairsOpt(n int) int {
    if n <= 1 {
        return 1
    }
    a, b := 1, 1
    for i := 2; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}
```

### Nhận diện bài toán QHĐ

| Dấu hiệu | Ví dụ |
|----------|-------|
| Hỏi **số cách** / **đếm** | "Bao nhiêu cách chia?" |
| Hỏi **tối ưu** (min/max) | "Chi phí nhỏ nhất?" |
| Có ràng buộc **chọn/không chọn** | "Chọn tập con thỏa điều kiện" |
| Bài toán trên **dãy**, **chuỗi**, **lưới** | "Đường đi trên bảng" |
| Đệ quy naïve bị **TLE** | → Thêm memo hoặc chuyển bottom-up |
| Input nhỏ nhưng brute-force quá chậm | n ≤ 1000 nhưng O(2ⁿ) không chấp nhận |

### Các dạng QHĐ phổ biến

| Dạng | Trạng thái | Ví dụ |
|------|-----------|-------|
| **1D** | `dp[i]` | Leo cầu thang, Fibonacci, House Robber |
| **2D** | `dp[i][j]` | Knapsack, Edit Distance, đường đi lưới |
| **Khoảng (Interval)** | `dp[i][j]` = tối ưu trên đoạn [i..j] | Nhân chuỗi ma trận, Burst Balloons |
| **Bitmask** | `dp[mask]` hoặc `dp[mask][i]` | TSP, phân công |
| **Trên cây** | `dp[node]` | Phủ cây, Independent Set trên cây |
| **Trên DAG** | `dp[node]` | Đường đi dài nhất trên DAG |
| **Chữ số (Digit DP)** | `dp[pos][tight][...]` | Đếm số trong khoảng thỏa tính chất |

### Kỹ thuật tối ưu bộ nhớ — Cuộn mảng (Rolling Array)

Khi `dp[i]` chỉ phụ thuộc vào `dp[i-1]` (hoặc vài hàng trước), không cần lưu toàn bộ bảng.

**Nguyên tắc**: Nếu `dp[i]` chỉ cần `dp[i-1]` → chỉ giữ 2 biến.
Nếu `dp[i][j]` chỉ cần hàng `dp[i-1][...]` → chỉ giữ 2 hàng (hoặc 1 hàng + duyệt ngược).

```go
// 2D → 1D: dp[i][j] phụ thuộc dp[i-1][j] và dp[i-1][j-1]
// Chỉ cần 1 hàng, duyệt NGƯỢC j
for i := 1; i <= n; i++ {
    for j := W; j >= weight[i]; j-- {
        dp[j] = max(dp[j], dp[j-weight[i]]+value[i])
    }
}
```

### Lỗi thường gặp — §2

- **Sai trạng thái**: thiếu chiều → không phân biệt được các trường hợp.
- **Sai base case**: dp[0] = 0 hay dp[0] = 1? Tùy bài toán!
- **Sai thứ tự tính**: dùng giá trị chưa tính xong → kết quả sai.
- **Quên cuộn mảng đúng chiều**: phải duyệt **ngược** khi tối ưu từ 2D xuống 1D trong knapsack 0/1.
- **Nhầm tối ưu / đếm**: bài đếm dùng `+=`, bài tối ưu dùng `min()`/`max()`.

---

## §3 — Các bài toán QHĐ kinh điển

### 3.1. Dãy con tăng dài nhất (LIS — Longest Increasing Subsequence)

**Đề bài**: Cho mảng `a[0..n-1]`, tìm độ dài **dãy con** (không nhất thiết liên tục) tăng ngặt dài nhất.

```
a = [10, 9, 2, 5, 3, 7, 101, 18]
LIS = [2, 3, 7, 101] → độ dài 4
```

#### Cách 1: QHĐ O(n²)

**Trạng thái**: `dp[i]` = độ dài LIS kết thúc tại `a[i]`.

**Chuyển trạng thái**:

$$dp[i] = \max_{j < i, \; a[j] < a[i]} (dp[j]) + 1$$

**Base case**: `dp[i] = 1` ∀i (mỗi phần tử tự nó là LIS độ dài 1).

**Đáp án**: `max(dp[0..n-1])`.

```go
// LeetCode 300 — Longest Increasing Subsequence (O(n²))
func lengthOfLIS(nums []int) int {
    n := len(nums)
    if n == 0 {
        return 0
    }
    dp := make([]int, n)
    for i := range dp {
        dp[i] = 1
    }

    best := 1
    for i := 1; i < n; i++ {
        for j := 0; j < i; j++ {
            if nums[j] < nums[i] && dp[j]+1 > dp[i] {
                dp[i] = dp[j] + 1
            }
        }
        if dp[i] > best {
            best = dp[i]
        }
    }
    return best
}
```

**Truy vết** — tìm dãy con thực tế:

```go
func findLIS(nums []int) []int {
    n := len(nums)
    dp := make([]int, n)
    prev := make([]int, n) // lưu vị trí trước đó
    for i := range dp {
        dp[i] = 1
        prev[i] = -1
    }

    bestIdx := 0
    for i := 1; i < n; i++ {
        for j := 0; j < i; j++ {
            if nums[j] < nums[i] && dp[j]+1 > dp[i] {
                dp[i] = dp[j] + 1
                prev[i] = j
            }
        }
        if dp[i] > dp[bestIdx] {
            bestIdx = i
        }
    }

    // Truy vết ngược
    var lis []int
    for idx := bestIdx; idx != -1; idx = prev[idx] {
        lis = append(lis, nums[idx])
    }
    // Đảo ngược
    for i, j := 0, len(lis)-1; i < j; i, j = i+1, j-1 {
        lis[i], lis[j] = lis[j], lis[i]
    }
    return lis
}
```

#### Cách 2: Tìm nhị phân + Patience Sorting — O(n log n)

Duy trì mảng `tails[]` trong đó `tails[i]` = phần tử cuối nhỏ nhất của mọi dãy tăng độ dài `i+1`.

```go
// LIS O(n log n)
func lengthOfLISFast(nums []int) int {
    tails := []int{}
    for _, x := range nums {
        // Tìm vị trí đầu tiên trong tails >= x (lower bound)
        lo, hi := 0, len(tails)
        for lo < hi {
            mid := lo + (hi-lo)/2
            if tails[mid] < x {
                lo = mid + 1
            } else {
                hi = mid
            }
        }
        if lo == len(tails) {
            tails = append(tails, x) // mở rộng LIS
        } else {
            tails[lo] = x // cập nhật tail nhỏ hơn
        }
    }
    return len(tails)
}
```

**Truy vết chi tiết**:

| i | nums[i] | tails (sau) | Hành động |
|---|---------|-------------|-----------|
| 0 | 10 | [10] | append |
| 1 | 9 | [9] | thay tails[0] |
| 2 | 2 | [2] | thay tails[0] |
| 3 | 5 | [2, 5] | append |
| 4 | 3 | [2, 3] | thay tails[1] |
| 5 | 7 | [2, 3, 7] | append |
| 6 | 101 | [2, 3, 7, 101] | append |
| 7 | 18 | [2, 3, 7, 18] | thay tails[3] |

Kết quả: `len(tails) = 4` ✓

> **Chú ý**: mảng `tails` cuối cùng KHÔNG phải LIS thực tế — chỉ cho biết **độ dài**. Cần thêm mảng phụ để truy vết.

---

### 3.2. Bài toán ba lô (Knapsack)

#### 3.2.1. Ba lô 0/1 (0/1 Knapsack)

**Đề bài**: Có `n` vật phẩm, mỗi vật có khối lượng `w[i]` và giá trị `v[i]`. Ba lô chịu được tối đa `W` khối lượng. Mỗi vật **chọn hoặc không** (không chia nhỏ). Tìm tổng giá trị lớn nhất.

**Trạng thái**: `dp[i][j]` = giá trị lớn nhất khi xét `i` vật đầu, ba lô còn chứa được `j` khối lượng.

**Chuyển trạng thái**:

$$dp[i][j] = \max\big(dp[i-1][j], \; dp[i-1][j-w_i] + v_i\big)$$

- `dp[i-1][j]`: **không chọn** vật i.
- `dp[i-1][j-w[i]] + v[i]`: **chọn** vật i (nếu `j ≥ w[i]`).

**Base case**: `dp[0][j] = 0` ∀j.

**Đáp án**: `dp[n][W]`.

```go
// 0/1 Knapsack — O(nW) thời gian, O(nW) bộ nhớ
func knapsack01(weights, values []int, W int) int {
    n := len(weights)
    dp := make([][]int, n+1)
    for i := range dp {
        dp[i] = make([]int, W+1)
    }

    for i := 1; i <= n; i++ {
        wi, vi := weights[i-1], values[i-1]
        for j := 0; j <= W; j++ {
            dp[i][j] = dp[i-1][j] // không chọn
            if j >= wi && dp[i-1][j-wi]+vi > dp[i][j] {
                dp[i][j] = dp[i-1][j-wi] + vi // chọn
            }
        }
    }
    return dp[n][W]
}
```

**Tối ưu bộ nhớ — 1D** (cuộn mảng, duyệt **ngược** j):

```go
// 0/1 Knapsack — O(nW) thời gian, O(W) bộ nhớ
func knapsack01Opt(weights, values []int, W int) int {
    dp := make([]int, W+1)

    for i := 0; i < len(weights); i++ {
        wi, vi := weights[i], values[i]
        // NGƯỢC: đảm bảo mỗi vật chỉ chọn tối đa 1 lần
        for j := W; j >= wi; j-- {
            if dp[j-wi]+vi > dp[j] {
                dp[j] = dp[j-wi] + vi
            }
        }
    }
    return dp[W]
}
```

> **Tại sao duyệt ngược?** Nếu duyệt xuôi, `dp[j-wi]` có thể đã được cập nhật bằng giá trị của hàng hiện tại → vật i bị chọn nhiều lần (giống Unbounded Knapsack).

**Truy vết** — tìm vật nào được chọn:

```go
func knapsack01WithTrace(weights, values []int, W int) (int, []int) {
    n := len(weights)
    dp := make([][]int, n+1)
    for i := range dp {
        dp[i] = make([]int, W+1)
    }
    for i := 1; i <= n; i++ {
        wi, vi := weights[i-1], values[i-1]
        for j := 0; j <= W; j++ {
            dp[i][j] = dp[i-1][j]
            if j >= wi && dp[i-1][j-wi]+vi > dp[i][j] {
                dp[i][j] = dp[i-1][j-wi] + vi
            }
        }
    }

    // Truy vết
    var chosen []int
    j := W
    for i := n; i >= 1; i-- {
        if dp[i][j] != dp[i-1][j] {
            chosen = append(chosen, i-1) // vật i-1 (0-indexed) được chọn
            j -= weights[i-1]
        }
    }
    return dp[n][W], chosen
}
```

#### 3.2.2. Ba lô không giới hạn (Unbounded Knapsack)

Mỗi vật có thể chọn **bao nhiêu lần tùy ý**.

**Chuyển trạng thái**:

$$dp[j] = \max_{w_i \leq j}\big(dp[j], \; dp[j-w_i] + v_i\big)$$

Chỉ khác 0/1: duyệt j **xuôi** (cho phép dùng lại vật i).

```go
// Unbounded Knapsack — O(nW)
func knapsackUnbounded(weights, values []int, W int) int {
    dp := make([]int, W+1)

    for i := 0; i < len(weights); i++ {
        wi, vi := weights[i], values[i]
        // XUÔI: cho phép chọn vật i nhiều lần
        for j := wi; j <= W; j++ {
            if dp[j-wi]+vi > dp[j] {
                dp[j] = dp[j-wi] + vi
            }
        }
    }
    return dp[W]
}
```

#### So sánh 0/1 vs Unbounded

| | 0/1 Knapsack | Unbounded Knapsack |
|---|---|---|
| Mỗi vật dùng | Tối đa 1 lần | Không giới hạn |
| Duyệt j | **Ngược** (W → wi) | **Xuôi** (wi → W) |
| Ví dụ thực tế | Chọn dự án đầu tư | Đổi tiền xu |

---

### 3.3. Khoảng cách chỉnh sửa (Edit Distance / Levenshtein Distance)

**Đề bài**: Cho 2 chuỗi `s` và `t`, tìm số phép toán **tối thiểu** để biến `s` thành `t`. Ba phép toán: chèn (insert), xóa (delete), thay thế (replace) — mỗi phép tốn 1 đơn vị.

**Trạng thái**: `dp[i][j]` = khoảng cách chỉnh sửa giữa `s[0..i-1]` và `t[0..j-1]`.

**Chuyển trạng thái**:

$$dp[i][j] = \begin{cases}
dp[i-1][j-1] & \text{nếu } s[i-1] = t[j-1] \\
1 + \min\big(dp[i-1][j], \; dp[i][j-1], \; dp[i-1][j-1]\big) & \text{nếu } s[i-1] \neq t[j-1]
\end{cases}$$

Trong đó:
- `dp[i-1][j] + 1`: **xóa** ký tự cuối của s
- `dp[i][j-1] + 1`: **chèn** ký tự vào s
- `dp[i-1][j-1] + 1`: **thay thế** ký tự cuối của s

**Base case**: `dp[i][0] = i` (xóa hết s), `dp[0][j] = j` (chèn hết t).

**Đáp án**: `dp[m][n]` (m = len(s), n = len(t)).

```go
// LeetCode 72 — Edit Distance — O(mn)
func minDistance(word1, word2 string) int {
    m, n := len(word1), len(word2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
        dp[i][0] = i
    }
    for j := 0; j <= n; j++ {
        dp[0][j] = j
    }

    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if word1[i-1] == word2[j-1] {
                dp[i][j] = dp[i-1][j-1]
            } else {
                dp[i][j] = 1 + min3(
                    dp[i-1][j],   // xóa
                    dp[i][j-1],   // chèn
                    dp[i-1][j-1], // thay thế
                )
            }
        }
    }
    return dp[m][n]
}

func min3(a, b, c int) int {
    if a < b { b = a }
    if c < b { return c }
    return b
}
```

**Truy vết** chi tiết — in ra các phép toán:

```go
func editDistanceTrace(s, t string) (int, []string) {
    m, n := len(s), len(t)
    dp := make([][]int, m+1)
    for i := range dp { dp[i] = make([]int, n+1); dp[i][0] = i }
    for j := 0; j <= n; j++ { dp[0][j] = j }

    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if s[i-1] == t[j-1] {
                dp[i][j] = dp[i-1][j-1]
            } else {
                dp[i][j] = 1 + min3(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
            }
        }
    }

    // Truy vết ngược
    var ops []string
    i, j := m, n
    for i > 0 || j > 0 {
        if i > 0 && j > 0 && s[i-1] == t[j-1] {
            i--; j--
        } else if i > 0 && j > 0 && dp[i][j] == dp[i-1][j-1]+1 {
            ops = append(ops, fmt.Sprintf("Replace s[%d]='%c' → '%c'", i-1, s[i-1], t[j-1]))
            i--; j--
        } else if i > 0 && dp[i][j] == dp[i-1][j]+1 {
            ops = append(ops, fmt.Sprintf("Delete s[%d]='%c'", i-1, s[i-1]))
            i--
        } else {
            ops = append(ops, fmt.Sprintf("Insert '%c' at position %d", t[j-1], i))
            j--
        }
    }
    // Đảo ngược
    for l, r := 0, len(ops)-1; l < r; l, r = l+1, r-1 {
        ops[l], ops[r] = ops[r], ops[l]
    }
    return dp[m][n], ops
}
```

**Ví dụ truy vết**: s = "kitten", t = "sitting"

| | "" | s | i | t | t | i | n | g |
|---|---|---|---|---|---|---|---|---|
| "" | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| k | 1 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| i | 2 | 2 | 1 | 2 | 3 | 4 | 5 | 6 |
| t | 3 | 3 | 2 | 1 | 2 | 3 | 4 | 5 |
| t | 4 | 4 | 3 | 2 | 1 | 2 | 3 | 4 |
| e | 5 | 5 | 4 | 3 | 2 | 2 | 3 | 4 |
| n | 6 | 6 | 5 | 4 | 3 | 3 | 2 | 3 |

Đáp án: **3** (replace k→s, replace e→i, insert g).

**Tối ưu bộ nhớ — O(min(m,n))**:

```go
func minDistanceOpt(word1, word2 string) int {
    m, n := len(word1), len(word2)
    if m < n {
        return minDistanceOpt(word2, word1) // đảm bảo n là chiều ngắn hơn
    }
    prev := make([]int, n+1)
    curr := make([]int, n+1)
    for j := 0; j <= n; j++ { prev[j] = j }

    for i := 1; i <= m; i++ {
        curr[0] = i
        for j := 1; j <= n; j++ {
            if word1[i-1] == word2[j-1] {
                curr[j] = prev[j-1]
            } else {
                curr[j] = 1 + min3(prev[j], curr[j-1], prev[j-1])
            }
        }
        prev, curr = curr, prev
    }
    return prev[n]
}
```

---

### 3.4. Dãy con chung dài nhất (LCS — Longest Common Subsequence)

**Đề bài**: Cho 2 chuỗi `s` và `t`, tìm độ dài **dãy con chung** dài nhất (subsequence — không cần liên tục).

```
s = "abcde", t = "ace" → LCS = "ace", độ dài 3
```

**Trạng thái**: `dp[i][j]` = LCS của `s[0..i-1]` và `t[0..j-1]`.

**Chuyển trạng thái**:

$$dp[i][j] = \begin{cases}
dp[i-1][j-1] + 1 & \text{nếu } s[i-1] = t[j-1] \\
\max\big(dp[i-1][j], \; dp[i][j-1]\big) & \text{nếu } s[i-1] \neq t[j-1]
\end{cases}$$

**Base case**: `dp[0][j] = dp[i][0] = 0`.

```go
// LeetCode 1143 — Longest Common Subsequence — O(mn)
func longestCommonSubsequence(text1, text2 string) int {
    m, n := len(text1), len(text2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }

    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if text1[i-1] == text2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}

func max(a, b int) int {
    if a > b { return a }
    return b
}
```

**Truy vết** — tìm chuỗi LCS:

```go
func findLCS(s, t string) string {
    m, n := len(s), len(t)
    dp := make([][]int, m+1)
    for i := range dp { dp[i] = make([]int, n+1) }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if s[i-1] == t[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }

    // Truy vết ngược
    var result []byte
    i, j := m, n
    for i > 0 && j > 0 {
        if s[i-1] == t[j-1] {
            result = append(result, s[i-1])
            i--; j--
        } else if dp[i-1][j] > dp[i][j-1] {
            i--
        } else {
            j--
        }
    }
    // Đảo ngược
    for l, r := 0, len(result)-1; l < r; l, r = l+1, r-1 {
        result[l], result[r] = result[r], result[l]
    }
    return string(result)
}
```

---

### 3.5. Nhân chuỗi ma trận (Matrix Chain Multiplication)

**Đề bài**: Cho `n` ma trận A₁, A₂, …, Aₙ cần nhân với nhau. Ma trận Aᵢ có kích thước `dims[i-1] × dims[i]`. Tìm cách đặt ngoặc sao cho **tổng số phép nhân scalar** là ít nhất.

> Nhân hai ma trận kích thước p×q và q×r tốn `p × q × r` phép nhân scalar.

**Trạng thái**: `dp[i][j]` = chi phí nhân tối thiểu cho chuỗi ma trận từ Aᵢ đến Aⱼ.

**Chuyển trạng thái** (QHĐ khoảng — Interval DP):

$$dp[i][j] = \min_{i \leq k < j}\big(dp[i][k] + dp[k+1][j] + d_{i-1} \times d_k \times d_j\big)$$

**Base case**: `dp[i][i] = 0` (ma trận đơn lẻ, không cần nhân).

**Thứ tự tính**: theo **độ dài khoảng** tăng dần (`len = 2, 3, ..., n`).

```go
// Matrix Chain Multiplication — O(n³)
func matrixChainOrder(dims []int) int {
    n := len(dims) - 1 // số ma trận
    if n <= 1 {
        return 0
    }

    dp := make([][]int, n)
    split := make([][]int, n) // để truy vết
    for i := range dp {
        dp[i] = make([]int, n)
        split[i] = make([]int, n)
    }

    // Duyệt theo độ dài khoảng
    for length := 2; length <= n; length++ {
        for i := 0; i <= n-length; i++ {
            j := i + length - 1
            dp[i][j] = 1<<63 - 1
            for k := i; k < j; k++ {
                cost := dp[i][k] + dp[k+1][j] + dims[i]*dims[k+1]*dims[j+1]
                if cost < dp[i][j] {
                    dp[i][j] = cost
                    split[i][j] = k
                }
            }
        }
    }
    return dp[0][n-1]
}

// In cách đặt ngoặc tối ưu
func printOptimalParens(split [][]int, i, j int) string {
    if i == j {
        return fmt.Sprintf("A%d", i+1)
    }
    k := split[i][j]
    left := printOptimalParens(split, i, k)
    right := printOptimalParens(split, k+1, j)
    return "(" + left + " × " + right + ")"
}
```

**Ví dụ**: dims = [10, 30, 5, 60] → 3 ma trận: A₁(10×30), A₂(30×5), A₃(5×60)

| Cách nhân | Chi phí |
|----------|---------|
| (A₁ × A₂) × A₃ | 10×30×5 + 10×5×60 = 1500 + 3000 = **4500** |
| A₁ × (A₂ × A₃) | 30×5×60 + 10×30×60 = 9000 + 18000 = **27000** |

Cách 1 tốt hơn 6 lần! Thứ tự đặt ngoặc **cực kỳ quan trọng**.

---

### 3.6. Đường đi trên lưới (Grid Path)

**Đề bài**: Cho lưới m×n, đi từ góc trên-trái đến góc dưới-phải, mỗi bước chỉ đi **phải** hoặc **xuống**. Đếm số đường đi.

**Trạng thái**: `dp[i][j]` = số đường đi đến ô (i, j).

**Chuyển trạng thái**: `dp[i][j] = dp[i-1][j] + dp[i][j-1]`

**Base case**: `dp[0][j] = dp[i][0] = 1` (chỉ có 1 cách đi dọc cạnh).

```go
// LeetCode 62 — Unique Paths
func uniquePaths(m, n int) int {
    dp := make([]int, n)
    for j := range dp {
        dp[j] = 1
    }
    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            dp[j] += dp[j-1]
        }
    }
    return dp[n-1]
}
```

**Biến thể — có chướng ngại vật** (LeetCode 63):

```go
func uniquePathsWithObstacles(grid [][]int) int {
    m, n := len(grid), len(grid[0])
    if grid[0][0] == 1 {
        return 0
    }
    dp := make([]int, n)
    dp[0] = 1
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == 1 {
                dp[j] = 0
            } else if j > 0 {
                dp[j] += dp[j-1]
            }
        }
    }
    return dp[n-1]
}
```

**Biến thể — chi phí tối thiểu** (LeetCode 64):

```go
func minPathSum(grid [][]int) int {
    m, n := len(grid), len(grid[0])
    dp := make([]int, n)
    dp[0] = grid[0][0]
    for j := 1; j < n; j++ {
        dp[j] = dp[j-1] + grid[0][j]
    }
    for i := 1; i < m; i++ {
        dp[0] += grid[i][0]
        for j := 1; j < n; j++ {
            dp[j] = min(dp[j], dp[j-1]) + grid[i][j]
        }
    }
    return dp[n-1]
}

func min(a, b int) int {
    if a < b { return a }
    return b
}
```

---

### 3.7. Bài toán đổi tiền (Coin Change)

**Đề bài**: Cho các mệnh giá tiền xu `coins[]` và số tiền `amount`. Tìm **số xu ít nhất** để tổng đúng bằng amount. Mỗi mệnh giá dùng không giới hạn.

**Trạng thái**: `dp[j]` = số xu tối thiểu để đạt tổng `j`.

**Chuyển trạng thái**: $dp[j] = \min_{c \in coins, \; c \leq j}\big(dp[j-c] + 1\big)$

**Base case**: `dp[0] = 0`.

```go
// LeetCode 322 — Coin Change
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1)
    for i := 1; i <= amount; i++ {
        dp[i] = amount + 1 // sentinel (không thể đạt)
    }

    for i := 1; i <= amount; i++ {
        for _, c := range coins {
            if c <= i && dp[i-c]+1 < dp[i] {
                dp[i] = dp[i-c] + 1
            }
        }
    }

    if dp[amount] > amount {
        return -1 // không thể đổi
    }
    return dp[amount]
}
```

**Biến thể — đếm số cách đổi** (LeetCode 518):

```go
// Coin Change 2 — số cách tổ hợp (không phân biệt thứ tự)
func change(amount int, coins []int) int {
    dp := make([]int, amount+1)
    dp[0] = 1

    // Duyệt theo từng loại xu (tránh đếm trùng hoán vị)
    for _, c := range coins {
        for j := c; j <= amount; j++ {
            dp[j] += dp[j-c]
        }
    }
    return dp[amount]
}
```

> **Chú ý thứ tự vòng lặp**: nếu duyệt `coins` ở vòng ngoài → đếm **tổ hợp** (không phân biệt thứ tự). Nếu duyệt `amount` ở vòng ngoài → đếm **hoán vị** (có phân biệt thứ tự).

---

### 3.8. House Robber (Trộm nhà)

**Đề bài**: Dãy `n` nhà, mỗi nhà có `nums[i]` tiền. Không được trộm 2 nhà liền kề. Tìm tổng tiền lớn nhất.

**Trạng thái**: `dp[i]` = tổng lớn nhất khi xét i nhà đầu.

**Chuyển trạng thái**: `dp[i] = max(dp[i-1], dp[i-2] + nums[i])`
- Không trộm nhà i → `dp[i-1]`
- Trộm nhà i → `dp[i-2] + nums[i]`

```go
// LeetCode 198 — House Robber
func rob(nums []int) int {
    if len(nums) == 0 { return 0 }
    if len(nums) == 1 { return nums[0] }

    prev2, prev1 := 0, 0
    for _, num := range nums {
        curr := max(prev1, prev2+num)
        prev2, prev1 = prev1, curr
    }
    return prev1
}
```

**Biến thể — nhà vòng tròn** (LeetCode 213):

```go
// House Robber II — nhà đầu và nhà cuối liền kề
func rob2(nums []int) int {
    n := len(nums)
    if n == 1 { return nums[0] }
    // Trường hợp 1: bỏ nhà cuối
    // Trường hợp 2: bỏ nhà đầu
    return max(robRange(nums, 0, n-2), robRange(nums, 1, n-1))
}

func robRange(nums []int, lo, hi int) int {
    prev2, prev1 := 0, 0
    for i := lo; i <= hi; i++ {
        curr := max(prev1, prev2+nums[i])
        prev2, prev1 = prev1, curr
    }
    return prev1
}
```

---

### 3.9. Xâu con chung dài nhất (Longest Palindromic Subsequence)

**Đề bài**: Tìm độ dài dãy con đối xứng (palindrome) dài nhất trong chuỗi `s`.

**Insight**: LPS(s) = LCS(s, reverse(s)).

Hoặc dùng **Interval DP** trực tiếp:

**Trạng thái**: `dp[i][j]` = LPS của `s[i..j]`.

**Chuyển trạng thái**:

$$dp[i][j] = \begin{cases}
dp[i+1][j-1] + 2 & \text{nếu } s[i] = s[j] \\
\max(dp[i+1][j], \; dp[i][j-1]) & \text{nếu } s[i] \neq s[j]
\end{cases}$$

**Base case**: `dp[i][i] = 1`.

```go
// LeetCode 516 — Longest Palindromic Subsequence
func longestPalinSubseq(s string) int {
    n := len(s)
    dp := make([][]int, n)
    for i := range dp {
        dp[i] = make([]int, n)
        dp[i][i] = 1
    }

    // Duyệt theo độ dài khoảng tăng dần
    for length := 2; length <= n; length++ {
        for i := 0; i <= n-length; i++ {
            j := i + length - 1
            if s[i] == s[j] {
                dp[i][j] = dp[i+1][j-1] + 2
            } else {
                dp[i][j] = max(dp[i+1][j], dp[i][j-1])
            }
        }
    }
    return dp[0][n-1]
}
```

---

### 3.10. Tổng tập con (Subset Sum)

**Đề bài**: Cho mảng `nums[]` và số nguyên `target`. Kiểm tra có tập con nào có tổng bằng `target`?

**Trạng thái**: `dp[j]` = true nếu có thể đạt tổng `j` bằng tập con.

**Chuyển trạng thái**: `dp[j] = dp[j] || dp[j - nums[i]]`

**Base case**: `dp[0] = true`.

```go
// LeetCode 416 — Partition Equal Subset Sum
// (Thiếu: target = sum/2)
func canPartition(nums []int) bool {
    sum := 0
    for _, v := range nums { sum += v }
    if sum%2 != 0 { return false }
    target := sum / 2

    dp := make([]bool, target+1)
    dp[0] = true

    for _, num := range nums {
        // Duyệt NGƯỢC (0/1 knapsack — mỗi số dùng 1 lần)
        for j := target; j >= num; j-- {
            dp[j] = dp[j] || dp[j-num]
        }
    }
    return dp[target]
}
```

---

## So sánh các bài toán QHĐ

| Bài toán | Trạng thái | Phức tạp | Dạng |
|---------|-----------|---------|------|
| Fibonacci | `dp[i]` | O(n) | 1D |
| LIS | `dp[i]` | O(n²) / O(n log n) | 1D |
| LCS | `dp[i][j]` | O(mn) | 2D |
| Edit Distance | `dp[i][j]` | O(mn) | 2D |
| 0/1 Knapsack | `dp[i][j]` | O(nW) | 2D |
| Matrix Chain | `dp[i][j]` | O(n³) | Interval |
| Grid Path | `dp[i][j]` | O(mn) | 2D |
| Coin Change | `dp[j]` | O(nW) | 1D (unbounded) |
| House Robber | `dp[i]` | O(n) | 1D |
| Palindromic Subseq | `dp[i][j]` | O(n²) | Interval |
| Subset Sum | `dp[j]` | O(nW) | 1D (0/1) |

---

## Bài tập (LeetCode)

### QHĐ 1D cơ bản

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Climbing Stairs | dp[i] = dp[i-1] + dp[i-2] | [70](https://leetcode.com/problems/climbing-stairs/) |
| House Robber | Chọn/bỏ, không liền kề | [198](https://leetcode.com/problems/house-robber/) |
| House Robber II | Vòng tròn | [213](https://leetcode.com/problems/house-robber-ii/) |
| Decode Ways | Counting DP | [91](https://leetcode.com/problems/decode-ways/) |
| Maximum Subarray | Kadane's algorithm (DP 1D) | [53](https://leetcode.com/problems/maximum-subarray/) |
| Jump Game | dp[i] hoặc greedy | [55](https://leetcode.com/problems/jump-game/) |
| Jump Game II | Min steps DP | [45](https://leetcode.com/problems/jump-game-ii/) |

### LIS & Dãy con

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Longest Increasing Subsequence | LIS O(n²) / O(n log n) | [300](https://leetcode.com/problems/longest-increasing-subsequence/) |
| Number of LIS | Đếm + LIS | [673](https://leetcode.com/problems/number-of-longest-increasing-subsequence/) |
| Russian Doll Envelopes | 2D LIS | [354](https://leetcode.com/problems/russian-doll-envelopes/) |
| Longest Common Subsequence | LCS 2D | [1143](https://leetcode.com/problems/longest-common-subsequence/) |
| Longest Palindromic Subsequence | Interval / LCS | [516](https://leetcode.com/problems/longest-palindromic-subsequence/) |
| Longest Palindromic Substring | Expand center / DP | [5](https://leetcode.com/problems/longest-palindromic-substring/) |

### Ba lô & Tổng tập con

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Partition Equal Subset Sum | 0/1 Knapsack boolean | [416](https://leetcode.com/problems/partition-equal-subset-sum/) |
| Target Sum | 0/1 Knapsack counting | [494](https://leetcode.com/problems/target-sum/) |
| Coin Change | Unbounded Knapsack min | [322](https://leetcode.com/problems/coin-change/) |
| Coin Change 2 | Unbounded counting | [518](https://leetcode.com/problems/coin-change-ii/) |
| Ones and Zeroes | 2D Knapsack | [474](https://leetcode.com/problems/ones-and-zeroes/) |
| Last Stone Weight II | 0/1 Knapsack | [1049](https://leetcode.com/problems/last-stone-weight-ii/) |

### Chuỗi (String DP)

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Edit Distance | 2D DP | [72](https://leetcode.com/problems/edit-distance/) |
| Distinct Subsequences | Counting DP | [115](https://leetcode.com/problems/distinct-subsequences/) |
| Interleaving String | 2D DP | [97](https://leetcode.com/problems/interleaving-string/) |
| Wildcard Matching | 2D DP | [44](https://leetcode.com/problems/wildcard-matching/) |
| Regular Expression Matching | 2D DP | [10](https://leetcode.com/problems/regular-expression-matching/) |

### Interval DP

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Matrix Chain (không có trên LC) | Interval DP O(n³) | — |
| Burst Balloons | Interval DP | [312](https://leetcode.com/problems/burst-balloons/) |
| Minimum Cost Tree From Leaf | Interval DP | [1130](https://leetcode.com/problems/minimum-cost-tree-from-leaf-values/) |
| Strange Printer | Interval DP | [664](https://leetcode.com/problems/strange-printer/) |
| Palindrome Partitioning II | Interval + 1D | [132](https://leetcode.com/problems/palindrome-partitioning-ii/) |

### Grid DP

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Unique Paths | Grid path counting | [62](https://leetcode.com/problems/unique-paths/) |
| Unique Paths II | Grid + chướng ngại vật | [63](https://leetcode.com/problems/unique-paths-ii/) |
| Minimum Path Sum | Grid min cost | [64](https://leetcode.com/problems/minimum-path-sum/) |
| Triangle | Bottom-up trên tam giác | [120](https://leetcode.com/problems/triangle/) |
| Maximal Square | 2D DP | [221](https://leetcode.com/problems/maximal-square/) |

### Nâng cao

| Bài tập | Kỹ thuật | Link |
|---------|-----------|------|
| Word Break | 1D DP + Set | [139](https://leetcode.com/problems/word-break/) |
| Perfect Squares | Unbounded Knapsack | [279](https://leetcode.com/problems/perfect-squares/) |
| Longest Valid Parentheses | Stack DP | [32](https://leetcode.com/problems/longest-valid-parentheses/) |
| Best Time to Buy/Sell Stock III | State machine DP | [123](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/) |
| Best Time IV | Generalized k txn | [188](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iv/) |

---

## Pattern liên quan

- [03-recursion.md](03-recursion.md) — Đệ quy là tiền đề; QHĐ = đệ quy + ghi nhớ.
- [09-enumeration.md](09-enumeration.md) — Quay lui liệt kê tất cả; QHĐ tối ưu hoặc đếm.
- [10-data-structures-algorithms.md](10-data-structures-algorithms.md) — Cấu trúc dữ liệu nền tảng (mảng, stack).
- **Tham lam (Greedy)**: Khi lựa chọn cục bộ tối ưu dẫn đến toàn cục tối ưu → không cần DP.
- **Chia để trị**: Khi bài toán con KHÔNG chồng lấn → Merge sort, Quick sort.
- **Bitmask DP**: Khi trạng thái là tập con → `dp[mask]`. Thường O(2ⁿ × n).

---

## Ghi nhớ nhanh

### Hai điều kiện áp dụng QHĐ
1. **Bài toán con tối ưu** — lời giải tối ưu chứa lời giải tối ưu của bài toán con.
2. **Bài toán con chồng lấn** — cùng bài toán con giải nhiều lần.

### 5 bước giải
1. Trạng thái → 2. Chuyển trạng thái → 3. Base case → 4. Thứ tự tính → 5. Đáp án.

### Top-down vs Bottom-up
- Top-down: đệ quy + memo. Dễ viết, có thể stack overflow.
- Bottom-up: vòng lặp điền bảng. Dễ tối ưu bộ nhớ.

### Hai pattern knapsack quan trọng
- 0/1 (mỗi vật 1 lần): duyệt j **ngược**.
- Unbounded (không giới hạn): duyệt j **xuôi**.

### Đếm vs Tối ưu
- **Đếm**: `dp[i] += dp[...]`
- **Tối ưu**: `dp[i] = min/max(dp[...] + cost)`

### Truy vết
- Lưu mảng `prev[]` hoặc `split[]` song song.
- Truy ngược từ đáp án cuối về base case.
- Nếu `dp[i][j] == dp[i-1][j]` → không chọn; ngược lại → có chọn.

### Tối ưu bộ nhớ
- 2D → 1D nếu chỉ cần hàng trước: **cuộn mảng**.
- Knapsack 0/1: duyệt **ngược** j.
- Hai hàng `prev[]` / `curr[]` rồi swap.

---

## Học tiếp

- [09-enumeration.md](09-enumeration.md) — Phần 1: Bài toán liệt kê (quay lui, nhánh cận)
- [10-data-structures-algorithms.md](10-data-structures-algorithms.md) — Phần 2: CTDL & Giải thuật cơ bản
- Phần 4: Thuật toán đồ thị — DFS/BFS, đường đi ngắn nhất, cây khung, luồng cực đại
