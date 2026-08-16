# Chương 8: Queue & Deque (Hàng đợi)

## 8.1. Khái niệm cốt lõi

### 8.1.1. Định nghĩa

Queue là cấu trúc dữ liệu tuyến tính hoạt động theo nguyên tắc **FIFO (First In, First Out)** — phần tử được thêm vào đầu tiên sẽ là phần tử được lấy ra đầu tiên, đối lập trực tiếp với nguyên tắc LIFO của Stack (Chương 7). Hai thao tác cơ bản: `enqueue` (thêm phần tử vào cuối hàng đợi) và `dequeue` (lấy và loại bỏ phần tử ở đầu hàng đợi).

### 8.1.2. Bản chất — vì sao cài đặt bằng Array thô sơ không đạt O(1)

Nếu cài đặt Queue bằng cách đơn giản dùng một Array, thêm phần tử ở cuối (`push_back`) đạt O(1) amortized, nhưng lấy phần tử ở đầu đòi hỏi loại bỏ `arr[0]` — theo mục 1.2, thao tác này tốn O(n) vì toàn bộ phần tử phía sau phải dịch chuyển lên. Đây chính là lý do Queue **không thể** cài đặt hiệu quả bằng Array theo cách trực quan nhất — cần một cấu trúc chuyên biệt hơn.

Hai giải pháp phổ biến để đạt O(1) cho cả hai đầu:

**Circular Queue (Hàng đợi vòng):** dùng một Array cố định kích thước, cùng hai chỉ số `front` và `rear` di chuyển vòng tròn (dùng phép chia dư `% capacity`) thay vì dịch chuyển phần tử. Khi `dequeue`, ta chỉ tăng `front` lên một mà không cần di chuyển bất kỳ phần tử nào — vùng nhớ đã "trống" phía trước được tái sử dụng khi `rear` vòng lại từ đầu mảng.

**Linked List:** dùng Doubly Linked List (mục 6.1.3) với con trỏ `head` và `tail`, tận dụng trực tiếp việc chèn/xóa tại hai đầu đều là O(1) đã chứng minh ở Chương 6.

**Minh họa Circular Queue** với capacity = 5, sau khi `enqueue(1), enqueue(2), enqueue(3)`, rồi `dequeue()`, rồi `enqueue(4), enqueue(5), enqueue(6)`:

```
Sau enqueue 1,2,3:      [1, 2, 3, _, _]
                          ↑front      ↑rear (chỉ vị trí tiếp theo sẽ ghi)

Sau dequeue():          [_, 2, 3, _, _]
                             ↑front

Sau enqueue 4,5,6:      [6, 2, 3, 4, 5]     ← 6 "vòng" lại ghi đè vị trí đã trống
                            ↑front       ↑rear vòng về index 0
```

### 8.1.3. Deque (Double-Ended Queue)

Deque là dạng tổng quát hóa của Queue, cho phép thêm và loại bỏ phần tử tại **cả hai đầu** trong O(1): `push_front`, `push_back`, `pop_front`, `pop_back`. Deque bao hàm cả khả năng của Stack (dùng một đầu) lẫn Queue (dùng cả hai đầu theo một chiều cố định), nên thường được dùng làm nền tảng cài đặt cho cả hai cấu trúc trong thư viện chuẩn (`std::deque` trong C++ chính là cấu trúc nền của `std::queue` và `std::stack` mặc định).

---

## 8.2. So sánh Queue và Stack

| Tiêu chí | Stack | Queue |
|---|---|---|
| Nguyên tắc | LIFO | FIFO |
| Thao tác thêm | push (một đầu) | enqueue (một đầu) |
| Thao tác lấy | pop (cùng đầu với push) | dequeue (đầu đối diện với enqueue) |
| Cấu trúc nền phù hợp | Array (cuối) hoặc Linked List (đầu) | Circular Array hoặc Linked List (hai đầu) |
| Ứng dụng tiêu biểu | DFS, Call Stack, Undo | BFS, xử lý theo thứ tự đến trước |

---

## 8.3. Ứng dụng quan trọng

### 8.3.1. Queue trong BFS

**Bản chất:** thuật toán BFS (Breadth-First Search, trình bày chi tiết ở chương Graph) khám phá đồ thị/cây theo từng **lớp khoảng cách** tăng dần — mọi node ở khoảng cách `k` phải được xử lý trước mọi node ở khoảng cách `k+1`. Đây chính xác là tính chất FIFO: node được đưa vào hàng đợi trước (khoảng cách gần hơn) phải được xử lý trước. Nếu dùng Stack (LIFO) thay vì Queue, thuật toán sẽ trở thành DFS, khám phá theo chiều sâu thay vì theo lớp.

### 8.3.2. Deque trong Sliding Window Maximum (Monotonic Deque)

**Bài toán:** cho mảng và kích thước cửa sổ `k`, tìm giá trị lớn nhất trong mỗi cửa sổ trượt.

**Bản chất:** đây là sự kết hợp giữa kỹ thuật Sliding Window (Chương 5) và tư tưởng Monotonic Stack (mục 7.2.3), áp dụng trên Deque thay vì Stack vì cần loại bỏ phần tử ở **cả hai đầu**: đầu bên phải khi phần tử mới lớn hơn (loại bỏ phần tử cũ hơn nhưng nhỏ hơn — chúng không bao giờ có cơ hội là giá trị lớn nhất khi phần tử mới còn nằm trong cửa sổ), và đầu bên trái khi phần tử ở đầu deque đã trượt ra khỏi phạm vi cửa sổ hiện tại.

**Deque lưu chỉ số**, duy trì tính chất: giá trị tại các chỉ số trong deque luôn **giảm dần từ đầu đến cuối** — phần tử lớn nhất của cửa sổ hiện tại luôn nằm ở đầu deque, tra cứu được trong O(1).

---

## 8.4. Cài đặt các bài toán kinh điển

### 8.4.1. Cài đặt Circular Queue từ đầu

**Minh họa bản chất** cách Circular Queue tránh dịch chuyển phần tử, dùng phép chia dư để "vòng" chỉ số:

```cpp
#include <vector>
using namespace std;

class CircularQueue {
private:
    vector<int> data;
    int front;
    int count;
    int capacity;

public:
    explicit CircularQueue(int k) : data(k), front(0), count(0), capacity(k) {}

    bool enqueue(int value) {
        if (count == capacity) return false; // hàng đợi đầy
        int rear = (front + count) % capacity;
        data[rear] = value;
        count++;
        return true;
    }

    bool dequeue() {
        if (count == 0) return false; // hàng đợi rỗng
        front = (front + 1) % capacity; // chỉ dịch chỉ số, không dịch phần tử
        count--;
        return true;
    }

    int Front() const {
        return count == 0 ? -1 : data[front];
    }

    bool isEmpty() const { return count == 0; }
    bool isFull() const { return count == capacity; }
};
```

**Độ phức tạp:** O(1) cho mọi thao tác `enqueue`/`dequeue`/`Front` — không có thao tác nào dịch chuyển phần tử, khác biệt căn bản so với cách dùng Array thô sơ đã phân tích ở mục 8.1.2.

### 8.4.2. Sliding Window Maximum (Monotonic Deque)

**Cài đặt C++:**

```cpp
#include <vector>
#include <deque>
using namespace std;

vector<int> maxSlidingWindow(const vector<int>& arr, int k) {
    deque<int> dq; // lưu chỉ số, giá trị tương ứng giảm dần từ đầu đến cuối
    vector<int> result;

    for (int i = 0; i < (int)arr.size(); i++) {
        // Loại bỏ chỉ số đã trượt ra khỏi cửa sổ [i-k+1, i]
        if (!dq.empty() && dq.front() <= i - k) {
            dq.pop_front();
        }

        // Loại bỏ mọi phần tử ở cuối deque nhỏ hơn arr[i]
        // (chúng không bao giờ còn cơ hội là max khi arr[i] còn trong cửa sổ)
        while (!dq.empty() && arr[dq.back()] < arr[i]) {
            dq.pop_back();
        }

        dq.push_back(i);

        // Khi cửa sổ đầu tiên đã đủ kích thước k, bắt đầu ghi nhận kết quả
        if (i >= k - 1) {
            result.push_back(arr[dq.front()]);
        }
    }

    return result;
}
```

**Độ phức tạp:** O(n) thời gian — mỗi chỉ số được `push_back` một lần và `pop` (từ đầu hoặc cuối) tối đa một lần trên toàn bộ thuật toán; O(k) bộ nhớ phụ cho deque. So với brute force O(n·k) (tìm max riêng cho từng cửa sổ), đây là cải tiến đáng kể, đặc biệt tốt hơn cả cách dùng Heap (O(n log k), sẽ trình bày ở Chương 9) vì Monotonic Deque đạt độ phức tạp tuyến tính.

### 8.4.3. Number of Recent Calls (Queue cơ bản ứng dụng thực tế)

**Bài toán:** đếm số lượng request đã ghi nhận trong 3000 mili-giây gần nhất, mỗi lần có request mới.

**Bản chất:** dùng Queue lưu thời điểm các request; mỗi khi có request mới, loại bỏ khỏi đầu Queue mọi thời điểm đã nằm ngoài cửa sổ 3000ms (chúng chắc chắn không còn liên quan cho mọi truy vấn sau này, vì thời gian chỉ tăng dần) — cùng bản chất amortized với Sliding Window đã trình bày ở Chương 5.

**Cài đặt C++:**

```cpp
#include <queue>
using namespace std;

class RecentCounter {
private:
    queue<int> requests;

public:
    int ping(int t) {
        requests.push(t);
        // Loại bỏ mọi request nằm ngoài cửa sổ [t-3000, t]
        while (requests.front() < t - 3000) {
            requests.pop();
        }
        return requests.size();
    }
};
```

**Độ phức tạp:** O(1) amortized cho mỗi lần gọi `ping` — mỗi request chỉ bị `pop` đúng một lần trong suốt vòng đời của nó.

---

## 8.5. Bảng tổng hợp

| Bài toán | Cấu trúc | Độ phức tạp |
|---|---|---|
| Circular Queue tự cài | Array + chỉ số vòng | O(1) mỗi thao tác |
| BFS trên Graph/Grid | Queue chuẩn | O(V + E) — chi tiết ở chương Graph |
| Sliding Window Maximum | Monotonic Deque | O(n) |
| Number of Recent Calls | Queue chuẩn | O(1) amortized mỗi lần gọi |

---

## 8.6. Danh sách bài tập luyện tập

### Mức Easy
1. Implement Queue using Stacks (so sánh đánh đổi ngược lại với bài tập Chương 7)
2. Number of Recent Calls
3. Design Circular Queue
4. Moving Average from Data Stream

### Mức Medium
5. Sliding Window Maximum
6. Design Circular Deque
7. Dota2 Senate (ứng dụng Queue mô phỏng vòng loại trừ)
8. Task Scheduler (kết hợp Queue + Heap — xem thêm Chương 9)

### Mức Hard
9. Shortest Subarray with Sum at Least K (kết hợp Monotonic Deque + Prefix Sum, mục 1.6.1)
10. Constrained Subsequence Sum (kết hợp Monotonic Deque + Dynamic Programming)

---

*Chương tiếp theo: **Chương 9 — Heap / Priority Queue**, giới thiệu cấu trúc chuyên biệt cho việc truy xuất phần tử lớn nhất/nhỏ nhất hiệu quả, nền tảng cho lớp bài toán Top-K.*
