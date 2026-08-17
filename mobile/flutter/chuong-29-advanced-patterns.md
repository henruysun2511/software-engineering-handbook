# Chương 29: Advanced Patterns (Kỹ thuật nâng cao tổng hợp)

## 29.1. Giới thiệu

Chương này hệ thống hóa lại bốn nhóm kỹ thuật đã xuất hiện rải rác từ các chương trước (Prefix Sum ở mục 1.6.1, Monotonic Stack ở mục 7.2.3, Interval ở mục 11.4 và 23.2, Heap cho Top K ở mục 9.3.1), đồng thời bổ sung các kỹ thuật/bài toán **mới chưa được trình bày** — đặc biệt là **Quick Select**, một thuật toán quan trọng cho lớp bài toán Top-K mà tài liệu chưa đề cập đến ở Chương 9.

---

## 29.2. Prefix Sum nâng cao — 2D Prefix Sum

### 29.2.1. Bản chất

Mục 1.6.1 đã trình bày Prefix Sum một chiều cho phép truy vấn tổng đoạn `[l, r]` trong O(1). **2D Prefix Sum** mở rộng ý tưởng này sang ma trận, cho phép truy vấn tổng của một **vùng chữ nhật bất kỳ** trong O(1) sau bước tiền xử lý O(m·n).

**Công thức xây dựng**, dùng nguyên lý bù trừ (inclusion-exclusion) để tránh cộng trùng vùng giao nhau:

```
prefix[i][j] = matrix[i-1][j-1] + prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1]
```

**Minh họa nguyên lý bù trừ khi tính tổng vùng chữ nhật** `(r1,c1)` đến `(r2,c2)`:

```
   A  A  B          Vùng cần tính là D.
   A  A  B          Tổng(0,0 → r2,c2) = A+B+C+D
   C  C  D          Trừ đi phần A+B (phía trên) và A+C (bên trái)
                      sẽ trừ A hai lần → cần cộng lại A một lần

sum(r1,c1,r2,c2) = prefix[r2+1][c2+1] - prefix[r1][c2+1] - prefix[r2+1][c1] + prefix[r1][c1]
```

### 29.2.2. Cài đặt C++

```cpp
#include <vector>
using namespace std;

class NumMatrix {
private:
    vector<vector<long long>> prefix;

public:
    explicit NumMatrix(vector<vector<int>>& matrix) {
        int m = matrix.size(), n = matrix[0].size();
        prefix.assign(m + 1, vector<long long>(n + 1, 0));

        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) {
                prefix[i + 1][j + 1] = matrix[i][j] + prefix[i][j + 1]
                                      + prefix[i + 1][j] - prefix[i][j];
            }
        }
    }

    long long sumRegion(int r1, int c1, int r2, int c2) {
        return prefix[r2 + 1][c2 + 1] - prefix[r1][c2 + 1]
             - prefix[r2 + 1][c1] + prefix[r1][c1];
    }
};
```

**Độ phức tạp:** O(m·n) xây dựng, O(1) mỗi truy vấn — so với brute force O(m·n) cho **mỗi lần** truy vấn nếu tính trực tiếp, 2D Prefix Sum vượt trội khi có nhiều truy vấn trên cùng một ma trận tĩnh.

---

## 29.3. Monotonic Stack nâng cao

### 29.3.1. Ôn lại bản chất

Đã trình bày đầy đủ ở mục 7.2.3: Monotonic Stack duy trì tính đơn điệu, giải quyết lớp bài toán "phần tử lớn hơn/nhỏ hơn gần nhất" trong O(n).

### 29.3.2. Remove K Digits — ứng dụng mới

**Bài toán:** cho chuỗi số `num` và số nguyên `k`, xóa đúng `k` chữ số để số còn lại (giữ nguyên thứ tự) là **nhỏ nhất có thể**.

**Bản chất:** để số kết quả nhỏ nhất, các chữ số ở vị trí cao (trọng số lớn) cần nhỏ nhất có thể — nếu một chữ số đang xét **nhỏ hơn** chữ số ngay trước nó (đỉnh stack), việc xóa chữ số trước đó (dù đứng ở vị trí cao hơn) sẽ luôn cho kết quả tốt hơn hoặc bằng. Đây chính là Monotonic Stack tăng dần, kết hợp ràng buộc "chỉ xóa tối đa k lần".

```cpp
#include <string>
#include <stack>
using namespace std;

string removeKdigits(string num, int k) {
    string stackStr; // dùng string làm stack để tiện dựng kết quả cuối

    for (char digit : num) {
        // Khi chữ số hiện tại nhỏ hơn đỉnh stack và vẫn còn lượt xóa, xóa đỉnh stack
        while (!stackStr.empty() && k > 0 && stackStr.back() > digit) {
            stackStr.pop_back();
            k--;
        }
        stackStr.push_back(digit);
    }

    // Nếu chưa dùng hết k lượt xóa (chuỗi đã tăng dần từ đầu), xóa nốt từ cuối
    while (k > 0) {
        stackStr.pop_back();
        k--;
    }

    // Loại bỏ số 0 dẫn đầu
    int start = 0;
    while (start < (int)stackStr.size() - 1 && stackStr[start] == '0') start++;
    stackStr = stackStr.substr(start);

    return stackStr.empty() ? "0" : stackStr;
}
```

**Độ phức tạp:** O(n) thời gian (mỗi chữ số push/pop tối đa một lần — tương tự phân tích amortized ở mục 7.2.3), O(n) bộ nhớ phụ.

---

## 29.4. Interval Problems nâng cao

### 29.4.1. Insert Interval

**Bài toán:** cho danh sách khoảng đã sắp xếp và không chồng lấn, chèn thêm một khoảng mới và gộp nếu cần.

**Bản chất:** vì danh sách gốc **đã sắp xếp sẵn**, không cần sắp xếp lại từ đầu (khác với Merge Intervals ở mục 11.4.1) — chỉ cần duyệt một lượt, chia thành ba giai đoạn: (1) các khoảng hoàn toàn đứng trước khoảng mới, thêm trực tiếp; (2) các khoảng chồng lấn với khoảng mới, gộp dần vào nó; (3) các khoảng hoàn toàn đứng sau, thêm trực tiếp.

```cpp
#include <vector>
using namespace std;

vector<vector<int>> insertInterval(vector<vector<int>>& intervals, vector<int> newInterval) {
    vector<vector<int>> result;
    int i = 0, n = intervals.size();

    // Giai đoạn 1: các khoảng kết thúc trước khi newInterval bắt đầu
    while (i < n && intervals[i][1] < newInterval[0]) {
        result.push_back(intervals[i++]);
    }

    // Giai đoạn 2: gộp mọi khoảng chồng lấn với newInterval
    while (i < n && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = min(newInterval[0], intervals[i][0]);
        newInterval[1] = max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.push_back(newInterval);

    // Giai đoạn 3: các khoảng còn lại, hoàn toàn đứng sau
    while (i < n) {
        result.push_back(intervals[i++]);
    }

    return result;
}
```

**Độ phức tạp:** O(n) thời gian, O(n) bộ nhớ phụ — so với việc thêm khoảng mới rồi chạy lại toàn bộ Merge Intervals O(n log n), cách này tận dụng tính chất "đã sắp xếp sẵn" để đạt O(n).

### 29.4.2. Meeting Rooms II — kết hợp Interval và Heap

**Bài toán:** cho danh sách khoảng thời gian họp, tìm số phòng họp **tối thiểu** cần thiết để không có hai cuộc họp nào trùng giờ trong cùng phòng.

**Bản chất:** đây là ứng dụng kết hợp Sort (mục 11.4) và Heap (Chương 9) — sắp xếp các cuộc họp theo giờ bắt đầu, dùng Min-Heap lưu giờ **kết thúc** của các cuộc họp đang diễn ra. Với mỗi cuộc họp mới, nếu giờ bắt đầu của nó **≥** giờ kết thúc sớm nhất trong Heap (đỉnh Heap), phòng đó đã trống, có thể tái sử dụng (pop khỏi Heap rồi push giờ kết thúc mới); nếu không, cần thêm phòng mới. Kích thước lớn nhất của Heap trong suốt quá trình chính là số phòng tối thiểu cần thiết.

```cpp
#include <vector>
#include <queue>
#include <algorithm>
using namespace std;

int minMeetingRooms(vector<vector<int>>& intervals) {
    if (intervals.empty()) return 0;

    sort(intervals.begin(), intervals.end(),
         [](const vector<int>& a, const vector<int>& b) { return a[0] < b[0]; });

    priority_queue<int, vector<int>, greater<int>> minHeap; // lưu giờ kết thúc, Min-Heap

    for (auto& interval : intervals) {
        if (!minHeap.empty() && minHeap.top() <= interval[0]) {
            minHeap.pop(); // phòng có cuộc họp kết thúc sớm nhất đã trống, tái sử dụng
        }
        minHeap.push(interval[1]);
    }

    return minHeap.size(); // số phòng tối đa cần dùng đồng thời tại một thời điểm
}
```

**Độ phức tạp:** O(n log n) thời gian (chi phí sắp xếp và thao tác Heap), O(n) bộ nhớ phụ.

---

## 29.5. Top K nâng cao — Quick Select

### 29.5.1. Bản chất

Mục 9.3.1 và 9.4.1 đã giải bài toán Kth Largest Element bằng Min-Heap kích thước K, đạt O(n log k). **Quick Select** là một cách tiếp cận khác, dựa trên tư tưởng **Divide and Conquer** (mục 12.1.4) của thuật toán Quick Sort, đạt độ phức tạp **trung bình O(n)** — nhanh hơn cách dùng Heap.

**Ý tưởng cốt lõi:** áp dụng thao tác **partition** của Quick Sort — chọn một phần tử ngẫu nhiên làm **pivot**, sắp xếp lại mảng sao cho mọi phần tử nhỏ hơn pivot nằm bên trái, mọi phần tử lớn hơn nằm bên phải, pivot nằm đúng vị trí sắp xếp cuối cùng của nó. Sau một lần partition, ta biết chính xác **thứ hạng** của pivot trong mảng đã sắp xếp — nếu đúng bằng thứ hạng cần tìm, dừng lại; nếu không, **chỉ cần đệ quy tiếp tục vào một nửa** (trái hoặc phải) chứa vị trí cần tìm, **bỏ hẳn nửa còn lại** — đây chính là điểm khác biệt so với Quick Sort (phải đệ quy cả hai nửa để sắp xếp toàn bộ).

**Minh họa trực giác độ phức tạp:** vì mỗi lần chỉ cần đệ quy vào **một nửa** thay vì cả hai nửa như Quick Sort, tổng công việc qua các lần đệ quy giảm theo cấp số nhân (`n + n/2 + n/4 + ... ≈ 2n`), cho độ phức tạp trung bình **O(n)** — khác biệt căn bản so với Quick Sort trung bình O(n log n).

### 29.5.2. Cài đặt C++

```cpp
#include <vector>
#include <algorithm>
#include <cstdlib>
using namespace std;

int partition(vector<int>& nums, int left, int right, int pivotIndex) {
    int pivotValue = nums[pivotIndex];
    swap(nums[pivotIndex], nums[right]); // đưa pivot ra cuối tạm thời

    int storeIndex = left;
    for (int i = left; i < right; i++) {
        if (nums[i] < pivotValue) {
            swap(nums[i], nums[storeIndex]);
            storeIndex++;
        }
    }
    swap(nums[storeIndex], nums[right]); // đưa pivot về đúng vị trí sắp xếp cuối cùng

    return storeIndex; // vị trí (thứ hạng) thực sự của pivot
}

int quickSelect(vector<int>& nums, int left, int right, int targetIndex) {
    if (left == right) return nums[left];

    int pivotIndex = left + rand() % (right - left + 1); // chọn pivot ngẫu nhiên
    pivotIndex = partition(nums, left, right, pivotIndex);

    if (pivotIndex == targetIndex) {
        return nums[pivotIndex];
    } else if (pivotIndex < targetIndex) {
        return quickSelect(nums, pivotIndex + 1, right, targetIndex); // chỉ đệ quy nửa phải
    } else {
        return quickSelect(nums, left, pivotIndex - 1, targetIndex); // chỉ đệ quy nửa trái
    }
}

int findKthLargestQuickSelect(vector<int>& nums, int k) {
    int n = nums.size();
    int targetIndex = n - k; // phần tử lớn thứ k = phần tử tại vị trí (n-k) sau khi sắp xếp tăng dần
    return quickSelect(nums, 0, n - 1, targetIndex);
}
```

**Chọn pivot ngẫu nhiên — vì sao quan trọng:** nếu luôn chọn pivot cố định (ví dụ luôn lấy phần tử đầu hoặc cuối), một đầu vào được "thiết kế" đặc biệt (ví dụ mảng đã sắp xếp sẵn) có thể khiến thuật toán suy biến về O(n²) — mỗi lần partition chỉ loại bỏ đúng một phần tử thay vì một nửa. Chọn pivot ngẫu nhiên (hoặc dùng kỹ thuật median-of-three) làm giảm xác suất gặp trường hợp xấu nhất xuống gần như không đáng kể trong thực hành.

**Độ phức tạp:** O(n) thời gian trung bình, O(n²) trường hợp xấu nhất (hiếm khi xảy ra với pivot ngẫu nhiên); O(1) bộ nhớ phụ ngoài Call Stack đệ quy — vượt trội hơn cách dùng Heap (O(n log k)) về mặt lý thuyết trung bình, nhưng cài đặt phức tạp hơn và có rủi ro trường hợp xấu nhất mà Heap không gặp phải.

### 29.5.3. So sánh Heap và Quick Select cho bài toán Top K

| Tiêu chí | Min-Heap (mục 9.4.1) | Quick Select |
|---|---|---|
| Độ phức tạp trung bình | O(n log k) | O(n) |
| Độ phức tạp xấu nhất | O(n log k) — ổn định | O(n²) — hiếm khi xảy ra với pivot ngẫu nhiên |
| Phù hợp dữ liệu dạng stream (không biết trước toàn bộ) | Có — xử lý từng phần tử một | Không — cần toàn bộ mảng có sẵn để partition |
| Trả về K phần tử đã sắp xếp | Có, tự nhiên (pop dần từ Heap) | Không trực tiếp — chỉ xác định phần tử thứ K, cần thêm bước nếu muốn cả danh sách |
| Độ phức tạp cài đặt trong phỏng vấn | Đơn giản hơn (dùng `priority_queue` có sẵn) | Phức tạp hơn (tự cài partition) |

**Khi nào dùng cách nào:** Min-Heap phù hợp hơn khi dữ liệu đến dưới dạng stream hoặc khi cần K phần tử đã sắp xếp; Quick Select phù hợp hơn khi toàn bộ mảng có sẵn và chỉ cần xác định **một** giá trị thứ K (không cần thứ tự đầy đủ của K phần tử), ưu tiên độ phức tạp trung bình tốt nhất.

---

## 29.6. Bảng tổng hợp

| Bài toán | Kỹ thuật | Độ phức tạp |
|---|---|---|
| Range Sum Query 2D | 2D Prefix Sum | O(1) mỗi truy vấn sau O(m·n) xây dựng |
| Remove K Digits | Monotonic Stack tăng dần | O(n) |
| Insert Interval | Duyệt tuyến tính ba giai đoạn | O(n) |
| Meeting Rooms II | Sort + Min-Heap | O(n log n) |
| Kth Largest (Quick Select) | Divide and Conquer + partition | O(n) trung bình |

---

## 29.7. Danh sách bài tập luyện tập

### Mức Medium
1. Range Sum Query 2D — Immutable
2. Insert Interval
3. Meeting Rooms II
4. Kth Largest Element in an Array (giải lại bằng Quick Select, so sánh thời gian chạy thực tế với bản Heap ở mục 9.4.1)
5. Remove K Digits
6. Car Pooling (ứng dụng Difference Array trên bài toán khoảng thời gian, ôn lại mục 1.6.2)

### Mức Hard
7. Median of Two Sorted Arrays (có thể giải bằng biến thể Quick Select cho bài toán hai mảng)
8. Count of Range Sum (kết hợp Prefix Sum + cấu trúc dữ liệu nâng cao)
9. The Skyline Problem (kết hợp Sort + Heap nâng cao)

---

*Chương tiếp theo: **Chương 30 — Problem Recognition**, tổng hợp toàn bộ kỹ thuật đã học thành một bảng tra cứu nhanh theo dấu hiệu nhận diện đề bài — công cụ phản xạ quan trọng nhất cho buổi live coding interview thực tế.*
