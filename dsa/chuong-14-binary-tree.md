# Chương 14: Binary Tree (Cây nhị phân)

## 14.1. Khái niệm cốt lõi

### 14.1.1. Định nghĩa

Binary Tree là cấu trúc dữ liệu phi tuyến tính, gồm các **node** liên kết với nhau theo quan hệ phân cấp, trong đó mỗi node có tối đa **hai node con** (con trái và con phải). Node trên cùng gọi là **root** (gốc), node không có con gọi là **leaf** (lá). Đây là cấu trúc tổng quát hơn Linked List (Chương 6, mỗi node chỉ có tối đa một "con" là `next`) — sự phân nhánh này cho phép biểu diễn quan hệ phân cấp và mở ra các thuật toán duyệt phong phú hơn nhiều.

### 14.1.2. Các khái niệm cơ bản

- **Height (chiều cao)** của một node: số cạnh trên đường đi dài nhất từ node đó xuống một lá.
- **Depth (độ sâu)** của một node: số cạnh trên đường đi từ root đến node đó.
- **Subtree (cây con):** một node bất kỳ cùng toàn bộ con cháu của nó tạo thành một cây con độc lập — đây chính là cơ sở cho tính chất đệ quy tự nhiên của Binary Tree: **mỗi cây con cũng là một Binary Tree hợp lệ**, cho phép định nghĩa hầu hết thuật toán trên cây bằng đệ quy (Chương 12).

**Minh họa cấu trúc** với các khái niệm chú thích:

```
              1              ← root, depth 0
            /   \
           2     3           ← depth 1
          / \      \
         4   5      6        ← depth 2, node 4/5/6 là leaf

Chiều cao của node 1 (toàn cây) = 2
Chiều cao của node 2 = 1
```

### 14.1.3. Cài đặt cấu trúc node trong C++

```cpp
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};
```

---

## 14.2. Duyệt cây — bản chất và các dạng

### 14.2.1. Bản chất chung

Duyệt cây (Tree Traversal) là quá trình thăm mọi node của cây theo một thứ tự xác định. Vì mỗi cây con cũng là một Binary Tree (mục 14.1.2), việc duyệt được định nghĩa đệ quy: xử lý node hiện tại và đệ quy vào hai cây con — điểm khác biệt giữa các loại duyệt chỉ nằm ở **thứ tự tương đối** giữa việc xử lý node hiện tại so với việc đệ quy vào con trái/phải.

### 14.2.2. Preorder, Inorder, Postorder — bản chất và minh họa

Với cây minh họa ở mục 14.1.2:

```
              1
            /   \
           2     3
          / \      \
         4   5      6
```

**Preorder (Node → Trái → Phải):** xử lý node hiện tại **trước**, rồi mới đệ quy vào con trái, con phải.
```
Thứ tự thăm: 1, 2, 4, 5, 3, 6
```
**Khi nào dùng:** khi cần xử lý node cha trước con (ví dụ sao chép/serialize cấu trúc cây — cần biết root trước để biết cách dựng lại cây).

**Inorder (Trái → Node → Phải):** đệ quy vào con trái trước, xử lý node hiện tại ở giữa, rồi mới đệ quy vào con phải.
```
Thứ tự thăm: 4, 2, 5, 1, 3, 6
```
**Khi nào dùng:** đây là loại duyệt đặc biệt quan trọng với **Binary Search Tree** (Chương 15) — Inorder Traversal trên BST luôn trả về dãy giá trị đã sắp xếp tăng dần, hệ quả trực tiếp từ định nghĩa thứ tự của BST.

**Postorder (Trái → Phải → Node):** đệ quy vào cả hai con trước, xử lý node hiện tại **sau cùng**.
```
Thứ tự thăm: 4, 5, 2, 6, 3, 1
```
**Khi nào dùng:** khi cần thông tin từ cây con **trước khi** xử lý node cha (ví dụ tính chiều cao cây — cần biết chiều cao hai cây con trước khi tính chiều cao node hiện tại, như đã minh họa ở mục 12.3.4; hoặc khi cần giải phóng bộ nhớ cây — phải giải phóng con trước khi giải phóng cha).

**Level Order (Duyệt theo tầng):** thăm các node theo từng tầng từ trên xuống, trái sang phải trong mỗi tầng — **không** phải dạng đệ quy tự nhiên như ba loại trên, mà dùng Queue (Chương 8) để đạt tính chất FIFO đúng theo tầng, bản chất chính là BFS áp dụng trên cây (sẽ trình bày sâu hơn ở chương Graph).
```
Thứ tự thăm: 1, 2, 3, 4, 5, 6
```

### 14.2.3. So sánh Recursive và Iterative Traversal

| Tiêu chí | Recursive | Iterative (dùng Stack tường minh) |
|---|---|---|
| Độ rõ ràng code | Ngắn gọn, tự nhiên | Dài hơn, cần quản lý Stack thủ công |
| Bộ nhớ phụ | O(h) qua Call Stack ẩn | O(h) qua Stack tường minh — tương đương |
| Kiểm soát luồng thực thi | Hạn chế (khó dừng giữa chừng, khó tạm dừng/tiếp tục) | Linh hoạt hơn (có thể tạm dừng, resume — hữu ích cho Iterator pattern) |

Về bản chất, cả hai cách đều mô phỏng đúng cùng một cơ chế: Recursive tận dụng Call Stack có sẵn của hệ thống (mục 12.1.2), Iterative tự quản lý một Stack tường minh để đạt hiệu quả tương đương — không có khác biệt về độ phức tạp thời gian/không gian giữa hai cách.

---

## 14.3. Cài đặt các bài toán kinh điển

### 14.3.1. Ba dạng duyệt DFS (Recursive)

```cpp
#include <vector>
using namespace std;

void preorder(TreeNode* root, vector<int>& result) {
    if (root == nullptr) return; // Base Case
    result.push_back(root->val);  // xử lý NODE trước
    preorder(root->left, result);
    preorder(root->right, result);
}

void inorder(TreeNode* root, vector<int>& result) {
    if (root == nullptr) return;
    inorder(root->left, result);
    result.push_back(root->val);  // xử lý NODE ở giữa
    inorder(root->right, result);
}

void postorder(TreeNode* root, vector<int>& result) {
    if (root == nullptr) return;
    postorder(root->left, result);
    postorder(root->right, result);
    result.push_back(root->val);  // xử lý NODE sau cùng
}
```

**Độ phức tạp (áp dụng cho cả ba dạng):** O(n) thời gian (mỗi node được thăm đúng một lần), O(h) bộ nhớ phụ với `h` là chiều cao cây.

### 14.3.2. Inorder Traversal — dạng Iterative (minh họa dùng Stack tường minh)

```cpp
#include <vector>
#include <stack>
using namespace std;

vector<int> inorderIterative(TreeNode* root) {
    vector<int> result;
    stack<TreeNode*> st;
    TreeNode* curr = root;

    while (curr != nullptr || !st.empty()) {
        // Đi hết về phía trái, đẩy toàn bộ node trên đường đi vào stack
        while (curr != nullptr) {
            st.push(curr);
            curr = curr->left;
        }

        curr = st.top();
        st.pop();
        result.push_back(curr->val); // xử lý node khi không còn đi trái được nữa

        curr = curr->right; // chuyển sang cây con phải
    }

    return result;
}
```

### 14.3.3. Level Order Traversal (dùng Queue)

```cpp
#include <vector>
#include <queue>
using namespace std;

vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> result;
    if (root == nullptr) return result;

    queue<TreeNode*> q;
    q.push(root);

    while (!q.empty()) {
        int levelSize = q.size(); // số node ở tầng hiện tại, cố định trước khi xử lý
        vector<int> currentLevel;

        for (int i = 0; i < levelSize; i++) {
            TreeNode* node = q.front();
            q.pop();
            currentLevel.push_back(node->val);

            if (node->left) q.push(node->left);
            if (node->right) q.push(node->right);
        }

        result.push_back(currentLevel);
    }

    return result;
}
```

**Giải thích kỹ thuật `levelSize`:** chốt số lượng node trong Queue **trước khi** bắt đầu vòng lặp xử lý tầng hiện tại — vì trong lúc xử lý, các node con sẽ được thêm vào Queue, nếu không chốt số lượng trước, vòng lặp sẽ lẫn cả node của tầng tiếp theo vào cùng nhóm xử lý.

**Độ phức tạp:** O(n) thời gian, O(w) bộ nhớ phụ với `w` là bề rộng tối đa của cây (số node nhiều nhất ở một tầng).

### 14.3.4. Maximum Depth of Binary Tree

*(Đã trình bày chi tiết ở mục 12.3.4, nhắc lại để hoàn chỉnh mạch bài toán chương này — đây là ứng dụng Postorder: cần biết chiều sâu hai cây con trước khi tính chiều sâu node hiện tại.)*

```cpp
int maxDepth(TreeNode* root) {
    if (root == nullptr) return 0;
    return 1 + max(maxDepth(root->left), maxDepth(root->right));
}
```

### 14.3.5. Diameter of Binary Tree

**Bài toán:** tìm đường đi dài nhất giữa hai node bất kỳ trong cây (không nhất thiết đi qua root), tính bằng số cạnh.

**Bản chất:** đường kính đi qua một node bất kỳ bằng tổng chiều cao cây con trái và cây con phải của node đó. Điểm tinh tế: đường kính lớn nhất **không nhất thiết đi qua root** — có thể nằm hoàn toàn trong một cây con. Vì vậy, thuật toán cần tính chiều cao **và đồng thời** cập nhật đường kính lớn nhất tại **mọi** node trong cùng một lượt duyệt Postorder, tránh phải duyệt lại nhiều lần (tránh độ phức tạp O(n²) nếu tính chiều cao riêng cho từng node).

```cpp
#include <algorithm>
using namespace std;

int diameterHelper(TreeNode* root, int& diameter) {
    if (root == nullptr) return 0;

    int leftHeight = diameterHelper(root->left, diameter);
    int rightHeight = diameterHelper(root->right, diameter);

    // Cập nhật đường kính lớn nhất TÍNH ĐẾN node hiện tại
    diameter = max(diameter, leftHeight + rightHeight);

    return 1 + max(leftHeight, rightHeight); // trả về chiều cao để node cha sử dụng
}

int diameterOfBinaryTree(TreeNode* root) {
    int diameter = 0;
    diameterHelper(root, diameter);
    return diameter;
}
```

**Độ phức tạp:** O(n) thời gian — một lượt duyệt Postorder duy nhất, tận dụng việc tính chiều cao để đồng thời cập nhật đường kính, thay vì O(n²) nếu tính chiều cao lặp lại cho từng node riêng biệt.

### 14.3.6. Same Tree và Invert Binary Tree

```cpp
bool isSameTree(TreeNode* p, TreeNode* q) {
    if (p == nullptr && q == nullptr) return true;   // cả hai cùng rỗng: giống nhau
    if (p == nullptr || q == nullptr) return false;  // chỉ một bên rỗng: khác nhau
    if (p->val != q->val) return false;

    return isSameTree(p->left, q->left) && isSameTree(p->right, q->right);
}

TreeNode* invertTree(TreeNode* root) {
    if (root == nullptr) return nullptr;

    // Đảo ngược trước, rồi đệ quy vào hai con (đã bị hoán đổi vị trí)
    swap(root->left, root->right);
    invertTree(root->left);
    invertTree(root->right);

    return root;
}
```

**Độ phức tạp (cả hai hàm):** O(n) thời gian, O(h) bộ nhớ phụ.

### 14.3.7. Path Sum

**Bài toán:** kiểm tra có tồn tại đường đi từ root đến một lá sao cho tổng giá trị các node trên đường đi bằng `targetSum` hay không.

```cpp
bool hasPathSum(TreeNode* root, int targetSum) {
    if (root == nullptr) return false;

    // Nếu là lá, kiểm tra giá trị còn lại có khớp chính xác giá trị node này không
    if (root->left == nullptr && root->right == nullptr) {
        return targetSum == root->val;
    }

    int remaining = targetSum - root->val;
    return hasPathSum(root->left, remaining) || hasPathSum(root->right, remaining);
}
```

**Độ phức tạp:** O(n) thời gian trong trường hợp xấu nhất, O(h) bộ nhớ phụ.

### 14.3.8. Lowest Common Ancestor (LCA)

**Bài toán:** tìm node tổ tiên chung gần nhất (LCA) của hai node `p` và `q` trong cây (không giả định là BST — trường hợp tổng quát; trường hợp BST có cách tối ưu hơn, xem mục 15.3.4).

**Bản chất:** LCA của `p` và `q` là node **đầu tiên** mà khi duyệt Postorder, cả `p` lẫn `q` đều được tìm thấy trong hai nhánh **khác nhau** của nó (một bên trái, một bên phải) — hoặc chính node đó là `p`/`q`.

```cpp
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (root == nullptr || root == p || root == q) return root;

    TreeNode* left = lowestCommonAncestor(root->left, p, q);
    TreeNode* right = lowestCommonAncestor(root->right, p, q);

    // Nếu p và q được tìm thấy ở hai nhánh khác nhau, root chính là LCA
    if (left != nullptr && right != nullptr) return root;

    // Ngược lại, LCA nằm hoàn toàn ở một nhánh — trả về nhánh đó
    return (left != nullptr) ? left : right;
}
```

**Độ phức tạp:** O(n) thời gian (trường hợp xấu nhất phải duyệt toàn bộ cây), O(h) bộ nhớ phụ.

---

## 14.4. Bảng tổng hợp

| Thao tác | Kỹ thuật | Thời gian | Không gian |
|---|---|---|---|
| Preorder / Inorder / Postorder | Đệ quy hoặc Stack tường minh | O(n) | O(h) |
| Level Order | Queue (BFS) | O(n) | O(w), w = bề rộng lớn nhất |
| Maximum Depth | Postorder | O(n) | O(h) |
| Diameter | Postorder, cập nhật kết quả tại mọi node | O(n) | O(h) |
| Same Tree / Invert Tree | Preorder hoặc bất kỳ dạng nào | O(n) | O(h) |
| Path Sum | Preorder, truyền giá trị còn lại | O(n) | O(h) |
| Lowest Common Ancestor | Postorder | O(n) | O(h) |

*Lưu ý: `h = O(log n)` nếu cây cân bằng, `h = O(n)` nếu cây suy biến thành một chuỗi thẳng (trường hợp xấu nhất) — đây là động lực trực tiếp dẫn đến khái niệm cây tự cân bằng sẽ được đề cập ở Chương 15.*

---

## 14.5. Danh sách bài tập luyện tập

### Mức Easy
1. Maximum Depth of Binary Tree
2. Same Tree
3. Invert Binary Tree
4. Binary Tree Inorder Traversal (thử cả hai cách Recursive và Iterative)
5. Symmetric Tree
6. Path Sum
7. Balanced Binary Tree (kiểm tra chiều cao hai cây con chênh lệch tối đa 1)

### Mức Medium
8. Binary Tree Level Order Traversal
9. Diameter of Binary Tree
10. Lowest Common Ancestor of a Binary Tree
11. Binary Tree Zigzag Level Order Traversal
12. Construct Binary Tree from Preorder and Inorder Traversal
13. Path Sum II (liệt kê mọi đường đi thỏa mãn — kết hợp Backtracking, Chương 13)
14. Binary Tree Right Side View

### Mức Hard
15. Binary Tree Maximum Path Sum (mở rộng của Diameter — đường đi có trọng số)
16. Serialize and Deserialize Binary Tree
17. Vertical Order Traversal of a Binary Tree

---

*Chương tiếp theo: **Chương 15 — Binary Search Tree**, bổ sung ràng buộc thứ tự vào Binary Tree, mở khóa khả năng tìm kiếm hiệu quả O(log n) tương tự Binary Search (Chương 10) nhưng trên cấu trúc cây.*
