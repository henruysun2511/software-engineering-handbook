# Chương 18: BFS (Breadth-First Search — Tìm kiếm theo chiều rộng)

## 18.1. Khái niệm cốt lõi

### 18.1.1. Định nghĩa

BFS là thuật toán duyệt đồ thị theo **từng lớp khoảng cách tăng dần** tính từ đỉnh xuất phát: mọi đỉnh cách đỉnh xuất phát đúng `k` bước phải được thăm **trước** mọi đỉnh cách `k+1` bước. Đây chính là Level Order Traversal (mục 14.2.2) tổng quát hóa từ Tree sang Graph.

### 18.1.2. Bản chất — vì sao dùng Queue

Tính chất "lớp gần hơn phải xử lý trước lớp xa hơn" chính xác là nguyên tắc **FIFO** (First In, First Out) đã trình bày ở Chương 8: đỉnh được phát hiện trước (thuộc lớp gần hơn) phải được xử lý và mở rộng trước đỉnh phát hiện sau (thuộc lớp xa hơn). Nếu dùng Stack (LIFO) thay vì Queue, thuật toán sẽ ưu tiên đỉnh **mới phát hiện gần nhất**, đi sâu theo một nhánh trước khi quay lại các nhánh khác — đó chính là bản chất của DFS (Chương 19), không phải BFS.

**Minh họa BFS** trên đồ thị vô hướng, xuất phát từ đỉnh A:

```
      A
    / | \
   B  C  D
   |     |
   E     F

Bước 1: thăm A (lớp 0), enqueue B, C, D
Bước 2: dequeue B (lớp 1), thăm B, enqueue E
Bước 3: dequeue C (lớp 1), thăm C (không có đỉnh kề mới)
Bước 4: dequeue D (lớp 1), thăm D, enqueue F
Bước 5: dequeue E (lớp 2), thăm E
Bước 6: dequeue F (lớp 2), thăm F

Thứ tự thăm: A, B, C, D, E, F   ← đúng theo từng lớp khoảng cách
```

### 18.1.3. Vai trò của tập hợp Visited

**Bản chất:** khác với Tree (mỗi đỉnh chỉ có đúng một đường đi từ root, không cần lo trùng lặp), Graph tổng quát cho phép **nhiều đường đi** dẫn đến cùng một đỉnh, và có thể chứa **chu trình**. Nếu không đánh dấu đỉnh đã thăm, thuật toán có thể quay lại thăm cùng một đỉnh nhiều lần — thậm chí lặp vô hạn nếu đồ thị có chu trình. Tập hợp `visited` (thường dùng HashSet — Chương 3, hoặc mảng boolean nếu đỉnh được đánh số liên tục) đảm bảo mỗi đỉnh chỉ được enqueue **đúng một lần**, giữ độ phức tạp tuyến tính theo số đỉnh và cạnh.

**Điểm cần đặc biệt lưu ý trong cài đặt:** đỉnh phải được đánh dấu `visited` **ngay tại thời điểm enqueue**, không phải đợi đến khi dequeue mới đánh dấu — nếu đánh dấu muộn, cùng một đỉnh có thể bị enqueue nhiều lần bởi các đỉnh lân cận khác nhau trước khi nó được dequeue và đánh dấu, gây lãng phí và trong một số bài toán có thể dẫn đến kết quả sai.

---

## 18.2. Cài đặt BFS cơ bản

### 18.2.1. BFS trên đồ thị (Adjacency List)

```cpp
#include <vector>
#include <queue>
using namespace std;

vector<int> bfs(int start, const vector<vector<int>>& adjList) {
    int n = adjList.size();
    vector<bool> visited(n, false);
    vector<int> order; // thứ tự thăm các đỉnh

    queue<int> q;
    q.push(start);
    visited[start] = true; // đánh dấu NGAY khi enqueue

    while (!q.empty()) {
        int curr = q.front();
        q.pop();
        order.push_back(curr);

        for (int neighbor : adjList[curr]) {
            if (!visited[neighbor]) {
                visited[neighbor] = true; // đánh dấu ngay, tránh enqueue trùng lặp
                q.push(neighbor);
            }
        }
    }

    return order;
}
```

**Độ phức tạp:** O(V + E) thời gian — mỗi đỉnh được enqueue/dequeue đúng một lần (O(V)), mỗi cạnh được xét đúng một lần khi duyệt danh sách kề của đỉnh hai đầu (O(E)); O(V) bộ nhớ phụ cho `visited` và Queue.

### 18.2.2. Shortest Path trong đồ thị không trọng số

**Bản chất:** vì BFS thăm đỉnh theo đúng thứ tự khoảng cách tăng dần, **lần đầu tiên** một đỉnh được thăm chính là khi đạt được khoảng cách **ngắn nhất** từ đỉnh xuất phát đến nó — không cần thuật toán phức tạp hơn cho đồ thị không trọng số (đây là trường hợp đặc biệt sẽ được so sánh với Dijkstra ở Chương 22).

```cpp
#include <vector>
#include <queue>
using namespace std;

vector<int> shortestPathUnweighted(int start, const vector<vector<int>>& adjList) {
    int n = adjList.size();
    vector<int> distance(n, -1); // -1 nghĩa là chưa thể tới được

    queue<int> q;
    q.push(start);
    distance[start] = 0;

    while (!q.empty()) {
        int curr = q.front();
        q.pop();

        for (int neighbor : adjList[curr]) {
            if (distance[neighbor] == -1) { // chưa thăm
                distance[neighbor] = distance[curr] + 1;
                q.push(neighbor);
            }
        }
    }

    return distance;
}
```

---

## 18.3. BFS trên Grid (Lưới ô vuông)

### 18.3.1. Bản chất

Áp dụng trực tiếp khuôn mẫu đỉnh-kề-tính-trực-tiếp đã giới thiệu ở mục 17.4.2: thay vì Adjacency List tường minh, đỉnh kề của một ô `(row, col)` được tính bằng cách cộng lần lượt các vector hướng di chuyển. Cần kiểm tra biên lưới và điều kiện hợp lệ (ví dụ ô không phải chướng ngại vật) trước khi coi một ô lân cận là đỉnh kề hợp lệ.

### 18.3.2. Number of Islands

**Bài toán:** cho lưới nhị phân gồm `'1'` (đất) và `'0'` (nước), đếm số lượng "đảo" (nhóm ô đất liên thông theo 4 hướng).

**Bản chất:** mỗi lần gặp một ô đất **chưa được thăm**, đó chính là điểm khởi đầu của một đảo mới — chạy BFS (hoặc DFS, xem Chương 19) từ ô đó để đánh dấu **toàn bộ đảo** này là đã thăm, đảm bảo không đếm trùng.

```cpp
#include <vector>
#include <queue>
using namespace std;

int numIslands(vector<vector<char>>& grid) {
    int rows = grid.size(), cols = grid[0].size();
    vector<vector<bool>> visited(rows, vector<bool>(cols, false));
    const vector<pair<int,int>> directions = {{-1,0},{1,0},{0,-1},{0,1}};
    int islandCount = 0;

    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            if (grid[r][c] == '1' && !visited[r][c]) {
                islandCount++; // phát hiện đảo mới, BFS để đánh dấu toàn bộ đảo này

                queue<pair<int,int>> q;
                q.push({r, c});
                visited[r][c] = true;

                while (!q.empty()) {
                    auto [row, col] = q.front();
                    q.pop();

                    for (auto& [dr, dc] : directions) {
                        int nr = row + dr, nc = col + dc;
                        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols &&
                            grid[nr][nc] == '1' && !visited[nr][nc]) {
                            visited[nr][nc] = true;
                            q.push({nr, nc});
                        }
                    }
                }
            }
        }
    }

    return islandCount;
}
```

**Độ phức tạp:** O(rows · cols) thời gian — mỗi ô được thăm đúng một lần trên toàn bộ thuật toán; O(rows · cols) bộ nhớ phụ trong trường hợp xấu nhất (toàn bộ lưới là đất, Queue chứa gần hết số ô).

### 18.3.3. Multi-source BFS — Rotting Oranges

**Bài toán:** lưới chứa `0` (ô trống), `1` (cam tươi), `2` (cam thối). Mỗi phút, cam thối làm thối các cam tươi liền kề (4 hướng). Tìm số phút tối thiểu để mọi cam đều thối, hoặc -1 nếu không thể.

**Bản chất Multi-source BFS:** thay vì chạy BFS riêng lẻ từ từng cam thối (tốn kém và không phản ánh đúng việc chúng lan tỏa **đồng thời**), ta khởi tạo Queue với **toàn bộ** các cam thối ban đầu cùng một lúc trước khi bắt đầu vòng lặp BFS. Vì mọi nguồn lan tỏa cùng tốc độ, BFS từ nhiều nguồn đồng thời tự nhiên mô phỏng đúng quá trình lan tỏa song song theo từng phút — số "lớp" BFS chính là số phút cần thiết.

```cpp
#include <vector>
#include <queue>
using namespace std;

int orangesRotting(vector<vector<int>>& grid) {
    int rows = grid.size(), cols = grid[0].size();
    queue<pair<int,int>> q;
    int freshCount = 0;
    const vector<pair<int,int>> directions = {{-1,0},{1,0},{0,-1},{0,1}};

    // Khởi tạo Queue với TOÀN BỘ cam thối ban đầu — đây chính là Multi-source BFS
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            if (grid[r][c] == 2) q.push({r, c});
            else if (grid[r][c] == 1) freshCount++;
        }
    }

    if (freshCount == 0) return 0; // không có cam tươi nào cần làm thối

    int minutes = 0;

    while (!q.empty() && freshCount > 0) {
        int levelSize = q.size(); // tương tự kỹ thuật Level Order (mục 14.3.3)
        minutes++;

        for (int i = 0; i < levelSize; i++) {
            auto [row, col] = q.front();
            q.pop();

            for (auto& [dr, dc] : directions) {
                int nr = row + dr, nc = col + dc;
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == 1) {
                    grid[nr][nc] = 2;
                    freshCount--;
                    q.push({nr, nc});
                }
            }
        }
    }

    return freshCount == 0 ? minutes : -1; // còn cam tươi không thể tới được
}
```

**Độ phức tạp:** O(rows · cols) thời gian, O(rows · cols) bộ nhớ phụ — tận dụng kỹ thuật `levelSize` giống hệt Level Order Traversal (mục 14.3.3) để tính chính xác số phút (số lớp BFS) đã trôi qua.

### 18.3.4. Word Ladder

**Bài toán:** cho `beginWord`, `endWord`, và một danh sách từ điển `wordList`, tìm độ dài đường biến đổi ngắn nhất từ `beginWord` đến `endWord`, mỗi bước chỉ đổi đúng một ký tự và từ mới phải nằm trong từ điển.

**Bản chất:** đây là bài toán tìm đường đi ngắn nhất trên một đồ thị **ẩn** — mỗi từ trong từ điển là một đỉnh, và tồn tại cạnh giữa hai từ nếu chúng chỉ khác nhau đúng một ký tự. Không cần xây dựng Adjacency List tường minh (tốn kém nếu từ điển lớn); thay vào đó, tại mỗi từ, tạo mọi biến thể thay đổi một ký tự tại một vị trí và kiểm tra biến thể đó có trong từ điển hay không.

```cpp
#include <string>
#include <vector>
#include <unordered_set>
#include <queue>
using namespace std;

int ladderLength(const string& beginWord, const string& endWord,
                  const vector<string>& wordList) {
    unordered_set<string> dict(wordList.begin(), wordList.end());
    if (dict.find(endWord) == dict.end()) return 0;

    queue<string> q;
    q.push(beginWord);
    unordered_set<string> visited = {beginWord};
    int steps = 1;

    while (!q.empty()) {
        int levelSize = q.size();

        for (int i = 0; i < levelSize; i++) {
            string curr = q.front();
            q.pop();

            if (curr == endWord) return steps;

            // Sinh mọi biến thể đổi một ký tự tại mỗi vị trí — đây là "đỉnh kề" ẩn
            for (int pos = 0; pos < (int)curr.size(); pos++) {
                char original = curr[pos];
                for (char c = 'a'; c <= 'z'; c++) {
                    if (c == original) continue;
                    curr[pos] = c;

                    if (dict.count(curr) && !visited.count(curr)) {
                        visited.insert(curr);
                        q.push(curr);
                    }
                }
                curr[pos] = original; // khôi phục để thử vị trí tiếp theo
            }
        }

        steps++;
    }

    return 0; // không tìm được đường biến đổi
}
```

**Độ phức tạp:** O(N · L² · 26) thời gian với `N` là số từ trong từ điển, `L` là độ dài mỗi từ (sinh biến thể tốn O(L·26), tạo chuỗi mới mỗi lần tốn thêm O(L)); O(N · L) bộ nhớ phụ.

---

## 18.4. Bảng tổng hợp

| Bài toán | Đặc điểm | Độ phức tạp |
|---|---|---|
| BFS Traversal cơ bản | Một nguồn xuất phát | O(V + E) |
| Shortest Path (không trọng số) | Lần đầu thăm = khoảng cách ngắn nhất | O(V + E) |
| Number of Islands | BFS lặp lại cho từng thành phần liên thông | O(rows · cols) |
| Rotting Oranges | Multi-source BFS | O(rows · cols) |
| Word Ladder | Đồ thị ẩn, sinh đỉnh kề động | O(N · L² · 26) |

---

## 18.5. Khi nào dùng BFS

- Cần tìm đường đi **ngắn nhất** trên đồ thị **không trọng số** (hoặc trọng số đều bằng nhau).
- Bài toán có tính chất "lan tỏa đồng thời từ nhiều nguồn" (Multi-source BFS) — nhận diện qua các từ khóa như "cùng lúc", "đồng thời", "mỗi đơn vị thời gian".
- Cần duyệt theo từng "lớp" hoặc "mức độ" rõ ràng (level-by-level).
- Bài toán trên lưới yêu cầu khoảng cách ngắn nhất giữa hai ô hoặc số bước lan truyền tối thiểu.

**Khi nào KHÔNG dùng BFS mà nên dùng DFS (Chương 19):** khi chỉ cần kiểm tra tồn tại đường đi (không quan tâm độ dài), khi cần liệt kê/khám phá toàn bộ nhánh (ví dụ tìm mọi đường đi khả dĩ, thuộc phạm vi Backtracking Chương 13), hoặc khi bài toán yêu cầu duyệt theo thứ tự ngăn xếp tự nhiên (ví dụ Topological Sort kiểu DFS, Chương 20).

---

## 18.6. Danh sách bài tập luyện tập

### Mức Easy
1. Flood Fill
2. Find if Path Exists in Graph

### Mức Medium
3. Number of Islands
4. Rotting Oranges
5. 01 Matrix (BFS đa nguồn tính khoảng cách ngắn nhất đến ô 0 gần nhất)
6. Shortest Path in Binary Matrix
7. Course Schedule (thử giải lại bằng BFS — chi tiết đầy đủ ở Chương 20 với Kahn's Algorithm)
8. Open the Lock

### Mức Hard
9. Word Ladder
10. Word Ladder II (liệt kê mọi đường biến đổi ngắn nhất — kết hợp BFS + truy vết đường đi)
11. Bus Routes

---

*Chương tiếp theo: **Chương 19 — DFS**, khai thác Recursion (Chương 12) hoặc Stack (Chương 7) để khám phá đồ thị theo chiều sâu, nền tảng cho bài toán phát hiện chu trình và liệt kê thành phần liên thông.*
