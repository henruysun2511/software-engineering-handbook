# Chương 26: 2D Dynamic Programming

## 26.1. Khái niệm cốt lõi

### 26.1.1. Đặc điểm nhận diện

2D DP là lớp bài toán mà State cần **hai chiều thông tin** để mô tả đầy đủ một bài toán con — thường gặp dưới hai dạng chính: **Grid DP** (hai chiều là tọa độ hàng/cột trên lưới) và **Knapsack** (một chiều là chỉ số vật phẩm đang xét, chiều còn lại là "sức chứa" hoặc "giá trị mục tiêu" còn lại). Việc nhận ra bài toán cần **hai chiều** State (thay vì một chiều như Chương 25) là bước mở rộng tự nhiên của khung tư duy 5 bước (mục 24.5): khi một chiều thông tin không đủ để mô tả đầy đủ bài toán con, cần bổ sung thêm chiều thứ hai.

---

## 26.2. Grid DP

### 26.2.1. Unique Paths

**Bài toán:** một robot đứng ở góc trên-trái của lưới `m × n`, chỉ có thể di chuyển xuống hoặc sang phải, đếm số đường đi khác nhau để đến góc dưới-phải.

**Áp dụng khung 5 bước:**

- **State:** `dp[i][j]` = số đường đi từ điểm xuất phát đến ô `(i, j)`.
- **Transition:** để đến `(i, j)`, robot phải đến từ **ô trên** `(i-1, j)` hoặc **ô trái** `(i, j-1)` ngay trước đó:
```
dp[i][j] = dp[i-1][j] + dp[i][j-1]
```
- **Base Case:** `dp[0][j] = 1` với mọi `j` (hàng đầu tiên chỉ có đúng 1 cách đến — luôn đi sang phải); `dp[i][0] = 1` với mọi `i` (cột đầu tiên tương tự, luôn đi xuống).
- **Answer:** `dp[m-1][n-1]`.

**Minh họa** với lưới `3 × 3`:

```
1   1   1
1   2   3
1   3   6

dp[1][1] = dp[0][1] + dp[1][0] = 1 + 1 = 2
dp[2][2] = dp[1][2] + dp[2][1] = 3 + 3 = 6   ← đáp án
```

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

int uniquePaths(int m, int n) {
    vector<vector<int>> dp(m, vector<int>(n, 1)); // khởi tạo toàn bộ hàng 0, cột 0 = 1 (Base Case)

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }

    return dp[m - 1][n - 1];
}
```

**Độ phức tạp:** O(m·n) thời gian, O(m·n) bộ nhớ phụ — có thể tối ưu xuống O(n) bằng Rolling Array (mục 26.4), vì `dp[i][j]` chỉ phụ thuộc hàng `i-1` và giá trị cùng hàng `i` bên trái.

### 26.2.2. Minimum Path Sum

**Bài toán:** cho lưới số nguyên, tìm đường đi từ góc trên-trái đến góc dưới-phải (chỉ di chuyển xuống/phải) có tổng nhỏ nhất.

**Bản chất khác biệt so với Unique Paths:** thay vì **cộng dồn số lượng** đường đi từ hai hướng, ta lấy **giá trị nhỏ nhất** giữa hai hướng cộng với chi phí ô hiện tại — cùng cấu trúc Grid DP nhưng Transition dùng `min` thay vì `+`.

```
dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])
```

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int minPathSum(vector<vector<int>>& grid) {
    int m = grid.size(), n = grid[0].size();
    vector<vector<int>> dp(m, vector<int>(n));

    dp[0][0] = grid[0][0];
    for (int j = 1; j < n; j++) dp[0][j] = dp[0][j - 1] + grid[0][j]; // Base Case hàng đầu
    for (int i = 1; i < m; i++) dp[i][0] = dp[i - 1][0] + grid[i][0]; // Base Case cột đầu

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = grid[i][j] + min(dp[i - 1][j], dp[i][j - 1]);
        }
    }

    return dp[m - 1][n - 1];
}
```

**Độ phức tạp:** O(m·n) thời gian, O(m·n) bộ nhớ phụ (tối ưu được xuống O(n) tương tự Unique Paths).

---

## 26.3. Knapsack (Bài toán cái túi)

### 26.3.1. 0/1 Knapsack — bản chất

**Bài toán:** cho `n` vật phẩm, mỗi vật có trọng lượng `weight[i]` và giá trị `value[i]`, và một cái túi sức chứa `capacity`, chọn tập hợp vật phẩm (mỗi vật **chỉ được chọn tối đa một lần** — đây là ý nghĩa của "0/1": chọn hoặc không chọn) sao cho tổng trọng lượng không vượt `capacity` và tổng giá trị lớn nhất.

**Áp dụng khung 5 bước:**

- **State:** `dp[i][w]` = giá trị lớn nhất đạt được khi chỉ xét `i` vật phẩm đầu tiên, với sức chứa còn lại là `w`. Đây là ví dụ điển hình cho việc **cần hai chiều**: một chiều theo dõi "đã xét đến vật nào", một chiều theo dõi "còn bao nhiêu sức chứa" — thiếu một trong hai chiều đều không đủ để mô tả đúng bài toán con.
- **Transition:** với vật phẩm thứ `i`, có đúng hai lựa chọn:
  - **Không chọn:** `dp[i][w] = dp[i-1][w]`.
  - **Chọn** (chỉ khả thi nếu `weight[i] ≤ w`): `dp[i][w] = value[i] + dp[i-1][w - weight[i]]`.
```
dp[i][w] = max(dp[i-1][w], value[i] + dp[i-1][w - weight[i]])  nếu weight[i] ≤ w
dp[i][w] = dp[i-1][w]                                           nếu weight[i] > w
```
- **Base Case:** `dp[0][w] = 0` với mọi `w` (chưa xét vật phẩm nào, giá trị bằng 0).
- **Answer:** `dp[n][capacity]`.

**Minh họa bảng DP** với 3 vật phẩm `(weight, value)` = `(1,1), (3,4), (4,5)`, `capacity = 4`:

```
        w=0  w=1  w=2  w=3  w=4
i=0      0    0    0    0    0
i=1(1,1) 0    1    1    1    1
i=2(3,4) 0    1    1    4    5
i=3(4,5) 0    1    1    4    5

dp[2][3] = max(dp[1][3]=1, value=4 + dp[1][0]=0) = 4
dp[3][4] = max(dp[2][4]=5, value=5 + dp[2][0]=0) = 5   (giữ nguyên, không chọn vật 3 tốt hơn)
```

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int knapsack01(int capacity, const vector<int>& weight, const vector<int>& value) {
    int n = weight.size();
    vector<vector<int>> dp(n + 1, vector<int>(capacity + 1, 0));

    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= capacity; w++) {
            dp[i][w] = dp[i - 1][w]; // mặc định: không chọn vật phẩm i

            if (weight[i - 1] <= w) {
                dp[i][w] = max(dp[i][w], value[i - 1] + dp[i - 1][w - weight[i - 1]]);
            }
        }
    }

    return dp[n][capacity];
}
```

**Độ phức tạp:** O(n · capacity) thời gian, O(n · capacity) bộ nhớ phụ.

### 26.3.2. Tối ưu bộ nhớ cho 0/1 Knapsack — kỹ thuật duyệt ngược

**Bản chất:** quan sát Transition chỉ phụ thuộc vào hàng `i-1` (không phụ thuộc các hàng xa hơn), có thể áp dụng Rolling Array (mục 24.4) để giảm từ O(n·capacity) xuống O(capacity) bộ nhớ. Tuy nhiên, **điểm tinh tế quan trọng**: nếu dùng chung một mảng `dp[w]` cho cả "hàng cũ" và "hàng mới", phải duyệt `w` theo chiều **giảm dần** (từ `capacity` về 0) — vì nếu duyệt tăng dần, giá trị `dp[w - weight[i-1]]` có thể đã bị **ghi đè bởi chính vật phẩm `i`** (vi phạm ràng buộc "mỗi vật chỉ chọn tối đa một lần", vô tình cho phép chọn lại vật `i` nhiều lần).

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int knapsack01Optimized(int capacity, const vector<int>& weight, const vector<int>& value) {
    int n = weight.size();
    vector<int> dp(capacity + 1, 0);

    for (int i = 0; i < n; i++) {
        // Duyệt NGƯỢC để đảm bảo dp[w - weight[i]] vẫn là giá trị của "hàng cũ" (chưa xét vật i)
        for (int w = capacity; w >= weight[i]; w--) {
            dp[w] = max(dp[w], value[i] + dp[w - weight[i]]);
        }
    }

    return dp[capacity];
}
```

**Độ phức tạp:** O(n · capacity) thời gian (không đổi), O(capacity) bộ nhớ phụ — cải thiện đáng kể so với O(n · capacity) của bản đầy đủ hai chiều.

### 26.3.3. Unbounded Knapsack — khác biệt với 0/1 Knapsack

**Bản chất khác biệt:** Unbounded Knapsack cho phép **chọn lại cùng một loại vật phẩm không giới hạn số lần** — đây chính xác là cấu trúc của bài toán Coin Change đã giải ở mục 25.5 (mỗi loại đồng xu dùng được không giới hạn). Về mặt cài đặt, khác biệt duy nhất so với bản tối ưu bộ nhớ ở mục 26.3.2 là hướng duyệt: dùng **duyệt xuôi** (tăng dần) thay vì duyệt ngược, vì việc "cho phép dùng lại vật `i` vừa chọn" chính là mục tiêu, không phải lỗi cần tránh.

```cpp
int knapsackUnbounded(int capacity, const vector<int>& weight, const vector<int>& value) {
    int n = weight.size();
    vector<int> dp(capacity + 1, 0);

    for (int i = 0; i < n; i++) {
        // Duyệt XUÔI: cho phép dp[w] sử dụng lại giá trị vừa cập nhật của chính vật phẩm i
        for (int w = weight[i]; w <= capacity; w++) {
            dp[w] = max(dp[w], value[i] + dp[w - weight[i]]);
        }
    }

    return dp[capacity];
}
```

**So sánh trực quan hướng duyệt:** đây là điểm dễ nhầm lẫn nhất giữa hai biến thể Knapsack — ghi nhớ theo nguyên tắc "0/1 (chọn tối đa 1 lần) → duyệt ngược; Unbounded (chọn không giới hạn) → duyệt xuôi".

### 26.3.4. Subset Sum — ứng dụng trực tiếp của 0/1 Knapsack

**Bài toán:** cho mảng số nguyên dương, xác định có tồn tại tập con nào có tổng bằng đúng `target` hay không.

**Bản chất:** đây chính là 0/1 Knapsack với `value[i] = weight[i]` (mỗi phần tử vừa là "trọng lượng" vừa là "giá trị"), và câu hỏi không phải "giá trị lớn nhất" mà là "có đạt được chính xác `target` hay không" — Transition đổi từ `max` sang phép **OR logic** (`dp[w]` là kiểu boolean thay vì số nguyên).

```cpp
#include <vector>
using namespace std;

bool canPartition(const vector<int>& nums, int target) {
    vector<bool> dp(target + 1, false);
    dp[0] = true; // luôn đạt được tổng 0 (không chọn gì)

    for (int num : nums) {
        for (int w = target; w >= num; w--) { // duyệt ngược, giống 0/1 Knapsack
            dp[w] = dp[w] || dp[w - num];
        }
    }

    return dp[target];
}
```

**Độ phức tạp:** O(n · target) thời gian, O(target) bộ nhớ phụ.

---

## 26.4. So sánh các biến thể Knapsack

| Biến thể | Ràng buộc chọn vật phẩm | Hướng duyệt khi tối ưu 1D | Bài toán tương ứng đã học |
|---|---|---|---|
| 0/1 Knapsack | Mỗi vật tối đa 1 lần | Duyệt ngược (giảm dần) | Subset Sum |
| Unbounded Knapsack | Mỗi vật không giới hạn số lần | Duyệt xuôi (tăng dần) | Coin Change (mục 25.5) |

---

## 26.5. Bảng tổng hợp độ phức tạp

| Bài toán | State | Độ phức tạp thời gian | Bộ nhớ (đã tối ưu) |
|---|---|---|---|
| Unique Paths | Số đường đi đến (i,j) | O(m·n) | O(n) |
| Minimum Path Sum | Tổng nhỏ nhất đến (i,j) | O(m·n) | O(n) |
| 0/1 Knapsack | Giá trị lớn nhất với i vật, sức chứa w | O(n·capacity) | O(capacity) |
| Unbounded Knapsack | Tương tự, không giới hạn số lần chọn | O(n·capacity) | O(capacity) |
| Subset Sum | Có đạt tổng w hay không, với i vật | O(n·target) | O(target) |

---

## 26.6. Danh sách bài tập luyện tập

### Mức Medium
1. Unique Paths
2. Unique Paths II (lưới có chướng ngại vật)
3. Minimum Path Sum
4. Triangle (Grid DP dạng tam giác)
5. Partition Equal Subset Sum (chính là Subset Sum với target = tổng mảng / 2)
6. Target Sum (biến thể Subset Sum với phép cộng/trừ)
7. Coin Change II (đếm SỐ CÁCH — Unbounded Knapsack đếm thay vì tối ưu giá trị)

### Mức Hard
8. Maximal Square (Grid DP tìm hình vuông lớn nhất toàn số 1)
9. Dungeon Game (Grid DP duyệt ngược từ đích về nguồn)
10. Profitable Schemes (0/1 Knapsack hai ràng buộc đồng thời)

---

*Chương tiếp theo: **Chương 27 — String DP**, áp dụng State hai chiều lên **hai chuỗi** đồng thời, giải quyết lớp bài toán so sánh/biến đổi chuỗi như Longest Common Subsequence và Edit Distance.*
