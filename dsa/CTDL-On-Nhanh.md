# 📘 CTDL & GT – Ôn Nhanh Trước Phỏng Vấn Software Engineer

> Tài liệu tra cứu nhanh – tập trung bản chất, độ phức tạp, code Java ngắn gọn, lỗi thường gặp, câu hỏi phỏng vấn.

---

# 1. Big O – Độ Phức Tạp Thuật Toán

## 1. Khái niệm
Big O mô tả **tốc độ tăng trưởng** của thời gian/bộ nhớ khi input tăng, bỏ qua hằng số.

## 2. Khi nào sử dụng
* Dùng để **so sánh thuật toán** độc lập với phần cứng.
* Không dùng để đo thời gian chạy thực tế (đó là benchmark).

## 3. Độ phức tạp phổ biến (từ nhanh → chậm)

| Big O | Tên | Ví dụ |
|---|---|---|
| O(1) | Hằng số | Truy cập mảng theo index |
| O(log n) | Logarit | Binary Search |
| O(n) | Tuyến tính | Duyệt mảng |
| O(n log n) | Tuyến-log | Merge Sort, Quick Sort (avg) |
| O(n²) | Bình phương | Bubble Sort, nested loop |
| O(2ⁿ) | Mũ | Backtracking không cắt tỉa |
| O(n!) | Giai thừa | Hoán vị toàn bộ |

## 4. Cách tính nhanh
```java
for (int i = 0; i < n; i++) { ... }        // O(n)
for (int i = 0; i < n; i++)
  for (int j = 0; j < n; j++) { ... }      // O(n^2)
for (int i = 1; i < n; i *= 2) { ... }     // O(log n)
```
* **Cộng** khi tuần tự (loop rồi loop khác): O(n) + O(m) = O(n+m)
* **Nhân** khi lồng nhau: O(n) * O(m) = O(n*m)
* Bỏ hằng số, bỏ hạng tử bậc thấp: O(2n + 100) → O(n)

## 5. Ưu điểm
* Ngôn ngữ chung để trao đổi độ hiệu quả.

## 6. Nhược điểm
* Không phản ánh hằng số thực tế (O(n) với hằng số lớn có thể chậm hơn O(n log n)).

## 7. Lỗi thường gặp
* Nhầm Best/Average/Worst case (VD: Quick Sort worst O(n²)).
* Quên tính độ phức tạp **không gian** (space), chỉ chú ý time.
* Nhầm O(n) của thao tác thư viện (VD: `ArrayList.remove(0)` là O(n), không phải O(1)).

## 8. Câu hỏi phỏng vấn thường gặp
* **Big O là gì, dùng để làm gì?** → Đo tốc độ tăng trưởng theo input, so sánh thuật toán.
* **Amortized O(1) là gì?** → Trung bình O(1) qua nhiều lần gọi, dù có lúc tốn O(n) (VD: `ArrayList.add`).
* **O(log n) đến từ đâu?** → Mỗi bước loại bỏ một nửa không gian tìm kiếm.

## 9. LeetCode tiêu biểu
* Không có bài riêng – đây là nền tảng phân tích cho mọi bài.

---

## 🔑 Cheat Sheet Chương 1
* n ≤ 10 → O(n!) / O(2ⁿ) chấp nhận được
* n ≤ 1000 → O(n²) ổn
* n ≤ 10⁵–10⁶ → cần O(n log n) hoặc O(n)
* n > 10⁸ → cần O(log n) hoặc O(1)

## ⚠️ Dễ nhầm
* HashMap get/put là O(1) **trung bình**, worst case O(n) (collision).

## ✅ Checklist
- [ ] Phân biệt được Best/Avg/Worst
- [ ] Biết ước lượng độ phức tạp từ số vòng lặp
- [ ] Biết suy ra giới hạn n cho phép từ time limit

## ❓ 5 câu hỏi phổ biến
1. So sánh O(n log n) và O(n²)?
2. Amortized complexity là gì?
3. Vì sao Quick Sort worst case O(n²)?
4. Space complexity tính đệ quy thế nào? (stack depth)
5. Khi nào O(n²) vẫn chấp nhận được?


# 2. Array & String

## 1. Khái niệm
* **Array**: vùng nhớ liên tục, truy cập theo index O(1).
* **String**: trong Java là **immutable** – mỗi lần sửa tạo object mới.

## 2. Khi nào sử dụng
* Dùng khi cần **truy cập ngẫu nhiên nhanh**, kích thước biết trước (hoặc dùng ArrayList).
* Không nên dùng khi cần **thêm/xóa đầu mảng** thường xuyên → dùng LinkedList/Deque.

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Truy cập index | O(1) | - |
| Tìm kiếm (chưa sort) | O(n) | - |
| Thêm/xóa cuối (ArrayList) | O(1) amortized | - |
| Thêm/xóa giữa/đầu | O(n) | - |
| String concat trong loop (`+`) | O(n²) tổng | O(n) |
| StringBuilder append | O(1) amortized | O(n) |

## 4. Cách cài đặt cơ bản
```java
int[] arr = new int[]{5, 3, 8, 1};
Arrays.sort(arr);                          // O(n log n)

StringBuilder sb = new StringBuilder();
for (char c : "hello".toCharArray()) sb.append(c);
String result = sb.reverse().toString();   // "olleh"
```

## 5. Ưu điểm
* Array: truy cập nhanh, cache-friendly.
* String immutable: an toàn khi share, dùng làm HashMap key.

## 6. Nhược điểm
* Array kích thước cố định (Java native array).
* String immutable → tốn bộ nhớ khi nối chuỗi nhiều lần.

## 7. Lỗi thường gặp
* **Off-by-one**: `for (i = 0; i <= n; i++)` → ArrayIndexOutOfBounds.
* Dùng `+` nối String trong vòng lặp lớn → chậm.
* Nhầm `==` với `.equals()` khi so sánh String.
* Quên `Arrays.sort` là O(n log n), không phải O(n).

## 8. Câu hỏi phỏng vấn thường gặp
* **Vì sao String immutable trong Java?** → An toàn thread, dùng làm key HashMap (hashcode cache), bảo mật (String pool).
* **StringBuilder vs StringBuffer?** → StringBuffer synchronized (thread-safe), StringBuilder nhanh hơn không đồng bộ.
* **Array vs ArrayList?** → Array fixed-size, primitive được; ArrayList resizable, chỉ chứa object.

## 9. LeetCode tiêu biểu
* Two Sum (1)
* Best Time to Buy/Sell Stock (121)
* Product of Array Except Self (238)
* Valid Anagram (242)
* Longest Common Prefix (14)

---

# 3. Linked List

## 1. Khái niệm
Chuỗi các **node**, mỗi node chứa data + con trỏ tới node tiếp theo (và trước đó nếu doubly).

## 2. Khi nào sử dụng
* Dùng khi cần **thêm/xóa đầu danh sách** nhanh, không cần random access.
* Không dùng khi cần truy cập ngẫu nhiên nhiều (dùng Array).

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Truy cập theo index | O(n) | - |
| Thêm/xóa đầu | O(1) | - |
| Thêm/xóa cuối (có tail pointer) | O(1) | - |
| Tìm kiếm | O(n) | - |

## 4. Cách cài đặt cơ bản
```java
class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
}

// Đảo ngược Linked List
ListNode reverse(ListNode head) {
    ListNode prev = null;
    while (head != null) {
        ListNode next = head.next;
        head.next = prev;
        prev = head;
        head = next;
    }
    return prev;
}
```

## 5. Ưu điểm
* Thêm/xóa O(1) nếu có con trỏ tại vị trí.
* Không cần cấp phát liên tục bộ nhớ.

## 6. Nhược điểm
* Không random access.
* Tốn thêm bộ nhớ cho con trỏ.
* Cache-unfriendly (node rải rác trong RAM).

## 7. Lỗi thường gặp
* **NullPointerException** khi quên check `node.next != null`.
* Tạo **vòng lặp vô hạn** khi nối node sai (cycle).
* Mất tham chiếu node khi không lưu `next` trước khi đổi `head.next`.

## 8. Câu hỏi phỏng vấn thường gặp
* **Phát hiện cycle trong Linked List?** → Floyd's Cycle Detection (slow/fast pointer), gặp nhau ⇒ có cycle.
* **Tìm node giữa danh sách?** → 2 con trỏ slow/fast, fast đi gấp đôi.
* **Singly vs Doubly Linked List?** → Doubly có con trỏ `prev`, xóa node dễ hơn nhưng tốn bộ nhớ hơn.

## 9. LeetCode tiêu biểu
* Reverse Linked List (206)
* Linked List Cycle (141)
* Merge Two Sorted Lists (21)
* Remove Nth Node From End (19)
* LRU Cache (146)

---

## 🔑 Cheat Sheet Chương 2–3
* Slow/Fast pointer: tìm giữa, phát hiện cycle, tìm cycle start.
* Dummy node: luôn dùng khi thao tác có thể đổi `head`.

## ⚠️ Dễ nhầm
* `ArrayList.remove(0)` là O(n) (shift toàn bộ), khác LinkedList O(1).

## ✅ Checklist
- [ ] Thuộc lòng reverse linked list
- [ ] Biết kỹ thuật slow/fast pointer
- [ ] Biết dùng dummy node

## ❓ 5 câu hỏi phổ biến
1. Làm sao phát hiện cycle mà không dùng thêm bộ nhớ?
2. Vì sao String nối trong loop chậm?
3. Khi nào chọn LinkedList thay vì ArrayList?
4. Dummy node dùng để làm gì?
5. Cách tìm điểm bắt đầu của cycle?


# 4. Stack

## 1. Khái niệm
Cấu trúc **LIFO** (Last In First Out) – vào sau ra trước.

## 2. Khi nào sử dụng
* Dùng cho: undo/redo, duyệt DFS, kiểm tra dấu ngoặc, đánh giá biểu thức, monotonic stack.
* Không dùng khi cần truy cập giữa danh sách.

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| push | O(1) | O(1) |
| pop | O(1) | O(1) |
| peek | O(1) | O(1) |

## 4. Cách cài đặt cơ bản
```java
Deque<Integer> stack = new ArrayDeque<>(); // ưu tiên hơn java.util.Stack
stack.push(1);
stack.push(2);
int top = stack.pop(); // 2
```

## 5. Ưu điểm
* Đơn giản, thao tác O(1).
* Phù hợp mô phỏng đệ quy (dùng stack thay recursion tránh Stack Overflow).

## 6. Nhược điểm
* Không truy cập ngẫu nhiên.

## 7. Lỗi thường gặp
* Pop khi stack rỗng → Exception.
* Dùng `java.util.Stack` (synchronized, chậm) thay vì `ArrayDeque`.
* **Stack Overflow** khi đệ quy quá sâu thay vì dùng stack tường minh.

## 8. Câu hỏi phỏng vấn thường gặp
* **Vì sao dùng ArrayDeque thay Stack class?** → Stack kế thừa Vector, bị synchronized → chậm hơn.
* **Stack dùng để làm gì trong DFS?** → Lưu trạng thái quay lui (backtrack).

## 9. LeetCode tiêu biểu
* Valid Parentheses (20)
* Min Stack (155)
* Daily Temperatures (739)
* Evaluate Reverse Polish Notation (150)
* Largest Rectangle in Histogram (84)

---

# 5. Queue & Deque

## 1. Khái niệm
* **Queue**: FIFO (First In First Out).
* **Deque**: thêm/xóa được cả 2 đầu.

## 2. Khi nào sử dụng
* Queue: BFS, xử lý theo thứ tự đến (task scheduling).
* Deque: sliding window, cần thêm/xóa 2 đầu.

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| offer/poll (Queue) | O(1) | O(1) |
| addFirst/addLast (Deque) | O(1) | O(1) |

## 4. Cách cài đặt cơ bản
```java
Queue<Integer> queue = new LinkedList<>();
queue.offer(1);
int front = queue.poll();

Deque<Integer> deque = new ArrayDeque<>();
deque.addFirst(1);
deque.addLast(2);
```

## 5. Ưu điểm
* Queue: đúng thứ tự xử lý (fairness).
* Deque: linh hoạt 2 đầu, thay thế được cả Stack và Queue.

## 6. Nhược điểm
* Queue không truy cập ngẫu nhiên.

## 7. Lỗi thường gặp
* Dùng `poll()` (trả null nếu rỗng) nhầm với `remove()` (ném Exception).
* Quên `PriorityQueue` **không phải** FIFO.

## 8. Câu hỏi phỏng vấn thường gặp
* **BFS cần cấu trúc gì?** → Queue.
* **Deque implement Stack và Queue thế nào?** → addFirst/removeFirst (stack), addLast/removeFirst (queue).

## 9. LeetCode tiêu biểu
* Number of Islands (200) – BFS
* Sliding Window Maximum (239) – Deque
* Design Circular Queue (622)
* Rotting Oranges (994)

---

## 🔑 Cheat Sheet Chương 4–5
* Ngoặc hợp lệ / biểu thức → Stack.
* BFS / shortest path unweighted → Queue.
* Sliding window max/min → Monotonic Deque.

## ⚠️ Dễ nhầm
* `ArrayDeque` không cho phép `null`.

## ✅ Checklist
- [ ] Biết dùng ArrayDeque làm cả Stack và Queue
- [ ] Biết khi nào cần Deque thay vì Queue thường

## ❓ 5 câu hỏi phổ biến
1. Vì sao ArrayDeque nhanh hơn Stack/LinkedList?
2. Monotonic Deque hoạt động thế nào trong sliding window?
3. Queue dùng trong thuật toán nào là chính?
4. Sự khác biệt poll() vs remove()?
5. Có thể dùng 2 Stack để làm Queue không?


# 6. HashMap & HashSet

## 1. Khái niệm
Lưu trữ key-value (Map) hoặc unique values (Set) dựa trên **hash function** để tính vị trí bucket.

## 2. Khi nào sử dụng
* Dùng khi cần tra cứu/kiểm tra tồn tại nhanh O(1).
* Không dùng khi cần **thứ tự** (dùng LinkedHashMap/TreeMap).

## 3. Độ phức tạp

| Thao tác | Average | Worst |
|---|---|---|
| get/put/contains | O(1) | O(n) (nhiều collision) |
| TreeMap get/put | O(log n) | O(log n) |

## 4. Cách cài đặt cơ bản
```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.merge("a", 1, Integer::sum);   // tăng giá trị nếu tồn tại
map.getOrDefault("b", 0);

Set<Integer> set = new HashSet<>();
set.add(5);
boolean exists = set.contains(5);
```

## 5. Ưu điểm
* Tra cứu, thêm, xóa trung bình O(1).

## 6. Nhược điểm
* Không giữ thứ tự (trừ LinkedHashMap).
* Tốn bộ nhớ hơn Array.
* Worst case O(n) nếu hash function tệ.

## 7. Lỗi thường gặp
* Dùng object làm key mà **không override `equals()`/`hashCode()`**.
* Sửa đổi key sau khi đã insert (hash thay đổi → không tìm lại được).
* Nhầm `HashMap` (không thứ tự) với `LinkedHashMap` (thứ tự insert) hay `TreeMap` (thứ tự sort).

## 8. Câu hỏi phỏng vấn thường gặp
* **Tại sao HashMap O(1)?** → Hash function ánh xạ key → bucket index trực tiếp, không cần duyệt tuần tự.
* **Xử lý collision thế nào?** → Separate chaining (linked list/tree trong bucket, Java 8+ chuyển sang Red-Black Tree khi bucket >8 phần tử).
* **HashMap vs HashTable?** → HashTable synchronized, không cho null key/value; HashMap thì ngược lại.

## 9. LeetCode tiêu biểu
* Two Sum (1)
* Group Anagrams (49)
* Longest Consecutive Sequence (128)
* Top K Frequent Elements (347)
* LRU Cache (146)

---

## 🔑 Cheat Sheet Chương 6
* Cần đếm tần suất → HashMap<K, Integer>.
* Cần kiểm tra tồn tại nhanh → HashSet.
* Cần giữ thứ tự chèn → LinkedHashMap/LinkedHashSet.
* Cần sắp xếp theo key → TreeMap/TreeSet (O(log n)).

## ⚠️ Dễ nhầm
* HashMap cho phép 1 null key, HashTable thì không.

## ✅ Checklist
- [ ] Biết override equals/hashCode khi dùng object key
- [ ] Phân biệt HashMap/LinkedHashMap/TreeMap

## ❓ 5 câu hỏi phổ biến
1. Load factor là gì, ảnh hưởng thế nào?
2. Vì sao cần override cả equals và hashCode cùng lúc?
3. HashSet cài đặt dựa trên gì? (HashMap bên trong)
4. Khi nào HashMap suy biến thành O(n)?
5. TreeMap dùng cấu trúc gì bên trong? (Red-Black Tree)

---

# 7. Tree (Binary Tree)

## 1. Khái niệm
Cấu trúc phân cấp, mỗi node có tối đa 2 con (binary tree). Duyệt: Preorder, Inorder, Postorder, Level-order.

## 2. Khi nào sử dụng
* Biểu diễn quan hệ phân cấp (file system, org chart, DOM).
* Không nên dùng nếu chỉ cần danh sách phẳng.

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Duyệt toàn bộ (DFS/BFS) | O(n) | O(h) hoặc O(n) |
| Tìm kiếm (không cân bằng) | O(n) worst | - |

*h = chiều cao cây*

## 4. Cách cài đặt cơ bản
```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}

void inorder(TreeNode root, List<Integer> res) {
    if (root == null) return;
    inorder(root.left, res);
    res.add(root.val);
    inorder(root.right, res);
}

// BFS level order
void levelOrder(TreeNode root) {
    Queue<TreeNode> q = new LinkedList<>();
    if (root != null) q.offer(root);
    while (!q.isEmpty()) {
        TreeNode node = q.poll();
        // process node.val
        if (node.left != null) q.offer(node.left);
        if (node.right != null) q.offer(node.right);
    }
}
```

## 5. Ưu điểm
* Biểu diễn tự nhiên dữ liệu phân cấp.
* Duyệt linh hoạt (pre/in/post/level).

## 6. Nhược điểm
* Cây lệch (skewed) → thao tác suy biến O(n) như linked list.

## 7. Lỗi thường gặp
* Quên check `null` trước khi đệ quy → NullPointerException.
* Đệ quy quá sâu với cây lớn → **Stack Overflow**.
* Nhầm thứ tự Preorder/Inorder/Postorder.

## 8. Câu hỏi phỏng vấn thường gặp
* **BST khác Binary Tree ở đâu?** → BST có ràng buộc: trái < gốc < phải tại mọi node.
* **Height vs Depth?** → Height: số cạnh xa nhất tới lá; Depth: số cạnh từ gốc tới node đó.
* **Làm sao xây lại cây từ Preorder + Inorder?** → Dùng Preorder xác định gốc, Inorder chia trái/phải.

## 9. LeetCode tiêu biểu
* Maximum Depth of Binary Tree (104)
* Binary Tree Level Order Traversal (102)
* Lowest Common Ancestor (236)
* Diameter of Binary Tree (543)
* Serialize and Deserialize Binary Tree (297)


# 8. Binary Search Tree (BST)

## 1. Khái niệm
Binary Tree với ràng buộc: node trái < node gốc < node phải (áp dụng đệ quy toàn bộ subtree).

## 2. Khi nào sử dụng
* Cần dữ liệu **luôn sắp xếp** + thao tác insert/delete/search hiệu quả.
* Không nên dùng BST thường nếu dữ liệu có thể vào theo thứ tự tăng dần liên tục → cây lệch, cần **cây tự cân bằng** (AVL, Red-Black Tree, TreeMap).

## 3. Độ phức tạp

| Thao tác | Average | Worst (lệch) |
|---|---|---|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |

## 4. Cách cài đặt cơ bản
```java
TreeNode insert(TreeNode root, int val) {
    if (root == null) return new TreeNode(val);
    if (val < root.val) root.left = insert(root.left, val);
    else root.right = insert(root.right, val);
    return root;
}

boolean search(TreeNode root, int val) {
    if (root == null) return false;
    if (root.val == val) return true;
    return val < root.val ? search(root.left, val) : search(root.right, val);
}
```

## 5. Ưu điểm
* Duyệt Inorder cho ra dãy sắp xếp.
* Search/Insert nhanh hơn Array (O(log n) vs O(n)) khi cân bằng.

## 6. Nhược điểm
* Dễ suy biến thành linked list nếu insert theo thứ tự tăng/giảm dần.
* Cần cây tự cân bằng để đảm bảo O(log n) trong thực tế (Java `TreeMap` dùng Red-Black Tree).

## 7. Lỗi thường gặp
* **Duplicate**: không xử lý rõ trường hợp giá trị trùng khi insert.
* Validate BST sai bằng cách chỉ so `node.left.val < node.val < node.right.val` (không đủ – cần so với cả khoảng min/max toàn subtree).

## 8. Câu hỏi phỏng vấn thường gặp
* **BST khác Binary Tree?** → BST có ràng buộc thứ tự, Binary Tree thì không.
* **Vì sao BST có thể suy biến O(n)?** → Nếu insert dữ liệu đã sắp xếp, cây trở thành chuỗi thẳng như linked list.
* **Cách validate BST đúng?** → Truyền khoảng (min, max) xuống mỗi node đệ quy.

## 9. LeetCode tiêu biểu
* Validate Binary Search Tree (98)
* Kth Smallest Element in a BST (230)
* Insert into a BST (701)
* Lowest Common Ancestor of a BST (235)
* Convert Sorted Array to BST (108)

---

## 🔑 Cheat Sheet Chương 7–8
* Inorder BST = dãy tăng dần.
* Validate BST → truyền (min, max) qua đệ quy.
* Cây lệch = worst case cho mọi BST cơ bản.

## ⚠️ Dễ nhầm
* `TreeMap`/`TreeSet` trong Java tự cân bằng (Red-Black Tree) → luôn O(log n), khác BST tự cài đặt.

## ✅ Checklist
- [ ] Thuộc 3 kiểu duyệt DFS + BFS
- [ ] Biết validate BST đúng cách
- [ ] Hiểu vì sao cần cây tự cân bằng

## ❓ 5 câu hỏi phổ biến
1. Sự khác nhau Preorder/Inorder/Postorder dùng khi nào?
2. AVL Tree cân bằng bằng cách nào? (rotation)
3. Làm sao tìm LCA trong BST hiệu quả hơn Binary Tree thường?
4. Red-Black Tree đảm bảo gì? (height ≤ 2*log(n+1))
5. Xóa node trong BST xử lý 3 trường hợp nào? (lá, 1 con, 2 con)

---

# 9. Heap & Priority Queue

## 1. Khái niệm
Cây nhị phân gần hoàn chỉnh (complete binary tree) thỏa **heap property**: Min-Heap (cha ≤ con) hoặc Max-Heap (cha ≥ con). Thường cài bằng array.

## 2. Khi nào sử dụng
* Cần liên tục lấy phần tử **nhỏ nhất/lớn nhất** (top-K, scheduling, Dijkstra).
* Không dùng nếu cần duyệt toàn bộ theo thứ tự sort đầy đủ nhiều lần (Sort 1 lần rẻ hơn).

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| peek (min/max) | O(1) | O(n) |
| insert (offer) | O(log n) | - |
| extract (poll) | O(log n) | - |
| build heap từ array | O(n) | - |

## 4. Cách cài đặt cơ bản
```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();          // mặc định min-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

minHeap.offer(5);
minHeap.offer(1);
int min = minHeap.poll(); // 1

// Heap với Comparator tùy chỉnh (VD: theo tần suất)
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
```

## 5. Ưu điểm
* Lấy min/max O(1), insert/extract O(log n) – nhanh hơn sort lại toàn bộ mỗi lần.

## 6. Nhược điểm
* Không hỗ trợ tìm kiếm phần tử tùy ý nhanh (O(n)).
* Không giữ thứ tự đầy đủ như sorted array.

## 7. Lỗi thường gặp
* Tưởng nhầm `PriorityQueue` mặc định là max-heap (thực ra là **min-heap**).
* Dùng heap khi chỉ cần top-1 (lãng phí, dùng biến đơn giản đủ).
* Comparator viết ngược dấu gây sai thứ tự.

## 8. Câu hỏi phỏng vấn thường gặp
* **Heap dùng để làm gì?** → Truy xuất min/max nhanh, nền tảng Heap Sort, Dijkstra, Top-K.
* **Heap cài đặt bằng array thế nào?** → Node tại index i có con tại 2i+1, 2i+2, cha tại (i-1)/2.
* **Top K frequent elements làm sao O(n log k)?** → Dùng min-heap kích thước k, thay phần tử nhỏ nhất khi có phần tử tần suất cao hơn.

## 9. LeetCode tiêu biểu
* Kth Largest Element in an Array (215)
* Top K Frequent Elements (347)
* Merge K Sorted Lists (23)
* Find Median from Data Stream (295)
* Task Scheduler (621)


## 🔑 Cheat Sheet Chương 9
* Top-K lớn nhất → Min-heap kích thước K.
* Top-K nhỏ nhất → Max-heap kích thước K.
* `PriorityQueue` Java mặc định = Min-Heap.

## ⚠️ Dễ nhầm
* `PriorityQueue` **không** duyệt theo thứ tự sort khi iterate trực tiếp (chỉ `poll()` mới đúng thứ tự).

## ✅ Checklist
- [ ] Biết build min-heap và max-heap trong Java
- [ ] Biết kỹ thuật Top-K với heap kích thước K
- [ ] Hiểu heapify O(n)

## ❓ 5 câu hỏi phổ biến
1. Vì sao heapify từ array là O(n) chứ không phải O(n log n)?
2. Heap Sort có ổn định (stable) không? (Không)
3. Khi nào nên dùng heap thay vì TreeMap?
4. Median trong data stream dùng cấu trúc gì? (2 heap)
5. So sánh Heap vs BST về use case?

---

# 10. Trie (Prefix Tree)

## 1. Khái niệm
Cây đặc biệt lưu trữ chuỗi, mỗi cạnh là 1 ký tự, các chuỗi có chung tiền tố sẽ dùng chung nhánh.

## 2. Khi nào sử dụng
* Autocomplete, tìm kiếm theo tiền tố, kiểm tra từ điển.
* Không dùng nếu chỉ cần kiểm tra tồn tại chuỗi đơn lẻ (HashSet đủ và tốn ít bộ nhớ hơn).

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Insert | O(L) | O(L × N) worst |
| Search | O(L) | - |
| StartsWith (prefix) | O(L) | - |

*L = độ dài chuỗi, N = số chuỗi*

## 4. Cách cài đặt cơ bản
```java
class TrieNode {
    Map<Character, TrieNode> children = new HashMap<>();
    boolean isEnd;
}

class Trie {
    TrieNode root = new TrieNode();

    void insert(String word) {
        TrieNode node = root;
        for (char c : word.toCharArray()) {
            node = node.children.computeIfAbsent(c, k -> new TrieNode());
        }
        node.isEnd = true;
    }

    boolean search(String word) {
        TrieNode node = find(word);
        return node != null && node.isEnd;
    }

    boolean startsWith(String prefix) {
        return find(prefix) != null;
    }

    private TrieNode find(String s) {
        TrieNode node = root;
        for (char c : s.toCharArray()) {
            node = node.children.get(c);
            if (node == null) return null;
        }
        return node;
    }
}
```

## 5. Ưu điểm
* Tìm kiếm theo tiền tố cực nhanh O(L), không phụ thuộc số lượng từ N.
* Tiết kiệm bộ nhớ khi nhiều chuỗi chung tiền tố.

## 6. Nhược điểm
* Tốn bộ nhớ nếu chuỗi ít chung tiền tố.
* Cài đặt phức tạp hơn HashSet.

## 7. Lỗi thường gặp
* Quên đánh dấu `isEnd` → không phân biệt được "car" là từ hoàn chỉnh hay chỉ là tiền tố của "card".
* NullPointerException khi truy cập `children.get(c)` mà không check null.

## 8. Câu hỏi phỏng vấn thường gặp
* **Trie khác HashSet ở điểm nào?** → Trie hỗ trợ tìm theo **tiền tố** hiệu quả, HashSet chỉ kiểm tra tồn tại chính xác.
* **Độ phức tạp không gian của Trie?** → O(tổng độ dài tất cả chuỗi) trong trường hợp xấu nhất.

## 9. LeetCode tiêu biểu
* Implement Trie (208)
* Word Search II (212)
* Design Add and Search Words Data Structure (211)
* Longest Word in Dictionary (720)

---

## 🔑 Cheat Sheet Chương 10
* Bài toán có "prefix", "autocomplete", "dictionary" → nghĩ ngay đến Trie.

## ⚠️ Dễ nhầm
* Trie không phải lúc nào cũng tốt hơn HashSet – chỉ thắng khi cần thao tác **prefix**.

## ✅ Checklist
- [ ] Cài đặt được Trie từ đầu (insert/search/startsWith)
- [ ] Biết khi nào Trie hiệu quả hơn HashSet

## ❓ 5 câu hỏi phổ biến
1. Vì sao Trie tốt cho autocomplete?
2. Làm sao dùng Trie giải Word Search II hiệu quả hơn brute force?
3. Trie có thể dùng mảng 26 thay vì HashMap không? Ưu/nhược?
4. Không gian lưu trữ Trie phụ thuộc yếu tố nào?
5. So sánh Trie và Suffix Tree?


# 11. Graph

## 1. Khái niệm
Tập đỉnh (vertices) và cạnh (edges) biểu diễn quan hệ. Có hướng/vô hướng, có trọng số/không trọng số.

## 2. Khi nào sử dụng
* Mô hình mạng lưới: mạng xã hội, bản đồ, dependency graph.
* Không cần dùng Graph nếu quan hệ đơn giản là cây/dãy tuyến tính.

## 3. Độ phức tạp

| Thuật toán | Time | Space | Dùng khi |
|---|---|---|---|
| BFS/DFS | O(V+E) | O(V) | Duyệt, tìm đường đi không trọng số |
| Dijkstra (heap) | O(E log V) | O(V) | Đường đi ngắn nhất, trọng số dương |
| Bellman-Ford | O(V×E) | O(V) | Có cạnh âm |
| Floyd-Warshall | O(V³) | O(V²) | Tất cả cặp đỉnh |
| Topological Sort | O(V+E) | O(V) | DAG, thứ tự phụ thuộc |

## 4. Cách cài đặt cơ bản
```java
Map<Integer, List<Integer>> graph = new HashMap<>();
graph.computeIfAbsent(0, k -> new ArrayList<>()).add(1);

void bfs(int start, Map<Integer, List<Integer>> graph) {
    Queue<Integer> queue = new LinkedList<>();
    Set<Integer> visited = new HashSet<>();
    queue.offer(start);
    visited.add(start);
    while (!queue.isEmpty()) {
        int node = queue.poll();
        for (int next : graph.getOrDefault(node, List.of())) {
            if (!visited.contains(next)) {
                visited.add(next);
                queue.offer(next);
            }
        }
    }
}

void dfs(int node, Map<Integer, List<Integer>> graph, Set<Integer> visited) {
    if (visited.contains(node)) return;
    visited.add(node);
    for (int next : graph.getOrDefault(node, List.of())) dfs(next, graph, visited);
}
```

## 5. Ưu điểm
* Mô hình hóa được mọi loại quan hệ phức tạp.

## 6. Nhược điểm
* Nhiều thuật toán phức tạp, dễ sai khi cài đặt.
* Tốn bộ nhớ với đồ thị dày đặc (dùng adjacency matrix O(V²)).

## 7. Lỗi thường gặp
* Quên đánh dấu **visited** → **infinite loop** khi có chu trình (cycle).
* Nhầm đồ thị có hướng và vô hướng khi build adjacency list.
* Dùng DFS đệ quy cho graph quá lớn → **Stack Overflow** (nên dùng iterative + stack tường minh).

## 8. Câu hỏi phỏng vấn thường gặp
* **BFS vs DFS dùng khi nào?** → BFS: đường đi ngắn nhất (unweighted), duyệt theo tầng. DFS: duyệt toàn bộ, phát hiện cycle, backtracking.
* **Dijkstra không hoạt động khi nào?** → Khi đồ thị có cạnh trọng số âm (dùng Bellman-Ford).
* **Union-Find vs DFS để detect cycle?** → Union-Find nhanh hơn cho đồ thị vô hướng động (thêm cạnh dần).

## 9. LeetCode tiêu biểu
* Number of Islands (200)
* Course Schedule (207) – Topological Sort
* Clone Graph (133)
* Network Delay Time (743) – Dijkstra
* Word Ladder (127) – BFS

---

## 🔑 Cheat Sheet Chương 11
* Đường đi ngắn nhất không trọng số → BFS.
* Đường đi ngắn nhất có trọng số dương → Dijkstra.
* Có cạnh âm → Bellman-Ford.
* Thứ tự phụ thuộc (DAG) → Topological Sort (BFS Kahn's hoặc DFS).
* Kết nối động (union) → Union-Find.

## ⚠️ Dễ nhầm
* Topological Sort chỉ áp dụng cho **DAG** (Directed Acyclic Graph) – có cycle thì không tồn tại thứ tự.

## ✅ Checklist
- [ ] Cài đặt được BFS/DFS iterative và recursive
- [ ] Biết chọn đúng thuật toán shortest path theo điều kiện đồ thị
- [ ] Biết detect cycle bằng DFS (3 màu: white/gray/black)

## ❓ 5 câu hỏi phổ biến
1. Adjacency List vs Adjacency Matrix, khi nào dùng cái nào?
2. Làm sao detect cycle trong directed graph?
3. Topological Sort có bao nhiêu cách cài đặt? (Kahn's BFS, DFS + stack)
4. Dijkstra dùng cấu trúc dữ liệu gì để tối ưu? (Priority Queue)
5. Bipartite graph là gì, kiểm tra bằng cách nào? (2-coloring BFS/DFS)


# 12. Sorting

## 1. Khái niệm
Sắp xếp phần tử theo thứ tự. Các thuật toán chia làm 2 nhóm: **so sánh** (comparison-based) và **không so sánh** (counting, radix).

## 2. Khi nào sử dụng
* Dùng khi cần dữ liệu có thứ tự để binary search, two-pointer, greedy...
* Không cần tự cài đặt sort trong phỏng vấn thực tế (dùng `Arrays.sort`/`Collections.sort`) trừ khi được yêu cầu.

## 3. Độ phức tạp

| Thuật toán | Best | Average | Worst | Space | Stable |
|---|---|---|---|---|---|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | ❌ |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | ✅ |

## 4. Cách cài đặt cơ bản
```java
// Quick Sort (Lomuto partition)
void quickSort(int[] a, int lo, int hi) {
    if (lo >= hi) return;
    int pivot = a[hi], i = lo;
    for (int j = lo; j < hi; j++) {
        if (a[j] < pivot) { swap(a, i, j); i++; }
    }
    swap(a, i, hi);
    quickSort(a, lo, i - 1);
    quickSort(a, i + 1, hi);
}

// Merge Sort
void mergeSort(int[] a, int lo, int hi) {
    if (lo >= hi) return;
    int mid = (lo + hi) / 2;
    mergeSort(a, lo, mid);
    mergeSort(a, mid + 1, hi);
    merge(a, lo, mid, hi);
}
```

## 5. Ưu điểm
* Merge Sort: ổn định, O(n log n) đảm bảo mọi trường hợp.
* Quick Sort: nhanh trong thực tế, ít bộ nhớ phụ.

## 6. Nhược điểm
* Quick Sort: worst case O(n²) nếu pivot chọn tệ (VD: mảng đã sort + chọn pivot đầu/cuối).
* Merge Sort: tốn O(n) bộ nhớ phụ.

## 7. Lỗi thường gặp
* Chọn pivot cố định (luôn lấy phần tử đầu) → worst case với mảng đã sort.
* Quên `Arrays.sort()` với **object** dùng Dual-Pivot Quicksort (không stable), còn `Arrays.sort()` với **primitive** khác **Collections.sort()** (Timsort, stable) cho object.
* Off-by-one khi chia mid trong merge sort.

## 8. Câu hỏi phỏng vấn thường gặp
* **Vì sao Quick Sort worst case O(n²)?** → Khi pivot luôn là min/max, partition không cân, giống chọn 1 phần tử mỗi lần.
* **Stable sort là gì, khi nào cần?** → Giữ thứ tự tương đối của phần tử bằng nhau; cần khi sort nhiều tiêu chí.
* **Java dùng thuật toán sort gì?** → `Arrays.sort(primitives)` = Dual-Pivot Quicksort; `Arrays.sort(objects)`/`Collections.sort` = Timsort (Merge Sort biến thể).

## 9. LeetCode tiêu biểu
* Sort Colors (75)
* Merge Intervals (56)
* Kth Largest Element in an Array (215)
* Largest Number (179)

---

## 🔑 Cheat Sheet Chương 12
* Cần stable + đảm bảo O(n log n) → Merge Sort.
* Cần nhanh, ít bộ nhớ, chấp nhận rủi ro → Quick Sort.
* Dữ liệu giá trị nhỏ, giới hạn (0-100) → Counting Sort O(n).

## ⚠️ Dễ nhầm
* "Sort tại chỗ" (in-place) không đồng nghĩa với O(1) space nếu dùng đệ quy (Quick Sort tốn O(log n) stack).

## ✅ Checklist
- [ ] Nhớ được bảng độ phức tạp 7 thuật toán sort
- [ ] Cài được Quick Sort & Merge Sort từ đầu
- [ ] Biết khi nào dùng Counting Sort

## ❓ 5 câu hỏi phổ biến
1. Vì sao Merge Sort luôn O(n log n) mà Quick Sort thì không?
2. Cách chọn pivot để tránh worst case? (random, median-of-three)
3. Sort ổn định quan trọng trong tình huống nào?
4. Radix Sort hoạt động ra sao?
5. Có thể sort O(n) không? Khi nào? (Counting/Radix/Bucket sort với điều kiện đặc biệt)

---

# 13. Binary Search

## 1. Khái niệm
Tìm kiếm trong không gian **đã sắp xếp** (hoặc có tính đơn điệu) bằng cách chia đôi mỗi bước.

## 2. Khi nào sử dụng
* Mảng đã sort, hoặc bài toán có tính chất "đơn điệu" (monotonic) để binary search trên **đáp án**.
* Không dùng khi dữ liệu không có thứ tự/tính đơn điệu.

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Binary Search | O(log n) | O(1) iterative / O(log n) recursive |

## 4. Cách cài đặt cơ bản
```java
int binarySearch(int[] a, int target) {
    int lo = 0, hi = a.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;   // tránh overflow
        if (a[mid] == target) return mid;
        else if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

// Binary Search trên đáp án (lower bound)
int lowerBound(int[] a, int target) {
    int lo = 0, hi = a.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] < target) lo = mid + 1;
        else hi = mid;
    }
    return lo;
}
```

## 5. Ưu điểm
* O(log n) – rất nhanh với dữ liệu lớn.

## 6. Nhược điểm
* Yêu cầu dữ liệu đã sort hoặc có tính đơn điệu.
* Sort trước khi search tốn O(n log n) nếu chỉ search 1 lần thì không lợi.

## 7. Lỗi thường gặp
* **Off-by-one**: dùng `hi = a.length` thay vì `a.length - 1` không nhất quán với điều kiện loop (`<` vs `<=`).
* **Integer overflow**: `mid = (lo + hi) / 2` với lo, hi lớn → dùng `lo + (hi - lo) / 2`.
* Infinite loop khi cập nhật `lo`/`hi` sai (không thu hẹp khoảng).

## 8. Câu hỏi phỏng vấn thường gặp
* **Binary Search trên đáp án là gì?** → Áp dụng khi bài toán có tính đơn điệu (VD: "có thể hoàn thành trong X ngày không?"), binary search giá trị X thay vì index.
* **Tìm lower bound / upper bound khác gì search thường?** → Lower bound tìm vị trí đầu tiên ≥ target, upper bound tìm vị trí đầu tiên > target.

## 9. LeetCode tiêu biểu
* Binary Search (704)
* Search in Rotated Sorted Array (33)
* Find First and Last Position of Element (34)
* Koko Eating Bananas (875) – Binary search on answer
* Median of Two Sorted Arrays (4)

---

## 🔑 Cheat Sheet Chương 13
* Từ khóa "minimize the maximum" / "maximize the minimum" → Binary Search on Answer.
* Luôn dùng `mid = lo + (hi - lo)/2`.

## ⚠️ Dễ nhầm
* Rotated Sorted Array vẫn binary search được – chỉ cần xác định nửa nào đang sorted.

## ✅ Checklist
- [ ] Nhớ template chuẩn (tránh off-by-one)
- [ ] Biết binary search on answer
- [ ] Xử lý được rotated sorted array

## ❓ 5 câu hỏi phổ biến
1. Làm sao binary search trên mảng có phần tử trùng?
2. Binary search on answer áp dụng khi nào?
3. Rotated Sorted Array binary search thế nào?
4. Vì sao dùng `lo + (hi-lo)/2` thay vì `(lo+hi)/2`?
5. Tìm peak element trong mảng bằng binary search ra sao?


# 14. Recursion & Backtracking

## 1. Khái niệm
* **Recursion**: hàm tự gọi chính nó, cần base case để dừng.
* **Backtracking**: thử 1 lựa chọn, đệ quy tiếp, nếu không hợp lệ thì "quay lui" (undo) và thử lựa chọn khác.

## 2. Khi nào sử dụng
* Backtracking: sinh tổ hợp/hoán vị/subset, giải bài toán constraint (N-Queens, Sudoku).
* Không dùng backtracking nếu có công thức toán học trực tiếp (VD: nCr có công thức).

## 3. Độ phức tạp

| Bài toán | Time |
|---|---|
| Permutations | O(n!) |
| Subsets | O(2ⁿ) |
| Combinations C(n,k) | O(C(n,k) × k) |

## 4. Cách cài đặt cơ bản
```java
void backtrack(List<Integer> path, boolean[] used, int[] nums, List<List<Integer>> res) {
    if (path.size() == nums.length) {
        res.add(new ArrayList<>(path));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        path.add(nums[i]);
        backtrack(path, used, nums, res);      // đệ quy
        path.remove(path.size() - 1);          // quay lui (undo)
        used[i] = false;
    }
}
```

## 5. Ưu điểm
* Giải quyết được bài toán tổ hợp phức tạp một cách có hệ thống.
* Có thể **cắt tỉa (pruning)** để giảm không gian tìm kiếm.

## 6. Nhược điểm
* Độ phức tạp mũ – chỉ dùng được với n nhỏ.
* Dễ **Stack Overflow** nếu đệ quy quá sâu.

## 7. Lỗi thường gặp
* Quên **undo** (remove/reset) sau đệ quy → kết quả sai.
* Thiếu base case → **infinite recursion** → Stack Overflow.
* Copy sai list (`res.add(path)` thay vì `res.add(new ArrayList<>(path))`) → tất cả kết quả trỏ chung 1 reference.
* Không pruning → Timeout với n lớn.

## 8. Câu hỏi phỏng vấn thường gặp
* **Backtracking khác DFS thường ở đâu?** → Backtracking có bước "undo" tường minh để thử nhánh khác.
* **Làm sao tối ưu backtracking?** → Pruning: cắt sớm nhánh không thể đạt kết quả hợp lệ.
* **Vì sao phải `new ArrayList<>(path)`?** → path là reference dùng chung xuyên suốt đệ quy, không copy sẽ bị thay đổi sau đó.

## 9. LeetCode tiêu biểu
* Permutations (46)
* Subsets (78)
* Combination Sum (39)
* N-Queens (51)
* Word Search (79)

---

## 🔑 Cheat Sheet Chương 14
* Template: chọn → đệ quy → **bỏ chọn (undo)**.
* Subset: mỗi phần tử có 2 lựa chọn (lấy/không lấy) → O(2ⁿ).
* Permutation: dùng mảng `used[]` đánh dấu.

## ⚠️ Dễ nhầm
* Combination (không quan tâm thứ tự) khác Permutation (quan tâm thứ tự).

## ✅ Checklist
- [ ] Nhớ template backtracking chuẩn
- [ ] Biết pruning để tối ưu
- [ ] Luôn copy list khi thêm vào kết quả

## ❓ 5 câu hỏi phổ biến
1. Sự khác nhau giữa recursion thường và backtracking?
2. Làm sao xử lý duplicate trong Subsets II / Permutations II?
3. Time complexity của N-Queens là gì?
4. Khi nào nên dừng backtracking sớm (pruning)?
5. Tail recursion là gì, Java có tối ưu nó không? (Không)

---

# 15. Greedy

## 1. Khái niệm
Tại mỗi bước chọn lựa chọn **tốt nhất cục bộ** (local optimal), hy vọng dẫn đến kết quả tối ưu toàn cục.

## 2. Khi nào sử dụng
* Bài toán có tính chất **greedy choice property** + **optimal substructure** (chứng minh được).
* Không dùng khi lựa chọn cục bộ không đảm bảo tối ưu toàn cục → cần DP thay thế.

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Thường cần sort trước | O(n log n) | O(1)–O(n) |

## 4. Cách cài đặt cơ bản
```java
// Activity Selection / Merge Intervals kiểu greedy
int eraseOverlapIntervals(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[1] - b[1]);  // sort theo điểm kết thúc
    int count = 0, end = Integer.MIN_VALUE;
    for (int[] iv : intervals) {
        if (iv[0] >= end) end = iv[1];   // giữ lại
        else count++;                     // phải xóa (overlap)
    }
    return count;
}
```

## 5. Ưu điểm
* Đơn giản, hiệu quả O(n log n), dễ cài đặt hơn DP.

## 6. Nhược điểm
* **Không phải lúc nào cũng đúng** – cần chứng minh tính đúng đắn (greedy proof), nếu không dễ sai.

## 7. Lỗi thường gặp
* Áp dụng greedy cho bài toán cần DP (VD: 0/1 Knapsack không giải được bằng greedy, nhưng Fractional Knapsack thì được).
* Chọn sai tiêu chí sort (VD: sort theo start thay vì end trong interval scheduling).

## 8. Câu hỏi phỏng vấn thường gặp
* **Greedy khác DP ở đâu?** → Greedy chọn 1 lần không quay lại; DP xét mọi khả năng, lưu kết quả con.
* **Làm sao biết bài toán giải được bằng Greedy?** → Chứng minh Greedy Choice Property: chọn cục bộ tối ưu không làm mất khả năng đạt tối ưu toàn cục.

## 9. LeetCode tiêu biểu
* Jump Game (55)
* Gas Station (134)
* Non-overlapping Intervals (435)
* Task Scheduler (621)
* Partition Labels (763)

---

## 🔑 Cheat Sheet Chương 15
* Interval scheduling → sort theo **end time**.
* Greedy thường kết hợp sort trước khi duyệt tuyến tính.

## ⚠️ Dễ nhầm
* Fractional Knapsack giải được bằng Greedy, nhưng 0/1 Knapsack thì **không** (cần DP).

## ✅ Checklist
- [ ] Biết chứng minh greedy choice property cơ bản
- [ ] Nhận diện bài toán interval → sort theo end

## ❓ 5 câu hỏi phổ biến
1. Vì sao Greedy không đúng cho 0/1 Knapsack?
2. Interval Scheduling Maximization sort theo tiêu chí gì?
3. Greedy có luôn nhanh hơn DP không?
4. Cho ví dụ Greedy sai nếu áp dụng nhầm chỗ?
5. Huffman Coding dùng chiến lược gì? (Greedy + Heap)


# 16. Dynamic Programming (DP)

## 1. Khái niệm
Giải bài toán lớn bằng cách chia thành **bài toán con chồng lấp** (overlapping subproblems) + **cấu trúc con tối ưu** (optimal substructure), lưu kết quả để tránh tính lại (memoization/tabulation).

## 2. Khi nào sử dụng
* Bài toán đếm/tối ưu có thể chia nhỏ và các subproblem lặp lại.
* Không dùng nếu subproblem không lặp lại (dùng Divide & Conquer thường hoặc Greedy).

## 3. Độ phức tạp

| Cách tiếp cận | Time | Space |
|---|---|---|
| Top-down (memoization) | O(states × transition) | O(states) + stack |
| Bottom-up (tabulation) | O(states × transition) | O(states), có thể tối ưu O(1)–O(n) |

## 4. Cách cài đặt cơ bản
```java
// Fibonacci - Top-down memoization
Map<Integer, Long> memo = new HashMap<>();
long fib(int n) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n);
    long result = fib(n - 1) + fib(n - 2);
    memo.put(n, result);
    return result;
}

// Fibonacci - Bottom-up tabulation (tối ưu space O(1))
long fibBottomUp(int n) {
    if (n <= 1) return n;
    long prev2 = 0, prev1 = 1;
    for (int i = 2; i <= n; i++) {
        long cur = prev1 + prev2;
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}

// 0/1 Knapsack
int knapsack(int[] w, int[] v, int cap) {
    int n = w.length;
    int[][] dp = new int[n + 1][cap + 1];
    for (int i = 1; i <= n; i++) {
        for (int c = 0; c <= cap; c++) {
            dp[i][c] = dp[i - 1][c];
            if (w[i - 1] <= c)
                dp[i][c] = Math.max(dp[i][c], dp[i - 1][c - w[i - 1]] + v[i - 1]);
        }
    }
    return dp[n][cap];
}
```

## 5. Ưu điểm
* Giải quyết bài toán mũ (exponential) trong thời gian đa thức.

## 6. Nhược điểm
* Khó nhận diện state và transition đúng.
* Có thể tốn nhiều bộ nhớ nếu không tối ưu chiều space.

## 7. Lỗi thường gặp
* Xác định sai **state** (thiếu chiều biến số cần thiết).
* Quên **base case** → sai kết quả hoặc StackOverflow (top-down).
* Duplicate tính toán do quên memoize.
* Off-by-one khi index dp array (dp[i] ứng với phần tử thứ i-1 hay i?).

## 8. Câu hỏi phỏng vấn thường gặp
* **DP khác Recursion thường ở đâu?** → DP lưu lại (cache) kết quả subproblem đã tính để tránh tính lại.
* **Top-down vs Bottom-up?** → Top-down: đệ quy + memo, dễ viết hơn. Bottom-up: vòng lặp + bảng, tránh stack overflow, dễ tối ưu space.
* **Làm sao nhận diện bài toán DP?** → Có từ khóa "số cách", "tối ưu (max/min)", "có thể đạt được không", và subproblem chồng lấp.

## 9. LeetCode tiêu biểu
* Climbing Stairs (70)
* Coin Change (322)
* Longest Common Subsequence (1143)
* Longest Increasing Subsequence (300)
* House Robber (198)

---

## 🔑 Cheat Sheet Chương 16
* Xác định: **State** (biến gì đại diện subproblem) → **Transition** (công thức truy hồi) → **Base case** → **Thứ tự tính**.
* Có thể tối ưu space từ O(n²) → O(n) nếu dp[i] chỉ phụ thuộc dp[i-1].

## ⚠️ Dễ nhầm
* Memoization (top-down) vẫn tốn O(n) stack đệ quy, không tự động tiết kiệm space như bottom-up.

## ✅ Checklist
- [ ] Biết xác định state/transition cho bài toán mới
- [ ] Cài được cả top-down và bottom-up
- [ ] Biết tối ưu space 2D → 1D khi có thể

## ❓ 5 câu hỏi phổ biến
1. Làm sao nhận biết 1 bài toán giải được bằng DP?
2. Sự khác nhau giữa Memoization và Tabulation?
3. Optimal Substructure là gì?
4. Vì sao Fibonacci đệ quy thường là O(2ⁿ) nhưng DP là O(n)?
5. Làm sao tối ưu space DP 2 chiều xuống 1 chiều?


# 17. Union Find (Disjoint Set Union - DSU)

## 1. Khái niệm
Cấu trúc quản lý các tập hợp rời nhau, hỗ trợ 2 thao tác chính: `find` (tìm đại diện nhóm) và `union` (gộp 2 nhóm).

## 2. Khi nào sử dụng
* Kiểm tra kết nối động (dynamic connectivity), phát hiện cycle trong đồ thị vô hướng, Kruskal's MST.
* Không dùng nếu chỉ cần duyệt tĩnh 1 lần (BFS/DFS đơn giản hơn).

## 3. Độ phức tạp

| Thao tác | Time (với Union by Rank + Path Compression) |
|---|---|
| find | O(α(n)) ≈ O(1) |
| union | O(α(n)) ≈ O(1) |

*α(n) = hàm nghịch đảo Ackermann, tăng cực chậm, coi như hằng số.*

## 4. Cách cài đặt cơ bản
```java
class UnionFind {
    int[] parent, rank;

    UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]); // path compression
        return parent[x];
    }

    boolean union(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false; // đã cùng nhóm -> có cycle
        if (rank[rx] < rank[ry]) { int t = rx; rx = ry; ry = t; }
        parent[ry] = rx;
        if (rank[rx] == rank[ry]) rank[rx]++;
        return true;
    }
}
```

## 5. Ưu điểm
* Gần như O(1) cho mỗi thao tác với 2 tối ưu (path compression + union by rank).
* Cực kỳ hiệu quả cho bài toán kết nối động.

## 6. Nhược điểm
* Không hỗ trợ tách nhóm (chỉ gộp, không split).
* Không lưu thứ tự/đường đi cụ thể như BFS/DFS.

## 7. Lỗi thường gặp
* Quên **path compression** → find suy biến O(n).
* Quên **union by rank** → cây lệch, hiệu năng giảm.
* Nhầm `find(x) == find(y)` (cùng nhóm) với `x == y`.

## 8. Câu hỏi phỏng vấn thường gặp
* **Union Find dùng để làm gì?** → Quản lý và kiểm tra các thành phần liên thông một cách hiệu quả khi có thao tác gộp động.
* **Vì sao gần O(1)?** → Nhờ path compression (làm phẳng cây) + union by rank (tránh cây lệch).
* **Union Find vs DFS để detect cycle?** → Union Find phù hợp hơn khi cạnh được thêm dần (online), DFS phù hợp khi đồ thị tĩnh.

## 9. LeetCode tiêu biểu
* Number of Provinces (547)
* Redundant Connection (684)
* Accounts Merge (721)
* Number of Islands II (305)
* Graph Valid Tree (261)

---

## 🔑 Cheat Sheet Chương 17
* Luôn implement kèm **path compression** + **union by rank/size**.
* Từ khóa "connected components", "merge groups", "detect cycle undirected" → Union Find.

## ⚠️ Dễ nhầm
* Union Find không dùng được trực tiếp cho đồ thị **có hướng**.

## ✅ Checklist
- [ ] Cài được UnionFind với cả 2 tối ưu
- [ ] Biết dùng cho Kruskal's MST
- [ ] Nhận diện bài toán "connected components" động

## ❓ 5 câu hỏi phổ biến
1. Path compression hoạt động thế nào?
2. Union by rank khác union by size ra sao?
3. Union Find dùng trong thuật toán MST nào? (Kruskal)
4. Độ phức tạp thực sự của α(n) là gì?
5. Khi nào nên dùng Union Find thay vì BFS/DFS?

---

# 18. Prefix Sum

## 1. Khái niệm
M��ng phụ `prefix[i]` = tổng các phần tử từ 0 đến i-1, giúp tính tổng đoạn `[l, r]` trong O(1).

## 2. Khi nào sử dụng
* Nhiều truy vấn tổng đoạn con trên mảng **tĩnh** (không đổi).
* Không dùng nếu mảng thay đổi liên tục (dùng Fenwick Tree/Segment Tree thay thế).

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Xây prefix sum | O(n) | O(n) |
| Truy vấn tổng đoạn | O(1) | - |

## 4. Cách cài đặt cơ bản
```java
int[] buildPrefix(int[] nums) {
    int[] prefix = new int[nums.length + 1];
    for (int i = 0; i < nums.length; i++) prefix[i + 1] = prefix[i] + nums[i];
    return prefix;
}

int rangeSum(int[] prefix, int l, int r) { // sum [l, r] inclusive
    return prefix[r + 1] - prefix[l];
}
```

## 5. Ưu điểm
* Truy vấn tổng đoạn O(1) sau khi tiền xử lý O(n).

## 6. Nhược điểm
* Chỉ hiệu quả khi mảng **không đổi**; nếu update thường xuyên thì mất lợi thế.

## 7. Lỗi thường gặp
* **Off-by-one** giữa prefix[i] và index thực trong mảng gốc.
* Quên khởi tạo `prefix[0] = 0`.
* Áp dụng cho mảng 2D (prefix sum ma trận) sai công thức include-exclude.

## 8. Câu hỏi phỏng vấn thường gặp
* **Prefix Sum giải quyết vấn đề gì?** → Biến truy vấn tổng đoạn từ O(n) mỗi lần thành O(1) sau tiền xử lý O(n).
* **Prefix Sum + HashMap dùng khi nào?** → Đếm số subarray có tổng = k (Subarray Sum Equals K).

## 9. LeetCode tiêu biểu
* Range Sum Query - Immutable (303)
* Subarray Sum Equals K (560)
* Product of Array Except Self (238)
* Continuous Subarray Sum (523)

---

## 🔑 Cheat Sheet Chương 18
* Từ khóa "sum of subarray", nhiều truy vấn → Prefix Sum.
* Kết hợp **HashMap<prefixSum, count>** để đếm subarray tổng = k trong O(n).

## ⚠️ Dễ nhầm
* Prefix Sum không tốt cho mảng có update thường xuyên (dùng Fenwick/Segment Tree).

## ✅ Checklist
- [ ] Biết xây prefix sum 1D và 2D
- [ ] Biết kết hợp prefix sum với HashMap để đếm subarray

## ❓ 5 câu hỏi phổ biến
1. Vì sao Prefix Sum + HashMap giải Subarray Sum = K trong O(n)?
2. Prefix Sum 2D tính thế nào? (include-exclude)
3. Khi nào Prefix Sum không còn hiệu quả?
4. So sánh Prefix Sum với Fenwick Tree?
5. Difference Array là gì, liên hệ gì với Prefix Sum? (ngược lại của nhau)


# 19. Sliding Window

## 1. Khái niệm
Kỹ thuật duy trì 1 "cửa sổ" (window) liên tục trên mảng/chuỗi, mở rộng/thu hẹp thay vì duyệt lại từ đầu.

## 2. Khi nào sử dụng
* Bài toán tìm subarray/substring thỏa điều kiện (dài nhất/ngắn nhất/số lượng) với cửa sổ **liên tục**.
* Không dùng nếu phần tử không cần liên tục (dùng DP hoặc Two Pointer khác kiểu).

## 3. Độ phức tạp

| Loại | Time | Space |
|---|---|---|
| Fixed size window | O(n) | O(1)–O(k) |
| Variable size window | O(n) (mỗi phần tử vào/ra tối đa 1 lần) | O(1)–O(k) |

## 4. Cách cài đặt cơ bản
```java
// Longest substring without repeating characters
int lengthOfLongestSubstring(String s) {
    Set<Character> window = new HashSet<>();
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        while (window.contains(s.charAt(right))) {
            window.remove(s.charAt(left));
            left++;
        }
        window.add(s.charAt(right));
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

## 5. Ưu điểm
* Giảm độ phức tạp từ O(n²)/O(n³) (brute force) xuống O(n).

## 6. Nhược điểm
* Chỉ áp dụng khi điều kiện window có tính **đơn điệu** (mở rộng/thu hẹp hợp lý).

## 7. Lỗi thường gặp
* Quên cập nhật `left` khi thu hẹp window → sai kết quả.
* **Off-by-one** khi tính độ dài window (`right - left + 1`).
* Nhầm điều kiện while/if khi thu hẹp window.

## 8. Câu hỏi phỏng vấn thường gặp
* **Khi nào dùng Sliding Window thay vì brute force?** → Khi bài toán yêu cầu subarray/substring liên tục và điều kiện có tính đơn điệu khi mở rộng/thu hẹp.
* **Sliding Window khác Two Pointer ở đâu?** → Sliding Window luôn xét đoạn liên tục [left, right]; Two Pointer tổng quát hơn (có thể 2 con trỏ từ 2 đầu mảng).

## 9. LeetCode tiêu biểu
* Longest Substring Without Repeating Characters (3)
* Minimum Window Substring (76)
* Longest Repeating Character Replacement (424)
* Maximum Sum Subarray of Size K
* Sliding Window Maximum (239)

---

## 🔑 Cheat Sheet Chương 19
* Từ khóa "longest/shortest subarray/substring thỏa điều kiện" → nghĩ Sliding Window trước.
* Fixed size: dùng khi k cố định. Variable size: mở rộng right, thu hẹp left khi vi phạm.

## ⚠️ Dễ nhầm
* Sliding window chỉ đúng khi thêm phần tử làm điều kiện "tệ đi đơn điệu" (không dùng được nếu có phần tử âm phá vỡ tính đơn điệu, ví dụ tổng có số âm).

## ✅ Checklist
- [ ] Phân biệt fixed vs variable window
- [ ] Biết dùng HashMap đếm tần suất trong window

## ❓ 5 câu hỏi phổ biến
1. Sliding Window áp dụng được khi nào (điều kiện gì)?
2. Vì sao độ phức tạp vẫn là O(n) dù có vòng while lồng trong for?
3. Minimum Window Substring giải thế nào?
4. Sliding Window Maximum dùng cấu trúc gì để đạt O(n)? (Monotonic Deque)
5. Khi mảng có số âm, sliding window tổng còn dùng được không?

---

# 20. Two Pointer

## 1. Khái niệm
Dùng 2 con trỏ di chuyển trên mảng/chuỗi (thường đã sort) để giảm độ phức tạp từ O(n²) xuống O(n).

## 2. Khi nào sử dụng
* Mảng đã sort, tìm cặp/bộ ba thỏa điều kiện tổng; so sánh từ 2 đầu.
* Không dùng nếu dữ liệu chưa sort mà thứ tự quan trọng (cần sort trước, tốn O(n log n)).

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Two Pointer (đã sort) | O(n) | O(1) |
| 3Sum (sort + two pointer) | O(n²) | O(1)–O(n) |

## 4. Cách cài đặt cơ bản
```java
// Two Sum trên mảng đã sort
int[] twoSumSorted(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) return new int[]{left, right};
        else if (sum < target) left++;
        else right--;
    }
    return new int[]{-1, -1};
}
```

## 5. Ưu điểm
* Giảm O(n²) → O(n) cho bài toán tìm cặp trên mảng sort.

## 6. Nhược điểm
* Cần mảng đã sort (tốn O(n log n) nếu chưa sort).

## 7. Lỗi thường gặp
* Quên bỏ qua **duplicate** trong 3Sum/4Sum → kết quả trùng lặp.
* Điều kiện dừng vòng lặp sai (`left < right` vs `left <= right`).

## 8. Câu hỏi phỏng vấn thường gặp
* **Two Pointer khác Sliding Window?** → Two Pointer: 2 con trỏ có thể di chuyển độc lập từ 2 đầu; Sliding Window: luôn duy trì đoạn liên tục [left,right] cùng hướng.
* **Vì sao cần sort trước khi Two Pointer?** → Để đảm bảo tính đơn điệu, quyết định di chuyển trái/phải đúng đắn.

## 9. LeetCode tiêu biểu
* Two Sum II - Input Array Is Sorted (167)
* 3Sum (15)
* Container With Most Water (11)
* Trapping Rain Water (42)
* Valid Palindrome (125)

---

## 🔑 Cheat Sheet Chương 19–20
* Mảng sort + tìm cặp/bộ tổng = target → Two Pointer.
* Subarray/substring liên tục + điều kiện đơn điệu → Sliding Window.

## ⚠️ Dễ nhầm
* Two Pointer đôi khi bị nhầm là Sliding Window – phân biệt: Two Pointer không nhất thiết duy trì "window" liên tục.

## ✅ Checklist
- [ ] Biết xử lý duplicate trong 3Sum
- [ ] Phân biệt rõ Two Pointer vs Sliding Window

## ❓ 5 câu hỏi phổ biến
1. Container With Most Water dùng chiến lược two pointer thế nào?
2. Làm sao bỏ qua duplicate hiệu quả trong 3Sum?
3. Two Pointer có áp dụng được cho Linked List không? (Có – slow/fast)
4. Trapping Rain Water giải bằng Two Pointer ra sao?
5. Khi nào Two Pointer không hiệu quả bằng HashMap?


# 21. Monotonic Stack

## 1. Khái niệm
Stack luôn duy trì thứ tự **tăng dần hoặc giảm dần** – khi phần tử mới phá vỡ thứ tự, pop các phần tử vi phạm trước khi push.

## 2. Khi nào sử dụng
* Bài toán "next greater/smaller element", tính diện tích lớn nhất histogram.
* Không dùng nếu bài toán không liên quan đến so sánh phần tử trước/sau gần nhất.

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Xử lý toàn mảng | O(n) (mỗi phần tử push/pop tối đa 1 lần) | O(n) |

## 4. Cách cài đặt cơ bản
```java
// Next Greater Element
int[] nextGreaterElement(int[] nums) {
    int[] res = new int[nums.length];
    Arrays.fill(res, -1);
    Deque<Integer> stack = new ArrayDeque<>(); // lưu index, giá trị giảm dần từ đáy
    for (int i = 0; i < nums.length; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            res[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return res;
}
```

## 5. Ưu điểm
* Giảm O(n²) (brute force so sánh từng cặp) xuống O(n).

## 6. Nhược điểm
* Chỉ giải quyết được dạng bài "phần tử liền kề lớn hơn/nhỏ hơn gần nhất", không tổng quát.

## 7. Lỗi thường gặp
* Nhầm điều kiện `<` và `<=` khi pop (ảnh hưởng xử lý phần tử bằng nhau).
* Lưu **giá trị** thay vì **index** trong stack (thường cần index để tính khoảng cách).

## 8. Câu hỏi phỏng vấn thường gặp
* **Monotonic Stack dùng để làm gì?** → Tìm phần tử lớn hơn/nhỏ hơn gần nhất bên trái/phải hiệu quả O(n).
* **Vì sao độ phức tạp là O(n) dù có while lồng trong for?** → Mỗi phần tử chỉ được push và pop tối đa 1 lần trong suốt quá trình.

## 9. LeetCode tiêu biểu
* Daily Temperatures (739)
* Next Greater Element I (496)
* Largest Rectangle in Histogram (84)
* Trapping Rain Water (42)
* Remove K Digits (402)

---

## 🔑 Cheat Sheet Chương 21
* Từ khóa "next greater/smaller", "largest rectangle" → Monotonic Stack.
* Luôn lưu **index** trong stack, không lưu giá trị.

## ⚠️ Dễ nhầm
* Monotonic Increasing Stack dùng cho "next smaller", Monotonic Decreasing Stack dùng cho "next greater" – dễ nhầm lẫn hướng.

## ✅ Checklist
- [ ] Nhớ template Monotonic Stack
- [ ] Biết phân biệt increasing/decreasing stack theo bài toán

## ❓ 5 câu hỏi phổ biến
1. Vì sao Monotonic Stack đạt O(n) tổng thể?
2. Daily Temperatures giải bằng Monotonic Stack thế nào?
3. Largest Rectangle in Histogram dùng stack ra sao?
4. Khi nào dùng Monotonic Stack thay vì Monotonic Deque?
5. Next Greater Element II (mảng vòng tròn) xử lý khác gì bản thường?

---

# 22. Bit Manipulation

## 1. Khái niệm
Thao tác trực tiếp trên các bit nhị phân bằng các phép: `& | ^ ~ << >> `.

## 2. Khi nào sử dụng
* Tối ưu không gian (bitmask thay set), kiểm tra tính chẵn/lẻ, tìm phần tử duy nhất, bật/tắt cờ trạng thái.
* Không dùng nếu làm giảm độ dễ đọc mà không cải thiện hiệu năng đáng kể.

## 3. Độ phức tạp

| Thao tác | Time | Space |
|---|---|---|
| Các phép bit cơ bản | O(1) | O(1) |
| Đếm số bit 1 (Brian Kernighan) | O(số bit 1) | O(1) |

## 4. Cách cài đặt cơ bản
```java
int a = 5, b = 3;
a & b;   // AND
a | b;   // OR
a ^ b;   // XOR
~a;      // NOT
a << 1;  // nhân 2
a >> 1;  // chia 2

boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}

// XOR: tìm phần tử duy nhất khi các phần tử khác xuất hiện 2 lần
int singleNumber(int[] nums) {
    int result = 0;
    for (int n : nums) result ^= n;
    return result;
}

// Đếm số bit 1
int countBits(int n) {
    int count = 0;
    while (n != 0) { n &= (n - 1); count++; } // xóa bit 1 thấp nhất
    return count;
}
```

## 5. Ưu điểm
* Cực nhanh O(1), tiết kiệm bộ nhớ (bitmask thay HashSet/boolean array).

## 6. Nhược điểm
* Code khó đọc, dễ gây lỗi logic nếu không quen thao tác bit.

## 7. Lỗi thường gặp
* Nhầm `^` (XOR) với `**` (không tồn tại trong Java) hoặc phép lũy thừa.
* Overflow khi shift quá số bit của kiểu dữ liệu (`int` 32-bit).
* Quên `a & (a-1)` xóa bit 1 thấp nhất — nhầm dấu.

## 8. Câu hỏi phỏng vấn thường gặp
* **XOR có tính chất gì hữu ích?** → `x ^ x = 0`, `x ^ 0 = x` → dùng tìm phần tử duy nhất/lẻ.
* **Kiểm tra n có phải lũy thừa của 2?** → `n > 0 && (n & (n-1)) == 0`.
* **Bitmask dùng để làm gì trong DP?** → Biểu diễn tập con (subset) làm state cho DP (Traveling Salesman Problem...).

## 9. LeetCode tiêu biểu
* Single Number (136)
* Number of 1 Bits (191)
* Counting Bits (338)
* Sum of Two Integers (371)
* Bitwise AND of Numbers Range (201)

---

## 🔑 Cheat Sheet Chương 22
* `n & (n-1)` → xóa bit 1 thấp nhất.
* `n & (-n)` → lấy bit 1 thấp nhất.
* `x ^ x = 0`, dùng để tìm phần tử lẻ/duy nhất.
* Bitmask → biểu diễn subset cho DP trạng thái.

## ⚠️ Dễ nhầm
* `>>` (arithmetic shift, giữ dấu) khác `>>>` (logical shift, không giữ dấu) trong Java.

## ✅ Checklist
- [ ] Thuộc các trick XOR/AND cơ bản
- [ ] Biết dùng bitmask làm DP state
- [ ] Phân biệt `>>` và `>>>`

## ❓ 5 câu hỏi phổ biến
1. XOR dùng để giải bài "single number" như thế nào?
2. Sự khác nhau giữa `>>` và `>>>` trong Java?
3. Làm sao đếm số bit 1 hiệu quả (Brian Kernighan)?
4. Bitmask DP áp dụng cho bài toán nào điển hình? (TSP)
5. Kiểm tra 2 số có cùng dấu bằng bit trick thế nào?

---

# 📋 Tổng Kết – Bảng Tra Cứu Nhanh Toàn Bộ

| Chủ đề | Time điển hình | Dùng khi |
|---|---|---|
| Array/String | O(n) | Truy cập nhanh, dữ liệu tuyến tính |
| Linked List | O(n), O(1) đầu | Thêm/xóa đầu thường xuyên |
| Stack | O(1)/op | LIFO, DFS, biểu thức |
| Queue/Deque | O(1)/op | FIFO, BFS, sliding window |
| HashMap/Set | O(1) avg | Tra cứu nhanh, đếm tần suất |
| Tree/BST | O(log n) | Dữ liệu phân cấp có thứ tự |
| Heap | O(log n) | Top-K, min/max động |
| Trie | O(L) | Prefix, autocomplete |
| Graph (BFS/DFS) | O(V+E) | Duyệt, kết nối |
| Sorting | O(n log n) | Chuẩn bị dữ liệu có thứ tự |
| Binary Search | O(log n) | Dữ liệu sort/đơn điệu |
| Backtracking | O(2ⁿ)/O(n!) | Tổ hợp, hoán vị, constraint |
| Greedy | O(n log n) | Chọn cục bộ tối ưu chứng minh được |
| DP | O(states×transition) | Subproblem chồng lấp |
| Union Find | O(α(n))≈O(1) | Kết nối động |
| Prefix Sum | O(1) query | Tổng đoạn tĩnh nhiều truy vấn |
| Sliding Window | O(n) | Subarray/substring liên tục |
| Two Pointer | O(n) | Mảng sort, tìm cặp |
| Monotonic Stack | O(n) | Next greater/smaller |
| Bit Manipulation | O(1) | Tối ưu không gian, cờ trạng thái |

## 🎯 Chiến lược nhận diện bài toán qua từ khóa

| Từ khóa trong đề bài | Kỹ thuật gợi ý |
|---|---|
| "top K", "K largest/smallest" | Heap |
| "prefix", "autocomplete" | Trie |
| "shortest path unweighted" | BFS |
| "shortest path weighted" | Dijkstra |
| "connected components", "merge" | Union Find |
| "subarray sum" | Prefix Sum / Sliding Window |
| "longest/shortest substring" | Sliding Window |
| "sorted array, pair sum" | Two Pointer |
| "next greater/smaller" | Monotonic Stack |
| "all permutations/subsets" | Backtracking |
| "minimize max / maximize min" | Binary Search on Answer |
| "number of ways", "min/max cost" | Dynamic Programming |
| "can we schedule/select optimally" | Greedy (cần chứng minh) |
