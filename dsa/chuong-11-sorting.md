# Chương 11: Sorting (Sắp xếp)

## 11.1. Khái niệm cốt lõi

### 11.1.1. Định nghĩa và phạm vi chương này

Sorting là thao tác sắp xếp lại các phần tử của một tập dữ liệu theo một thứ tự xác định (tăng dần, giảm dần, hoặc theo một tiêu chí tùy chỉnh). Trong phạm vi live coding interview, việc **tự cài đặt thuật toán sắp xếp từ đầu** (Merge Sort, Quick Sort...) hiếm khi được yêu cầu trực tiếp — hầu hết ngôn ngữ lập trình đều cung cấp hàm sắp xếp built-in hiệu năng cao (`std::sort` trong C++, tận dụng thuật toán Introsort với độ phức tạp trung bình O(n log n)). Vì vậy, chương này tập trung vào **cách vận dụng thao tác sắp xếp như một bước tiền xử lý** để đơn giản hóa các bài toán phức tạp hơn, kết hợp với các kỹ thuật đã học ở những chương trước.

### 11.1.2. Bản chất — vì sao sắp xếp trước lại giúp giải bài dễ hơn

Nhiều bài toán trở nên khó vì dữ liệu đầu vào ở dạng "ngẫu nhiên" — không có mối quan hệ thứ tự rõ ràng giữa các phần tử để khai thác. Sắp xếp trước biến dữ liệu ngẫu nhiên thành dữ liệu có **tính đơn điệu**, đúng điều kiện tiên quyết để áp dụng Two Pointers (Chương 4) hoặc Binary Search (Chương 10) — hai kỹ thuật vốn chỉ hoạt động đúng trên dữ liệu đã sắp xếp. Cái giá phải trả là chi phí O(n log n) cho bước sắp xếp, nhưng đổi lại các bước xử lý sau đó thường giảm từ O(n²) xuống O(n) — tổng thể độ phức tạp bài toán vẫn được cải thiện đáng kể so với brute force O(n²) hoặc O(n³) trên dữ liệu chưa sắp xếp.

### 11.1.3. Stable và Unstable Sort

**Bản chất:** một thuật toán sắp xếp được gọi là **ổn định (stable)** nếu nó giữ nguyên **thứ tự tương đối** giữa các phần tử có giá trị bằng nhau sau khi sắp xếp. Tính chất này quan trọng khi sắp xếp dữ liệu có nhiều tiêu chí (ví dụ sắp xếp danh sách nhân viên theo phòng ban, rồi trong từng phòng ban sắp xếp theo tên — nếu bước sắp xếp theo phòng ban không ổn định, thứ tự tên đã sắp xếp trước đó có thể bị xáo trộn).

**Minh họa:** sắp xếp danh sách `[(A,2), (B,1), (C,2)]` theo giá trị số:

```
Stable sort:    [(B,1), (A,2), (C,2)]   ← A vẫn đứng trước C (giữ thứ tự gốc)
Unstable sort:  [(B,1), (C,2), (A,2)]   ← thứ tự A, C có thể bị đảo (không đảm bảo)
```

`std::sort` trong C++ **không đảm bảo ổn định**; khi cần tính ổn định, dùng `std::stable_sort` (đánh đổi hiệu năng: O(n log n) trong hầu hết trường hợp nhưng có thể cần thêm O(n) bộ nhớ phụ tùy cài đặt).

---

## 11.2. Custom Comparator (Bộ so sánh tùy chỉnh)

**Bản chất:** hàm `sort` mặc định chỉ hiểu cách so sánh các kiểu dữ liệu cơ bản (số, ký tự). Khi cần sắp xếp theo tiêu chí phức tạp hơn (nhiều trường dữ liệu, thứ tự ngược, hoặc quy tắc đặc thù bài toán), cần cung cấp một **hàm so sánh (comparator)** định nghĩa quan hệ "phần tử nào đứng trước phần tử nào".

**Cài đặt C++ — sắp xếp mảng cặp theo trường thứ hai giảm dần, nếu bằng nhau thì theo trường đầu tăng dần:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

void customSort(vector<pair<int,int>>& arr) {
    sort(arr.begin(), arr.end(), [](const pair<int,int>& a, const pair<int,int>& b) {
        if (a.second != b.second) {
            return a.second > b.second; // giảm dần theo .second
        }
        return a.first < b.first;       // bằng nhau: tăng dần theo .first
    });
}
```

**Nguyên tắc viết comparator đúng:** hàm phải trả về `true` khi và chỉ khi phần tử thứ nhất **thực sự cần đứng trước** phần tử thứ hai trong kết quả cuối cùng, và phải tuân theo quan hệ thứ tự chặt (strict weak ordering) — nếu `cmp(a, b)` và `cmp(b, a)` cùng trả về `true` cho hai phần tử khác nhau, hành vi của `sort` sẽ không xác định (undefined behavior), một lỗi tinh vi cần đặc biệt lưu ý khi viết comparator trong lúc phỏng vấn.

---

## 11.3. Interview Pattern: Sort + Two Pointers

**Bản chất:** đã minh họa trực tiếp ở Chương 4 (mục 4.3.2 — 3Sum): khi bài toán yêu cầu tìm bộ phần tử thỏa mãn điều kiện tổng/hiệu mà **không có ràng buộc thứ tự xuất hiện gốc** (chỉ cần đúng tập giá trị), sắp xếp trước cho phép áp dụng Two Pointers, biến độ phức tạp từ O(n³) (brute force ba vòng lặp lồng) xuống O(n²) (một vòng lặp ngoài kết hợp Two Pointers O(n) bên trong).

**Ví dụ bổ sung — Merge Intervals (thuộc pattern Sort + Greedy, trình bày ở mục 11.4, nhưng minh họa thêm ở đây về vai trò của bước sắp xếp):** nếu không sắp xếp các khoảng theo điểm bắt đầu trước, việc xác định hai khoảng có giao nhau hay không đòi hỏi so sánh **mọi cặp** khoảng — O(n²). Sau khi sắp xếp, chỉ cần so sánh khoảng hiện tại với khoảng **liền trước** đã xử lý — O(n).

---

## 11.4. Interview Pattern: Sort + Greedy

### 11.4.1. Merge Intervals

**Bài toán:** cho danh sách các khoảng `[start, end]`, gộp các khoảng chồng lấn (overlapping) thành khoảng lớn nhất có thể.

**Bản chất:** sau khi sắp xếp các khoảng theo `start` tăng dần, hai khoảng chỉ có thể chồng lấn nếu chúng **liền kề nhau** trong thứ tự đã sắp xếp (nếu khoảng A không chồng lấn với khoảng liền sau B, A chắc chắn không chồng lấn với bất kỳ khoảng nào sau B nữa, vì các khoảng sau B đều có `start` lớn hơn hoặc bằng `start` của B). Đây là ví dụ điển hình của **Sort + Greedy**: sắp xếp trước biến bài toán so sánh mọi cặp thành bài toán chỉ cần xét từng cặp liền kề theo một lượt duyệt.

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

vector<vector<int>> mergeIntervals(vector<vector<int>>& intervals) {
    if (intervals.empty()) return {};

    sort(intervals.begin(), intervals.end(),
         [](const vector<int>& a, const vector<int>& b) {
             return a[0] < b[0]; // sắp xếp theo điểm bắt đầu
         });

    vector<vector<int>> result;
    result.push_back(intervals[0]);

    for (int i = 1; i < (int)intervals.size(); i++) {
        // So sánh khoảng hiện tại chỉ với khoảng liền trước đã gộp (result.back())
        if (intervals[i][0] <= result.back()[1]) {
            result.back()[1] = max(result.back()[1], intervals[i][1]); // gộp
        } else {
            result.push_back(intervals[i]); // không chồng lấn, thêm khoảng mới
        }
    }

    return result;
}
```

**Độ phức tạp:** O(n log n) thời gian (chi phí chính nằm ở bước sắp xếp), O(n) hoặc O(log n) bộ nhớ phụ tùy cài đặt sắp xếp — so với brute force O(n²) so sánh mọi cặp khoảng.

### 11.4.2. Non-overlapping Intervals

**Bài toán:** tìm số lượng khoảng tối thiểu cần loại bỏ để các khoảng còn lại không chồng lấn nhau.

**Bản chất:** đây là ứng dụng của **Activity Selection Problem** kinh điển — chiến lược Greedy tối ưu là sắp xếp các khoảng theo **điểm kết thúc** tăng dần (không phải điểm bắt đầu), sau đó luôn ưu tiên giữ lại khoảng có điểm kết thúc sớm nhất trong số các khoảng đang xét, vì nó "nhường chỗ" nhiều nhất cho các khoảng tiếp theo.

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int eraseOverlapIntervals(vector<vector<int>>& intervals) {
    if (intervals.empty()) return 0;

    sort(intervals.begin(), intervals.end(),
         [](const vector<int>& a, const vector<int>& b) {
             return a[1] < b[1]; // sắp xếp theo điểm KẾT THÚC
         });

    int count = 0;
    int lastEnd = intervals[0][1];

    for (int i = 1; i < (int)intervals.size(); i++) {
        if (intervals[i][0] < lastEnd) {
            count++; // khoảng này chồng lấn với khoảng vừa giữ lại, phải loại bỏ nó
        } else {
            lastEnd = intervals[i][1]; // giữ lại khoảng này, cập nhật mốc kết thúc
        }
    }

    return count;
}
```

**Độ phức tạp:** O(n log n) thời gian, O(1) bộ nhớ phụ (ngoài chi phí sắp xếp).

---

## 11.5. Bảng tổng hợp

| Pattern | Bước sắp xếp theo tiêu chí | Kỹ thuật kết hợp sau khi sắp xếp | Độ phức tạp tổng |
|---|---|---|---|
| Sort + Two Pointers | Giá trị phần tử | Two Pointers | O(n log n) hoặc O(n² log n) tùy bài |
| Sort + Greedy (Merge Intervals) | Điểm bắt đầu khoảng | So sánh với phần tử liền trước | O(n log n) |
| Sort + Greedy (Non-overlapping) | Điểm kết thúc khoảng | Đếm số lần vi phạm | O(n log n) |
| Custom Comparator | Tiêu chí tùy chỉnh | Tùy bài toán | O(n log n) |

---

## 11.6. Khi nào dùng Sort như bước tiền xử lý

- Bài toán liên quan đến khoảng (interval) chồng lấn — gần như luôn cần sắp xếp trước theo điểm bắt đầu hoặc kết thúc tùy chiến lược Greedy cụ thể.
- Bài toán tìm bộ phần tử thỏa điều kiện tổng/hiệu mà không quan tâm thứ tự xuất hiện gốc — mở khóa khả năng dùng Two Pointers.
- Bài toán Greedy nói chung — phần lớn chiến lược Greedy đòi hỏi xử lý phần tử theo một thứ tự ưu tiên nhất định, đạt được bằng cách sắp xếp trước (xem thêm ở chương Greedy).
- Khi cần loại bỏ trùng lặp hoặc nhóm các phần tử giống nhau lại gần nhau (ví dụ chuẩn hóa Anagram bằng cách sắp xếp ký tự, đã trình bày ở mục 2.4.3).

---

## 11.7. Danh sách bài tập luyện tập

### Mức Easy
1. Sort Array By Parity
2. Height Checker
3. Relative Sort Array

### Mức Medium
4. Merge Intervals
5. Non-overlapping Intervals
6. Meeting Rooms II (kết hợp Sort + Heap — xem lại Chương 9)
7. Largest Number (Custom Comparator trên chuỗi số)
8. Sort Colors (đã gợi ý ở Chương 4 dưới góc nhìn Two Pointers — Dutch National Flag; thử so sánh với cách giải bằng đếm tần suất + sắp xếp)
9. Minimum Number of Arrows to Burst Balloons

### Mức Hard
10. Merge k Sorted Lists (so sánh cách giải bằng Sort toàn bộ vs cách dùng Heap ở Chương 9)
11. Count of Smaller Numbers After Self (kết hợp Merge Sort biến thể — đếm nghịch thế)

---

*Chương tiếp theo: **Chương 12 — Recursion**, quay lại nền tảng đệ quy đã được nhắc đến ở nhiều chương trước (Reverse Linked List, Call Stack), làm rõ bản chất và chuẩn bị cho Chương 13 — Backtracking.*
