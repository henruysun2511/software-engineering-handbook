# Chương 9: Heap / Priority Queue (Đống ưu tiên)

## 9.1. Khái niệm cốt lõi

### 9.1.1. Định nghĩa

Heap là cấu trúc dữ liệu dạng cây nhị phân đặc biệt, cho phép truy xuất phần tử **nhỏ nhất** (Min-Heap) hoặc **lớn nhất** (Max-Heap) trong O(1), đồng thời chèn hoặc loại bỏ phần tử đó trong O(log n). Heap là cấu trúc nền tảng để cài đặt **Priority Queue** (hàng đợi ưu tiên) — một dạng Queue nhưng thứ tự xử lý dựa trên **độ ưu tiên** thay vì thứ tự đến trước như FIFO thông thường.

### 9.1.2. Complete Binary Tree — bản chất cấu trúc

Heap là một **Complete Binary Tree** (cây nhị phân đầy đủ): mọi tầng của cây đều được lấp đầy hoàn toàn, ngoại trừ tầng cuối cùng có thể chưa đầy nhưng phải được lấp từ trái sang phải. Tính chất này quan trọng vì nó cho phép Heap được **biểu diễn trực tiếp bằng Array** mà không cần con trỏ — một điểm khác biệt căn bản so với các cây nhị phân tổng quát (Chương 14).

Với phần tử tại chỉ số `i` (0-based) trong mảng biểu diễn Heap:

```
Con trái:   2*i + 1
Con phải:   2*i + 2
Cha:        (i - 1) / 2   (chia lấy phần nguyên)
```

**Minh họa Min-Heap** biểu diễn dưới dạng cây và dạng mảng, với giá trị `[1, 3, 2, 7, 4, 5]`:

```
Dạng cây:
              1
            /   \
           3     2
          / \   /
         7   4 5

Dạng mảng: [1, 3, 2, 7, 4, 5]
Chỉ số:     0  1  2  3  4  5
```

Kiểm chứng công thức: con trái của node tại chỉ số 1 (giá trị 3) là `2*1+1=3` → `arr[3]=7` ✓; cha của node tại chỉ số 4 (giá trị 4) là `(4-1)/2=1` → `arr[1]=3` ✓.

### 9.1.3. Heap Property — tính chất thứ tự

Khác với Binary Search Tree (Chương 15) yêu cầu thứ tự nghiêm ngặt giữa con trái/phải, Heap chỉ yêu cầu **quan hệ cha-con**: trong Min-Heap, giá trị mỗi node cha luôn **nhỏ hơn hoặc bằng** giá trị hai node con (Max-Heap thì ngược lại). Vì Heap **không** đảm bảo thứ tự giữa hai node anh em hay giữa các node không cùng nhánh cha-con trực tiếp, việc tìm một phần tử bất kỳ (không phải min/max) trong Heap vẫn tốn O(n) — đây là đánh đổi có chủ đích để đổi lấy O(log n) cho insert/extract thay vì O(log n) cho mọi thao tác tìm kiếm như BST cân bằng.

### 9.1.4. Bản chất thao tác Sift Up và Sift Down — vì sao O(log n)

**Insert (chèn phần tử mới):** phần tử mới được thêm vào cuối mảng (vị trí lấp đầy tiếp theo trong Complete Binary Tree), sau đó được "nổi lên" (**sift up** / bubble up) bằng cách so sánh và hoán đổi với node cha cho đến khi tính chất Heap được khôi phục.

**Extract (lấy và loại bỏ phần tử gốc):** phần tử tại đỉnh (gốc) được lấy ra, sau đó phần tử **cuối cùng** của mảng được chuyển lên đỉnh để lấp chỗ trống, rồi "chìm xuống" (**sift down** / bubble down) bằng cách so sánh và hoán đổi với node con nhỏ hơn (Min-Heap) cho đến khi tính chất Heap được khôi phục.

**Vì sao O(log n):** vì Heap là Complete Binary Tree, chiều cao của cây với `n` phần tử luôn là O(log n) (mỗi tầng chứa gấp đôi số node của tầng trước, nên số tầng chỉ tăng theo logarit của tổng số phần tử). Cả sift up và sift down đều di chuyển tối đa một đường đi từ đỉnh xuống đáy (hoặc ngược lại), nên tốn tối đa O(log n) bước so sánh/hoán đổi.

**Minh họa Sift Up** khi chèn giá trị `0` vào Min-Heap `[1, 3, 2, 7, 4, 5]`:

```
Bước 1: thêm 0 vào cuối:        [1, 3, 2, 7, 4, 5, 0]
Bước 2: so sánh 0 với cha (2, tại chỉ số (6-1)/2=2): 0 < 2 → hoán đổi
                                 [1, 3, 0, 7, 4, 5, 2]
Bước 3: so sánh 0 với cha mới (1, tại chỉ số (2-1)/2=0): 0 < 1 → hoán đổi
                                 [0, 3, 1, 7, 4, 5, 2]
Bước 4: 0 đã ở gốc, dừng lại — Heap Property được khôi phục
```

### 9.1.5. Build Heap từ mảng có sẵn — vì sao O(n) chứ không phải O(n log n)

Một chi tiết dễ gây hiểu lầm: xây dựng Heap từ một mảng `n` phần tử có sẵn (thao tác `heapify` toàn bộ) chỉ tốn **O(n)**, không phải O(n log n) như trực giác "gọi sift down n lần, mỗi lần O(log n)" gợi ý. Lý do: thuật toán xây dựng chuẩn chỉ gọi sift down bắt đầu từ các node **không phải lá**, đi ngược từ dưới lên. Phần lớn node nằm ở các tầng gần đáy có chiều cao rất nhỏ (gần lá thì sift down gần như không tốn bước nào), chỉ một số ít node gần gốc mới có chiều cao lớn — tổng công việc khi cộng dồn theo cấp số nhân giảm dần hội tụ về O(n), tương tự bản chất chuỗi hội tụ đã gặp ở phân tích Amortized Analysis (mục 0.1.5).

---

## 9.2. So sánh Heap với các cấu trúc liên quan

| Tiêu chí | Heap | Sorted Array | BST cân bằng |
|---|---|---|---|
| Tìm min/max | O(1) | O(1) | O(log n) |
| Insert | O(log n) | O(n) | O(log n) |
| Extract min/max | O(log n) | O(n) hoặc O(1) nếu extract đúng đầu | O(log n) |
| Tìm phần tử bất kỳ | O(n) | O(log n) | O(log n) |
| Duyệt theo thứ tự | O(n log n) (phải extract lần lượt) | O(n) | O(n) |
| Build từ mảng có sẵn | O(n) | O(n log n) | O(n log n) |

**Khi nào dùng Heap:** khi chỉ cần liên tục truy xuất/loại bỏ phần tử min hoặc max mà không cần duyệt toàn bộ theo thứ tự hay tìm kiếm phần tử bất kỳ — đây là điểm khác biệt cốt lõi khiến Heap nhẹ hơn và nhanh hơn BST cân bằng cho đúng lớp bài toán này.

---

## 9.3. Ứng dụng quan trọng

### 9.3.1. Top K Problems

**Bản chất:** để tìm K phần tử lớn nhất trong `n` phần tử, thay vì sắp xếp toàn bộ O(n log n), ta duy trì một **Min-Heap kích thước K**: duyệt qua từng phần tử, nếu Heap chưa đủ K phần tử thì thêm vào, nếu đã đủ mà phần tử mới lớn hơn phần tử nhỏ nhất trong Heap (gốc của Min-Heap) thì loại gốc và thêm phần tử mới vào. Vì Heap chỉ giữ tối đa K phần tử, mỗi thao tác insert/extract chỉ tốn O(log K) thay vì O(log n) — hiệu quả khi K nhỏ hơn nhiều so với n.

### 9.3.2. Median Maintenance (Hai Heap)

**Bản chất:** để duy trì trung vị của một dòng dữ liệu (data stream) liên tục, dùng đồng thời hai Heap: một **Max-Heap** lưu nửa nhỏ hơn của dữ liệu (gốc là giá trị lớn nhất của nửa nhỏ), một **Min-Heap** lưu nửa lớn hơn (gốc là giá trị nhỏ nhất của nửa lớn), luôn giữ cân bằng kích thước giữa hai heap (chênh lệch tối đa 1 phần tử). Trung vị luôn có thể truy xuất trực tiếp từ gốc của một hoặc cả hai heap trong O(1), trong khi mỗi lần thêm phần tử mới chỉ tốn O(log n) để duy trì cân bằng.

---

## 9.4. Cài đặt các bài toán kinh điển

*(Trong C++, `std::priority_queue` là cách cài đặt Heap chuẩn của thư viện, mặc định là Max-Heap; dùng `priority_queue<int, vector<int>, greater<int>>` để có Min-Heap.)*

### 9.4.1. Kth Largest Element in an Array

**Bài toán:** tìm phần tử lớn thứ K trong mảng chưa sắp xếp.

**Cài đặt C++ (Min-Heap kích thước K, áp dụng trực tiếp mục 9.3.1):**

```cpp
#include <vector>
#include <queue>
using namespace std;

int findKthLargest(const vector<int>& arr, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap; // Min-Heap

    for (int x : arr) {
        minHeap.push(x);
        if ((int)minHeap.size() > k) {
            minHeap.pop(); // loại bỏ phần tử nhỏ nhất, giữ heap luôn có đúng K phần tử lớn nhất
        }
    }

    return minHeap.top(); // gốc của Min-Heap kích thước K chính là phần tử lớn thứ K
}
```

**Độ phức tạp:** O(n log k) thời gian, O(k) bộ nhớ phụ — so với O(n log n) nếu sắp xếp toàn bộ mảng. Khi K nhỏ hơn đáng kể so với n, cách dùng Heap vượt trội.

### 9.4.2. Top K Frequent Elements

**Bài toán:** tìm K phần tử xuất hiện nhiều nhất trong mảng.

**Bản chất:** kết hợp Frequency Counting (mục 3.3.1) để đếm tần suất bằng HashMap, sau đó áp dụng kỹ thuật Top K Problems (mục 9.3.1) trên tần suất thay vì trên giá trị gốc.

**Cài đặt C++:**

```cpp
#include <vector>
#include <unordered_map>
#include <queue>
using namespace std;

vector<int> topKFrequent(const vector<int>& arr, int k) {
    unordered_map<int, int> freq;
    for (int x : arr) freq[x]++;

    // Min-Heap theo tần suất, kích thước tối đa k
    auto compare = [](const pair<int,int>& a, const pair<int,int>& b) {
        return a.second > b.second; // đảo ngược để tạo Min-Heap theo .second
    };
    priority_queue<pair<int,int>, vector<pair<int,int>>, decltype(compare)> minHeap(compare);

    for (auto& [value, count] : freq) {
        minHeap.push({value, count});
        if ((int)minHeap.size() > k) {
            minHeap.pop();
        }
    }

    vector<int> result;
    while (!minHeap.empty()) {
        result.push_back(minHeap.top().first);
        minHeap.pop();
    }

    return result;
}
```

**Độ phức tạp:** O(n log k) thời gian, O(n) bộ nhớ phụ cho HashMap.

### 9.4.3. Task Scheduler

**Bài toán:** cho danh sách task (biểu diễn bằng ký tự) và số nguyên `n` là thời gian nghỉ tối thiểu bắt buộc giữa hai lần thực hiện cùng một loại task, tìm tổng thời gian tối thiểu để hoàn thành tất cả task (có thể xen kẽ thời gian rỗi — idle).

**Bản chất:** đây là bài toán Greedy kết hợp Heap — tại mỗi đơn vị thời gian, luôn ưu tiên thực hiện task có **tần suất còn lại cao nhất** (Max-Heap theo tần suất), vì các task này cần được "trải" ra xa nhau nhất để tránh vi phạm ràng buộc nghỉ `n` đơn vị thời gian.

**Cài đặt C++:**

```cpp
#include <vector>
#include <queue>
#include <unordered_map>
#include <algorithm>
using namespace std;

int leastInterval(const vector<char>& tasks, int n) {
    unordered_map<char, int> freq;
    for (char t : tasks) freq[t]++;

    priority_queue<int> maxHeap; // Max-Heap theo tần suất còn lại
    for (auto& [task, count] : freq) maxHeap.push(count);

    int time = 0;
    while (!maxHeap.empty()) {
        vector<int> temp; // lưu tạm các task đã dùng trong "chu kỳ" n+1 đơn vị thời gian
        int cycleLength = n + 1;

        for (int i = 0; i < cycleLength && !maxHeap.empty(); i++) {
            int count = maxHeap.top();
            maxHeap.pop();
            if (count > 1) temp.push_back(count - 1); // còn lượt thực hiện, đưa lại vào heap sau
            time++;
        }

        for (int count : temp) maxHeap.push(count);

        // Nếu heap vẫn còn task nhưng chu kỳ chưa dùng hết, cộng thêm thời gian idle
        if (!maxHeap.empty()) {
            time += cycleLength - (int)temp.size() - (cycleLength - (int)temp.size() > 0 ? 0 : 0);
        }
    }

    return time;
}
```

*Ghi chú: cài đặt trên minh họa đúng bản chất Greedy + Heap; trong thực hành phỏng vấn, phiên bản dùng công thức toán học trực tiếp (dựa trên tần suất lớn nhất và số task đạt tần suất đó) thường được trình bày như cách tối ưu hơn về hằng số thời gian, nhưng cách dùng Heap thể hiện rõ bản chất Greedy hơn và ít cần chứng minh công thức.*

**Độ phức tạp:** O(n_task · log 26) ≈ O(n_task) thời gian (kích thước Heap bị chặn bởi 26 loại task tối đa), O(26) = O(1) bộ nhớ phụ.

### 9.4.4. Find Median from Data Stream

**Bài toán:** thiết kế cấu trúc dữ liệu hỗ trợ thêm số liên tục vào dòng dữ liệu và truy vấn trung vị hiện tại bất kỳ lúc nào.

**Cài đặt C++ (áp dụng trực tiếp mục 9.3.2 — Hai Heap):**

```cpp
#include <queue>
using namespace std;

class MedianFinder {
private:
    priority_queue<int> maxHeapLower;                              // nửa nhỏ hơn
    priority_queue<int, vector<int>, greater<int>> minHeapUpper;    // nửa lớn hơn

public:
    void addNum(int num) {
        // Luôn thêm vào maxHeapLower trước, sau đó cân bằng lại nếu cần
        maxHeapLower.push(num);
        minHeapUpper.push(maxHeapLower.top());
        maxHeapLower.pop();

        // Duy trì maxHeapLower luôn có kích thước bằng hoặc nhiều hơn minHeapUpper đúng 1
        if (minHeapUpper.size() > maxHeapLower.size()) {
            maxHeapLower.push(minHeapUpper.top());
            minHeapUpper.pop();
        }
    }

    double findMedian() {
        if (maxHeapLower.size() > minHeapUpper.size()) {
            return maxHeapLower.top();
        }
        return (maxHeapLower.top() + minHeapUpper.top()) / 2.0;
    }
};
```

**Độ phức tạp:** O(log n) cho mỗi lần `addNum`, O(1) cho mỗi lần `findMedian`.

---

## 9.5. Bảng tổng hợp

| Bài toán | Kỹ thuật | Độ phức tạp |
|---|---|---|
| Kth Largest Element | Min-Heap kích thước K | O(n log k) |
| Top K Frequent Elements | Frequency Counting + Min-Heap kích thước K | O(n log k) |
| Task Scheduler | Greedy + Max-Heap | O(n_task) |
| Find Median from Data Stream | Hai Heap cân bằng | O(log n) insert, O(1) query |

---

## 9.6. Danh sách bài tập luyện tập

### Mức Easy
1. Kth Largest Element in a Stream
2. Last Stone Weight
3. Relative Ranks

### Mức Medium
4. Kth Largest Element in an Array
5. Top K Frequent Elements
6. K Closest Points to Origin
7. Task Scheduler
8. Sort Characters By Frequency
9. Reorganize String (kỹ thuật tương tự Task Scheduler)

### Mức Hard
10. Find Median from Data Stream
11. Merge K Sorted Lists (kết hợp Linked List — xem lại Chương 6)
12. Smallest Range Covering Elements from K Lists
13. IPO (kết hợp Greedy + Hai Heap)

---

*Chương tiếp theo: **Chương 10 — Binary Search**, chuyển sang nhóm kỹ thuật tìm kiếm trên không gian đã sắp xếp hoặc có tính đơn điệu, mở rộng ra cả khái niệm "tìm kiếm trên không gian đáp án".*
