# Chương 15: Binary Search Tree (Cây tìm kiếm nhị phân)

## 15.1. Khái niệm cốt lõi

### 15.1.1. Định nghĩa

Binary Search Tree (BST) là Binary Tree (Chương 14) bổ sung thêm một **ràng buộc thứ tự** nghiêm ngặt tại mọi node: giá trị của node bất kỳ **lớn hơn** mọi giá trị trong cây con trái của nó, và **nhỏ hơn** mọi giá trị trong cây con phải của nó. Ràng buộc này áp dụng **đệ quy cho mọi cây con**, không chỉ giữa node và hai con trực tiếp.

### 15.1.2. Bản chất — vì sao BST đạt O(log n) cho tìm kiếm

Ràng buộc thứ tự của BST cho phép áp dụng đúng tư tưởng **chia đôi không gian tìm kiếm** đã trình bày ở Binary Search (mục 10.1.2): tại mỗi node đang xét, so sánh giá trị cần tìm với giá trị node đó cho biết ngay lập tức **toàn bộ một nhánh** (trái hoặc phải) chắc chắn không chứa giá trị cần tìm, loại bỏ được cả một cây con chỉ bằng một phép so sánh — không cần duyệt qua từng phần tử trong nhánh đó.

**Minh họa tìm kiếm giá trị 5 trong BST:**

```
              8
            /   \
           3     10
          / \       \
         1   6       14
            / \      /
           4   7    13

Bước 1: so sánh 5 với root=8: 5 < 8 → loại bỏ TOÀN BỘ cây con phải (10,14,13), sang trái
Bước 2: so sánh 5 với 3: 5 > 3 → loại bỏ TOÀN BỘ cây con trái của 3 (chỉ có node 1), sang phải
Bước 3: so sánh 5 với 6: 5 < 6 → sang trái
Bước 4: so sánh 5 với 4: 5 > 4 → sang phải → hết cây con, không tìm thấy 5
```

**Điều kiện để đạt O(log n):** độ phức tạp tìm kiếm phụ thuộc trực tiếp vào **chiều cao** của cây (giống nhận định ở cuối Chương 14). Nếu cây **cân bằng** (chiều cao O(log n)), tìm kiếm đạt O(log n). Nhưng nếu các giá trị được chèn vào theo thứ tự đã sắp xếp sẵn (ví dụ chèn liên tiếp 1, 2, 3, 4, 5...), BST sẽ suy biến thành một chuỗi thẳng — thực chất là một Linked List (Chương 6) — khiến chiều cao trở thành O(n) và mọi thao tác suy biến về O(n), mất hoàn toàn ưu thế của cấu trúc cây.

**Minh họa BST suy biến** khi chèn tuần tự `1, 2, 3, 4`:

```
1
 \
  2
   \
    3
     \
      4          ← đây thực chất là Linked List, chiều cao = n, mất ưu thế O(log n)
```

Đây chính là động lực cho các cấu trúc **cây tự cân bằng (self-balancing tree)** như AVL Tree hay Red-Black Tree, tự động thực hiện các phép xoay (rotation) sau mỗi lần chèn/xóa để đảm bảo chiều cao luôn duy trì O(log n) — nội dung này thuộc phạm vi nâng cao, ít khi cần tự cài đặt trong live coding interview, nhưng cần nắm bản chất để giải thích được đánh đổi giữa BST thường và BST cân bằng khi được hỏi.

---

## 15.2. Các thao tác cơ bản

### 15.2.1. Search (Tìm kiếm)

```cpp
TreeNode* searchBST(TreeNode* root, int val) {
    if (root == nullptr || root->val == val) return root;

    return (val < root->val) ? searchBST(root->left, val) : searchBST(root->right, val);
}
```

### 15.2.2. Insert (Chèn)

**Bản chất:** đi theo đúng con đường mà `search` sẽ đi (dựa vào so sánh giá trị), cho đến khi gặp một vị trí trống (`nullptr`) — đó chính là vị trí hợp lệ duy nhất để chèn giá trị mới mà vẫn giữ đúng tính chất BST.

```cpp
TreeNode* insertIntoBST(TreeNode* root, int val) {
    if (root == nullptr) return new TreeNode(val); // tìm thấy vị trí trống, chèn tại đây

    if (val < root->val) {
        root->left = insertIntoBST(root->left, val);
    } else {
        root->right = insertIntoBST(root->right, val);
    }

    return root;
}
```

### 15.2.3. Delete (Xóa) — thao tác phức tạp nhất

**Bản chất:** xóa một node trong BST cần xử lý ba trường hợp khác nhau, vì phải đảm bảo tính chất BST vẫn đúng sau khi xóa:

1. **Node là lá (không có con):** xóa trực tiếp, không cần xử lý thêm.
2. **Node có đúng một con:** thay thế node bị xóa bằng chính con của nó.
3. **Node có đủ hai con:** đây là trường hợp khó nhất — không thể xóa trực tiếp vì sẽ để lại "lỗ hổng" không rõ thay bằng gì mà vẫn giữ đúng thứ tự. Giải pháp: tìm **giá trị kế tiếp** trong thứ tự sắp xếp (successor) — chính là giá trị **nhỏ nhất của cây con phải** (node cực trái của cây con phải, đảm bảo lớn hơn mọi giá trị bên trái nhưng nhỏ hơn mọi giá trị còn lại bên phải) — dùng giá trị này thay thế cho node cần xóa, sau đó xóa node successor gốc (node này chắc chắn thuộc trường hợp 1 hoặc 2, không thể có con trái vì nó là nhỏ nhất của nhánh phải).

```cpp
TreeNode* findMin(TreeNode* root) {
    while (root->left != nullptr) root = root->left; // cực trái = giá trị nhỏ nhất
    return root;
}

TreeNode* deleteNode(TreeNode* root, int key) {
    if (root == nullptr) return nullptr;

    if (key < root->val) {
        root->left = deleteNode(root->left, key);
    } else if (key > root->val) {
        root->right = deleteNode(root->right, key);
    } else {
        // Đã tìm thấy node cần xóa
        if (root->left == nullptr) return root->right;   // trường hợp 1 hoặc chỉ có con phải
        if (root->right == nullptr) return root->left;    // chỉ có con trái

        // Trường hợp có đủ hai con: thay bằng successor (nhỏ nhất của cây con phải)
        TreeNode* successor = findMin(root->right);
        root->val = successor->val;
        root->right = deleteNode(root->right, successor->val); // xóa successor gốc
    }

    return root;
}
```

**Độ phức tạp (cả ba thao tác Search/Insert/Delete):** O(h) với `h` là chiều cao cây — O(log n) nếu cây cân bằng, O(n) trong trường hợp xấu nhất (cây suy biến, mục 15.1.2).

### 15.2.4. Inorder Traversal — tính chất đặc trưng của BST

*(Đã giới thiệu ở mục 14.2.2, nhấn mạnh lại trong ngữ cảnh BST):* Inorder Traversal (Trái → Node → Phải) trên BST luôn cho ra dãy giá trị **đã sắp xếp tăng dần** — đây là hệ quả trực tiếp từ định nghĩa BST (mọi giá trị bên trái nhỏ hơn node, mọi giá trị bên phải lớn hơn node, áp dụng đệ quy). Tính chất này là chìa khóa cho nhiều bài toán liên quan đến BST.

---

## 15.3. Cài đặt các bài toán kinh điển

### 15.3.1. Validate Binary Search Tree

**Bài toán:** kiểm tra một Binary Tree có phải BST hợp lệ hay không.

**Lỗi thường gặp:** chỉ so sánh node với hai con trực tiếp (`node->val > node->left->val && node->val < node->right->val`) là **không đủ** — ràng buộc BST yêu cầu mọi node trong toàn bộ cây con trái phải nhỏ hơn node hiện tại, không chỉ riêng con trực tiếp. Cách đúng: truyền một **khoảng giá trị hợp lệ (lower, upper)** xuống mỗi lời gọi đệ quy, thu hẹp dần khoảng này khi đi sâu vào cây.

**Minh họa lỗi:** cây `[5, 1, 4, null, null, 3, 6]` — node `4` có con trái `3` hợp lệ cục bộ, nhưng `4` lại nằm trong cây con phải của `5`, nên `4` bắt buộc phải lớn hơn `5` — vi phạm, dù so sánh cục bộ node `4` với con `3` là đúng.

```cpp
#include <climits>
using namespace std;

bool validateHelper(TreeNode* root, long long lower, long long upper) {
    if (root == nullptr) return true;

    if (root->val <= lower || root->val >= upper) return false;

    return validateHelper(root->left, lower, root->val) &&
           validateHelper(root->right, root->val, upper);
}

bool isValidBST(TreeNode* root) {
    return validateHelper(root, LLONG_MIN, LLONG_MAX);
}
```

**Độ phức tạp:** O(n) thời gian, O(h) bộ nhớ phụ.

### 15.3.2. Kth Smallest Element in a BST

**Bài toán:** tìm giá trị nhỏ thứ K trong BST.

**Bản chất:** áp dụng trực tiếp tính chất Inorder Traversal (mục 15.2.4) — phần tử nhỏ thứ K chính là phần tử thứ K trong dãy Inorder. Có thể dừng ngay khi đếm đủ K phần tử, không cần duyệt hết toàn bộ cây.

```cpp
void inorderKth(TreeNode* root, int k, int& count, int& result) {
    if (root == nullptr || count >= k) return;

    inorderKth(root->left, k, count, result);

    count++;
    if (count == k) {
        result = root->val;
        return;
    }

    inorderKth(root->right, k, count, result);
}

int kthSmallest(TreeNode* root, int k) {
    int count = 0, result = -1;
    inorderKth(root, k, count, result);
    return result;
}
```

**Độ phức tạp:** O(h + k) thời gian trong trường hợp tốt (dừng sớm khi đủ K phần tử), O(n) trường hợp xấu nhất nếu K gần bằng n; O(h) bộ nhớ phụ.

### 15.3.3. Search trong BST

*(Đã trình bày ở mục 15.2.1 — bài toán cơ bản, độ phức tạp O(h).)*

### 15.3.4. Lowest Common Ancestor trong BST — tối ưu so với Binary Tree tổng quát

**Bản chất:** khác với LCA trên Binary Tree tổng quát (mục 14.3.8, cần O(n) vì không có thông tin thứ tự để định hướng), BST cho phép dùng chính ràng buộc thứ tự để **định hướng** tìm kiếm mà không cần duyệt cả hai nhánh: nếu cả `p` và `q` đều nhỏ hơn node hiện tại, LCA chắc chắn nằm ở cây con trái; nếu cả hai đều lớn hơn, LCA nằm ở cây con phải; nếu một lớn một nhỏ (hoặc một trong hai bằng node hiện tại), node hiện tại chính là LCA — đây là điểm "phân tách" đầu tiên giữa đường đi đến `p` và đường đi đến `q`.

```cpp
TreeNode* lowestCommonAncestorBST(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (p->val < root->val && q->val < root->val) {
        return lowestCommonAncestorBST(root->left, p, q);
    }
    if (p->val > root->val && q->val > root->val) {
        return lowestCommonAncestorBST(root->right, p, q);
    }
    return root; // p và q nằm ở hai nhánh khác nhau (hoặc trùng root) — đây là điểm phân tách
}
```

**Độ phức tạp:** O(h) thời gian — vượt trội so với O(n) của trường hợp Binary Tree tổng quát, nhờ khai thác được ràng buộc thứ tự để định hướng ngay từ đầu thay vì phải khám phá cả hai nhánh.

---

## 15.4. So sánh BST với các cấu trúc tra cứu khác

*(Bổ sung so với bảng đã trình bày ở mục 3.4, xét riêng góc độ BST)*

| Tiêu chí | BST (không cân bằng) | BST cân bằng (AVL/Red-Black) | Hash Table |
|---|---|---|---|
| Tìm kiếm/Chèn/Xóa | O(h), xấu nhất O(n) | O(log n) đảm bảo | O(1) trung bình |
| Duyệt theo thứ tự | O(n), tự nhiên qua Inorder | O(n) | Không hỗ trợ trực tiếp |
| Tìm khoảng giá trị (range query) | O(log n + k), k = số kết quả | O(log n + k) | Không hỗ trợ hiệu quả |
| Độ phức tạp cài đặt | Đơn giản | Phức tạp (cần cơ chế cân bằng) | Trung bình |

**Khi nào dùng BST thay vì Hash Table:** khi cần duyệt dữ liệu theo thứ tự tăng/giảm dần, khi cần truy vấn theo khoảng (ví dụ "tìm mọi giá trị trong khoảng [10, 50]"), hoặc khi cần tìm phần tử nhỏ nhất lớn hơn một giá trị cho trước (successor/predecessor) — những thao tác Hash Table không hỗ trợ hiệu quả (đã nêu ở mục 3.4).

---

## 15.5. Bảng tổng hợp độ phức tạp

| Thao tác | BST cân bằng | BST xấu nhất (suy biến) |
|---|---|---|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |
| Kth Smallest (qua Inorder) | O(h + k) | O(n) |
| LCA | O(log n) | O(n) |

---

## 15.6. Danh sách bài tập luyện tập

### Mức Easy
1. Search in a Binary Search Tree
2. Insert into a Binary Search Tree
3. Range Sum of BST
4. Minimum Absolute Difference in BST (dùng Inorder Traversal)
5. Convert Sorted Array to Binary Search Tree (dựng BST cân bằng từ mảng đã sắp xếp)

### Mức Medium
6. Validate Binary Search Tree
7. Kth Smallest Element in a BST
8. Lowest Common Ancestor of a Binary Search Tree
9. Delete Node in a BST
10. Balanced Binary Search Tree from Preorder Traversal
11. Two Sum IV — Input is a BST (kết hợp HashSet, Chương 3, hoặc Two Pointers qua Inorder, Chương 4)

### Mức Hard
12. Recover Binary Search Tree (phát hiện và sửa hai node bị hoán đổi sai vị trí)
13. Binary Search Tree Iterator (thiết kế Iterator dùng Stack, độ phức tạp amortized O(1) cho `next()`)

---

*Chương tiếp theo: **Chương 16 — Trie**, giới thiệu cấu trúc cây chuyên biệt cho việc lưu trữ và truy vấn tập hợp chuỗi theo tiền tố, mở rộng khái niệm cây sang miền dữ liệu chuỗi đã học ở Chương 2.*
