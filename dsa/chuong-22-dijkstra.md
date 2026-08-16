# Chương 22: Dijkstra (Shortest Path — Đường đi ngắn nhất)

## 22.1. Khái niệm cốt lõi

### 22.1.1. Định nghĩa

Dijkstra's Algorithm là thuật toán tìm đường đi ngắn nhất từ một đỉnh nguồn đến tất cả các đỉnh còn lại trong đồ thị **có trọng số không âm**. Đây là mở rộng trực tiếp của bài toán Shortest Path trên đồ thị không trọng số đã giải bằng BFS ở mục 18.2.2, cho trường hợp tổng quát hơn khi các cạnh có "chi phí" khác nhau.

### 22.1.2. Bản chất — vì sao BFS không còn đúng khi có trọng số

BFS đảm bảo tìm đường đi ngắn nhất trên đồ thị không trọng số nhờ tính chất: đỉnh được thăm theo đúng thứ tự khoảng cách tăng dần (từng đơn vị một, mục 18.1.1). Khi cạnh có trọng số khác nhau, tính chất này bị phá vỡ — một đỉnh được BFS "thăm sớm" (do ít cạnh trung gian hơn) chưa chắc có tổng chi phí đường đi nhỏ hơn một đỉnh được thăm muộn hơn nhưng đi qua các cạnh nhẹ.

**Minh họa vấn đề:** từ đỉnh A, đường đi `A → B` (trọng số 10) chỉ qua 1 cạnh, còn đường đi `A → C → B` (trọng số 1 + 1 = 2) qua 2 cạnh. BFS sẽ thăm B trước C (ít cạnh hơn), nhưng đường đi thực sự ngắn nhất đến B lại là 2 (qua C), không phải 10 (trực tiếp) — BFS thuần túy sẽ báo cáo sai nếu áp dụng trực tiếp trên đồ thị có trọng số.

### 22.1.3. Bản chất Greedy của Dijkstra — vai trò của Heap

**Ý tưởng cốt lõi:** tại mỗi bước, Dijkstra luôn chọn xử lý đỉnh có **tổng khoảng cách tạm thời nhỏ nhất** trong số các đỉnh chưa được "chốt" (finalized) khoảng cách ngắn nhất. Đây là chiến lược Greedy: một khi đỉnh có khoảng cách nhỏ nhất trong số các ứng viên còn lại được chọn, khoảng cách đó **chắc chắn đã là khoảng cách ngắn nhất thực sự** đến đỉnh đó — vì mọi đường đi khác đến đỉnh này phải đi qua một đỉnh khác có khoảng cách tạm thời lớn hơn hoặc bằng (do trọng số không âm, không có cách nào đường đi dài hơn về số bước lại có tổng nhỏ hơn một cách bất ngờ).

**Vì sao cần Min-Heap (Chương 9):** để luôn truy xuất được đỉnh có khoảng cách tạm thời nhỏ nhất một cách hiệu quả (O(log n) thay vì quét tuyến tính O(n) mỗi bước), Dijkstra sử dụng Min-Heap lưu các cặp `(khoảng cách tạm thời, đỉnh)` — chính là ứng dụng trực tiếp của Priority Queue đã học ở Chương 9, nơi độ ưu tiên chính là khoảng cách (càng nhỏ càng ưu tiên xử lý trước).

**Minh họa từng bước** trên đồ thị có trọng số, tìm đường ngắn nhất từ A:

```
        A
      1/ \4
      B   C
      |\ /|
     2| X |1
      | / \|
      D    

Cạnh: A-B(1), A-C(4), B-D(2), B-C(3 qua đường chéo), C-D(1)

Bước 1: dist[A]=0, Heap: {(0,A)}
Bước 2: pop (0,A), chốt dist[A]=0. Cập nhật: dist[B]=1, dist[C]=4
         Heap: {(1,B), (4,C)}
Bước 3: pop (1,B), chốt dist[B]=1 (đây LÀ khoảng cách ngắn nhất thực sự đến B)
         Cập nhật: dist[D] = min(∞, 1+2) = 3, dist[C] = min(4, 1+3) = 4 (không đổi)
         Heap: {(3,D), (4,C)}
Bước 4: pop (3,D), chốt dist[D]=3
         Cập nhật: dist[C] = min(4, 3+1) = 4 (không đổi)
Bước 5: pop (4,C), chốt dist[C]=4

Kết quả: dist[A]=0, dist[B]=1, dist[C]=4, dist[D]=3
```

### 22.1.4. Vì sao Dijkstra KHÔNG hoạt động đúng với trọng số âm

**Bản chất:** toàn bộ lập luận Greedy ở mục 22.1.3 dựa trên giả định "một khi đỉnh có khoảng cách tạm thời nhỏ nhất được chốt, không có đường đi nào khác có thể cho kết quả nhỏ hơn". Giả định này **sụp đổ** nếu tồn tại cạnh trọng số âm: một đường đi tưởng chừng dài hơn (đã bị bỏ qua vì có tổng tạm thời lớn hơn) có thể đột ngột trở nên ngắn hơn sau khi đi qua một cạnh âm làm giảm tổng — nhưng vì đỉnh trung gian đã bị "chốt" (finalized) và không bao giờ được xét lại, thuật toán sẽ bỏ lỡ đường đi thực sự ngắn nhất này. Đây chính là lý do các bài toán có trọng số âm cần thuật toán khác (Bellman-Ford), nằm ngoài phạm vi trọng tâm của tài liệu ôn tập live coding interview thông thường, nhưng cần nắm được **ranh giới áp dụng** của Dijkstra khi được hỏi.

---

## 22.2. Cài đặt Dijkstra

```cpp
#include <vector>
#include <queue>
#include <climits>
using namespace std;

vector<long long> dijkstra(int start, int n, const vector<vector<pair<int,int>>>& adjList) {
    // adjList[u] chứa các cặp (đỉnh kề v, trọng số cạnh u->v)
    vector<long long> dist(n, LLONG_MAX);
    dist[start] = 0;

    // Min-Heap lưu (khoảng cách tạm thời, đỉnh) — ưu tiên khoảng cách nhỏ nhất
    priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> minHeap;
    minHeap.push({0, start});

    while (!minHeap.empty()) {
        auto [d, u] = minHeap.top();
        minHeap.pop();

        if (d > dist[u]) continue; // đã có đường đi ngắn hơn được chốt trước đó, bỏ qua entry cũ

        for (auto& [v, weight] : adjList[u]) {
            long long newDist = dist[u] + weight;
            if (newDist < dist[v]) {
                dist[v] = newDist;
                minHeap.push({newDist, v});
            }
        }
    }

    return dist;
}
```

**Giải thích điều kiện `if (d > dist[u]) continue`:** vì cùng một đỉnh có thể được `push` vào Heap **nhiều lần** với các giá trị khoảng cách khác nhau (mỗi lần tìm thấy đường đi ngắn hơn), khi `pop` ra một entry, cần kiểm tra xem entry đó có còn "hợp lệ" hay không — nếu `dist[u]` đã được cập nhật nhỏ hơn giá trị `d` trong entry đang xét (do một entry tốt hơn đã được xử lý trước đó), entry hiện tại đã lỗi thời, bỏ qua để tránh xử lý lại không cần thiết. Đây là kỹ thuật **lazy deletion** (xóa trễ) — thay vì tìm và cập nhật trực tiếp một phần tử giữa Heap (Heap không hỗ trợ hiệu quả thao tác này), ta chấp nhận Heap chứa các entry dư thừa và lọc bỏ chúng khi `pop` ra.

**Độ phức tạp:** O((V + E) log V) thời gian — mỗi cạnh có thể góp phần tạo ra một lần `push` vào Heap (O(E) lần push, mỗi lần O(log V)), tổng cộng O(E log V); O(V + E) bộ nhớ phụ cho `dist` và Heap.

---

## 22.3. So sánh Dijkstra với BFS

| Tiêu chí | BFS | Dijkstra |
|---|---|---|
| Loại đồ thị áp dụng | Không trọng số (hoặc trọng số đều bằng nhau) | Có trọng số không âm |
| Cấu trúc nền | Queue (FIFO) | Min-Heap (Priority Queue) |
| Độ phức tạp | O(V + E) | O((V + E) log V) |
| Bản chất | Mỗi bước mở rộng đồng đều theo "vòng tròn" khoảng cách | Mỗi bước ưu tiên mở rộng từ đỉnh có khoảng cách nhỏ nhất |

**Nhận định quan trọng:** BFS thực chất là **trường hợp đặc biệt** của Dijkstra khi mọi cạnh có trọng số bằng nhau (ví dụ đều bằng 1) — khi đó, Min-Heap luôn xử lý đỉnh theo đúng thứ tự mà Queue (FIFO) cũng sẽ cho ra, nên hai thuật toán trở nên tương đương, nhưng BFS đơn giản hơn và nhanh hơn (O(V+E) so với O((V+E) log V)) nên luôn được ưu tiên khi đồ thị không có trọng số.

---

## 22.4. Bảng lựa chọn thuật toán tìm đường đi ngắn nhất

| Đặc điểm đồ thị | Thuật toán phù hợp | Độ phức tạp |
|---|---|---|
| Không trọng số | BFS (Chương 18) | O(V + E) |
| Trọng số 0/1 | 0-1 BFS (Deque, mở rộng của Chương 8) | O(V + E) |
| Trọng số dương | Dijkstra | O((V + E) log V) |
| Có trọng số âm (không chu trình âm) | Bellman-Ford | O(V · E) |
| Cần khoảng cách giữa **mọi cặp** đỉnh | Floyd-Warshall | O(V³) |

*Bảng này giúp định hướng nhanh trong phỏng vấn: xác định đặc điểm trọng số của đồ thị trước, sau đó chọn thuật toán tương ứng — tránh áp dụng nhầm Dijkstra cho đồ thị có trọng số âm (mục 22.1.4).*

---

## 22.5. Khi nào dùng Dijkstra

- Bài toán tìm đường đi ngắn nhất/chi phí nhỏ nhất trên đồ thị có trọng số **không âm** (chi phí, thời gian, khoảng cách đều là số không âm trong hầu hết ngữ cảnh thực tế).
- Nhận diện qua từ khóa: "chi phí nhỏ nhất", "đường đi rẻ nhất", "thời gian ngắn nhất" trên đồ thị có trọng số cạnh khác nhau.
- Khi chỉ cần khoảng cách từ **một đỉnh nguồn** đến các đỉnh khác (khác với Floyd-Warshall cần mọi cặp).

---

## 22.6. Danh sách bài tập luyện tập

### Mức Medium
1. Network Delay Time (ứng dụng Dijkstra trực tiếp và cơ bản nhất)
2. Path with Maximum Probability (biến thể: tối đa hóa tích xác suất thay vì tối thiểu hóa tổng)
3. Cheapest Flights Within K Stops (Dijkstra có ràng buộc thêm số bước tối đa — cần điều chỉnh điều kiện dừng)

### Mức Hard
4. Path with Minimum Effort (Dijkstra kết hợp Binary Search on Answer, xem lại Chương 10)
5. Swim in Rising Water (đã gợi ý ở Chương 21 — có thể giải bằng Dijkstra biến thể "minimax path" thay vì tổng đường đi)
6. Minimum Cost to Make at Least One Valid Path in a Grid (0-1 BFS trên lưới)

---

*Đến đây, tài liệu đã hoàn thành Part VII — Graph (Chương 17-22), bao phủ toàn bộ nền tảng biểu diễn đồ thị, hai thuật toán duyệt cơ bản (BFS/DFS), sắp xếp tô-pô, cấu trúc liên thông động (Union-Find), và bài toán đường đi ngắn nhất có trọng số (Dijkstra). Chương tiếp theo: **Chương 23 — Greedy Algorithm**, hệ thống hóa tư duy "lựa chọn tối ưu cục bộ" đã xuất hiện rải rác ở Kruskal's (mục 21.4.3) và Dijkstra (mục 22.1.3) thành một kỹ thuật giải thuật độc lập.*
