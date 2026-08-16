# Chương 19: DFS (Depth-First Search — Tìm kiếm theo chiều sâu)

## 19.1. Khái niệm cốt lõi

### 19.1.1. Định nghĩa

DFS là thuật toán duyệt đồ thị theo nguyên tắc **đi sâu nhất có thể** theo một nhánh trước khi quay lại (backtrack) để khám phá nhánh khác. Đối lập trực tiếp với BFS (Chương 18, khám phá theo từng lớp khoảng cách), DFS ưu tiên độ sâu hơn độ rộng — đây chính là Preorder/Postorder Traversal (mục 14.2.2) tổng quát hóa từ Tree sang Graph.

### 19.1.2. Bản chất — vì sao dùng Stack (tường minh hoặc qua đệ quy)

Nguyên tắc "đi sâu theo nhánh mới phát hiện gần nhất trước khi quay lại nhánh cũ" chính xác là nguyên tắc **LIFO** đã trình bày ở Chương 7: đỉnh vừa được phát hiện (thuộc nhánh đang khám phá) phải được xử lý ngay, trước khi quay lại các đỉnh đã phát hiện từ sớm hơn. Cài đặt đệ quy (mục 12.1.2) tận dụng trực tiếp Call Stack có sẵn của hệ thống — mỗi lời gọi đệ quy tương đương một lần `push`; cài đặt lặp dùng một Stack tường minh để đạt hiệu quả tương đương, giống mối quan hệ giữa Recursive và Iterative Traversal đã bàn ở mục 14.2.3.

**Minh họa DFS** trên cùng đồ thị đã dùng ở mục 18.1.2, xuất phát từ đỉnh A:

```
      A
    / | \
   B  C  D
   |     |
   E     F

Bước 1: thăm A, đi sâu vào nhánh đầu tiên: B
Bước 2: thăm B, đi sâu vào nhánh của B: E
Bước 3: thăm E, hết nhánh (E không có đỉnh kề chưa thăm) → quay lui về B
Bước 4: B hết nhánh chưa thăm → quay lui về A
Bước 5: A còn nhánh C chưa thăm → thăm C, hết nhánh → quay lui về A
Bước 6: A còn nhánh D chưa thăm → thăm D, đi sâu vào nhánh của D: F
Bước 7: thăm F, hết nhánh → quay lui

Thứ tự thăm: A, B, E, C, D, F   ← đi sâu triệt để từng nhánh trước khi chuyển nhánh khác
```

So sánh với thứ tự BFS trên cùng đồ thị (A, B, C, D, E, F) ở mục 18.1.2 cho thấy rõ khác biệt căn bản: BFS thăm hết "lớp 1" (B, C, D) trước khi sang "lớp 2", còn DFS đi thẳng xuống hết một nhánh (A→B→E) trước khi quay lại thử nhánh khác.

---

## 19.2. Cài đặt DFS cơ bản

### 19.2.1. Recursive DFS

```cpp
#include <vector>
using namespace std;

void dfsRecursive(int curr, const vector<vector<int>>& adjList,
                   vector<bool>& visited, vector<int>& order) {
    visited[curr] = true;
    order.push_back(curr);

    for (int neighbor : adjList[curr]) {
        if (!visited[neighbor]) {
            dfsRecursive(neighbor, adjList, visited, order);
        }
    }
}

vector<int> dfs(int start, const vector<vector<int>>& adjList) {
    int n = adjList.size();
    vector<bool> visited(n, false);
    vector<int> order;

    dfsRecursive(start, adjList, visited, order);
    return order;
}
```

**Độ phức tạp:** O(V + E) thời gian (giống BFS — mỗi đỉnh và mỗi cạnh chỉ được xét một lần), O(V) bộ nhớ phụ cho `visited`, cộng thêm O(h) cho Call Stack với `h` là chiều sâu đệ quy tối đa (có thể lên đến O(V) trong trường hợp xấu nhất — đồ thị dạng chuỗi dài).

### 19.2.2. Iterative DFS (dùng Stack tường minh)

**Điểm cần lưu ý:** khác với BFS chỉ cần đánh dấu `visited` khi đưa vào Queue, DFS dùng Stack tường minh cần cẩn trọng hơn — vì một đỉnh có thể được đẩy vào Stack **nhiều lần** từ các đỉnh cha khác nhau trước khi nó thực sự được xử lý, nên việc kiểm tra `visited` cần thực hiện **cả khi push lẫn khi pop** để tránh xử lý trùng lặp.

```cpp
#include <vector>
#include <stack>
using namespace std;

vector<int> dfsIterative(int start, const vector<vector<int>>& adjList) {
    int n = adjList.size();
    vector<bool> visited(n, false);
    vector<int> order;

    stack<int> st;
    st.push(start);

    while (!st.empty()) {
        int curr = st.top();
        st.pop();

        if (visited[curr]) continue; // có thể đã bị push trùng trước đó, bỏ qua nếu đã xử lý
        visited[curr] = true;
        order.push_back(curr);

        for (int neighbor : adjList[curr]) {
            if (!visited[neighbor]) {
                st.push(neighbor);
            }
        }
    }

    return order;
}
```

**Độ phức tạp:** O(V + E) thời gian, O(V) bộ nhớ phụ — tương đương bản đệ quy, nhưng tránh rủi ro Stack Overflow khi đồ thị có đường đi rất dài (đổi lấy code phức tạp hơn một chút).

---

## 19.3. DFS trên Grid và Connected Components

### 19.3.1. Number of Islands — giải lại bằng DFS

**So sánh với lời giải BFS (mục 18.3.2):** cùng bản chất "khám phá toàn bộ thành phần liên thông từ một điểm chưa thăm", nhưng DFS thể hiện gọn hơn nhờ đệ quy tự nhiên, không cần quản lý Queue tường minh.

```cpp
#include <vector>
using namespace std;

void dfsIsland(vector<vector<char>>& grid, int row, int col,
               vector<vector<bool>>& visited) {
    int rows = grid.size(), cols = grid[0].size();

    if (row < 0 || row >= rows || col < 0 || col >= cols) return;
    if (visited[row][col] || grid[row][col] == '0') return;

    visited[row][col] = true;

    dfsIsland(grid, row + 1, col, visited);
    dfsIsland(grid, row - 1, col, visited);
    dfsIsland(grid, row, col + 1, visited);
    dfsIsland(grid, row, col - 1, visited);
}

int numIslandsDFS(vector<vector<char>>& grid) {
    int rows = grid.size(), cols = grid[0].size();
    vector<vector<bool>> visited(rows, vector<bool>(cols, false));
    int islandCount = 0;

    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            if (grid[r][c] == '1' && !visited[r][c]) {
                islandCount++;
                dfsIsland(grid, r, c, visited);
            }
        }
    }

    return islandCount;
}
```

**Độ phức tạp:** O(rows · cols) thời gian — tương đương bản BFS, khác biệt chỉ nằm ở O(rows · cols) bộ nhớ Call Stack trong trường hợp xấu nhất (thay vì O(rows · cols) cho Queue ở bản BFS) — cả hai đều cùng bậc độ phức tạp không gian.

### 19.3.2. Đếm số thành phần liên thông (Connected Components)

**Bản chất:** áp dụng đúng khuôn mẫu của Number of Islands nhưng tổng quát hóa từ Grid sang Graph bất kỳ — mỗi lần gặp một đỉnh chưa thăm, đó là điểm khởi đầu của một thành phần liên thông mới.

```cpp
#include <vector>
using namespace std;

void dfsComponent(int curr, const vector<vector<int>>& adjList, vector<bool>& visited) {
    visited[curr] = true;
    for (int neighbor : adjList[curr]) {
        if (!visited[neighbor]) {
            dfsComponent(neighbor, adjList, visited);
        }
    }
}

int countConnectedComponents(int n, const vector<vector<int>>& adjList) {
    vector<bool> visited(n, false);
    int count = 0;

    for (int v = 0; v < n; v++) {
        if (!visited[v]) {
            count++;
            dfsComponent(v, adjList, visited);
        }
    }

    return count;
}
```

**Độ phức tạp:** O(V + E) thời gian, O(V) bộ nhớ phụ. *(So sánh với cách tiếp cận khác cho cùng bài toán này bằng Union-Find, sẽ trình bày ở Chương 21 — mục 21.4.1.)*

---

## 19.4. Cycle Detection (Phát hiện chu trình)

### 19.4.1. Bản chất — khác biệt giữa đồ thị vô hướng và có hướng

Đây là điểm dễ nhầm lẫn quan trọng nhất của DFS trên Graph: cách phát hiện chu trình **khác nhau hoàn toàn** giữa đồ thị vô hướng và có hướng, vì khái niệm "quay lại đỉnh đã thăm" mang ý nghĩa khác nhau trong hai trường hợp.

**Đồ thị vô hướng:** một cạnh vô hướng `(u, v)` khi duyệt từ `u` sẽ luôn "nhìn thấy" `v`, và khi duyệt tiếp từ `v` sẽ lại "nhìn thấy" `u` (cạnh đôi chiều) — đây **không phải** chu trình, chỉ là việc đi ngược lại đúng cạnh vừa dùng để đến. Vì vậy, cần loại trừ trường hợp "quay lại chính đỉnh cha trực tiếp" khi kiểm tra chu trình.

**Đồ thị có hướng:** không có vấn đề "cạnh đôi chiều" như trên (vì cạnh chỉ đi được một chiều), nhưng lại phát sinh vấn đề khác: một đỉnh có thể đã được **thăm xong hoàn toàn** ở một nhánh DFS khác (không còn nằm trên đường đi hiện tại) — gặp lại đỉnh này **không phải** chu trình. Chu trình chỉ thực sự tồn tại khi gặp một đỉnh **đang nằm trên đường đi đệ quy hiện tại** (tức vẫn còn trong Call Stack, chưa `return`). Đây là lý do cần một cấu trúc gọi là **"ba màu" (three-color)**: WHITE (chưa thăm), GRAY (đang nằm trên đường đi đệ quy hiện tại), BLACK (đã thăm xong hoàn toàn, đã rời khỏi đường đi đệ quy).

### 19.4.2. Cycle Detection trên đồ thị vô hướng

```cpp
#include <vector>
using namespace std;

bool hasCycleUndirectedHelper(int curr, int parent, const vector<vector<int>>& adjList,
                                vector<bool>& visited) {
    visited[curr] = true;

    for (int neighbor : adjList[curr]) {
        if (!visited[neighbor]) {
            if (hasCycleUndirectedHelper(neighbor, curr, adjList, visited)) return true;
        } else if (neighbor != parent) {
            // Gặp lại một đỉnh đã thăm mà KHÔNG PHẢI đỉnh cha trực tiếp → chu trình thực sự
            return true;
        }
    }

    return false;
}

bool hasCycleUndirected(int n, const vector<vector<int>>& adjList) {
    vector<bool> visited(n, false);

    for (int v = 0; v < n; v++) {
        if (!visited[v]) {
            if (hasCycleUndirectedHelper(v, -1, adjList, visited)) return true;
        }
    }

    return false;
}
```

### 19.4.3. Cycle Detection trên đồ thị có hướng (Ba màu / Recursion Stack)

```cpp
#include <vector>
using namespace std;

enum Color { WHITE, GRAY, BLACK };

bool hasCycleDirectedHelper(int curr, const vector<vector<int>>& adjList,
                              vector<Color>& color) {
    color[curr] = GRAY; // đánh dấu đang nằm trên đường đi đệ quy hiện tại

    for (int neighbor : adjList[curr]) {
        if (color[neighbor] == GRAY) {
            return true; // gặp đỉnh ĐANG nằm trên đường đi hiện tại → chu trình thực sự
        }
        if (color[neighbor] == WHITE && hasCycleDirectedHelper(neighbor, adjList, color)) {
            return true;
        }
        // color[neighbor] == BLACK: đã thăm xong ở nhánh khác, KHÔNG phải chu trình, bỏ qua
    }

    color[curr] = BLACK; // hoàn tất khám phá đỉnh này, rời khỏi đường đi đệ quy hiện tại
    return false;
}

bool hasCycleDirected(int n, const vector<vector<int>>& adjList) {
    vector<Color> color(n, WHITE);

    for (int v = 0; v < n; v++) {
        if (color[v] == WHITE) {
            if (hasCycleDirectedHelper(v, adjList, color)) return true;
        }
    }

    return false;
}
```

**Độ phức tạp (cả hai trường hợp):** O(V + E) thời gian, O(V) bộ nhớ phụ. Kỹ thuật ba màu này chính là nền tảng trực tiếp cho việc phát hiện chu trình trong DAG — vấn đề cốt lõi của **Topological Sort** sẽ trình bày ở Chương 20.

---

## 19.5. So sánh BFS và DFS

| Tiêu chí | BFS | DFS |
|---|---|---|
| Cấu trúc nền | Queue (FIFO) | Stack / Call Stack (LIFO) |
| Thứ tự khám phá | Theo từng lớp khoảng cách | Đi sâu triệt để từng nhánh |
| Tìm đường đi ngắn nhất (không trọng số) | Có, tự nhiên | Không trực tiếp (cần thêm xử lý) |
| Bộ nhớ trong trường hợp xấu nhất | O(bề rộng lớn nhất của đồ thị) | O(chiều sâu lớn nhất của đồ thị) |
| Phát hiện chu trình | Có thể (qua Topological Sort — Kahn's, Chương 20) | Trực tiếp và tự nhiên hơn (mục 19.4) |
| Liệt kê mọi đường đi/tổ hợp | Kém tự nhiên | Tự nhiên (nền tảng của Backtracking, Chương 13) |

---

## 19.6. Danh sách bài tập luyện tập

### Mức Easy
1. Flood Fill (thử giải lại bằng DFS, so sánh với bản BFS ở mục 18.6)
2. Find if Path Exists in Graph (thử giải lại bằng DFS)

### Mức Medium
3. Number of Islands (thử giải bằng cả DFS và BFS, so sánh)
4. Number of Provinces (đếm connected components)
5. Course Schedule (phát hiện chu trình trong đồ thị có hướng — chi tiết đầy đủ ở Chương 20)
6. Pacific Atlantic Water Flow
7. Surrounded Regions
8. Graph Valid Tree (kết hợp kiểm tra liên thông + không chu trình)

### Mức Hard
9. Number of Distinct Islands (kết hợp DFS + chuẩn hóa hình dạng để so sánh)
10. Critical Connections in a Network (Bridges — mở rộng nâng cao của DFS với khái niệm discovery time/low-link)

---

*Chương tiếp theo: **Chương 20 — Topological Sort**, ứng dụng trực tiếp kỹ thuật phát hiện chu trình vừa học để sắp xếp các đỉnh của DAG theo đúng thứ tự phụ thuộc.*
