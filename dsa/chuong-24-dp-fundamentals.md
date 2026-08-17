# Chương 24: DP Fundamentals (Nền tảng Quy hoạch động)

## 24.1. Khái niệm cốt lõi

### 24.1.1. Định nghĩa

Dynamic Programming (DP — Quy hoạch động) là kỹ thuật giải bài toán tối ưu hoặc đếm số lượng bằng cách chia thành các **bài toán con chồng lấn (overlapping subproblems)**, giải từng bài toán con **đúng một lần duy nhất**, lưu lại kết quả để tái sử dụng thay vì tính lại — đây chính là hình thức hóa đầy đủ ý tưởng đã manh nha xuất hiện ở kỹ thuật Memoization (mục 12.3.2, Chương 12).

### 24.1.2. Bản chất — hai điều kiện tiên quyết

DP chỉ áp dụng được khi bài toán thỏa mãn đồng thời hai tính chất:

**Overlapping Subproblems (Bài toán con chồng lấn):** quá trình giải bài toán gốc bằng đệ quy sinh ra **cùng một bài toán con nhiều lần**. Đây chính là vấn đề đã minh họa cụ thể ở mục 12.1.3 với cây gọi đệ quy của Fibonacci — nếu các bài toán con **không** chồng lấn (như Divide and Conquer thuần túy, mục 12.1.4), việc lưu kết quả không mang lại lợi ích gì vì mỗi bài toán con chỉ được tính đúng một lần dù có lưu hay không.

**Optimal Substructure (Cấu trúc con tối ưu):** đã giới thiệu ở mục 23.1.2 khi so sánh với Greedy — lời giải tối ưu của bài toán gốc có thể được xây dựng từ lời giải tối ưu của các bài toán con nhỏ hơn. Khác với Greedy (chỉ xét một lựa chọn tại mỗi bước), DP xét **toàn bộ các khả năng lựa chọn** tại mỗi bước và chọn ra phương án tốt nhất dựa trên kết quả tối ưu đã biết của các bài toán con.

**Minh họa lại vấn đề Fibonacci** (đã trình bày ở mục 12.1.3 và 12.3.2) để làm rõ tại sao đây là ví dụ DP kinh điển: `fib(5)` cần `fib(4)` và `fib(3)`; nhưng `fib(4)` cũng cần `fib(3)` — đây chính là **overlapping subproblems**. Đồng thời, `fib(5) = fib(4) + fib(3)` cho thấy lời giải bài toán lớn được xây dựng trực tiếp từ lời giải bài toán con — đây là **optimal substructure** (dù Fibonacci không phải bài toán "tối ưu hóa" theo nghĩa thông thường, nguyên lý xây dựng từ bài toán con vẫn hoàn toàn tương tự).

---

## 24.2. Bốn thành phần của một lời giải DP

### 24.2.1. State (Trạng thái)

**Bản chất:** State là tập hợp thông tin **tối thiểu và đầy đủ** cần thiết để mô tả một bài toán con cụ thể — "tối thiểu" nghĩa là không dư thừa (mỗi chiều thông tin trong State đều thực sự ảnh hưởng đến kết quả), "đầy đủ" nghĩa là chỉ cần biết đúng các giá trị này, không cần biết thêm gì khác (kể cả "lịch sử" đã đi qua như thế nào), là có thể xác định được kết quả của bài toán con đó. Việc xác định đúng State là bước **quan trọng nhất và khó nhất** khi giải một bài toán DP — chọn sai hoặc thiếu chiều State sẽ khiến công thức truy hồi không thể xây dựng đúng.

Với Fibonacci, State chỉ cần một chiều: `dp[i]` = giá trị Fibonacci thứ `i`.

### 24.2.2. Transition (Công thức truy hồi)

**Bản chất:** Transition mô tả cách tính giá trị của một State **dựa trên** giá trị của các State nhỏ hơn (đã biết trước đó) — đây chính là biểu diễn hình thức của Optimal Substructure. Với Fibonacci: `dp[i] = dp[i-1] + dp[i-2]`.

### 24.2.3. Base Case (Trường hợp cơ sở)

**Bản chất:** giống hệt vai trò của Base Case trong đệ quy (mục 12.1.1) — các giá trị State nhỏ nhất được xác định **trực tiếp**, không cần thông qua Transition, làm điểm khởi đầu cho toàn bộ quá trình tính toán. Với Fibonacci: `dp[0] = 0, dp[1] = 1`.

### 24.2.4. Answer (Cách trích xuất đáp án)

**Bản chất:** sau khi đã tính toán đầy đủ bảng DP, đáp án cuối cùng của bài toán gốc không phải lúc nào cũng đơn giản là "giá trị tại State lớn nhất" — đôi khi cần lấy max/min qua nhiều State, hoặc tổng hợp từ nhiều giá trị trong bảng. Với Fibonacci, đáp án đơn giản là `dp[n]`, nhưng với các bài toán phức tạp hơn (ví dụ House Robber, mục 25.3), việc xác định đúng cách trích xuất đáp án cần được suy luận cẩn thận.

---

## 24.3. Top-down (Memoization) và Bottom-up (Tabulation)

### 24.3.1. Bản chất chung — hai cách hiện thực hóa cùng một tư tưởng

Cả hai cách tiếp cận đều dựa trên đúng bốn thành phần ở mục 24.2, chỉ khác nhau ở **thứ tự tính toán các State**:

**Top-down (Memoization):** giữ nguyên cấu trúc đệ quy tự nhiên của bài toán (đi từ bài toán lớn xuống bài toán con nhỏ hơn), nhưng lưu lại (memoize) kết quả mỗi bài toán con đã tính vào một bảng tra cứu (thường là mảng hoặc HashMap, Chương 3) — trước khi tính một bài toán con, kiểm tra xem nó đã được tính trước đó chưa. Đây chính xác là kỹ thuật đã trình bày ở mục 12.3.2.

**Bottom-up (Tabulation):** đảo ngược hoàn toàn hướng tính toán — bắt đầu từ Base Case, tính dần các State theo thứ tự tăng dần cho đến khi đạt đến State cần cho đáp án cuối cùng, dùng vòng lặp thay vì đệ quy.

### 24.3.2. So sánh Top-down và Bottom-up

| Tiêu chí | Top-down (Memoization) | Bottom-up (Tabulation) |
|---|---|---|
| Cấu trúc code | Đệ quy + bảng tra cứu | Vòng lặp + bảng DP |
| Độ trực quan | Thường dễ viết hơn — bám sát tư duy đệ quy tự nhiên của bài toán | Cần suy luận đúng thứ tự tính toán trước khi viết code |
| Chỉ tính bài toán con thực sự cần thiết | Có — chỉ đệ quy vào nhánh cần thiết | Không — thường tính toàn bộ bảng dù một số State có thể không cần đến |
| Rủi ro Stack Overflow | Có (do đệ quy sâu) — tương tự phân tích ở mục 12.2 | Không |
| Dễ tối ưu bộ nhớ (rolling array) | Khó hơn | Dễ hơn — vì biết trước thứ tự truy cập State |

**Khi nào ưu tiên Top-down:** khi mới tiếp cận bài toán, muốn tư duy tự nhiên theo hướng đệ quy trước, hoặc khi không phải mọi State đều cần được tính (một số bài toán có nhánh không bao giờ được đệ quy tới, Top-down tự động bỏ qua chúng).

**Khi nào ưu tiên Bottom-up:** khi cần tối ưu hiệu năng tối đa (tránh overhead của lời gọi hàm đệ quy), khi cần tối ưu bộ nhớ bằng kỹ thuật rolling array (mục 24.4), hoặc khi độ sâu đệ quy tiềm ẩn quá lớn có nguy cơ Stack Overflow.

### 24.3.3. Minh họa cả hai cách trên bài toán Climbing Stairs

**Bài toán:** có `n` bậc thang, mỗi bước có thể leo 1 hoặc 2 bậc, tìm số cách khác nhau để leo hết `n` bậc.

**Xác định 4 thành phần:** State `dp[i]` = số cách leo đến bậc thứ `i`. Transition: `dp[i] = dp[i-1] + dp[i-2]` (đến bậc `i` bằng cách leo 1 bậc từ `i-1`, hoặc leo 2 bậc từ `i-2`). Base Case: `dp[0] = 1` (một cách duy nhất: không leo bậc nào), `dp[1] = 1`. Answer: `dp[n]`.

*(Nhận xét: công thức truy hồi giống hệt Fibonacci — đây là một minh chứng cho thấy nhiều bài toán DP khác nhau về ngữ cảnh có thể có cùng bản chất toán học.)*

**Cài đặt Top-down (Memoization):**

```cpp
#include <vector>
using namespace std;

int climbStairsTopDown(int n, vector<int>& memo) {
    if (n <= 1) return 1; // Base Case
    if (memo[n] != -1) return memo[n]; // đã tính trước đó, tái sử dụng

    memo[n] = climbStairsTopDown(n - 1, memo) + climbStairsTopDown(n - 2, memo);
    return memo[n];
}

int climbStairs(int n) {
    vector<int> memo(n + 1, -1);
    return climbStairsTopDown(n, memo);
}
```

**Cài đặt Bottom-up (Tabulation):**

```cpp
#include <vector>
using namespace std;

int climbStairsBottomUp(int n) {
    if (n <= 1) return 1;

    vector<int> dp(n + 1);
    dp[0] = 1; // Base Case
    dp[1] = 1;

    for (int i = 2; i <= n; i++) { // tính dần theo thứ tự tăng, đảm bảo dp[i-1], dp[i-2] đã có sẵn
        dp[i] = dp[i - 1] + dp[i - 2];
    }

    return dp[n];
}
```

**Độ phức tạp (cả hai cách):** O(n) thời gian (mỗi State chỉ tính một lần — điểm khác biệt căn bản so với đệ quy thuần túy O(2ⁿ) đã phân tích ở mục 12.1.3), O(n) bộ nhớ phụ cho bảng `dp`/`memo`.

---

## 24.4. Tối ưu bộ nhớ — Rolling Array (giới thiệu bản chất)

**Bản chất:** quan sát kỹ công thức `dp[i] = dp[i-1] + dp[i-2]` của Climbing Stairs, ta thấy tại mỗi bước chỉ cần **hai giá trị gần nhất**, không cần lưu toàn bộ mảng `dp` kích thước `n`. Đây là cơ hội tối ưu bộ nhớ từ O(n) xuống O(1) bằng cách chỉ giữ lại một số lượng biến cố định thay vì toàn bộ bảng.

```cpp
int climbStairsOptimized(int n) {
    if (n <= 1) return 1;

    int prev2 = 1, prev1 = 1; // tương ứng dp[0], dp[1]

    for (int i = 2; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }

    return prev1;
}
```

**Điều kiện áp dụng kỹ thuật này:** chỉ khả thi khi Bottom-up **và** công thức Transition chỉ phụ thuộc vào một số lượng **cố định** các State liền kề trước đó (không phụ thuộc toàn bộ lịch sử) — kỹ thuật này sẽ được áp dụng lặp lại nhiều lần ở Chương 25 (1D DP) và Chương 26 (2D DP, nơi rolling array giảm từ O(m·n) xuống O(n) bộ nhớ).

---

## 24.5. Quy trình 5 bước giải bài toán DP

Đúc kết thành khung tư duy có thể áp dụng nhất quán cho mọi bài toán DP:

1. **Nhận diện bài toán có phải DP hay không:** kiểm tra dấu hiệu Overlapping Subproblems và Optimal Substructure (mục 24.1.2) — thường qua từ khóa "số cách", "giá trị lớn nhất/nhỏ nhất", "có thể đạt được hay không", kết hợp với việc bài toán có cấu trúc đệ quy tự nhiên nhưng brute force tốn hàm mũ.
2. **Xác định State:** trả lời câu hỏi "cần biết những thông tin gì để mô tả một bài toán con?" — đây là bước quan trọng nhất, thường quyết định bài toán là 1D DP (Chương 25), 2D DP (Chương 26), hay String DP (Chương 27).
3. **Xây dựng Transition:** với mỗi State, liệt kê **mọi lựa chọn khả dĩ** có thể dẫn đến State đó, biểu diễn quan hệ với các State nhỏ hơn.
4. **Xác định Base Case:** các State nhỏ nhất có thể xác định trực tiếp không cần Transition.
5. **Xác định Answer và thứ tự tính toán:** đảm bảo khi tính một State, mọi State nó phụ thuộc đã được tính từ trước (đúng thứ tự Bottom-up), và xác định đúng vị trí lấy đáp án cuối cùng.

---

## 24.6. Danh sách bài tập luyện tập

### Mức Easy
1. Climbing Stairs (thực hành cả ba cách: Top-down, Bottom-up, và tối ưu bộ nhớ)
2. Fibonacci Number (ôn lại từ mục 12.3.2 dưới góc nhìn đầy đủ khung DP)
3. N-th Tribonacci Number (mở rộng Transition phụ thuộc 3 State liền trước thay vì 2)

### Mức Medium
4. Min Cost Climbing Stairs
5. House Robber (bài toán mở đầu cho Chương 25 — thử tự xác định State/Transition trước khi đọc lời giải chi tiết)

---

*Chương tiếp theo: **Chương 25 — 1D Dynamic Programming**, áp dụng khung tư duy 5 bước vừa xây dựng vào lớp bài toán có State chỉ phụ thuộc một chiều chỉ số.*
