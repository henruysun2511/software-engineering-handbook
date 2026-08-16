# Chương 12: Recursion (Đệ quy)

## 12.1. Khái niệm cốt lõi

### 12.1.1. Định nghĩa

Recursion (đệ quy) là kỹ thuật giải quyết một bài toán bằng cách chia nó thành các bài toán con **có cùng bản chất** nhưng kích thước nhỏ hơn, thông qua việc hàm **tự gọi lại chính nó**. Mọi lời giải đệ quy đều bắt buộc gồm hai thành phần:

- **Base Case (trường hợp cơ sở):** điều kiện dừng, giải trực tiếp không cần gọi đệ quy thêm — nếu thiếu thành phần này, đệ quy sẽ lặp vô hạn.
- **Recursive Case (trường hợp đệ quy):** cách chia bài toán hiện tại thành (các) bài toán con nhỏ hơn, và cách kết hợp kết quả của bài toán con để tạo ra kết quả bài toán hiện tại.

### 12.1.2. Bản chất — mối liên hệ với Call Stack

Đã được giới thiệu ở mục 7.1.3: mỗi lần một hàm gọi đệ quy chính nó, hệ thống `push` một **khung (frame)** mới vào Call Stack, lưu trữ biến cục bộ và địa chỉ trở về của lần gọi đó. Khi hàm chạm Base Case và bắt đầu trả về giá trị, các khung được `pop` lần lượt theo đúng thứ tự LIFO — khung được `push` sau cùng (bài toán con nhỏ nhất) được xử lý và trả về **trước tiên**.

**Minh họa Call Stack** khi tính `factorial(4)`:

```
Gọi factorial(4)
  → push frame(n=4), gọi factorial(3)
    → push frame(n=3), gọi factorial(2)
      → push frame(n=2), gọi factorial(1)
        → push frame(n=1), CHẠM BASE CASE, trả về 1
      ← pop frame(n=1), frame(n=2) nhận về 1, tính 2*1=2, trả về 2
    ← pop frame(n=2), frame(n=3) nhận về 2, tính 3*2=6, trả về 6
  ← pop frame(n=3), frame(n=4) nhận về 6, tính 4*6=24, trả về 24
← pop frame(n=4), kết quả cuối cùng: 24
```

Chiều sâu tối đa của Call Stack (số frame đồng thời tồn tại) chính là **độ phức tạp không gian** của lời giải đệ quy — đây là căn cứ trực tiếp cho việc phân tích O(n) bộ nhớ phụ của các lời giải đệ quy đã gặp ở mục 6.4.1 (Reverse Linked List Recursive) và bài tập mục 0.3.

### 12.1.3. Linear Recursion và Tree Recursion

**Linear Recursion:** mỗi lời gọi chỉ tạo ra **một** lời gọi đệ quy con (ví dụ `factorial`). Cây gọi đệ quy có dạng một chuỗi thẳng, chiều sâu O(n), tổng số lời gọi cũng O(n).

**Tree Recursion:** mỗi lời gọi tạo ra **nhiều hơn một** lời gọi đệ quy con (ví dụ `fib(n) = fib(n-1) + fib(n-2)`, mỗi lời gọi phân nhánh thành hai). Cây gọi đệ quy phân nhánh theo cấp số nhân, dẫn đến tổng số lời gọi tăng theo hàm mũ nếu không có biện pháp tối ưu.

**Minh họa cây gọi đệ quy của `fib(4)`, thể hiện rõ vấn đề chồng lấn (overlapping subproblems):**

```
                    fib(4)
                   /      \
              fib(3)        fib(2)
             /      \       /      \
        fib(2)    fib(1) fib(1)   fib(0)
        /    \
    fib(1)  fib(0)

→ fib(2) được tính LẶP LẠI 2 lần, fib(1) được tính LẶP LẠI 3 lần
```

**Bản chất vấn đề:** vì các bài toán con **chồng lấn** (overlapping subproblems) — cùng một bài toán con `fib(2)` bị tính toán lại nhiều lần một cách lãng phí — độ phức tạp thời gian của `fib` đệ quy thuần túy là **O(2ⁿ)**, dù bài toán bản chất chỉ có `n` giá trị khác nhau cần tính. Đây chính là động lực trực tiếp dẫn đến kỹ thuật **Dynamic Programming** (Chương 25): lưu lại (ghi nhớ — memoize) kết quả các bài toán con đã tính, tránh tính lại, đưa độ phức tạp từ O(2ⁿ) xuống O(n).

### 12.1.4. Divide and Conquer — trường hợp đặc biệt của Tree Recursion

**Bản chất:** Divide and Conquer là dạng Tree Recursion trong đó các bài toán con **không chồng lấn nhau** (khác với Fibonacci) — mỗi bài toán con là một phần dữ liệu tách biệt hoàn toàn. Ba bước kinh điển: **Divide** (chia bài toán thành các bài toán con độc lập), **Conquer** (giải từng bài toán con bằng đệ quy), **Combine** (kết hợp kết quả các bài toán con thành kết quả cuối). Merge Sort (mục 11.1.1, ứng dụng chia đôi mảng liên tục) và Binary Search (Chương 10, dù chỉ đệ quy trên một nhánh) đều là ứng dụng của tư tưởng này.

---

## 12.2. So sánh Recursion và Iteration

| Tiêu chí | Recursion (Đệ quy) | Iteration (Vòng lặp) |
|---|---|---|
| Độ phức tạp không gian | Thường O(depth) do Call Stack | Thường O(1) nếu không dùng cấu trúc phụ |
| Độ rõ ràng code | Thường ngắn gọn, gần với định nghĩa toán học của bài toán | Có thể dài hơn nhưng tường minh luồng thực thi |
| Rủi ro | Stack Overflow nếu độ sâu quá lớn | Không có rủi ro tương tự |
| Phù hợp | Bài toán có cấu trúc đệ quy tự nhiên (cây, đồ thị, chia để trị) | Bài toán tuyến tính đơn giản, cần tối ưu bộ nhớ |

**Khi nào ưu tiên Recursion:** khi bài toán có cấu trúc đệ quy rõ ràng (duyệt cây — Chương 14, duyệt đồ thị bằng DFS — chương Graph, Backtracking — Chương 13), khi độ sâu đệ quy được đảm bảo nhỏ (ví dụ cây cân bằng có chiều cao O(log n)).

**Khi nào ưu tiên Iteration:** khi cần tối ưu bộ nhớ nghiêm ngặt, khi độ sâu đệ quy tiềm ẩn có thể rất lớn (ví dụ duyệt Linked List dài hàng triệu phần tử — nên dùng vòng lặp thay vì đệ quy để tránh Stack Overflow, như đã minh họa ở mục 6.4.1).

---

## 12.3. Cài đặt minh họa

### 12.3.1. Factorial (Linear Recursion)

```cpp
long long factorial(int n) {
    if (n <= 1) return 1; // Base Case
    return n * factorial(n - 1); // Recursive Case
}
```

**Độ phức tạp:** O(n) thời gian, O(n) bộ nhớ phụ (Call Stack sâu n).

### 12.3.2. Fibonacci — minh họa vấn đề Tree Recursion và cách khắc phục

**Cách đệ quy thuần túy (kém hiệu quả — minh họa vấn đề):**

```cpp
int fibNaive(int n) {
    if (n <= 1) return n;
    return fibNaive(n - 1) + fibNaive(n - 2); // gọi lặp lại các bài toán con chồng lấn
}
```

**Độ phức tạp:** O(2ⁿ) thời gian — như đã phân tích ở mục 12.1.3.

**Cách đệ quy có Memoization (khắc phục bằng cách lưu kết quả bài toán con):**

```cpp
#include <vector>
using namespace std;

int fibMemo(int n, vector<int>& memo) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n]; // bài toán con đã tính trước đó, tái sử dụng

    memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
    return memo[n];
}

int fibonacci(int n) {
    vector<int> memo(n + 1, -1);
    return fibMemo(n, memo);
}
```

**Độ phức tạp:** O(n) thời gian (mỗi giá trị từ 0 đến n chỉ được tính đúng một lần), O(n) bộ nhớ phụ cho mảng `memo` cộng với O(n) Call Stack. Đây chính là bước đệm trực tiếp dẫn vào kỹ thuật Dynamic Programming Top-down sẽ trình bày chi tiết ở Chương 25.

### 12.3.3. Binary Search dạng đệ quy (minh họa Divide and Conquer)

```cpp
#include <vector>
using namespace std;

int binarySearchRecursive(const vector<int>& arr, int target, int lo, int hi) {
    if (lo > hi) return -1; // Base Case: không gian tìm kiếm rỗng

    int mid = lo + (hi - lo) / 2;
    if (arr[mid] == target) return mid;
    else if (arr[mid] < target) return binarySearchRecursive(arr, target, mid + 1, hi);
    else return binarySearchRecursive(arr, target, lo, mid - 1);
}
```

**Độ phức tạp:** O(log n) thời gian (giống bản iterative ở mục 10.1.3), nhưng O(log n) bộ nhớ phụ do Call Stack — trong khi bản iterative chỉ tốn O(1). Đây là một ví dụ cụ thể khác cho đánh đổi không gian đã nêu ở mục 12.2.

### 12.3.4. Duyệt cây nhị phân đệ quy (minh họa cấu trúc đệ quy tự nhiên)

```cpp
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

int maxDepth(TreeNode* root) {
    if (root == nullptr) return 0; // Base Case: cây rỗng có chiều sâu 0

    // Recursive Case: chiều sâu = 1 (node hiện tại) + max(chiều sâu hai cây con)
    int leftDepth = maxDepth(root->left);
    int rightDepth = maxDepth(root->right);

    return 1 + max(leftDepth, rightDepth);
}
```

**Độ phức tạp:** O(n) thời gian (mỗi node được thăm đúng một lần), O(h) bộ nhớ phụ với `h` là chiều cao cây (O(log n) nếu cây cân bằng, O(n) nếu cây suy biến thành danh sách liên kết) — sẽ được phân tích sâu hơn ở Chương 14.

---

## 12.4. Bảng tổng hợp độ phức tạp

| Ví dụ | Loại đệ quy | Thời gian | Không gian (Call Stack) |
|---|---|---|---|
| Factorial | Linear | O(n) | O(n) |
| Fibonacci (thuần túy) | Tree (chồng lấn) | O(2ⁿ) | O(n) |
| Fibonacci (memoized) | Tree + Memoization | O(n) | O(n) |
| Binary Search (đệ quy) | Divide and Conquer | O(log n) | O(log n) |
| Duyệt cây nhị phân | Tree (không chồng lấn) | O(n) | O(h), h = chiều cao cây |

---

## 12.5. Danh sách bài tập luyện tập

### Mức Easy
1. Fibonacci Number (thử cài cả ba cách: thuần túy, memoization, và iterative — so sánh thời gian chạy thực tế)
2. Power of Two (dùng đệ quy chia đôi)
3. Reverse String (thử giải lại bằng đệ quy, so sánh với cách Two Pointers ở mục 2.4.1 style)
4. Sum of Natural Numbers

### Mức Medium
5. Pow(x, n) — Fast Exponentiation bằng Divide and Conquer
6. Generate Parentheses (bước đệm chuẩn bị cho Chương 13 — Backtracking)
7. Merge Two Sorted Lists (thử giải lại bằng đệ quy, so sánh với cách iterative ở mục 6.4.4)
8. Validate Binary Search Tree (đệ quy trên cấu trúc cây)

### Mức Hard
9. Count Good Nodes in Binary Tree
10. Unique Binary Search Trees II (Tree Recursion tạo cấu trúc)

---

*Chương tiếp theo: **Chương 13 — Backtracking**, mở rộng Tree Recursion thành kỹ thuật khám phá có hệ thống toàn bộ không gian lời giải, với khả năng "quay lui" khi một nhánh không dẫn đến lời giải hợp lệ.*
