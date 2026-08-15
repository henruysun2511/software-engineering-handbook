# 📚 NCCSOFT CONTEST – Tổng hợp lời giải 6 bài

> **Mức độ:** Trung bình – Khó | **Ngôn ngữ:** JavaScript & C++

---

## Mục lục

1. [DEVIL'S GAME – Trò chơi với quỷ dữ](#1-devils-game)
2. [CONNECTION – Kết nối máy tính](#2-connection)
3. [XY – Nhân bản tế bào](#3-xy)
4. [AREA – Diện tích tốt nhất](#4-area)
5. [MANAGING – Quản lý dự án](#5-managing)
6. [FLOWERS – Đường hoa trang trí](#6-flowers)

---

## 1. DEVIL'S GAME

### 📋 Tóm tắt đề bài

Mario và quỷ chơi trò lấy đồng tiền vàng. Có **n đồng tiền**, mỗi lượt người chơi lấy **1, 2 hoặc 3** đồng. **Mario đi trước**. Người lấy đồng tiền **cuối cùng thắng**. Cả hai đều chơi tối ưu. Cho T ván đấu, mỗi ván có n đồng, hỏi Mario thắng hay thua?

- **Input:** T test cases, mỗi test một số nguyên n (1 ≤ n ≤ 10⁹)
- **Output:** `true` nếu Mario thắng, `false` nếu quỷ thắng

### 🧠 Kiến thức liên quan: Game Theory – Nim Game

**Nim Game** là một dạng trò chơi hai người, mỗi người lần lượt lấy một số vật thể theo quy tắc cho trước. Người lấy vật thể cuối cùng thắng (hoặc thua tùy biến thể).

**Khái niệm P-position và N-position:**
- **P-position (Previous player wins):** Người vừa đi xong thắng — tức người đang đến lượt đi **sẽ thua** nếu đối thủ chơi tối ưu.
- **N-position (Next player wins):** Người đang đến lượt đi **sẽ thắng** nếu chơi tối ưu.

**Bản chất:**
- Từ P-position, mọi nước đi đều dẫn đến N-position cho đối thủ.
- Từ N-position, luôn tồn tại ít nhất một nước đi dẫn đến P-position cho đối thủ.

### 💡 Ý tưởng

Mỗi lượt lấy 1, 2 hoặc 3 đồng. Tổng số đồng mỗi "chu kỳ" là 4 (1+3 hoặc 2+2). Nếu số đồng ban đầu là bội của 4, dù Mario lấy bao nhiêu (1, 2, 3), quỷ luôn lấy phần còn lại để về bội của 4 tiếp theo → Mario luôn thua.

### 📊 Minh họa

```
n=4: Mario lấy 1 → còn 3 (quỷ lấy 3, thắng)
     Mario lấy 2 → còn 2 (quỷ lấy 2, thắng)
     Mario lấy 3 → còn 1 (quỷ lấy 1, thắng)
     → Quỷ luôn thắng!

n=5: Mario lấy 1 → còn 4 (P-position cho quỷ → quỷ thua)
     → Mario thắng!

Quy luật: n % 4 == 0 → Quỷ thắng, ngược lại → Mario thắng
```

| n | n%4 | Kết quả |
|---|-----|---------|
| 1 | 1   | Mario thắng |
| 2 | 2   | Mario thắng |
| 3 | 3   | Mario thắng |
| 4 | 0   | Quỷ thắng |
| 5 | 1   | Mario thắng |
| 8 | 0   | Quỷ thắng |

### 💻 Code JavaScript

```javascript
let t = gets();
for (let i = 0; i < t; i++) {
    let n = gets();
    // n % 4 == 0: quỷ thắng (false), ngược lại Mario thắng (true)
    print(n % 4 == 0 ? "false" : "true");
}
```

### 💻 Code C++

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    int t;
    cin >> t;
    while (t--) {
        long long n;
        cin >> n;
        // n % 4 == 0: quỷ thắng
        cout << (n % 4 == 0 ? "false" : "true") << "\n";
    }
    return 0;
}
```

**Độ phức tạp:** O(T) – Cực nhanh, xử lý n ≤ 10⁹ dễ dàng.

---

## 2. CONNECTION

### 📋 Tóm tắt đề bài

Có **N máy tính** (đánh số 0 đến N-1) và **M đường dây cáp**. Hiện tại các máy chưa liên kết hết với nhau. Thiết muốn **tháo một số dây** hiện có rồi **kết nối lại vào cặp máy khác** sao cho tất cả N máy đều liên kết với nhau (trực tiếp hoặc gián tiếp). Tìm **số dây tháo tối thiểu**. Nếu không thể → in -1.

- **Input:** N, M và M cặp (u, v) mô tả các đường dây (N ≤ 5×10⁵, M ≤ 10⁶)
- **Output:** Số dây cần tháo tối thiểu, hoặc -1 nếu không đủ dây

### 🧠 Kiến thức liên quan: Union-Find (Disjoint Set Union)

**Union-Find** là cấu trúc dữ liệu quản lý các tập hợp rời nhau, hỗ trợ 2 thao tác:
- `find(x)`: Tìm đại diện (root) của tập chứa x.
- `union(x, y)`: Gộp tập chứa x và tập chứa y.

**Tối ưu hóa:**
- **Path compression:** Khi tìm root, nén đường đi để lần sau nhanh hơn.
- **Union by rank:** Ghép cây nhỏ vào cây lớn để giảm chiều cao.

**Bản chất:** Mỗi connected component trong đồ thị tương ứng với một tập trong Union-Find. Hai node cùng root → cùng component.

### 💡 Ý tưởng

- Dùng Union-Find để tìm số **connected components** C hiện tại.
- Để nối C components thành 1 cần **C-1 dây mới**.
- Những dây "dư" (nối 2 node cùng component) có thể tháo ra tái sử dụng.
- Số dây dư = số lần `union` trả về false (2 node đã cùng component).
- Nếu dây dư ≥ C-1 → đáp án là C-1, ngược lại → -1 (thiếu dây).

### 📊 Minh họa

```
N=4, M=3: Cạnh (0,1), (0,2), (1,2)

Ban đầu: {0}, {1}, {2}, {3}
union(0,1): {0,1}, {2}, {3}         → thành công
union(0,2): {0,1,2}, {3}            → thành công
union(1,2): đã cùng nhóm!           → DƯ 1 dây

Components C = 2: {0,1,2} và {3}
Cần C-1 = 1 dây mới
Dây dư = 1 >= 1 → Đáp án: 1 ✓
```

```
N=6, M=4: Cạnh (0,1),(0,2),(0,3),(1,2)

Components: {0,1,2,3}, {4}, {5} → C=3
Dây dư = 1 (cạnh (1,2) dư)
Cần C-1 = 2 dây mới
1 < 2 → Đáp án: -1 ✓
```

### 💻 Code JavaScript

```javascript
const inp = gets().split(' ');
const N = parseInt(inp[0]);
const M = parseInt(inp[1]);

// Khởi tạo Union-Find
const parent = Array.from({length: N}, (_, i) => i);
const rank = new Array(N).fill(0);

// Tìm root với path compression
function find(x) {
    if (parent[x] !== x) parent[x] = find(parent[x]); // nén đường đi
    return parent[x];
}

// Gộp 2 tập, trả về false nếu đã cùng tập (dây dư)
function union(x, y) {
    const px = find(x), py = find(y);
    if (px === py) return false;
    // Union by rank: ghép cây nhỏ vào cây lớn
    if (rank[px] < rank[py]) parent[px] = py;
    else if (rank[px] > rank[py]) parent[py] = px;
    else { parent[py] = px; rank[px]++; }
    return true;
}

let extraEdges = 0; // đếm dây dư

for (let i = 0; i < M; i++) {
    const edge = gets().split(' ');
    const u = parseInt(edge[0]);
    const v = parseInt(edge[1]);
    if (!union(u, v)) extraEdges++; // dây này nối 2 node cùng component
}

// Đếm số components (số root phân biệt)
const roots = new Set();
for (let i = 0; i < N; i++) roots.add(find(i));
const C = roots.size;

const need = C - 1; // cần C-1 dây để nối tất cả

if (extraEdges >= need) {
    print(need);
} else {
    print(-1); // không đủ dây để tái sử dụng
}
```

### 💻 Code C++

```cpp
#include <bits/stdc++.h>
using namespace std;

int parent[500005], rnk[500005];

int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);
    return parent[x];
}

bool unite(int x, int y) {
    int px = find(x), py = find(y);
    if (px == py) return false;
    if (rnk[px] < rnk[py]) swap(px, py);
    parent[py] = px;
    if (rnk[px] == rnk[py]) rnk[px]++;
    return true;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    int N, M;
    cin >> N >> M;

    for (int i = 0; i < N; i++) { parent[i] = i; rnk[i] = 0; }

    int extraEdges = 0;
    for (int i = 0; i < M; i++) {
        int u, v;
        cin >> u >> v;
        if (!unite(u, v)) extraEdges++;
    }

    set<int> roots;
    for (int i = 0; i < N; i++) roots.insert(find(i));
    int C = roots.size();
    int need = C - 1;

    cout << (extraEdges >= need ? need : -1) << "\n";
    return 0;
}
```

**Độ phức tạp:** O(M × α(N)) ≈ O(M) – α là hàm Ackermann nghịch, gần như hằng số.

---

## 3. XY – NHÂN BẢN TẾ BÀO

### 📋 Tóm tắt đề bài

Các tế bào được xếp thành hàng ngang, nhân bản theo quy luật:
- Giây 1: chỉ có **X**
- Mỗi giây sau: **X → XY**, **Y → YX** (ghép nối tiếp từ trái sang phải)

```
Giây 1: X
Giây 2: X Y
Giây 3: XY YX
Giây 4: XY YX YX XY
```

Cho T truy vấn, mỗi truy vấn cho **n** (giây) và **k** (vị trí), hỏi tế bào ở vị trí k tại giây n là **X hay Y**?

- **Input:** T test cases, mỗi test cặp (n, k) với 1 ≤ n ≤ 30, 1 ≤ k ≤ 2^(n-1)
- **Output:** `X` hoặc `Y`

### 🧠 Kiến thức liên quan: Đệ quy & Chia đôi

**Tư duy chia đôi (Divide & Conquer):**
Thay vì tạo toàn bộ chuỗi (không thể vì n=30 → 2²⁹ ≈ 500 triệu tế bào), ta **truy ngược** vị trí k về dạng đơn giản hơn qua mỗi bước.

**Bản chất:**
- Giây n = nửa trái (= giây n-1 giữ nguyên) + nửa phải (= giây n-1 đảo X↔Y).
- Để tìm tế bào k ở giây n, ta xác định nó thuộc nửa nào, điều chỉnh k và flip nếu cần.
- Lặp lại đến giây 1 (chỉ có X).

### 📊 Minh họa

```
Giây 1: X
Giây 2: X  Y          (X→XY)
Giây 3: XY YX         (X→XY, Y→YX)
Giây 4: XY YX | YX XY (nửa trái = giây 3, nửa phải = giây 3 đảo)

Tìm n=3, k=4:
  half = 2^(3-2) = 2
  k=4 > 2 → k = 4-2 = 2, flipped = true, n=2
  half = 2^(2-2) = 1
  k=2 > 1 → k = 2-1 = 1, flipped = false, n=1
  Giây 1, vị trí 1 = X, flipped=false → X ✓
```

### 💻 Code JavaScript

```javascript
let t = gets();
for (let i = 0; i < t; i++) {
    const arr = gets().split(' ');
    let n = parseInt(arr[0]);
    let k = parseInt(arr[1]);

    let flipped = false;

    // Truy ngược từ giây n về giây 1
    while (n > 1) {
        const half = BigInt(1) << BigInt(n - 2); // 2^(n-2)
        if (BigInt(k) > half) {
            k = Number(BigInt(k) - half); // k thuộc nửa phải → trừ đi half
            flipped = !flipped;           // nửa phải là đảo của nửa trái
        }
        n--;
    }

    // Giây 1 chỉ có X ở vị trí 1
    // Nếu đã flip số lần lẻ thì X → Y
    print(flipped ? 'Y' : 'X');
}
```

### 💻 Code C++

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    int t;
    cin >> t;
    while (t--) {
        long long n, k;
        cin >> n >> k;

        bool flipped = false;
        while (n > 1) {
            long long half = 1LL << (n - 2); // 2^(n-2)
            if (k > half) {
                k -= half;
                flipped = !flipped;
            }
            n--;
        }

        cout << (flipped ? "Y" : "X") << "\n";
    }
    return 0;
}
```

**Độ phức tạp:** O(T × n) với n ≤ 30 → O(30T) – cực nhanh.

---

## 4. AREA – DIỆN TÍCH TỐT NHẤT

### 📋 Tóm tắt đề bài

Cho mảnh đất **N × M** ô, mỗi ô có giá trị 0 (đất xấu) hoặc 1 (đất tốt). Cần chọn một **hình chữ nhật con** sao cho **tất cả ô trong đó đều là đất tốt (= 1)**. Tìm **diện tích lớn nhất** của hình chữ nhật như vậy.

- **Input:** N, M và ma trận N×M gồm các số 0/1 (1 ≤ N, M ≤ 2000)
- **Output:** Diện tích hình chữ nhật toàn 1 lớn nhất

### 🧠 Kiến thức liên quan: Largest Rectangle in Histogram + Stack

**Largest Rectangle in Histogram** là bài toán tìm hình chữ nhật lớn nhất trong biểu đồ cột.

**Thuật toán Stack:**
- Duyệt từ trái sang phải, duy trì stack chứa các cột theo chiều tăng dần của chiều cao.
- Khi gặp cột thấp hơn top stack, pop stack và tính diện tích với cột đó làm chiều cao.
- Chiều rộng = khoảng cách từ phần tử dưới top đến vị trí hiện tại.

**Bản chất:** Stack giúp tìm nhanh "cột bên trái thấp hơn gần nhất" và "cột bên phải thấp hơn gần nhất" cho mỗi cột — đây chính là biên của hình chữ nhật lớn nhất có chiều cao bằng cột đó.

**Ứng dụng vào ma trận 0/1:**
- Với mỗi hàng i, tính `height[j]` = số ô 1 liên tiếp từ hàng i **lên trên** tại cột j.
- Áp dụng Histogram cho mảng height → tìm hình chữ nhật lớn nhất.
- Lặp qua N hàng, lấy max.

### 📊 Minh họa

```
Ma trận:          height sau mỗi hàng:
1 0 1 0 0         [1,0,1,0,0]  → histogram max = 1
1 0 1 1 1         [2,0,2,1,1]  → histogram max = 3
1 1 1 1 1         [3,1,3,2,2]  → histogram max = 6 ✓
1 0 0 1 0         [4,0,0,3,0]  → histogram max = 6

Hàng 3, height = [3,1,3,2,2]:
  Dùng stack tìm max rectangle:
  i=0: push(0)                stack=[0]
  i=1: h[1]=1 < h[0]=3 → pop 0, width=1, area=3*1=3; push(1) stack=[1]
  i=2: push(2)                stack=[1,2]
  i=3: h[3]=2 < h[2]=3 → pop 2, width=1, area=3*1=3; push(3) stack=[1,3]
  i=4: push(4)                stack=[1,3,4]
  end: pop 4 → h=2, width=1, area=2
       pop 3 → h=2, width=3, area=6 ✓ (từ vị trí 2 đến 4)
       pop 1 → h=1, width=5, area=5
  Max = 6 ✓
```

### 💻 Code JavaScript

```javascript
const nm = gets().split(' ');
const N = parseInt(nm[0]);
const M = parseInt(nm[1]);

const height = new Array(M).fill(0);
let maxArea = 0;

// Tìm hình chữ nhật lớn nhất trong histogram bằng stack
function largestInHistogram(h) {
    const stack = []; // lưu index các cột
    let maxA = 0;

    for (let i = 0; i <= h.length; i++) {
        // Thêm cột cao 0 ở cuối để flush hết stack
        const cur = i === h.length ? 0 : h[i];

        while (stack.length > 0 && cur < h[stack[stack.length - 1]]) {
            const topH = h[stack.pop()]; // chiều cao của hình chữ nhật
            // Chiều rộng: từ phần tử dưới top đến i
            const width = stack.length === 0 ? i : i - stack[stack.length - 1] - 1;
            maxA = Math.max(maxA, topH * width);
        }
        stack.push(i);
    }
    return maxA;
}

for (let i = 0; i < N; i++) {
    const row = gets().split(' ');

    // Cập nhật height: cộng dồn nếu ô = 1, reset về 0 nếu ô = 0
    for (let j = 0; j < M; j++) {
        height[j] = parseInt(row[j]) === 1 ? height[j] + 1 : 0;
    }

    // Tìm max rectangle trong histogram hiện tại
    maxArea = Math.max(maxArea, largestInHistogram(height));
}

print(maxArea);
```

### 💻 Code C++

```cpp
#include <bits/stdc++.h>
using namespace std;

int largestInHistogram(vector<int>& h) {
    stack<int> st;
    int maxA = 0;
    int n = h.size();

    for (int i = 0; i <= n; i++) {
        int cur = (i == n) ? 0 : h[i];
        while (!st.empty() && cur < h[st.top()]) {
            int topH = h[st.top()]; st.pop();
            int width = st.empty() ? i : i - st.top() - 1;
            maxA = max(maxA, topH * width);
        }
        st.push(i);
    }
    return maxA;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    int N, M;
    cin >> N >> M;

    vector<int> height(M, 0);
    int maxArea = 0;

    for (int i = 0; i < N; i++) {
        for (int j = 0; j < M; j++) {
            int x; cin >> x;
            height[j] = x == 1 ? height[j] + 1 : 0;
        }
        maxArea = max(maxArea, largestInHistogram(height));
    }

    cout << maxArea << "\n";
    return 0;
}
```

**Độ phức tạp:** O(N × M) – Mỗi phần tử được push/pop stack tối đa 1 lần.

---

## 5. MANAGING – QUẢN LÝ DỰ ÁN

### 📋 Tóm tắt đề bài

Có **N dự án** theo thứ tự cho trước, dự án thứ i có lợi nhuận **v_i**. Công ty muốn chọn một số dự án để thực hiện nhưng **không được chọn quá K dự án liên tiếp** (theo đúng thứ tự đã cho). Tìm **tổng lợi nhuận lớn nhất** có thể đạt được.

Ví dụ K=2, dãy [8,6,2,5,7]: không được chọn 3 dự án liên tiếp bất kỳ → bỏ 2 → lấy [8,6,_,5,7] = 26.

- **Input:** N, K và N số nguyên v_i (N ≤ 10⁵, v_i ≤ 2×10⁹)
- **Output:** Tổng lợi nhuận tối đa

### 🧠 Kiến thức liên quan: DP + Monotonic Deque (Sliding Window Maximum)

**DP với cửa sổ trượt:**
Bài toán DP dạng `dp[i] = max(f(j))` với j thuộc đoạn `[i-K-1, i-1]` — đây là dạng **Sliding Window Maximum**, giải bằng **Monotonic Deque** (deque đơn điệu).

**Monotonic Deque:**
- Deque luôn giữ các phần tử theo thứ tự **giảm dần** của giá trị.
- Khi thêm phần tử mới: pop hết các phần tử nhỏ hơn ở cuối deque.
- Khi cửa sổ trượt: pop phần tử cũ (ngoài cửa sổ) ở đầu deque.
- Phần tử đầu deque luôn là **max** của cửa sổ hiện tại.

**Bản chất:** Thay vì duyệt K phần tử để tìm max (O(K)), deque cho phép tìm max trong O(1) mỗi bước.

### 💡 Ý tưởng

Không được chọn quá K dự án liên tiếp → phải bỏ ít nhất 1 dự án trong mỗi đoạn K+1 liên tiếp.

Định nghĩa: `dp[i]` = lợi nhuận tốt nhất khi dự án i bị **bỏ qua**.

Công thức: `dp[i] = max(dp[j] + sum(j+1, i-1))` với `max(-1, i-K-1) ≤ j ≤ i-1`

Biến đổi: `dp[i] = max(dp[j] - prefix[j+1]) + prefix[i]`

→ Phần `prefix[i]` cố định, chỉ cần tìm `max(dp[j] - prefix[j+1])` trong cửa sổ → Deque!

### 📊 Minh họa

```
N=5, K=2, v=[8,6,2,5,7]
prefix = [0, 8, 14, 16, 21, 28]

val(j) = dp[j] - prefix[j+1]  (j=-1: val = 0 - 0 = 0)

i=0 (bỏ v[0]=8):
  Cửa sổ j ∈ [-1, -1], deque=[-1]
  dp[0] = val(-1) + prefix[0] = 0 + 0 = 0
  Thêm 0 vào deque: val(0) = dp[0]-prefix[1] = 0-8 = -8
  deque = [-1, 0]

i=1 (bỏ v[1]=6):
  Cửa sổ j ∈ [-1, 0], deque=[-1,0]
  dp[1] = val(-1) + prefix[1] = 0 + 8 = 8
  val(1) = dp[1]-prefix[2] = 8-14 = -6
  deque = [-1, 1] (pop 0 vì val(0)=-8 < val(1)=-6)

i=2 (bỏ v[2]=2):
  Cửa sổ j ∈ [0, 1], pop -1 (ngoài cửa sổ)
  deque = [1], dp[2] = val(1) + prefix[2] = -6+14 = 8... 
  Thực tế dp[2] = max = 8+6 = 14 (bỏ v[2], lấy v[0]+v[1])

Kết quả: bỏ v[2]=2 → tổng = 8+6+5+7 = 26 ✓
```

### 💻 Code JavaScript

```javascript
const line1 = gets().split(' ');
const N = parseInt(line1[0]);
const K = parseInt(line1[1]);

const v = [];
for (let i = 0; i < N; i++) v.push(parseInt(gets()));

// Prefix sum để tính tổng đoạn nhanh
const prefix = new Array(N + 1).fill(0);
for (let i = 0; i < N; i++) prefix[i + 1] = prefix[i] + v[i];

// dp[i] = lợi nhuận tốt nhất khi BỎ dự án i
const dp = new Array(N).fill(-Infinity);
const deque = []; // monotonic deque chứa index j

// val(j) = dp[j] - prefix[j+1]
// j=-1 là sentinel: val(-1) = 0 - prefix[0] = 0
function val(j) {
    return j === -1 ? 0 : dp[j] - prefix[j + 1];
}

deque.push(-1); // sentinel

for (let i = 0; i < N; i++) {
    // Loại phần tử ngoài cửa sổ [i-K-1, i-1]
    while (deque.length > 0 && deque[0] < i - K - 1) deque.shift();

    // dp[i] = max val trong cửa sổ + prefix[i]
    dp[i] = val(deque[0]) + prefix[i];

    // Duy trì deque giảm dần theo val: pop các phần tử nhỏ hơn
    while (deque.length > 0 && val(deque[deque.length - 1]) <= val(i)) deque.pop();
    deque.push(i);
}

// Tính kết quả: bỏ 1 dự án trong K+1 dự án cuối
let ans = 0;
for (let j = Math.max(-1, N - K - 1); j <= N - 1; j++) {
    const prev = j === -1 ? 0 : dp[j];
    const gain = prefix[N] - prefix[j + 1];
    if (prev + gain > ans) ans = prev + gain;
}

print(ans);
```

### 💻 Code C++

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    int N, K;
    cin >> N >> K;

    vector<long long> v(N);
    for (int i = 0; i < N; i++) cin >> v[i];

    vector<long long> prefix(N + 1, 0);
    for (int i = 0; i < N; i++) prefix[i + 1] = prefix[i] + v[i];

    vector<long long> dp(N, LLONG_MIN);
    deque<int> dq;
    dq.push_back(-1); // sentinel

    // val(j): giá trị để tối ưu dp
    auto val = [&](int j) -> long long {
        return j == -1 ? 0LL : dp[j] - prefix[j + 1];
    };

    for (int i = 0; i < N; i++) {
        while (!dq.empty() && dq.front() < i - K - 1) dq.pop_front();
        dp[i] = val(dq.front()) + prefix[i];
        while (!dq.empty() && val(dq.back()) <= val(i)) dq.pop_back();
        dq.push_back(i);
    }

    long long ans = 0;
    for (int j = max(-1, N - K - 1); j <= N - 1; j++) {
        long long prev = (j == -1) ? 0 : dp[j];
        long long gain = prefix[N] - prefix[j + 1];
        ans = max(ans, prev + gain);
    }

    cout << ans << "\n";
    return 0;
}
```

**Độ phức tạp:** O(N) – mỗi phần tử vào/ra deque đúng 1 lần.

---

## 6. FLOWERS – ĐƯỜNG HOA TRANG TRÍ

### 📋 Tóm tắt đề bài

Trồng **N cây hoa** dọc đường, mỗi cây thuộc 1 trong 5 loại: **Hồng, Ly, Mai, Lan, Tulip**. Vị trí đầu tiên tự do chọn. Các vị trí tiếp theo phải tuân quy tắc:

| Hoa | Chỉ được trồng liền sau |
|-----|------------------------|
| Hồng | Ly |
| Ly | Hồng hoặc Mai |
| Mai | **Không** được sau Mai |
| Lan | Mai hoặc Tulip |
| Tulip | Hồng |

Đếm số cách trồng N cây thỏa mãn, kết quả **mod 10⁹+7**.

- **Input:** Một số nguyên N (1 ≤ N ≤ 10¹²)
- **Output:** Số cách trồng mod 10⁹+7

### 🧠 Kiến thức liên quan: Matrix Exponentiation (Nhân ma trận nhanh)

**Bài toán:** Đếm số chuỗi hợp lệ độ dài N theo quy tắc chuyển tiếp giữa các ký tự → **DP tuyến tính**.

**Matrix Exponentiation:**
Khi DP có dạng: `state[n] = A × state[n-1]` (nhân ma trận), thì:
`state[n] = A^(n-1) × state[1]`

Tính `A^n` bằng **binary exponentiation** (bình phương nhanh):
- Nếu n lẻ: `A^n = A × A^(n-1)`
- Nếu n chẵn: `A^n = (A^(n/2))²`

→ Thay vì nhân n lần, chỉ cần O(log n) lần nhân ma trận.

**Bản chất:** Mỗi phép nhân ma trận 5×5 tốn O(5³)=125 phép tính. Với n ≤ 10¹², log₂(10¹²) ≈ 40 → tổng ≈ 5000 phép tính!

### 💡 Ý tưởng

Xây dựng **ma trận chuyển tiếp A** 5×5 từ quy tắc:
- Hồng(0): chỉ trồng sau Ly → L→H: `A[1][0]=1`
- Ly(1): trồng sau Hồng hoặc Mai → H→L, M→L: `A[0][1]=A[2][1]=1`
- Mai(2): không trồng sau Mai → H,L,La,T→M: `A[0][2]=A[1][2]=A[3][2]=A[4][2]=1`
- Lan(3): trồng sau Mai hoặc Tulip → M→La, T→La: `A[2][3]=A[4][3]=1`
- Tulip(4): trồng sau Hồng → H→T: `A[0][4]=1`

Đáp số = tổng tất cả phần tử của `A^(N-1)` (vì vị trí đầu chọn tự do = 5 cách).

### 📊 Minh họa

```
Ma trận A (A[i][j]: hoa i đứng trước hoa j):
       H  L  M  La  T
  H  [ 0  1  1   0  1 ]
  L  [ 1  0  1   0  0 ]
  M  [ 0  1  0   1  0 ]
  La [ 0  0  1   0  0 ]
  T  [ 0  0  1   1  0 ]

N=1: Tổng(A^0) = Tổng(I) = 5 ✓
N=2: Tổng(A^1) = (0+1+1+0+1)+(1+0+1+0+0)+(0+1+0+1+0)+(0+0+1+0+0)+(0+0+1+1+0)
               = 3+2+2+1+2 = 10 ✓
N=5: Tổng(A^4) = 68 ✓
```

### 💻 Code JavaScript

```javascript
const MOD = 1000000007n;

// Nhân 2 ma trận 5x5 với mod
function matMul(A, B) {
    const C = Array.from({length: 5}, () => new Array(5).fill(0n));
    for (let i = 0; i < 5; i++)
        for (let k = 0; k < 5; k++) if (A[i][k] !== 0n)
            for (let j = 0; j < 5; j++)
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
    return C;
}

// Ma trận đơn vị (identity)
function identity() {
    const I = Array.from({length: 5}, () => new Array(5).fill(0n));
    for (let i = 0; i < 5; i++) I[i][i] = 1n;
    return I;
}

// Lũy thừa ma trận: A^p bằng binary exponentiation
function matPow(A, p) {
    let result = identity();
    let base = A;
    while (p > 0n) {
        if (p % 2n === 1n) result = matMul(result, base); // p lẻ
        base = matMul(base, base);                          // bình phương
        p >>= 1n;                                           // p /= 2
    }
    return result;
}

// Ma trận chuyển tiếp: Index 0=Hồng, 1=Ly, 2=Mai, 3=Lan, 4=Tulip
const A = [
    [0n, 1n, 1n, 0n, 1n], // Hồng đứng trước: Ly, Mai, Tulip
    [1n, 0n, 1n, 0n, 0n], // Ly đứng trước: Hồng, Mai
    [0n, 1n, 0n, 1n, 0n], // Mai đứng trước: Ly, Lan
    [0n, 0n, 1n, 0n, 0n], // Lan đứng trước: Mai
    [0n, 0n, 1n, 1n, 0n], // Tulip đứng trước: Mai, Lan
];

let N = BigInt(gets().trim());

if (N === 1n) {
    print(5);
} else {
    const An = matPow(A, N - 1n); // A^(N-1)

    // Tổng tất cả phần tử = số chuỗi hợp lệ độ dài N
    let ans = 0n;
    for (let i = 0; i < 5; i++)
        for (let j = 0; j < 5; j++)
            ans = (ans + An[i][j]) % MOD;

    print(ans.toString());
}
```

### 💻 Code C++

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 1e9 + 7;
const int SZ = 5;

typedef vector<vector<long long>> Matrix;

Matrix matMul(const Matrix& A, const Matrix& B) {
    Matrix C(SZ, vector<long long>(SZ, 0));
    for (int i = 0; i < SZ; i++)
        for (int k = 0; k < SZ; k++) if (A[i][k])
            for (int j = 0; j < SZ; j++)
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
    return C;
}

Matrix identity() {
    Matrix I(SZ, vector<long long>(SZ, 0));
    for (int i = 0; i < SZ; i++) I[i][i] = 1;
    return I;
}

Matrix matPow(Matrix A, long long p) {
    Matrix result = identity();
    while (p > 0) {
        if (p & 1) result = matMul(result, A);
        A = matMul(A, A);
        p >>= 1;
    }
    return result;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    long long N;
    cin >> N;

    // Ma trận chuyển tiếp: 0=Hồng, 1=Ly, 2=Mai, 3=Lan, 4=Tulip
    Matrix A = {
        {0, 1, 1, 0, 1}, // Hồng → Ly, Mai, Tulip
        {1, 0, 1, 0, 0}, // Ly   → Hồng, Mai
        {0, 1, 0, 1, 0}, // Mai  → Ly, Lan
        {0, 0, 1, 0, 0}, // Lan  → Mai
        {0, 0, 1, 1, 0}, // Tulip → Mai, Lan
    };

    if (N == 1) {
        cout << 5 << "\n";
        return 0;
    }

    Matrix An = matPow(A, N - 1);

    long long ans = 0;
    for (int i = 0; i < SZ; i++)
        for (int j = 0; j < SZ; j++)
            ans = (ans + An[i][j]) % MOD;

    cout << ans << "\n";
    return 0;
}
```

**Độ phức tạp:** O(5³ × log N) ≈ O(125 × 40) = O(5000) – xử lý N ≤ 10¹² trong chớp mắt.

---

## 📌 Tổng kết

| Bài | Thuật toán chính | Độ phức tạp | Điểm |
|-----|-----------------|-------------|------|
| DEVIL'S GAME | Game Theory – Nim | O(T) | 10đ |
| CONNECTION | Union-Find (DSU) | O(M·α(N)) | 15đ |
| XY | Chia đôi đệ quy | O(T·n) | 10đ |
| AREA | DP Histogram + Stack | O(N·M) | 15đ |
| MANAGING | DP + Monotonic Deque | O(N) | 25đ |
| FLOWERS | Matrix Exponentiation | O(125·logN) | 25đ |

### 🔑 Kỹ năng cốt lõi cần nắm

1. **Tư duy P/N position** trong game theory
2. **Union-Find** với path compression & union by rank
3. **Chia đôi + truy ngược** cho bài toán fractal/đệ quy
4. **Stack đơn điệu** cho histogram và sliding window
5. **Deque đơn điệu** cho sliding window maximum
6. **Matrix exponentiation** cho DP với N cực lớn
