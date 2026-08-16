# Chương 20: Topological Sort (Sắp xếp tô-pô)

## 20.1. Khái niệm cốt lõi

### 20.1.1. Định nghĩa

Topological Sort là thuật toán sắp xếp các đỉnh của một **DAG** (Directed Acyclic Graph — đồ thị có hướng không chu trình, đã định nghĩa ở mục 17.1.3) thành một dãy tuyến tính sao cho với mọi cạnh có hướng `(u → v)`, đỉnh `u` luôn xuất hiện **trước** đỉnh `v` trong dãy kết quả. Đây chính là bài toán hình thức hóa của việc **sắp xếp thứ tự thực hiện các công việc có ràng buộc phụ thuộc** — ví dụ thứ tự học các môn học có môn tiên quyết, thứ tự biên dịch các module phần mềm phụ thuộc lẫn nhau.

### 20.1.2. Bản chất — vì sao chỉ áp dụng được trên DAG

Nếu đồ thị chứa chu trình, không thể tồn tại một thứ tự tuyến tính hợp lệ — vì chu trình `A → B → C → A` đòi hỏi đồng thời `A` phải đứng trước `B`, `B` phải đứng trước `C`, và `C` phải đứng trước `A`, một yêu cầu mâu thuẫn không thể thỏa mãn. Đây là lý do **phát hiện chu trình** (đã trình bày ở mục 19.4.3) và **Topological Sort** có mối quan hệ mật thiết — nhiều cách cài đặt Topological Sort đồng thời cũng là một phương pháp phát hiện chu trình: nếu không thể sắp xếp được đủ toàn bộ đỉnh, đồ thị chắc chắn chứa chu trình.

**Minh họa** với DAG biểu diễn môn học tiên quyết (`A → B` nghĩa là phải học A trước B):

```
A → B → D
 \      ↑
  → C →

Các thứ tự Topological hợp lệ (có thể có nhiều đáp án đúng):
  A, B, C, D
  A, C, B, D
```

---

## 20.2. Kahn's Algorithm (Dựa trên BFS và In-degree)

### 20.2.1. Bản chất

Kahn's Algorithm khai thác trực tiếp khái niệm **in-degree** (số cạnh đi vào một đỉnh, mục 17.1.2): một đỉnh có in-degree bằng 0 nghĩa là **không còn phụ thuộc** vào bất kỳ đỉnh nào chưa được xử lý — nó luôn có thể được đặt vào vị trí tiếp theo trong thứ tự Topological một cách an toàn. Sau khi "xử lý xong" một đỉnh, ta **loại bỏ** các cạnh đi ra từ nó (giảm in-degree của các đỉnh kề đi 1) — điều này có thể khiến một số đỉnh khác vừa trở thành in-degree 0, sẵn sàng được xử lý tiếp theo. Đây chính là cấu trúc BFS điển hình: dùng Queue lưu các đỉnh có in-degree 0, xử lý theo FIFO.

**Minh họa từng bước** với DAG ở mục 20.1.2:

```
In-degree ban đầu:  A=0, B=1, C=1, D=1

Bước 1: A có in-degree 0 → enqueue A, xử lý A
        Loại cạnh A→B, A→C: in-degree B=0, C=0
        Enqueue B, C

Bước 2: dequeue B, xử lý B
        Loại cạnh B→D: in-degree D=0
        (D chưa đủ điều kiện vì còn phụ thuộc C)

Bước 3: dequeue C, xử lý C
        Loại cạnh C→D: in-degree D = 0 - 1 = ... (D đã giảm về 0 từ bước 2, giờ tiếp tục giảm)
        Thực tế D có in-degree ban đầu = 2 (từ B và C), sau bước 2 còn 1, sau bước 3 còn 0
        Enqueue D

Bước 4: dequeue D, xử lý D

Kết quả: A, B, C, D
```

### 20.2.2. Cài đặt C++

```cpp
#include <vector>
#include <queue>
using namespace std;

vector<int> topologicalSortKahn(int n, const vector<vector<int>>& adjList) {
    vector<int> inDegree(n, 0);

    // Tính in-degree cho mọi đỉnh
    for (int u = 0; u < n; u++) {
        for (int v : adjList[u]) {
            inDegree[v]++;
        }
    }

    queue<int> q;
    for (int v = 0; v < n; v++) {
        if (inDegree[v] == 0) q.push(v); // mọi đỉnh không phụ thuộc gì, sẵn sàng xử lý ngay
    }

    vector<int> result;

    while (!q.empty()) {
        int curr = q.front();
        q.pop();
        result.push_back(curr);

        for (int neighbor : adjList[curr]) {
            inDegree[neighbor]--; // loại bỏ cạnh curr -> neighbor
            if (inDegree[neighbor] == 0) {
                q.push(neighbor); // neighbor vừa hết phụ thuộc, sẵn sàng xử lý
            }
        }
    }

    // Nếu không xử lý được đủ n đỉnh, đồ thị chứa chu trình — không tồn tại thứ tự hợp lệ
    if ((int)result.size() != n) return {};

    return result;
}
```

**Phát hiện chu trình qua Kahn's Algorithm:** nếu sau khi thuật toán kết thúc, số lượng đỉnh trong `result` **nhỏ hơn** tổng số đỉnh `n`, nghĩa là tồn tại những đỉnh có in-degree không bao giờ giảm về 0 — đây chính là các đỉnh nằm trong (hoặc phụ thuộc vào) một chu trình, không có cách nào "mở khóa" chúng.

**Độ phức tạp:** O(V + E) thời gian (giống BFS thông thường — tính in-degree O(E), mỗi đỉnh enqueue/dequeue đúng một lần O(V), mỗi cạnh xét đúng một lần khi giảm in-degree O(E)), O(V) bộ nhớ phụ.

---

## 20.3. DFS-based Topological Sort

### 20.3.1. Bản chất

Cách tiếp cận thứ hai dựa trên quan sát: nếu chạy DFS (Chương 19) trên DAG, đỉnh nào **hoàn tất khám phá trước** (tức đã xử lý xong toàn bộ các đỉnh con cháu của nó và chuẩn bị `return`, tương đương thời điểm chuyển màu từ GRAY sang BLACK ở mục 19.4.3) chắc chắn là đỉnh **không phụ thuộc bởi đỉnh nào khác chưa hoàn tất** — vì nếu có, đỉnh đó phải đợi con cháu hoàn tất trước. Ghi nhận thứ tự hoàn tất này (tương đương Postorder Traversal, mục 14.2.2) rồi **đảo ngược** sẽ cho ra đúng thứ tự Topological.

**Trực giác:** trong Postorder, một đỉnh được ghi nhận sau khi toàn bộ "hậu duệ" của nó (theo hướng cạnh) đã được ghi nhận — nghĩa là hậu duệ luôn đứng **trước** trong danh sách Postorder. Nhưng Topological Sort yêu cầu điều ngược lại (đỉnh nguồn đứng trước đỉnh đích), nên cần đảo ngược danh sách Postorder để có thứ tự đúng.

### 20.3.2. Cài đặt C++

```cpp
#include <vector>
#include <algorithm>
using namespace std;

void dfsTopo(int curr, const vector<vector<int>>& adjList,
             vector<bool>& visited, vector<int>& postorderResult) {
    visited[curr] = true;

    for (int neighbor : adjList[curr]) {
        if (!visited[neighbor]) {
            dfsTopo(neighbor, adjList, visited, postorderResult);
        }
    }

    postorderResult.push_back(curr); // ghi nhận SAU KHI đã khám phá xong toàn bộ hậu duệ
}

vector<int> topologicalSortDFS(int n, const vector<vector<int>>& adjList) {
    vector<bool> visited(n, false);
    vector<int> postorderResult;

    for (int v = 0; v < n; v++) {
        if (!visited[v]) {
            dfsTopo(v, adjList, visited, postorderResult);
        }
    }

    reverse(postorderResult.begin(), postorderResult.end()); // đảo ngược Postorder
    return postorderResult;
}
```

**Lưu ý về phát hiện chu trình:** cách DFS-based này **không tự động** phát hiện chu trình như Kahn's Algorithm — nếu cần đảm bảo tính đúng đắn trên đồ thị có thể chứa chu trình, cần kết hợp thêm kỹ thuật ba màu đã trình bày ở mục 19.4.3 để kiểm tra trước hoặc trong lúc chạy DFS.

**Độ phức tạp:** O(V + E) thời gian, O(V) bộ nhớ phụ.

---

## 20.4. So sánh Kahn's Algorithm và DFS-based

| Tiêu chí | Kahn's Algorithm (BFS-based) | DFS-based |
|---|---|---|
| Cấu trúc nền | Queue + mảng in-degree | Call Stack (đệ quy) |
| Tự động phát hiện chu trình | Có (đếm số đỉnh xử lý được) | Không (cần bổ sung kỹ thuật ba màu) |
| Thứ tự kết quả khi có nhiều đáp án đúng | Theo thứ tự in-degree giảm về 0 dần | Theo thứ tự đảo ngược Postorder DFS |
| Độ phức tạp | O(V + E) | O(V + E) |
| Trực quan/dễ giải thích | Rất trực quan (mô phỏng đúng "xử lý dần các việc hết phụ thuộc") | Cần hiểu rõ bản chất Postorder mới thấy trực quan |

**Khi nào dùng cách nào:** trong live coding interview, **Kahn's Algorithm thường được ưu tiên** vì tự nhiên tích hợp khả năng phát hiện chu trình (không cần thêm biến trạng thái phức tạp như ba màu), và logic "xử lý dần các việc hết phụ thuộc" dễ giải thích bằng lời hơn với người phỏng vấn.

---

## 20.5. Cài đặt bài toán kinh điển

### 20.5.1. Course Schedule

**Bài toán:** cho `numCourses` môn học và danh sách cặp `[a, b]` nghĩa là phải học `b` trước `a`, xác định có thể hoàn thành tất cả môn học hay không (tức đồ thị phụ thuộc không chứa chu trình).

**Cài đặt C++ (dùng Kahn's Algorithm, tận dụng khả năng phát hiện chu trình tự nhiên):**

```cpp
#include <vector>
#include <queue>
using namespace std;

bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
    vector<vector<int>> adjList(numCourses);
    vector<int> inDegree(numCourses, 0);

    for (auto& p : prerequisites) {
        int course = p[0], prereq = p[1];
        adjList[prereq].push_back(course); // cạnh prereq -> course
        inDegree[course]++;
    }

    queue<int> q;
    for (int v = 0; v < numCourses; v++) {
        if (inDegree[v] == 0) q.push(v);
    }

    int processedCount = 0;

    while (!q.empty()) {
        int curr = q.front();
        q.pop();
        processedCount++;

        for (int neighbor : adjList[curr]) {
            if (--inDegree[neighbor] == 0) {
                q.push(neighbor);
            }
        }
    }

    return processedCount == numCourses; // đủ n đỉnh được xử lý nghĩa là không có chu trình
}
```

**Độ phức tạp:** O(V + E) thời gian với V = numCourses, E = số cặp prerequisite; O(V + E) bộ nhớ phụ.

### 20.5.2. Course Schedule II

**Bài toán:** mở rộng bài toán trên, trả về **thứ tự cụ thể** để hoàn thành tất cả môn học, hoặc mảng rỗng nếu không thể.

**Cài đặt C++:** áp dụng trực tiếp mục 20.2.2, chỉ cần trả về `result` thay vì so sánh kích thước.

```cpp
#include <vector>
#include <queue>
using namespace std;

vector<int> findOrder(int numCourses, vector<vector<int>>& prerequisites) {
    vector<vector<int>> adjList(numCourses);
    vector<int> inDegree(numCourses, 0);

    for (auto& p : prerequisites) {
        int course = p[0], prereq = p[1];
        adjList[prereq].push_back(course);
        inDegree[course]++;
    }

    queue<int> q;
    for (int v = 0; v < numCourses; v++) {
        if (inDegree[v] == 0) q.push(v);
    }

    vector<int> order;

    while (!q.empty()) {
        int curr = q.front();
        q.pop();
        order.push_back(curr);

        for (int neighbor : adjList[curr]) {
            if (--inDegree[neighbor] == 0) {
                q.push(neighbor);
            }
        }
    }

    if ((int)order.size() != numCourses) return {}; // có chu trình, không tồn tại thứ tự hợp lệ

    return order;
}
```

**Độ phức tạp:** O(V + E) thời gian, O(V + E) bộ nhớ phụ.

---

## 20.6. Bảng tổng hợp

| Bài toán | Kỹ thuật | Độ phức tạp |
|---|---|---|
| Topological Sort (Kahn's) | BFS + in-degree | O(V + E) |
| Topological Sort (DFS-based) | Postorder DFS + đảo ngược | O(V + E) |
| Course Schedule | Kahn's, kiểm tra processedCount | O(V + E) |
| Course Schedule II | Kahn's, trả về thứ tự đầy đủ | O(V + E) |

---

## 20.7. Khi nào dùng Topological Sort

- Bài toán có ràng buộc phụ thuộc dạng "A phải xảy ra trước B" và cần tìm một thứ tự thực hiện hợp lệ (hoặc xác định thứ tự đó có tồn tại hay không).
- Bài toán lập lịch (scheduling) công việc có phụ thuộc.
- Xác định thứ tự biên dịch module phần mềm, thứ tự cài đặt gói phụ thuộc (dependency resolution).
- Nhận diện qua từ khóa: "tiên quyết" (prerequisite), "phụ thuộc" (dependency), "phải hoàn thành trước".

---

## 20.8. Danh sách bài tập luyện tập

### Mức Medium
1. Course Schedule
2. Course Schedule II
3. Alien Dictionary (xây dựng đồ thị thứ tự ký tự từ danh sách từ đã sắp xếp theo "bảng chữ cái ngoài hành tinh")
4. Minimum Height Trees
5. Sequence Reconstruction

### Mức Hard
6. Course Schedule III (kết hợp Greedy + Heap, xem lại Chương 8-9)
7. Parallel Courses (biến thể tính số bước tối thiểu thay vì chỉ thứ tự)
8. Sort Items by Groups Respecting Dependencies (Topological Sort hai cấp)

---

*Chương tiếp theo: **Chương 21 — Union-Find (DSU)**, giới thiệu cấu trúc dữ liệu chuyên biệt cho bài toán liên thông động, một cách tiếp cận khác so với BFS/DFS đã học ở hai chương trước.*
