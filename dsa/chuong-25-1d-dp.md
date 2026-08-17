# Chương 25: 1D Dynamic Programming

## 25.1. Khái niệm cốt lõi

### 25.1.1. Đặc điểm nhận diện

1D DP là lớp bài toán DP mà State chỉ cần **một chiều chỉ số** để mô tả đầy đủ một bài toán con — thường là `dp[i]` biểu diễn kết quả tối ưu tính đến vị trí thứ `i` của mảng/chuỗi đầu vào, hoặc kết quả cho một giá trị `i` cụ thể (như bài toán đổi tiền với giá trị `i`). Đây là dạng DP đơn giản nhất về mặt cấu trúc, phù hợp để rèn luyện khung tư duy 5 bước đã trình bày ở mục 24.5 trước khi chuyển sang các dạng phức tạp hơn.

---

## 25.2. Fibonacci Pattern và Climbing Stairs

*(Đã trình bày chi tiết ở mục 24.3.3 và mục 12.3.2 — đây là dạng 1D DP đơn giản nhất, Transition chỉ phụ thuộc đúng 2 State liền trước.)*

---

## 25.3. House Robber

### 25.3.1. Bài toán

Cho mảng `nums` biểu diễn số tiền tại mỗi nhà dọc một con phố, tìm số tiền tối đa có thể trộm được, biết rằng **không thể trộm hai nhà liền kề nhau** (sẽ kích hoạt báo động).

### 25.3.2. Áp dụng khung 5 bước

**Bước 1 — Nhận diện DP:** bài toán "giá trị lớn nhất" với ràng buộc lựa chọn (chọn/không chọn từng phần tử) — dấu hiệu rõ ràng của DP, khác với Kadane's Algorithm (mục 1.6.3, cũng là 1D DP nhưng không có ràng buộc "không liền kề").

**Bước 2 — State:** `dp[i]` = số tiền tối đa có thể trộm được, chỉ xét từ nhà `0` đến nhà `i`.

**Bước 3 — Transition:** tại nhà `i`, có đúng hai lựa chọn:
- **Không trộm nhà `i`:** số tiền tối đa bằng đúng `dp[i-1]` (kết quả tốt nhất đã có tính đến nhà `i-1`).
- **Trộm nhà `i`:** không được trộm nhà `i-1` (ràng buộc liền kề), nên số tiền bằng `nums[i] + dp[i-2]`.

```
dp[i] = max(dp[i-1], nums[i] + dp[i-2])
```

**Bước 4 — Base Case:** `dp[0] = nums[0]` (chỉ có một nhà, trộm luôn); `dp[1] = max(nums[0], nums[1])` (hai nhà liền kề, chỉ chọn nhà có giá trị lớn hơn).

**Bước 5 — Answer:** `dp[n-1]`.

**Minh họa** với `nums = [2, 7, 9, 3, 1]`:

```
i:      0   1   2   3   4
nums:   2   7   9   3   1
dp:     2   7   11  11  12

dp[2] = max(dp[1]=7, nums[2]+dp[0]=9+2=11) = 11
dp[3] = max(dp[2]=11, nums[3]+dp[1]=3+7=10) = 11
dp[4] = max(dp[3]=11, nums[4]+dp[2]=1+11=12) = 12
```

→ Kết quả: `12` (trộm nhà 0, 2, 4: `2+9+1=12`).

### 25.3.3. Cài đặt C++ (đã tối ưu bộ nhớ theo mục 24.4)

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int rob(const vector<int>& nums) {
    int n = nums.size();
    if (n == 0) return 0;
    if (n == 1) return nums[0];

    int prev2 = nums[0];
    int prev1 = max(nums[0], nums[1]);

    for (int i = 2; i < n; i++) {
        int curr = max(prev1, nums[i] + prev2);
        prev2 = prev1;
        prev1 = curr;
    }

    return prev1;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ (đã áp dụng Rolling Array, mục 24.4) — so với Backtracking thuần túy O(2ⁿ) thử mọi tổ hợp chọn/không chọn từng nhà.

---

## 25.4. Maximum Subarray — góc nhìn DP

*(Đã trình bày đầy đủ ở mục 1.6.3 dưới tên gọi Kadane's Algorithm — đáng nhắc lại ở đây dưới góc nhìn DP tường minh để củng cố khung tư duy 5 bước.)*

**State:** `dp[i]` = tổng lớn nhất của dãy con liên tiếp **kết thúc tại đúng vị trí `i`** (không phải "tính đến vị trí i" như House Robber — đây là khác biệt quan trọng cần phân biệt rõ).

**Transition:** `dp[i] = max(arr[i], dp[i-1] + arr[i])` — hoặc bắt đầu dãy con mới tại `i`, hoặc nối tiếp dãy con tốt nhất kết thúc tại `i-1`.

**Answer:** `max(dp[i])` với mọi `i` — **không phải** `dp[n-1]`, vì dãy con tối ưu có thể kết thúc ở bất kỳ vị trí nào, không nhất thiết ở cuối mảng. Đây là điểm khác biệt quan trọng so với House Robber (nơi Answer chính là `dp[n-1]`) — minh chứng cụ thể cho lưu ý ở mục 24.2.4 rằng cách trích xuất Answer cần được suy luận riêng cho từng bài toán, không có công thức chung.

---

## 25.5. Coin Change

### 25.5.1. Bài toán

Cho mảng `coins` (các mệnh giá đồng xu, số lượng mỗi loại không giới hạn) và `amount`, tìm số lượng đồng xu **tối thiểu** để đổi đủ `amount`, hoặc -1 nếu không thể.

### 25.5.2. Áp dụng khung 5 bước

**Bước 1 — Nhận diện DP:** đây chính là bài toán đã dùng làm counterexample cho Greedy ở mục 23.4.2 — với hệ đồng xu bất kỳ, chiến lược tham lam có thể sai, cần xét đầy đủ mọi khả năng bằng DP.

**Bước 2 — State:** `dp[i]` = số lượng đồng xu tối thiểu để đổi đủ giá trị `i`.

**Bước 3 — Transition:** với mỗi giá trị `i`, thử **mọi loại đồng xu** có thể dùng ở "bước cuối cùng":

```
dp[i] = min(dp[i - coin] + 1) với mọi coin trong coins sao cho coin ≤ i
```

**Bước 4 — Base Case:** `dp[0] = 0` (đổi giá trị 0 cần 0 đồng xu).

**Bước 5 — Answer:** `dp[amount]`, hoặc -1 nếu giá trị này vẫn là "vô cực" (không thể đổi được).

**Minh họa** với `coins = [1, 3, 4]`, `amount = 6` (đúng ví dụ phản chứng Greedy ở mục 23.4.2):

```
dp[0] = 0
dp[1] = dp[0]+1 = 1                                    (dùng đồng 1)
dp[2] = dp[1]+1 = 2                                    (dùng đồng 1)
dp[3] = min(dp[2]+1, dp[0]+1) = min(3, 1) = 1          (dùng đồng 3)
dp[4] = min(dp[3]+1, dp[1]+1, dp[0]+1) = min(2,2,1)=1  (dùng đồng 4)
dp[5] = min(dp[4]+1, dp[2]+1, dp[1]+1) = min(2,3,2)=2  (dùng đồng 4+1, hoặc 1+4)
dp[6] = min(dp[5]+1, dp[3]+1, dp[2]+1) = min(3,2,3)=2  (dùng đồng 3+3)
```

→ Kết quả `dp[6] = 2` — **khớp chính xác** với lời giải tối ưu thực sự đã nêu ở mục 23.4.2 (`6 = 3+3`), khác với kết quả sai của Greedy (3 đồng xu).

### 25.5.3. Cài đặt C++

```cpp
#include <vector>
#include <algorithm>
#include <climits>
using namespace std;

int coinChange(const vector<int>& coins, int amount) {
    vector<int> dp(amount + 1, INT_MAX);
    dp[0] = 0; // Base Case

    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i && dp[i - coin] != INT_MAX) {
                dp[i] = min(dp[i], dp[i - coin] + 1);
            }
        }
    }

    return dp[amount] == INT_MAX ? -1 : dp[amount];
}
```

**Độ phức tạp:** O(amount · số loại coin) thời gian — với mỗi giá trị từ 1 đến `amount`, thử mọi loại đồng xu; O(amount) bộ nhớ phụ. So với Backtracking thuần túy (thử mọi tổ hợp đồng xu), đây là cải tiến từ độ phức tạp hàm mũ xuống đa thức — minh chứng rõ ràng cho giá trị của DP so với brute force.

---

## 25.6. Decode Ways

### 25.6.1. Bài toán

Cho chuỗi số (ví dụ `"226"`), mỗi ký tự số từ '1' đến '9' có thể ánh xạ thành chữ cái A-I, và các cặp số từ "10" đến "26" có thể ánh xạ thành chữ cái J-Z (tương ứng A=1, B=2... Z=26). Đếm số cách giải mã chuỗi số thành chuỗi chữ cái hợp lệ.

### 25.6.2. Áp dụng khung 5 bước

**Bước 2 — State:** `dp[i]` = số cách giải mã cho `i` ký tự đầu tiên của chuỗi.

**Bước 3 — Transition:** tại vị trí `i`, có hai khả năng giải mã "bước cuối":
- **Giải mã 1 ký tự** `s[i-1]` riêng lẻ (nếu ký tự đó khác '0', vì '0' không tương ứng chữ cái nào): cộng thêm `dp[i-1]`.
- **Giải mã 2 ký tự** `s[i-2..i-1]` cùng nhau (nếu tạo thành số từ 10 đến 26): cộng thêm `dp[i-2]`.

```
dp[i] = dp[i-1] (nếu s[i-1] hợp lệ đứng riêng) + dp[i-2] (nếu s[i-2..i-1] hợp lệ đứng đôi)
```

**Bước 4 — Base Case:** `dp[0] = 1` (chuỗi rỗng có đúng 1 cách giải mã — "không giải mã gì cả").

### 25.6.3. Cài đặt C++

```cpp
#include <string>
#include <vector>
using namespace std;

int numDecodings(const string& s) {
    int n = s.size();
    if (n == 0 || s[0] == '0') return 0; // ký tự đầu là '0': không thể giải mã

    vector<int> dp(n + 1, 0);
    dp[0] = 1;
    dp[1] = 1; // s[0] chắc chắn khác '0' (đã kiểm tra ở trên)

    for (int i = 2; i <= n; i++) {
        // Khả năng 1: giải mã ký tự đơn s[i-1]
        if (s[i - 1] != '0') {
            dp[i] += dp[i - 1];
        }

        // Khả năng 2: giải mã cặp ký tự s[i-2..i-1]
        int twoDigit = (s[i - 2] - '0') * 10 + (s[i - 1] - '0');
        if (twoDigit >= 10 && twoDigit <= 26) {
            dp[i] += dp[i - 2];
        }
    }

    return dp[n];
}
```

**Độ phức tạp:** O(n) thời gian, O(n) bộ nhớ phụ (có thể tối ưu xuống O(1) bằng Rolling Array, mục 24.4, vì Transition chỉ phụ thuộc 2 State liền trước).

---

## 25.7. Bảng tổng hợp

| Bài toán | State | Đặc điểm Transition | Độ phức tạp |
|---|---|---|---|
| Climbing Stairs | Số cách đến bậc i | Phụ thuộc 2 State liền trước | O(n) |
| House Robber | Tiền tối đa tính đến nhà i | max giữa "bỏ qua" và "chọn + bước qua 1" | O(n) |
| Maximum Subarray | Tổng lớn nhất dãy con kết thúc tại i | max giữa "bắt đầu mới" và "nối tiếp" | O(n) |
| Coin Change | Số đồng xu tối thiểu cho giá trị i | min qua mọi loại đồng xu khả dĩ | O(amount · k) |
| Decode Ways | Số cách giải mã i ký tự đầu | Cộng dồn từ giải mã đơn và giải mã đôi | O(n) |

---

## 25.8. Danh sách bài tập luyện tập

### Mức Easy
1. Climbing Stairs
2. Min Cost Climbing Stairs
3. N-th Tribonacci Number

### Mức Medium
4. House Robber
5. House Robber II (biến thể vòng tròn — nhà đầu và nhà cuối coi như liền kề)
6. Maximum Subarray (thử giải lại theo góc nhìn DP tường minh)
7. Coin Change
8. Coin Change II (đếm SỐ CÁCH thay vì số đồng xu tối thiểu — Transition khác biệt tinh tế)
9. Decode Ways
10. Perfect Squares (cùng khuôn mẫu với Coin Change, "đồng xu" là các số chính phương)
11. Maximum Product Subarray (biến thể cần lưu cả max và min do có thể có số âm)

### Mức Hard
12. Longest Increasing Subsequence (chuẩn bị cho tư duy Subsequence DP sẽ mở rộng ở Chương 27)

---

*Chương tiếp theo: **Chương 26 — 2D Dynamic Programming**, mở rộng State sang hai chiều chỉ số, áp dụng cho bài toán trên lưới (Grid DP) và bài toán chọn vật phẩm có ràng buộc trọng lượng (Knapsack).*
