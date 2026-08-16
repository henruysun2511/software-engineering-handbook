# Chương 10: Binary Search (Tìm kiếm nhị phân)

## 10.1. Khái niệm cốt lõi

### 10.1.1. Định nghĩa

Binary Search là thuật toán tìm kiếm trên một không gian **đã sắp xếp** hoặc có **tính đơn điệu (monotonic)**, hoạt động bằng cách liên tục chia đôi không gian tìm kiếm, loại bỏ một nửa không thể chứa đáp án sau mỗi bước so sánh. Đây là ứng dụng trực tiếp của độ phức tạp O(log n) đã giới thiệu ở mục 0.1.4.

### 10.1.2. Bản chất — điều kiện tiên quyết để áp dụng được

Binary Search chỉ hoạt động đúng khi không gian tìm kiếm thỏa mãn **tính chất phân định (decision property)**: tồn tại một "ranh giới" sao cho mọi phần tử ở một phía ranh giới thỏa mãn một điều kiện, còn phía bên kia thì không — nói cách khác, hàm kiểm tra điều kiện phải là **đơn điệu** theo không gian tìm kiếm. Trường hợp phổ biến nhất là mảng đã sắp xếp (điều kiện "arr[i] ≥ target" đơn điệu từ false sang true), nhưng tính chất này áp dụng rộng hơn nhiều, kể cả trên các không gian không phải mảng số (mục 10.4).

**Minh họa trực quan** tìm `target = 23` trong mảng đã sắp xếp `[4, 8, 15, 16, 23, 42, 56]`:

```
[4, 8, 15, 16, 23, 42, 56]
 lo              mid          hi
 0                3            6

Bước 1: mid=3, arr[3]=16 < 23 → loại bỏ toàn bộ nửa trái [4,8,15,16], lo = 4

[.  .  .  .  23, 42, 56]
              lo  mid  hi
              4    5    6

Bước 2: mid=5, arr[5]=42 > 23 → loại bỏ nửa phải [42,56], hi = 4

[.  .  .  .  23]
              lo=mid=hi=4

Bước 3: mid=4, arr[4]=23 == 23 → tìm thấy, trả về chỉ số 4
```

Mỗi bước loại bỏ chính xác một nửa không gian còn lại, nên sau tối đa `⌈log₂n⌉` bước, không gian tìm kiếm thu hẹp về một phần tử duy nhất — đây là nguồn gốc trực tiếp của độ phức tạp O(log n).

### 10.1.3. Template chuẩn

**Cài đặt C++ (tìm chính xác một giá trị):**

```cpp
#include <vector>
using namespace std;

int binarySearch(const vector<int>& arr, int target) {
    int lo = 0, hi = (int)arr.size() - 1;

    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2; // tránh tràn số so với (lo+hi)/2

        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            lo = mid + 1; // target chắc chắn nằm bên phải mid
        } else {
            hi = mid - 1; // target chắc chắn nằm bên trái mid
        }
    }

    return -1; // không tìm thấy
}
```

**Lưu ý cài đặt quan trọng:** dùng `mid = lo + (hi - lo) / 2` thay vì `(lo + hi) / 2` để tránh tràn số nguyên (integer overflow) khi `lo` và `hi` đều lớn — đây là một lỗi tinh vi thường bị bỏ sót nhưng có thể được người phỏng vấn đặc biệt lưu ý.

---

## 10.2. Search Boundaries — Lower Bound và Upper Bound

### 10.2.1. Bản chất

Trong nhiều bài toán thực tế, mảng có thể chứa **phần tử trùng lặp**, và ta cần tìm không phải "một" vị trí khớp bất kỳ, mà là **vị trí đầu tiên** hoặc **vị trí cuối cùng** thỏa mãn điều kiện. Đây là lúc cần đến hai biến thể:

- **Lower Bound:** vị trí đầu tiên mà `arr[i] ≥ target` (vị trí nhỏ nhất có thể chèn `target` vào mà vẫn giữ mảng sắp xếp, chèn ở phía trước các phần tử bằng target nếu có).
- **Upper Bound:** vị trí đầu tiên mà `arr[i] > target` (vị trí chèn `target` vào mà giữ mảng sắp xếp, chèn ở phía sau các phần tử bằng target nếu có).

**Minh họa** với mảng `[1, 3, 3, 3, 5, 7]`, `target = 3`:

```
Chỉ số:   0  1  2  3  4  5
Giá trị:  1  3  3  3  5  7

Lower Bound(3) = 1   (vị trí đầu tiên có giá trị ≥ 3)
Upper Bound(3) = 4   (vị trí đầu tiên có giá trị > 3)

→ Số lượng phần tử bằng 3 = Upper Bound - Lower Bound = 4 - 1 = 3
```

### 10.2.2. Cài đặt C++ — Lower Bound Template

```cpp
#include <vector>
using namespace std;

int lowerBound(const vector<int>& arr, int target) {
    int lo = 0, hi = (int)arr.size(); // hi = size(), KHÔNG phải size()-1

    while (lo < hi) { // dùng < thay vì <=
        int mid = lo + (hi - lo) / 2;

        if (arr[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid; // arr[mid] >= target: có thể chính là đáp án, giữ lại mid trong phạm vi xét
        }
    }

    return lo; // lo == hi khi vòng lặp kết thúc, đây chính là Lower Bound
}

int upperBound(const vector<int>& arr, int target) {
    int lo = 0, hi = (int)arr.size();

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;

        if (arr[mid] <= target) { // khác biệt duy nhất so với lowerBound: dùng <=
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }

    return lo;
}
```

**Giải thích khác biệt cài đặt so với template chuẩn (mục 10.1.3):** thay vì dừng ngay khi tìm thấy khớp chính xác, ta tiếp tục thu hẹp về phía "còn khả năng cải thiện đáp án" ngay cả khi `arr[mid]` đã thỏa điều kiện — vì mục tiêu là tìm **vị trí biên**, không phải bất kỳ vị trí khớp nào. Đây là lý do vòng lặp dùng `lo < hi` (không có `=`) và `hi` khởi tạo bằng `size()` thay vì `size()-1`.

---

## 10.3. Binary Search on Answer (Tìm kiếm trên không gian đáp án)

### 10.3.1. Bản chất — nhận diện lớp bài toán này

Đây là ứng dụng mở rộng quan trọng nhất của Binary Search, thường bị bỏ sót vì đề bài **không hề nhắc đến mảng đã sắp xếp**. Nhận diện: khi đề bài hỏi "giá trị **nhỏ nhất/lớn nhất** sao cho điều kiện X được thỏa mãn", và có thể chứng minh rằng hàm kiểm tra "điều kiện X có thỏa mãn với giá trị `v` hay không" là **đơn điệu** theo `v` (nếu `v` thỏa mãn thì mọi giá trị lớn hơn/nhỏ hơn `v` cũng thỏa mãn), ta có thể Binary Search trực tiếp trên **không gian giá trị đáp án**, dùng một hàm `canAchieve(v)` như điều kiện phân định, thay vì tìm kiếm trên mảng dữ liệu gốc.

**Sơ đồ tư duy chuyển hóa bài toán:**

```
Bài toán gốc: "Tìm giá trị v nhỏ nhất sao cho canAchieve(v) = true"

Không gian đáp án:  false false false | true true true true
                                       ↑
                              Binary Search ranh giới này
```

### 10.3.2. Ví dụ minh họa — Koko Eating Bananas

**Bài toán:** Koko có `n` đống chuối, đống thứ `i` có `piles[i]` quả. Koko ăn với tốc độ `k` quả/giờ; mỗi giờ chỉ ăn từ một đống, nếu đống còn ít hơn `k` quả thì ăn hết đống đó trong giờ đó (không ăn sang đống khác). Tìm tốc độ ăn `k` nhỏ nhất để ăn hết tất cả trong `h` giờ.

**Bản chất:** đây không phải bài toán tìm kiếm trên mảng `piles`, mà là tìm kiếm trên **không gian giá trị của k** (từ 1 đến `max(piles)`). Hàm `canFinish(k)` — "với tốc độ k, có ăn hết trong h giờ không" — là đơn điệu: nếu `k` đủ nhanh để ăn hết trong `h` giờ, thì mọi tốc độ nhanh hơn `k` cũng chắc chắn đủ.

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
#include <cmath>
using namespace std;

bool canFinish(const vector<int>& piles, int k, int h) {
    long long hoursNeeded = 0;
    for (int pile : piles) {
        hoursNeeded += (pile + k - 1) / k; // làm tròn lên: ceil(pile / k)
    }
    return hoursNeeded <= h;
}

int minEatingSpeed(const vector<int>& piles, int h) {
    int lo = 1, hi = *max_element(piles.begin(), piles.end());

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;

        if (canFinish(piles, mid, h)) {
            hi = mid; // mid khả thi, thử tìm giá trị nhỏ hơn nữa
        } else {
            lo = mid + 1; // mid chưa đủ nhanh, cần tăng tốc độ
        }
    }

    return lo;
}
```

**Độ phức tạp:** O(n log m) với `n` là số đống chuối, `m` là giá trị lớn nhất trong `piles` — mỗi bước Binary Search (O(log m) bước) cần một lượt kiểm tra `canFinish` tốn O(n). So với brute force thử từng giá trị `k` từ 1 trở lên (O(n · m)), đây là cải tiến đáng kể khi `m` lớn.

### 10.3.3. Search in Rotated Sorted Array

**Bài toán:** mảng đã sắp xếp tăng dần bị "xoay" tại một điểm bất kỳ (ví dụ `[4,5,6,7,0,1,2]` là `[0,1,2,4,5,6,7]` xoay tại vị trí 4), tìm chỉ số của `target`.

**Bản chất:** dù mảng không còn sắp xếp toàn cục, tại **bất kỳ điểm chia đôi nào**, ít nhất một trong hai nửa vẫn còn giữ tính sắp xếp cục bộ. Ta xác định nửa nào còn sắp xếp, kiểm tra xem `target` có nằm trong phạm vi giá trị của nửa đó không để quyết định thu hẹp về nửa nào.

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

int searchRotated(const vector<int>& arr, int target) {
    int lo = 0, hi = (int)arr.size() - 1;

    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;

        if (arr[lo] <= arr[mid]) {
            // Nửa trái [lo, mid] đang sắp xếp bình thường
            if (arr[lo] <= target && target < arr[mid]) {
                hi = mid - 1; // target nằm trong nửa trái đã sắp xếp
            } else {
                lo = mid + 1;
            }
        } else {
            // Nửa phải [mid, hi] đang sắp xếp bình thường
            if (arr[mid] < target && target <= arr[hi]) {
                lo = mid + 1; // target nằm trong nửa phải đã sắp xếp
            } else {
                hi = mid - 1;
            }
        }
    }

    return -1;
}
```

**Độ phức tạp:** O(log n) thời gian, O(1) bộ nhớ phụ — vẫn giữ được bản chất chia đôi dù mảng không sắp xếp toàn cục.

---

## 10.4. So sánh các biến thể Binary Search

| Biến thể | Mục tiêu | Điều kiện phân định |
|---|---|---|
| Standard Binary Search | Tìm một vị trí khớp chính xác | `arr[mid] == / < / > target` |
| Lower Bound | Vị trí đầu tiên `≥ target` | `arr[mid] < target` |
| Upper Bound | Vị trí đầu tiên `> target` | `arr[mid] <= target` |
| Binary Search on Answer | Giá trị nhỏ nhất/lớn nhất thỏa điều kiện | `canAchieve(mid)` |
| Rotated Array Search | Vị trí khớp trong mảng đã xoay | Xác định nửa nào còn sắp xếp cục bộ |

---

## 10.5. Khi nào dùng Binary Search

- Dữ liệu đã sắp xếp (hoặc có thể sắp xếp mà không ảnh hưởng yêu cầu bài toán) và cần tra cứu nhanh hơn O(n).
- Đề bài hỏi giá trị nhỏ nhất/lớn nhất thỏa mãn một điều kiện, và điều kiện đó có thể chứng minh là đơn điệu — dấu hiệu ngôn ngữ thường gặp: "tối thiểu", "tối đa", "nhỏ nhất để...", "lớn nhất sao cho...".
- Cần tìm vị trí biên (lower/upper bound) trong mảng có phần tử trùng lặp.
- Không gian tìm kiếm rất lớn (đến mức duyệt tuyến tính không khả thi trong giới hạn thời gian) nhưng có tính đơn điệu.

---

## 10.6. Danh sách bài tập luyện tập

### Mức Easy
1. Binary Search (bài cơ bản, áp dụng trực tiếp mục 10.1.3)
2. Search Insert Position (chính là Lower Bound, mục 10.2.2)
3. First Bad Version (Binary Search on Answer đơn giản)
4. Sqrt(x)

### Mức Medium
5. Find First and Last Position of Element in Sorted Array (kết hợp Lower Bound và Upper Bound)
6. Search in Rotated Sorted Array
7. Find Minimum in Rotated Sorted Array
8. Find Peak Element
9. Koko Eating Bananas
10. Capacity To Ship Packages Within D Days
11. Search a 2D Matrix

### Mức Hard
12. Median of Two Sorted Arrays
13. Split Array Largest Sum (Binary Search on Answer nâng cao)
14. Find K-th Smallest Pair Distance

---

*Chương tiếp theo: **Chương 11 — Sorting**, đi qua các thao tác sắp xếp áp dụng thực tế và các pattern kết hợp Sort với những kỹ thuật đã học.*
