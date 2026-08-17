# Chương 32: Ôn tập tổng hợp và Lộ trình học

## 32.1. Tổng kết cấu trúc tài liệu

Tài liệu đã đi qua 32 chương, có thể nhóm lại thành 8 khối kiến thức lớn:

| Khối | Chương | Nội dung cốt lõi |
|---|---|---|
| Nền tảng | 0 | Độ phức tạp thuật toán, tư duy giải bài phỏng vấn |
| Cấu trúc dữ liệu tuyến tính | 1-3, 6-9 | Array, String, Hash Table, Linked List, Stack, Queue, Heap |
| Kỹ thuật trên mảng/chuỗi | 4-5, 10-11 | Two Pointers, Sliding Window, Binary Search, Sorting |
| Đệ quy và khám phá | 12-13 | Recursion, Backtracking |
| Cấu trúc cây | 14-16 | Binary Tree, BST, Trie |
| Đồ thị | 17-22 | Biểu diễn đồ thị, BFS, DFS, Topological Sort, Union-Find, Dijkstra |
| Tối ưu hóa | 23-28 | Greedy, Dynamic Programming (4 chương), Bit Manipulation |
| Tổng hợp | 29-32 | Advanced Patterns, Problem Recognition, Templates, Lộ trình học |

---

## 32.2. Mười nguyên lý xuyên suốt cần nhớ

Nhìn lại toàn bộ tài liệu, một số nguyên lý bản chất lặp lại xuyên suốt nhiều chương, đáng được khắc sâu hơn là ghi nhớ từng thuật toán riêng lẻ:

1. **Đánh đổi giữa cấu trúc liên tục và cấu trúc liên kết** (mục 1.1.2 vs 6.1.2): Array cho truy cập O(1) nhưng chèn/xóa giữa O(n); Linked List ngược lại — nguyên lý này lặp lại ở khắp nơi khi lựa chọn cấu trúc dữ liệu nền cho một cấu trúc phức tạp hơn (Stack, Queue).

2. **Amortized Analysis giải thích vì sao "trông có vẻ chậm nhưng thực ra nhanh"** (mục 0.1.5): từ `push_back` (1.1.4), Path Compression trong DSU (21.1.5), đến Sliding Window (5.1.2) — luôn quay lại việc chứng minh tổng chi phí qua nhiều thao tác, không nhìn từng thao tác đơn lẻ.

3. **Tiền xử lý đổi lấy truy vấn nhanh:** Prefix Sum (1.6.1), String Hashing (2.2.5), Sparse Table ẩn trong 2D Prefix Sum (29.2) — mẫu hình chung: tốn O(n) hoặc O(n log n) một lần, đổi lấy O(1) hoặc O(log n) mỗi truy vấn sau đó.

4. **Hai con trỏ luôn cần một lý do đơn điệu để loại bỏ khả năng:** Two Pointers (4.1.2), Sliding Window (5.1.2), Monotonic Stack (7.2.3) — nếu không chứng minh được tính đơn điệu, các kỹ thuật này áp dụng sai sẽ cho kết quả sai mà không báo lỗi.

5. **Overlapping Subproblems là ranh giới giữa đệ quy đơn thuần và DP:** từ Fibonacci (12.1.3) đến toàn bộ Chương 24-27 — luôn tự hỏi "bài toán con có bị tính lặp lại không?" trước khi quyết định cần Memoization.

6. **Chỉ một lựa chọn (Greedy) vs mọi lựa chọn (DP):** mục 23.4.1 — sự khác biệt cốt lõi giữa hai kỹ thuật tối ưu hóa, và Coin Change (25.5) là minh chứng cụ thể cho ranh giới này.

7. **BFS/DFS là hai mặt của cùng một bài toán khám phá, khác nhau ở cấu trúc dữ liệu nền (Queue vs Stack):** mục 19.1.2 — hiểu rõ điều này giúp không cần học thuộc hai thuật toán riêng biệt mà chỉ cần hiểu một nguyên lý chung áp dụng hai cách.

8. **Cây là đồ thị đặc biệt, đồ thị là cây mở rộng:** mục 17.1.1 — Binary Tree Traversal (14.2) chính là DFS/BFS thu hẹp; hiểu Tree trước giúp Graph (Chương 17-22) trở nên tự nhiên hơn.

9. **Mọi cấu trúc "tự cân bằng" đều nhằm tránh trường hợp suy biến O(n):** BST cân bằng (15.1.2), Union by Rank trong DSU (21.1.5) — nỗi sợ chung là cấu trúc cây ngầm bị kéo dài thành một chuỗi.

10. **Độ phức tạp không gian thường bị bỏ quên nhưng luôn đáng giá:** Rolling Array (24.4) áp dụng lặp lại ở 1D DP, 2D DP, Knapsack — luôn tự hỏi "Transition có thực sự cần toàn bộ lịch sử, hay chỉ cần vài giá trị gần nhất?"

---

## 32.3. Lộ trình ôn tập theo thời gian còn lại

### 32.3.1. Còn 1 tuần

Tập trung tuyệt đối vào nhóm **Core** — các kỹ thuật gần như chắc chắn xuất hiện trong bất kỳ buổi live coding interview nào:

```
Ngày 1: Chương 0 (Nền tảng) + Chương 1-3 (Array, String, Hash Table)
Ngày 2: Chương 4-5 (Two Pointers, Sliding Window)
Ngày 3: Chương 6-9 (Linked List, Stack, Queue, Heap) — đọc nhanh, tập trung code mẫu
Ngày 4: Chương 12-13 (Recursion, Backtracking) + Chương 14-15 (Binary Tree, BST)
Ngày 5: Chương 18-19 (BFS, DFS) — trọng tâm Graph cơ bản
Ngày 6: Chương 24-26 (DP Fundamentals, 1D DP, 2D DP) — trọng tâm House Robber, Coin Change, LCS
Ngày 7: Chương 30 (Problem Recognition) ôn lại toàn bộ + làm 3-5 bài Mock Interview theo khung UMPIRE (mục 0.2.2)
```

### 32.3.2. Còn 2-3 tuần

Bổ sung thêm nhóm **Should-know**, xen kẽ luyện tập:

```
Tuần 1: Chương 0-9 (Nền tảng + toàn bộ cấu trúc dữ liệu tuyến tính)
        — mỗi chương đọc xong, giải ngay 3-5 bài trong danh sách bài tập
Tuần 2: Chương 10-16 (Binary Search, Sorting, Recursion, Backtracking, Tree, BST, Trie)
        + Chương 17-22 (toàn bộ Graph)
Tuần 3: Chương 23-28 (Greedy, DP đầy đủ 4 chương, Bit Manipulation)
        + Chương 29-31 (Advanced Patterns, Problem Recognition, Templates)
        + Dành 2 ngày cuối cho Mock Interview đầy đủ, mô phỏng đúng thời gian thực (mục 0.2.9)
```

### 32.3.3. Còn hơn 1 tháng

Có thể học tuần tự đúng theo thứ tự chương (0 → 32), mỗi chương dành 1-2 ngày: đọc kỹ phần bản chất, tự tay chạy lại các đoạn minh họa từng bước bằng tay (không chỉ đọc code), giải toàn bộ bài tập Easy và Medium trong danh sách, chỉ làm Hard nếu còn thời gian dư. Dành tuần cuối cùng để ôn tập theo lộ trình 1 tuần ở mục 32.3.1, xem như một vòng "tổng duyệt" trước ngày phỏng vấn thật.

---

## 32.4. Checklist trước ngày phỏng vấn

- [ ] Đã ôn lại khung UMPIRE (mục 0.2.2) và có thể áp dụng tự nhiên không cần nhìn tài liệu.
- [ ] Đã tự giải ít nhất 2-3 bài Medium hoàn toàn mới (chưa từng thấy) trong 45 phút, có nói to suy nghĩ.
- [ ] Đã ôn lại bảng nhận diện dạng bài (Chương 30) — có thể nhìn đề bài lạ và đưa ra hướng tiếp cận trong dưới 2 phút.
- [ ] Đã ôn lại các template code (Chương 31) — có thể viết đúng cú pháp C++ cho từng khuôn mẫu mà không cần tra cứu.
- [ ] Đã chuẩn bị sẵn vài câu hỏi clarify mẫu (mục 0.2.3) để không bị lúng túng khi đề bài mơ hồ.
- [ ] Đã ôn lại cách phân tích độ phức tạp nhanh (mục 0.1.4) — nhìn code là nói ngay được Big O.
- [ ] Đã chuẩn bị tâm lý cho việc bị bug giữa buổi — nhớ lại quy trình debug có hệ thống (mục 0.2.7).
- [ ] Đã nghỉ ngơi đầy đủ trước ngày phỏng vấn — tư duy rõ ràng quan trọng hơn việc cố nhồi nhét thêm kiến thức vào phút chót.

---

## 32.5. Lời kết

Live coding interview không đánh giá việc thuộc lòng thuật toán, mà đánh giá **khả năng tư duy có hệ thống dưới áp lực thời gian**. Tài liệu này được xây dựng theo triết lý xuyên suốt: mỗi kỹ thuật đều đi kèm giải thích **bản chất** (vì sao nó hoạt động, không chỉ nó hoạt động như thế nào), vì hiểu bản chất là nền tảng duy nhất giúp ứng viên **tự tin ứng biến** khi gặp một biến thể bài toán chưa từng thấy — điều gần như chắc chắn sẽ xảy ra trong bất kỳ buổi phỏng vấn thực tế nào.

Chúc bạn ôn tập hiệu quả và tự tin bước vào buổi phỏng vấn.

---

*— Hết tài liệu —*
