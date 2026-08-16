# Chương 21: Union-Find / Disjoint Set Union (DSU)

## 21.1. Khái niệm cốt lõi

### 21.1.1. Định nghĩa

Union-Find (còn gọi là Disjoint Set Union — DSU) là cấu trúc dữ liệu quản lý một tập hợp các phần tử được chia thành nhiều **tập con rời nhau (disjoint sets)**, hỗ trợ hiệu quả hai thao tác: `find(x)` — xác định phần tử `x` thuộc tập con nào (thông qua một "đại diện" — representative — của tập đó), và `union(x, y)` — gộp hai tập con chứa `x` và `y` thành một tập duy nhất.

### 21.1.2. Bản chất — vì sao cần DSU thay vì chỉ dùng BFS/DFS

Bài toán "kiểm tra hai đỉnh có thuộc cùng thành phần liên thông hay không" đã có thể giải bằng DFS/BFS (mục 19.3.2): chạy một lượt duyệt, gán nhãn thành phần liên thông cho mọi đỉnh, sau đó so sánh nhãn. Tuy nhiên, cách này giả định đồ thị **cố định** — không hiệu quả khi các cạnh được **thêm vào dần dần theo thời gian** và cần trả lời liên tục câu hỏi liên thông sau mỗi lần thêm cạnh (bài toán liên thông động — dynamic connectivity). Chạy lại BFS/DFS từ đầu sau mỗi lần thêm cạnh sẽ tốn O(V + E) mỗi lần — rất lãng phí nếu có nhiều thao tác xen kẽ. DSU giải quyết đúng lớp bài toán này: `union` một cạnh mới và `find` kiểm tra liên thông đều đạt độ phức tạp gần O(1) (chính xác hơn ở mục 21.1.5).

### 21.1.3. Biểu diễn — mảng Parent (Rừng ngầm)

**Bản chất:** DSU biểu diễn mỗi tập con bằng một **cây ngầm (implicit tree)**, lưu trữ chỉ bằng một mảng `parent`, trong đó `parent[x]` là "cha" của `x` trong cây ngầm đó. Đỉnh **gốc** của mỗi cây (đỉnh có `parent[x] == x`) chính là **đại diện (representative)** cho toàn bộ tập con — hai phần tử thuộc cùng tập con khi và chỉ khi chúng có cùng đại diện gốc.

**Khởi tạo:** ban đầu mỗi phần tử tự làm đại diện cho chính nó — có `n` tập con rời rạc, mỗi tập chỉ chứa một phần tử.

```
Khởi tạo:  parent[i] = i với mọi i    (n cây, mỗi cây chỉ có 1 node — chính nó)
```

### 21.1.4. Find với Path Compression (Nén đường dẫn)

**Bản chất phép `find` cơ bản:** đi ngược từ `x` theo chuỗi `parent` cho đến khi gặp gốc (`parent[x] == x`). Nếu không tối ưu, cây ngầm có thể trở nên rất "cao" sau nhiều lần `union` liên tiếp theo một chuỗi, khiến `find` suy biến về O(n) — tương tự vấn đề BST suy biến đã gặp ở mục 15.1.2.

**Path Compression:** trong lúc thực hiện `find(x)`, sau khi đã xác định được gốc thực sự, ta **gắn trực tiếp** `x` (và mọi node trên đường đi vừa duyệt qua) về thẳng gốc đó, thay vì giữ nguyên cấu trúc cây cũ. Lần `find` tiếp theo trên các node này chỉ tốn O(1) vì chúng đã trỏ thẳng đến gốc — "làm phẳng" dần cấu trúc cây qua mỗi lần gọi.

**Minh họa Path Compression:**

```
Trước find(4):        Sau find(4) với Path Compression:

1                          1
|                        / | \
2                       2  3  4
|
3
|
4

(chuỗi dài 1-2-3-4)    (mọi node trên đường đi 4→3→2→1
                        được gắn thẳng về gốc 1)
```

### 21.1.5. Union by Rank/Size

**Bản chất:** khi gộp hai tập con, nếu luôn gắn gốc của tập này làm con của gốc tập kia một cách tùy tiện, cây ngầm có thể trở nên mất cân bằng, làm giảm hiệu quả của các lần `find` sau này (dù đã có Path Compression). **Union by Rank** (gắn theo độ cao ước lượng của cây) hoặc **Union by Size** (gắn theo số phần tử của tập) đảm bảo luôn gắn **cây nhỏ hơn vào cây lớn hơn**, giữ chiều cao cây ngầm tăng chậm nhất có thể — tương tự tư tưởng cân bằng đã gặp ở BST cân bằng (mục 15.1.2).

**Độ phức tạp khi kết hợp cả hai kỹ thuật:** khi áp dụng đồng thời Path Compression và Union by Rank/Size, độ phức tạp trung bình mỗi thao tác `find`/`union` đạt **O(α(n))**, với `α` là hàm nghịch đảo Ackermann — tăng trưởng cực kỳ chậm (với mọi giá trị `n` thực tế có thể tồn tại trong vũ trụ quan sát được, `α(n) ≤ 4`). Trong thực hành, độ phức tạp này được coi là **gần như O(1)** — đây là lý do DSU được xem là một trong những cấu trúc dữ liệu hiệu quả nhất cho bài toán liên thông động.

---

## 21.2. Cài đặt DSU hoàn chỉnh

```cpp
#include <vector>
using namespace std;

class DSU {
private:
    vector<int> parent;
    vector<int> rank_; // ước lượng độ cao của cây (không dùng tên "rank" để tránh trùng std::rank)

public:
    explicit DSU(int n) : parent(n), rank_(n, 0) {
        for (int i = 0; i < n; i++) parent[i] = i; // mỗi phần tử ban đầu tự làm đại diện
    }

    int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]); // Path Compression: gắn thẳng x về gốc thực sự
        }
        return parent[x];
    }

    bool unite(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);

        if (rootX == rootY) return false; // đã cùng tập con, không cần gộp (tránh tạo chu trình)

        // Union by Rank: gắn cây có rank thấp hơn vào cây có rank cao hơn
        if (rank_[rootX] < rank_[rootY]) {
            parent[rootX] = rootY;
        } else if (rank_[rootX] > rank_[rootY]) {
            parent[rootY] = rootX;
        } else {
            parent[rootY] = rootX;
            rank_[rootX]++; // hai cây cùng rank, cây gộp lại tăng rank thêm 1
        }

        return true;
    }

    bool connected(int x, int y) {
        return find(x) == find(y);
    }
};
```

**Giải thích giá trị trả về của `unite`:** trả về `true` nếu hai phần tử **trước đó** thuộc hai tập khác nhau (và vừa được gộp lại) — giá trị này đặc biệt hữu ích cho bài toán phát hiện chu trình (mục 21.4.2) và thuật toán Kruskal's MST (mục 21.4.3): nếu `unite` trả về `false`, nghĩa là hai đỉnh của cạnh đang xét **đã liên thông từ trước**, thêm cạnh này vào chắc chắn sẽ tạo ra chu trình.

**Độ phức tạp:** O(α(n)) amortized cho mỗi lần gọi `find`/`unite`/`connected` — gần như hằng số trong thực hành, như đã phân tích ở mục 21.1.5.

---

## 21.3. So sánh DSU với BFS/DFS cho bài toán liên thông

| Tiêu chí | BFS/DFS | DSU |
|---|---|---|
| Kiểm tra liên thông trên đồ thị tĩnh | O(V + E) một lần duyệt | O(α(n)) mỗi truy vấn sau khi xây dựng |
| Đồ thị có cạnh thêm dần theo thời gian (dynamic) | Phải chạy lại toàn bộ O(V + E) mỗi lần | O(α(n)) mỗi lần thêm cạnh — vượt trội |
| Hỗ trợ xóa cạnh | Tự nhiên (chạy lại DFS/BFS) | Không hỗ trợ hiệu quả (DSU không thiết kế cho việc "tách" tập hợp) |
| Đếm số thành phần liên thông | O(V + E) | O(V) khởi tạo + O(α(n)) mỗi lần `unite` thành công |
| Độ phức tạp cài đặt | Đơn giản, quen thuộc | Cần hiểu bản chất Path Compression + Union by Rank |

**Khi nào dùng DSU thay vì BFS/DFS:** khi bài toán có các cạnh được **thêm dần** và cần trả lời truy vấn liên thông xen kẽ (không thể gộp lại xử lý một lần), khi cần đếm số thành phần liên thông **giảm dần** theo từng lần thêm cạnh, hoặc khi cài đặt thuật toán Kruskal's cho bài toán cây khung nhỏ nhất (Minimum Spanning Tree).

---

## 21.4. Cài đặt các bài toán kinh điển

### 21.4.1. Number of Connected Components in an Undirected Graph

**Bài toán:** đếm số thành phần liên thông trong đồ thị vô hướng — giải lại bằng DSU, so sánh với cách DFS ở mục 19.3.2.

**Cài đặt C++:**

```cpp
int countComponentsDSU(int n, vector<vector<int>>& edges) {
    DSU dsu(n);
    int components = n; // ban đầu mỗi đỉnh là một thành phần riêng biệt

    for (auto& edge : edges) {
        if (dsu.unite(edge[0], edge[1])) {
            components--; // mỗi lần gộp thành công, số thành phần giảm đi 1
        }
    }

    return components;
}
```

**Độ phức tạp:** O(E · α(n)) thời gian, O(V) bộ nhớ phụ — so với O(V + E) của DFS/BFS (mục 19.3.2), độ phức tạp lý thuyết tương đương, nhưng DSU thể hiện ưu thế rõ rệt hơn khi bài toán yêu cầu xử lý cạnh **tuần tự theo thời gian** (xem mục 21.4.2).

### 21.4.2. Redundant Connection — phát hiện cạnh gây chu trình

**Bài toán:** cho một đồ thị vô hướng ban đầu là cây (n đỉnh, n-1 cạnh) và được thêm **đúng một cạnh dư thừa** gây ra chu trình, tìm cạnh dư thừa đó.

**Bản chất:** đây là ứng dụng trực tiếp giá trị trả về của `unite` (mục 21.2) — duyệt qua các cạnh theo đúng thứ tự đưa vào, cạnh đầu tiên khiến `unite` trả về `false` (hai đỉnh của nó đã liên thông từ trước) chính là cạnh dư thừa cần tìm, vì nó là cạnh **đầu tiên** tạo ra chu trình.

**Cài đặt C++:**

```cpp
vector<int> findRedundantConnection(vector<vector<int>>& edges) {
    int n = edges.size();
    DSU dsu(n + 1); // đỉnh đánh số từ 1 đến n theo đề bài gốc

    for (auto& edge : edges) {
        if (!dsu.unite(edge[0], edge[1])) {
            return edge; // hai đỉnh đã liên thông trước đó — đây chính là cạnh dư thừa
        }
    }

    return {}; // không xảy ra nếu đề bài đảm bảo luôn có đúng một cạnh dư thừa
}
```

**Độ phức tạp:** O(n · α(n)) thời gian, O(n) bộ nhớ phụ — so với cách tiếp cận bằng DFS (mỗi lần thêm cạnh phải kiểm tra chu trình bằng một lượt DFS riêng, tốn O(n) mỗi lần, tổng O(n²)), DSU vượt trội rõ rệt nhờ bản chất xử lý cạnh tuần tự với chi phí gần O(1) mỗi bước.

### 21.4.3. Kruskal's Algorithm — ứng dụng nâng cao (giới thiệu bản chất)

**Bối cảnh:** Minimum Spanning Tree (MST — cây khung nhỏ nhất) là bài toán tìm tập hợp cạnh nhỏ nhất có trọng số nhỏ nhất sao cho mọi đỉnh của đồ thị đều liên thông (không nhất thiết là trọng tâm chính của tài liệu ôn tập live coding interview thông thường, nhưng đáng giới thiệu bản chất vì thể hiện ứng dụng kinh điển của DSU).

**Bản chất:** Kruskal's Algorithm là thuật toán Greedy (sẽ trình bày chi tiết ở Chương 24) kết hợp trực tiếp với DSU: sắp xếp toàn bộ cạnh theo trọng số tăng dần (áp dụng Edge List, mục 17.2.3), sau đó duyệt qua từng cạnh theo thứ tự đó — chỉ **thêm** cạnh vào MST nếu hai đỉnh của nó **chưa liên thông** (kiểm tra và cập nhật bằng đúng thao tác `unite` đã cài đặt ở mục 21.2), đảm bảo không bao giờ tạo ra chu trình dư thừa trong khi vẫn ưu tiên các cạnh nhẹ nhất trước.

```cpp
#include <vector>
#include <algorithm>
using namespace std;

struct Edge {
    int u, v, weight;
};

int kruskalMST(int n, vector<Edge>& edges) {
    sort(edges.begin(), edges.end(), [](const Edge& a, const Edge& b) {
        return a.weight < b.weight; // Greedy: luôn xét cạnh nhẹ nhất trước
    });

    DSU dsu(n);
    int totalWeight = 0;

    for (auto& e : edges) {
        if (dsu.unite(e.u, e.v)) { // chỉ thêm nếu không tạo chu trình
            totalWeight += e.weight;
        }
    }

    return totalWeight;
}
```

**Độ phức tạp:** O(E log E) thời gian (chi phí sắp xếp cạnh chiếm ưu thế), O(V + E) bộ nhớ phụ.

---

## 21.5. Bảng tổng hợp

| Thao tác | Độ phức tạp (đã tối ưu Path Compression + Union by Rank) |
|---|---|
| Khởi tạo DSU với n phần tử | O(n) |
| `find(x)` | O(α(n)) amortized ≈ O(1) |
| `unite(x, y)` | O(α(n)) amortized ≈ O(1) |
| `connected(x, y)` | O(α(n)) amortized ≈ O(1) |

---

## 21.6. Khi nào dùng Union-Find

- Bài toán liên thông động — cạnh được thêm dần theo thời gian, cần truy vấn liên thông xen kẽ.
- Đếm/theo dõi số thành phần liên thông khi các cạnh lần lượt được thêm vào.
- Phát hiện chu trình khi thêm cạnh tuần tự (Redundant Connection).
- Cài đặt Kruskal's Algorithm cho bài toán Minimum Spanning Tree.
- Nhận diện qua đặc điểm bài toán: "nhóm", "liên thông", "kết nối dần", "cùng thuộc một tập hợp" xuất hiện qua nhiều bước xử lý tuần tự.

---

## 21.7. Danh sách bài tập luyện tập

### Mức Medium
1. Number of Connected Components in an Undirected Graph
2. Redundant Connection
3. Accounts Merge (DSU kết hợp HashMap, xem lại Chương 3)
4. Number of Provinces (thử giải lại bằng DSU, so sánh với DFS ở mục 19.6)
5. Graph Valid Tree (thử giải lại bằng DSU, so sánh với DFS ở mục 19.6)
6. Satisfiability of Equality Equations

### Mức Hard
7. Redundant Connection II (biến thể trên đồ thị có hướng, phức tạp hơn đáng kể)
8. Number of Islands II (liên thông động trên lưới — thêm đất dần theo thời gian)
9. Swim in Rising Water (kết hợp DSU + Binary Search on Answer, xem lại Chương 10)

---

*Chương tiếp theo: **Chương 22 — Dijkstra (Shortest Path)**, mở rộng bài toán đường đi ngắn nhất đã giải quyết bằng BFS (Chương 18) sang đồ thị có trọng số dương, kết hợp trực tiếp cấu trúc Heap đã học ở Chương 9.*
