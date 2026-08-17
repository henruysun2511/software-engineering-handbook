# Chương 31: Coding Templates (Khuôn mẫu code tổng hợp)

## 31.1. Vai trò của chương này

Chương này tổng hợp lại các khuôn mẫu code **cốt lõi, tối giản** của từng kỹ thuật đã trình bày chi tiết xuyên suốt tài liệu, phục vụ mục đích **tra cứu nhanh** trong giai đoạn ôn tập cuối cùng trước buổi phỏng vấn. Mỗi khuôn mẫu đi kèm một dòng ghi chú ngắn gọn về biến thể cần điều chỉnh tùy bài toán cụ thể, và tham chiếu đến mục đã giải thích đầy đủ bản chất. **Khuôn mẫu không thay thế việc hiểu bản chất** — chỉ nên dùng như một bản kiểm tra nhanh trí nhớ (recall check), không nên học thuộc lòng mà không hiểu vì sao mỗi dòng code tồn tại.

---

## 31.2. Binary Search Template

*(Chi tiết đầy đủ: mục 10.1.3, 10.2.2, 10.3.1)*

```cpp
// Standard: tìm vị trí khớp chính xác
int binarySearch(vector<int>& arr, int target) {
    int lo = 0, hi = arr.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

// Lower Bound: vị trí đầu tiên arr[i] >= target
int lowerBound(vector<int>& arr, int target) {
    int lo = 0, hi = arr.size();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] < target) lo = mid + 1;
        else hi = mid;
    }
    return lo;
}

// Binary Search on Answer: tìm giá trị nhỏ nhất thỏa canAchieve(v)
int binarySearchOnAnswer(int loBound, int hiBound) {
    while (loBound < hiBound) {
        int mid = loBound + (hiBound - loBound) / 2;
        if (canAchieve(mid)) hiBound = mid;   // định nghĩa canAchieve() theo bài toán cụ thể
        else loBound = mid + 1;
    }
    return loBound;
}
```

---

## 31.3. Two Pointers Template

*(Chi tiết đầy đủ: mục 4.3.1)*

```cpp
// Opposite Direction: mảng đã sắp xếp, tìm cặp thỏa tổng
int left = 0, right = arr.size() - 1;
while (left < right) {
    int sum = arr[left] + arr[right];
    if (sum == target) { /* xử lý kết quả */ break; }
    else if (sum < target) left++;
    else right--;
}
```

---

## 31.4. Sliding Window Template

*(Chi tiết đầy đủ: mục 5.1.3, 5.3.3)*

```cpp
// Variable Window: mở rộng right, thu hẹp left khi vi phạm điều kiện
int left = 0;
for (int right = 0; right < n; right++) {
    // Cập nhật trạng thái cửa sổ khi thêm arr[right]

    while (/* điều kiện cửa sổ vi phạm */) {
        // Cập nhật trạng thái khi loại arr[left]
        left++;
    }

    // Cập nhật kết quả dựa trên cửa sổ [left, right]
}
```

---

## 31.5. BFS Template

*(Chi tiết đầy đủ: mục 18.2.1)*

```cpp
vector<bool> visited(n, false);
queue<int> q;
q.push(start);
visited[start] = true;

while (!q.empty()) {
    int curr = q.front(); q.pop();
    // Xử lý curr

    for (int neighbor : adjList[curr]) {
        if (!visited[neighbor]) {
            visited[neighbor] = true;
            q.push(neighbor);
        }
    }
}
```

---

## 31.6. DFS Template

*(Chi tiết đầy đủ: mục 19.2.1, 19.2.2)*

```cpp
// Recursive
void dfs(int curr, vector<vector<int>>& adjList, vector<bool>& visited) {
    visited[curr] = true;
    // Xử lý curr

    for (int neighbor : adjList[curr]) {
        if (!visited[neighbor]) dfs(neighbor, adjList, visited);
    }
}
```

---

## 31.7. Backtracking Template

*(Chi tiết đầy đủ: mục 13.1.3)*

```cpp
void backtrack(/* trạng thái hiện tại */) {
    if (/* đã đạt lời giải hoàn chỉnh */) {
        // ghi nhận kết quả
        return;
    }

    for (/* mỗi lựa chọn khả dĩ */) {
        if (/* lựa chọn hợp lệ — Pruning */) {
            // THỰC HIỆN lựa chọn
            backtrack(/* trạng thái mới */);
            // HOÀN TÁC lựa chọn
        }
    }
}
```

---

## 31.8. Heap Template

*(Chi tiết đầy đủ: mục 9.4.1)*

```cpp
// Min-Heap kích thước K cho bài toán Top K
priority_queue<int, vector<int>, greater<int>> minHeap;
for (int x : nums) {
    minHeap.push(x);
    if (minHeap.size() > k) minHeap.pop();
}
// minHeap.top() là phần tử lớn thứ k
```

---

## 31.9. Topological Sort Template (Kahn's)

*(Chi tiết đầy đủ: mục 20.2.2)*

```cpp
vector<int> inDegree(n, 0);
for (int u = 0; u < n; u++)
    for (int v : adjList[u]) inDegree[v]++;

queue<int> q;
for (int v = 0; v < n; v++) if (inDegree[v] == 0) q.push(v);

vector<int> result;
while (!q.empty()) {
    int curr = q.front(); q.pop();
    result.push_back(curr);
    for (int neighbor : adjList[curr]) {
        if (--inDegree[neighbor] == 0) q.push(neighbor);
    }
}
// Nếu result.size() != n: đồ thị có chu trình
```

---

## 31.10. Union-Find Template

*(Chi tiết đầy đủ: mục 21.2)*

```cpp
vector<int> parent(n), rank_(n, 0);
for (int i = 0; i < n; i++) parent[i] = i;

function<int(int)> find = [&](int x) {
    if (parent[x] != x) parent[x] = find(parent[x]); // Path Compression
    return parent[x];
};

auto unite = [&](int x, int y) {
    int rx = find(x), ry = find(y);
    if (rx == ry) return false;
    if (rank_[rx] < rank_[ry]) swap(rx, ry);
    parent[ry] = rx;
    if (rank_[rx] == rank_[ry]) rank_[rx]++;
    return true;
};
```

---

## 31.11. Dijkstra Template

*(Chi tiết đầy đủ: mục 22.2)*

```cpp
vector<long long> dist(n, LLONG_MAX);
dist[start] = 0;
priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> minHeap;
minHeap.push({0, start});

while (!minHeap.empty()) {
    auto [d, u] = minHeap.top(); minHeap.pop();
    if (d > dist[u]) continue; // lazy deletion

    for (auto& [v, weight] : adjList[u]) {
        if (dist[u] + weight < dist[v]) {
            dist[v] = dist[u] + weight;
            minHeap.push({dist[v], v});
        }
    }
}
```

---

## 31.12. 1D DP Template

*(Chi tiết đầy đủ: Chương 25)*

```cpp
vector<int> dp(n + 1);
dp[0] = /* base case */;

for (int i = 1; i <= n; i++) {
    dp[i] = /* công thức truy hồi, dựa trên dp[i-1], dp[i-2], ... */;
}

return dp[n]; // hoặc max/min qua toàn bộ dp[], tùy bài toán (xem mục 24.2.4)
```

---

## 31.13. 2D DP Template

*(Chi tiết đầy đủ: Chương 26, 27)*

```cpp
vector<vector<int>> dp(m + 1, vector<int>(n + 1));

// Khởi tạo Base Case cho hàng 0 / cột 0 tùy bài toán

for (int i = 1; i <= m; i++) {
    for (int j = 1; j <= n; j++) {
        dp[i][j] = /* công thức truy hồi, dựa trên dp[i-1][j], dp[i][j-1], dp[i-1][j-1] */;
    }
}

return dp[m][n];
```

---

## 31.14. Bảng tra cứu nhanh — Template nào cho bài toán nào

| Template | Dấu hiệu sử dụng |
|---|---|
| Binary Search | Dữ liệu sắp xếp / điều kiện đơn điệu theo giá trị đáp án |
| Two Pointers | Mảng sắp xếp, tìm cặp/nhóm theo tổng/hiệu |
| Sliding Window | Đoạn con liên tục thỏa điều kiện |
| BFS | Đường đi ngắn nhất không trọng số, duyệt theo lớp |
| DFS | Khám phá toàn bộ nhánh, phát hiện chu trình, thành phần liên thông |
| Backtracking | Liệt kê toàn bộ tổ hợp/hoán vị/tập con |
| Heap | Top K, truy xuất min/max liên tục |
| Topological Sort | Thứ tự có ràng buộc phụ thuộc |
| Union-Find | Liên thông động, cạnh thêm dần |
| Dijkstra | Đường đi ngắn nhất có trọng số dương |
| 1D DP | Tối ưu/đếm phụ thuộc một chỉ số |
| 2D DP | Tối ưu/đếm phụ thuộc hai chiều (lưới, hai chuỗi, vật phẩm+sức chứa) |

---

*Chương tiếp theo và cũng là chương cuối cùng: **Chương 32 — Ôn tập tổng hợp và Lộ trình học**, tổng kết toàn bộ tài liệu thành một kế hoạch ôn luyện có thời hạn cụ thể.*
