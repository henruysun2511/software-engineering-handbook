# Chương 17: Graph Fundamentals (Nền tảng đồ thị)

## 17.1. Khái niệm cốt lõi

### 17.1.1. Định nghĩa

Graph (đồ thị) là cấu trúc dữ liệu phi tuyến tính tổng quát nhất trong số các cấu trúc đã học, gồm tập hợp các **đỉnh (vertex/node)** và tập hợp các **cạnh (edge)** biểu diễn quan hệ giữa các cặp đỉnh. Đây là sự mở rộng trực tiếp của Tree (Chương 14): một Tree thực chất là một Graph đặc biệt — **liên thông (connected)**, **không chu trình (acyclic)**, và mỗi đỉnh (trừ root) có đúng một đỉnh cha. Khi bỏ các ràng buộc này (cho phép chu trình, cho phép một đỉnh có nhiều "cha", cho phép đồ thị không liên thông), ta có Graph tổng quát.

### 17.1.2. Thuật ngữ cơ bản

- **Vertex (đỉnh):** đơn vị cơ bản của đồ thị, tương đương node trong Tree.
- **Edge (cạnh):** kết nối giữa hai đỉnh, biểu diễn một quan hệ.
- **Degree (bậc)** của một đỉnh: số cạnh kết nối trực tiếp với đỉnh đó. Trong đồ thị có hướng, tách thành **in-degree** (số cạnh đi vào) và **out-degree** (số cạnh đi ra) — khái niệm in-degree sẽ đóng vai trò trung tâm ở thuật toán Kahn's (Chương 20).
- **Path (đường đi):** dãy các đỉnh liên tiếp được nối với nhau bằng cạnh.
- **Cycle (chu trình):** đường đi bắt đầu và kết thúc tại cùng một đỉnh, không lặp lại đỉnh/cạnh giữa chừng.
- **Connected Component (thành phần liên thông):** tập hợp đỉnh lớn nhất mà giữa hai đỉnh bất kỳ trong tập đó luôn tồn tại đường đi nối chúng.

### 17.1.3. Phân loại đồ thị

| Tiêu chí | Loại 1 | Loại 2 |
|---|---|---|
| Hướng của cạnh | **Undirected** (vô hướng): cạnh (u,v) đi được cả hai chiều | **Directed** (có hướng): cạnh (u→v) chỉ đi được một chiều |
| Trọng số cạnh | **Unweighted** (không trọng số): mọi cạnh coi như "chi phí" bằng nhau | **Weighted** (có trọng số): mỗi cạnh gắn một giá trị chi phí/khoảng cách |
| Tồn tại chu trình | **Cyclic**: có ít nhất một chu trình | **Acyclic**: không có chu trình nào — đồ thị có hướng không chu trình gọi là **DAG** (Directed Acyclic Graph), nền tảng của Chương 20 |

**Minh họa trực quan** phân biệt Undirected và Directed:

```
Undirected:        Directed:
   A --- B             A --> B
   |     |             ^     |
   D --- C             D <-- C

(cạnh A-B đi được    (cạnh A→B chỉ đi được
 cả hai chiều)        một chiều, cần cạnh
                       riêng B→A nếu muốn
                       chiều ngược lại)
```

---

## 17.2. Biểu diễn đồ thị

### 17.2.1. Adjacency Matrix (Ma trận kề)

**Bản chất:** dùng một ma trận vuông kích thước `V × V` (V = số đỉnh), trong đó `matrix[u][v] = 1` (hoặc giá trị trọng số) nếu tồn tại cạnh từ `u` đến `v`, ngược lại là 0. Đây là cách biểu diễn **trực tiếp và tường minh nhất** quan hệ giữa mọi cặp đỉnh.

```cpp
vector<vector<int>> adjMatrix(V, vector<int>(V, 0));
adjMatrix[u][v] = 1; // thêm cạnh u -> v (undirected: thêm thêm cả adjMatrix[v][u] = 1)
```

**Đánh giá:** kiểm tra tồn tại cạnh `(u, v)` đạt O(1) — đây là ưu thế lớn nhất. Nhưng bộ nhớ luôn tốn O(V²) bất kể số cạnh thực tế nhiều hay ít, và duyệt toàn bộ các đỉnh kề của một đỉnh cũng tốn O(V) (phải quét cả hàng dù phần lớn có thể là 0) — rất lãng phí với đồ thị **thưa (sparse)**, tức số cạnh E nhỏ hơn nhiều so với V².

### 17.2.2. Adjacency List (Danh sách kề)

**Bản chất:** với mỗi đỉnh, chỉ lưu **danh sách các đỉnh kề trực tiếp** với nó (thường dùng `vector` hoặc Linked List, Chương 6), thay vì lưu quan hệ với toàn bộ V đỉnh như Adjacency Matrix. Đây là cách biểu diễn **được ưu tiên sử dụng trong tuyệt đại đa số bài toán phỏng vấn**, vì hầu hết đồ thị thực tế đều thưa.

```cpp
vector<vector<int>> adjList(V);
adjList[u].push_back(v); // thêm cạnh u -> v (undirected: thêm thêm adjList[v].push_back(u))

// Đồ thị có trọng số: lưu cặp (đỉnh kề, trọng số)
vector<vector<pair<int,int>>> weightedAdjList(V);
weightedAdjList[u].push_back({v, weight});
```

**Đánh giá:** bộ nhớ chỉ tốn O(V + E), tỉ lệ thuận với số cạnh **thực tế** tồn tại — hiệu quả hơn hẳn Adjacency Matrix với đồ thị thưa. Duyệt toàn bộ đỉnh kề của một đỉnh chỉ tốn đúng bằng số đỉnh kề thực sự (không lãng phí quét các đỉnh không kề). Đánh đổi: kiểm tra tồn tại cạnh cụ thể `(u,v)` tốn O(degree(u)) thay vì O(1) như Adjacency Matrix (phải duyệt qua danh sách kề của u để tìm v).

### 17.2.3. Edge List (Danh sách cạnh)

**Bản chất:** lưu trực tiếp một danh sách phẳng các cạnh dưới dạng bộ ba `(u, v, weight)`, không tổ chức theo đỉnh. Cách biểu diễn này đơn giản nhất nhưng kém hiệu quả nhất cho việc **tra cứu đỉnh kề** (phải quét toàn bộ O(E) để tìm mọi cạnh liên quan đến một đỉnh) — tuy nhiên lại tự nhiên và tiện lợi cho các thuật toán xử lý **toàn bộ tập cạnh cùng lúc**, đặc biệt là các thuật toán dựa trên sắp xếp cạnh theo trọng số (ví dụ Kruskal's Algorithm, sẽ nhắc đến ở Chương 21 khi bàn về Union-Find).

```cpp
struct Edge {
    int u, v, weight;
};
vector<Edge> edgeList;
```

---

## 17.3. So sánh ba cách biểu diễn

| Tiêu chí | Adjacency Matrix | Adjacency List | Edge List |
|---|---|---|---|
| Bộ nhớ | O(V²) | O(V + E) | O(E) |
| Kiểm tra tồn tại cạnh (u,v) | O(1) | O(degree(u)) | O(E) |
| Liệt kê đỉnh kề của u | O(V) | O(degree(u)) | O(E) |
| Phù hợp đồ thị thưa (E ≪ V²) | Kém (lãng phí bộ nhớ) | Tốt | Trung bình |
| Phù hợp đồ thị dày đặc (E ≈ V²) | Tốt | Tương đương | Kém |
| Phù hợp thuật toán xử lý theo cạnh (Kruskal) | Kém | Trung bình | Tốt |

**Khi nào dùng cách nào:** trong live coding interview, **Adjacency List là lựa chọn mặc định** cho hầu hết bài toán, vì phần lớn đồ thị trong đề bài (lưới ô vuông, sơ đồ quan hệ, cây phụ thuộc) đều thưa. Adjacency Matrix chỉ nên cân nhắc khi đồ thị dày đặc hoặc khi cần tra cứu tồn tại cạnh cụ thể lặp đi lặp lại nhiều lần. Edge List phù hợp khi thuật toán cần duyệt/sắp xếp toàn bộ cạnh (Kruskal's MST) hơn là duyệt theo từng đỉnh.

---

## 17.4. Cài đặt minh họa

### 17.4.1. Xây dựng Adjacency List từ danh sách cạnh

```cpp
#include <vector>
using namespace std;

vector<vector<int>> buildAdjList(int numVertices, const vector<vector<int>>& edges,
                                   bool isDirected) {
    vector<vector<int>> adjList(numVertices);

    for (const auto& edge : edges) {
        int u = edge[0], v = edge[1];
        adjList[u].push_back(v);
        if (!isDirected) {
            adjList[v].push_back(u); // đồ thị vô hướng: thêm cạnh ngược lại
        }
    }

    return adjList;
}
```

**Độ phức tạp:** O(V + E) thời gian xây dựng, O(V + E) bộ nhớ.

### 17.4.2. Adjacency List cho lưới ô vuông (Grid)

**Bản chất:** một lưới 2D thực chất là một đồ thị ẩn, trong đó mỗi ô là một đỉnh, và các ô liền kề (thường theo 4 hướng: lên/xuống/trái/phải) được nối bằng cạnh — không cần xây dựng Adjacency List tường minh, mà tính trực tiếp đỉnh kề bằng phép cộng tọa độ.

```cpp
// Bốn hướng di chuyển: lên, xuống, trái, phải
const vector<pair<int,int>> directions = {{-1,0}, {1,0}, {0,-1}, {0,1}};

bool isValidCell(int row, int col, int rows, int cols) {
    return row >= 0 && row < rows && col >= 0 && col < cols;
}
```

Đây là khuôn mẫu sẽ được sử dụng xuyên suốt Chương 18 (BFS) và Chương 19 (DFS) khi áp dụng trên bài toán dạng lưới (Number of Islands, Rotting Oranges...).

---

## 17.5. Danh sách bài tập luyện tập

*(Chương này chủ yếu là nền tảng khái niệm; bài tập thực hành thuật toán cụ thể sẽ tập trung ở Chương 18-22. Dưới đây là bài tập rèn kỹ năng biểu diễn đồ thị.)*

### Mức Easy
1. Find the Town Judge (bài toán dùng in-degree/out-degree trực tiếp)
2. Find Center of Star Graph

### Mức Medium
3. Clone Graph (yêu cầu hiểu rõ cách duyệt và tái tạo Adjacency List)
4. Course Schedule (đọc trước — sẽ giải chi tiết bằng Topological Sort ở Chương 20)
5. All Paths From Source to Target

---

*Chương tiếp theo: **Chương 18 — BFS**, áp dụng Queue (Chương 8) để khám phá đồ thị theo từng lớp khoảng cách, thuật toán nền tảng cho bài toán đường đi ngắn nhất trên đồ thị không trọng số.*
