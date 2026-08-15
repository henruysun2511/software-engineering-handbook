# Chương 4: Two Pointers (Hai con trỏ)

## 4.1. Khái niệm cốt lõi

### 4.1.1. Định nghĩa

Two Pointers là kỹ thuật sử dụng **hai biến chỉ số** (con trỏ) duyệt qua một cấu trúc dữ liệu tuyến tính (array, string) theo một quy luật phối hợp nhất định, nhằm thay thế cách duyệt lồng hai vòng lặp O(n²) bằng một lượt duyệt O(n). Đây không phải một cấu trúc dữ liệu mới, mà là một **kỹ thuật tối ưu hóa vòng lặp**, khai thác tính chất đã sắp xếp hoặc tính đơn điệu (monotonicity) của dữ liệu đầu vào.

### 4.1.2. Bản chất — vì sao Two Pointers loại bỏ được một bậc độ phức tạp

Xét bài toán tổng quát: "tìm hai phần tử trong mảng **đã sắp xếp** có tổng bằng target". Cách brute force duyệt mọi cặp `(i, j)` tốn O(n²) vì với mỗi `i`, ta thử lại toàn bộ `j` từ đầu, **bỏ phí thông tin** rằng mảng đã được sắp xếp.

Two Pointers khai thác tính đơn điệu: đặt `left` ở đầu, `right` ở cuối. Nếu tổng hiện tại nhỏ hơn target, ta biết chắc **mọi cặp liên quan đến `left`** với các chỉ số nhỏ hơn `right` hiện tại đều không đủ lớn — tăng `left` (không cần thử lại các `j` nhỏ hơn). Ngược lại nếu tổng lớn hơn target, giảm `right`. Mỗi bước, hoặc `left` tăng hoặc `right` giảm — tổng số bước tối đa là `n`, cho độ phức tạp O(n).

**Nguyên lý cốt lõi:** Two Pointers chỉ áp dụng được khi ta có thể **chứng minh** rằng việc di chuyển một con trỏ theo một hướng nhất định sẽ loại bỏ được một tập hợp các khả năng chắc chắn không phải đáp án, mà không cần kiểm tra riêng lẻ từng khả năng đó.

### 4.1.3. Hai mô hình Two Pointers

**Opposite Direction (Ngược chiều):** hai con trỏ xuất phát từ hai đầu đối diện của mảng, tiến dần vào giữa cho đến khi gặp nhau. Áp dụng cho bài toán trên mảng đã sắp xếp, hoặc bài toán đối xứng như kiểm tra palindrome (đã trình bày ở mục 2.2.2).

```
left →                                    ← right
[ 2,   7,  11,  15 ]
  ↑                  ↑
 left               right
```

**Same Direction (Cùng chiều — hay còn gọi Fast & Slow trong ngữ cảnh mảng):** hai con trỏ cùng xuất phát từ đầu (hoặc gần nhau) và di chuyển cùng hướng với tốc độ khác nhau, thường dùng một con trỏ `slow` đánh dấu vị trí ghi kết quả và một con trỏ `fast` (hay `i`) duyệt khám phá — đã áp dụng ở mục 1.6.6 (Move Zeroes).

```
slow                fast →
 ↑                    ↑
[0, 1, 0, 3, 12]
 write pos      duyệt tìm phần tử hợp lệ
```

---

## 4.2. Kiến thức liên quan và so sánh

### 4.2.1. Two Pointers vs Brute Force

| Tiêu chí | Brute Force | Two Pointers |
|---|---|---|
| Độ phức tạp thời gian | O(n²) | O(n) |
| Bộ nhớ phụ | O(1) | O(1) |
| Yêu cầu dữ liệu | Không | Thường cần đã sắp xếp hoặc có tính đơn điệu |
| Khả năng áp dụng | Mọi bài toán duyệt cặp | Chỉ khi chứng minh được tính đơn điệu |

### 4.2.2. Two Pointers vs Sliding Window

Hai kỹ thuật thường bị nhầm lẫn vì cùng dùng nhiều biến chỉ số di chuyển trên mảng. Điểm khác biệt bản chất: Two Pointers (mô hình ngược chiều) thường thao tác trên **hai đầu cố định tiến vào giữa**, giải bài toán về **cặp phần tử**; Sliding Window luôn duy trì một **đoạn liên tục (window)** giữa hai con trỏ cùng chiều, giải bài toán về **tính chất của đoạn con**. Chương 5 sẽ trình bày chi tiết Sliding Window như một biến thể chuyên biệt của mô hình Same Direction.

### 4.2.3. Khi nào dùng Two Pointers

- Mảng/chuỗi đã sắp xếp (hoặc có thể sắp xếp mà không ảnh hưởng đến yêu cầu bài toán) và cần tìm cặp/nhóm phần tử thỏa mãn điều kiện tổng, hiệu, tích.
- Bài toán đối xứng (palindrome).
- Bài toán cần sắp xếp lại phần tử tại chỗ theo một tiêu chí phân loại (partition), như Move Zeroes hay Dutch National Flag.
- Bài toán tìm diện tích/dung tích lớn nhất giới hạn bởi hai biên (Container With Most Water).

---

## 4.3. Cài đặt các bài toán kinh điển

### 4.3.1. Two Sum II — Input Array Is Sorted

**Bài toán:** cho mảng đã sắp xếp tăng dần, tìm chỉ số hai phần tử có tổng bằng target (đánh số từ 1).

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

vector<int> twoSumSorted(const vector<int>& arr, int target) {
    int left = 0, right = (int)arr.size() - 1;

    while (left < right) {
        int sum = arr[left] + arr[right];
        if (sum == target) {
            return {left + 1, right + 1}; // đánh số từ 1
        } else if (sum < target) {
            left++;  // tổng chưa đủ lớn, cần phần tử lớn hơn
        } else {
            right--; // tổng vượt quá, cần phần tử nhỏ hơn
        }
    }

    return {}; // không tìm thấy
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — so với O(n) thời gian nhưng O(n) bộ nhớ nếu dùng HashMap như mục 3.3.2 (Two Sum trên mảng chưa sắp xếp). Đây là minh chứng rõ ràng: khi dữ liệu đã sắp xếp, Two Pointers đạt cùng độ phức tạp thời gian với HashMap nhưng **tiết kiệm bộ nhớ**.

### 4.3.2. 3Sum

**Bài toán:** tìm tất cả bộ ba phần tử phân biệt có tổng bằng 0, không trùng lặp bộ kết quả.

**Bản chất:** cố định một phần tử `arr[i]`, bài toán còn lại thu về đúng dạng Two Sum trên phần mảng còn lại (đã sắp xếp) với target = `-arr[i]`. Việc **sắp xếp mảng trước** vừa cho phép áp dụng Two Pointers, vừa giúp dễ dàng bỏ qua các giá trị trùng lặp liên tiếp để tránh bộ ba kết quả trùng nhau.

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

vector<vector<int>> threeSum(vector<int> arr) {
    vector<vector<int>> result;
    sort(arr.begin(), arr.end());
    int n = arr.size();

    for (int i = 0; i < n - 2; i++) {
        if (i > 0 && arr[i] == arr[i - 1]) continue; // bỏ qua trùng lặp cho vị trí i

        int left = i + 1, right = n - 1;
        int target = -arr[i];

        while (left < right) {
            int sum = arr[left] + arr[right];
            if (sum == target) {
                result.push_back({arr[i], arr[left], arr[right]});
                left++;
                right--;
                // Bỏ qua trùng lặp cho left và right sau khi tìm được một bộ ba
                while (left < right && arr[left] == arr[left - 1]) left++;
                while (left < right && arr[right] == arr[right + 1]) right--;
            } else if (sum < target) {
                left++;
            } else {
                right--;
            }
        }
    }

    return result;
}
```

**Độ phức tạp:** O(n²) thời gian — O(n) cho vòng lặp ngoài, nhân với O(n) cho Two Pointers bên trong; O(n log n) cho bước sắp xếp (không ảnh hưởng bậc tổng). Đây là cải tiến đáng kể so với brute force O(n³) duyệt mọi bộ ba.

### 4.3.3. Container With Most Water

**Bài toán:** cho mảng `height` biểu diễn chiều cao các cột tại từng vị trí, tìm hai cột sao cho vùng chứa nước giữa chúng có diện tích lớn nhất. Diện tích = `min(height[i], height[j]) * (j - i)`.

**Bản chất:** đặt `left` và `right` ở hai đầu (khoảng cách lớn nhất có thể). Diện tích bị giới hạn bởi cột **thấp hơn** trong hai cột biên. Chứng minh tính đơn điệu: nếu di chuyển con trỏ tại cột **cao hơn** vào trong, chiều rộng chắc chắn giảm còn chiều cao giới hạn (min) không thể tăng — diện tích mới không thể lớn hơn diện tích cũ. Vì vậy, ta luôn di chuyển con trỏ tại cột **thấp hơn**, vì đó là hướng duy nhất còn khả năng tìm ra diện tích lớn hơn.

**Minh họa** với `height = [1,8,6,2,5,4,8,3,7]`:

```
left=0 (h=1), right=8 (h=7): diện tích = min(1,7)*8 = 8
→ cột trái thấp hơn → di chuyển left

left=1 (h=8), right=8 (h=7): diện tích = min(8,7)*7 = 49  ← lớn hơn
→ cột phải thấp hơn → di chuyển right
...
```

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int maxArea(const vector<int>& height) {
    int left = 0, right = (int)height.size() - 1;
    int maxWater = 0;

    while (left < right) {
        int width = right - left;
        int boundedHeight = min(height[left], height[right]);
        maxWater = max(maxWater, width * boundedHeight);

        // Luôn di chuyển con trỏ tại cột thấp hơn
        if (height[left] < height[right]) left++;
        else right--;
    }

    return maxWater;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — so với brute force O(n²) thử mọi cặp cột.

### 4.3.4. Trapping Rain Water

**Bài toán:** cho mảng `height`, tính tổng lượng nước mưa có thể đọng lại giữa các cột.

**Bản chất:** lượng nước đọng tại vị trí `i` bị giới hạn bởi cột cao nhất bên trái và cột cao nhất bên phải vị trí đó:

```
water[i] = max(0, min(maxLeft[i], maxRight[i]) - height[i])
```

Cách tính trực tiếp cần hai mảng phụ `maxLeft`, `maxRight` — O(n) bộ nhớ. Two Pointers tối ưu bộ nhớ xuống O(1) bằng quan sát: tại mỗi bước, ta chỉ cần biết **giá trị nhỏ hơn** giữa `maxLeft` và `maxRight` tính đến hiện tại để xác định lượng nước, mà không cần biết chính xác cả hai — nếu `maxLeft < maxRight`, mực nước tại `left` chắc chắn bị giới hạn bởi `maxLeft` (vì phía phải chắc chắn có một cột ≥ `maxRight > maxLeft`).

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int trap(const vector<int>& height) {
    int left = 0, right = (int)height.size() - 1;
    int maxLeft = 0, maxRight = 0;
    int totalWater = 0;

    while (left < right) {
        if (height[left] < height[right]) {
            // maxLeft chắc chắn là giới hạn thực sự tại vị trí left
            maxLeft = max(maxLeft, height[left]);
            totalWater += maxLeft - height[left];
            left++;
        } else {
            maxRight = max(maxRight, height[right]);
            totalWater += maxRight - height[right];
            right--;
        }
    }

    return totalWater;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ.

---

## 4.4. Bảng tổng hợp

| Bài toán | Mô hình | Độ phức tạp | So với Brute Force |
|---|---|---|---|
| Two Sum (sorted) | Opposite Direction | O(n) | O(n²) → O(n) |
| 3Sum | Opposite Direction + vòng lặp ngoài | O(n²) | O(n³) → O(n²) |
| Container With Most Water | Opposite Direction | O(n) | O(n²) → O(n) |
| Trapping Rain Water | Opposite Direction | O(n) | O(n) nhưng O(1) bộ nhớ thay vì O(n) |
| Move Zeroes | Same Direction | O(n) | O(n) nhưng O(1) bộ nhớ thay vì O(n) |

---

## 4.5. Danh sách bài tập luyện tập

### Mức Easy
1. Two Sum II — Input Array Is Sorted
2. Valid Palindrome (đã trình bày ở mục 2.4.1, ôn lại dưới góc nhìn Two Pointers)
3. Reverse String
4. Merge Sorted Array (đã trình bày ở mục 1.6.7, thử giải lại bằng Two Pointers từ cuối mảng)
5. Squares of a Sorted Array

### Mức Medium
6. 3Sum
7. 3Sum Closest
8. Container With Most Water
9. Sort Colors (Dutch National Flag — ba con trỏ)
10. Remove Duplicates from Sorted Array II
11. Boats to Save People

### Mức Hard
12. Trapping Rain Water
13. 4Sum
14. Minimum Window Substring (kết hợp Sliding Window — xem Chương 5)

---

*Chương tiếp theo: **Chương 5 — Sliding Window**, mở rộng mô hình Same Direction thành kỹ thuật chuyên biệt cho các bài toán về đoạn con liên tiếp thỏa mãn điều kiện.*
