# Chương 30: Problem Recognition (Nhận diện dạng bài)

## 30.1. Vai trò của chương này

### 30.1.1. Vì sao cần một chương riêng cho việc nhận diện

Trong suốt 29 chương trước, tài liệu đã trình bày từng kỹ thuật một cách độc lập và có hệ thống. Nhưng trong một buổi live coding interview thực tế, thử thách đầu tiên **không phải** là "có biết kỹ thuật này hay không", mà là "nhìn vào đề bài, biết ngay cần dùng kỹ thuật nào" — đây là kỹ năng **ánh xạ từ dấu hiệu đề bài sang kỹ thuật giải quyết**, thường được rèn qua số lượng lớn bài tập, nhưng có thể **tăng tốc đáng kể** bằng một hệ thống nhận diện tường minh. Chương này tổng hợp lại toàn bộ tài liệu thành các **quy tắc nhận diện** theo từ khóa và đặc điểm cấu trúc dữ liệu đầu vào của đề bài.

### 30.1.2. Cách sử dụng chương này hiệu quả

Khi đọc đề bài trong lúc phỏng vấn, hãy trả lời tuần tự ba câu hỏi: (1) **Cấu trúc dữ liệu đầu vào là gì?** (mảng, chuỗi, cây, đồ thị, khoảng...), (2) **Đề bài yêu cầu loại kết quả gì?** (một giá trị tối ưu, tồn tại hay không, liệt kê mọi khả năng, đếm số lượng...), (3) **Có ràng buộc đặc biệt nào về độ phức tạp hoặc kích thước đầu vào không?** (gợi ý độ phức tạp mục tiêu). Ba câu trả lời này, kết hợp với bảng nhận diện dưới đây, thường thu hẹp đáng kể phạm vi kỹ thuật cần cân nhắc.

---

## 30.2. Nhận diện theo từ khóa trong đề bài

| Từ khóa / Dấu hiệu | Kỹ thuật gợi ý | Chương tham chiếu | Giải thích ngắn gọn |
|---|---|---|---|
| "Hai số/phần tử có tổng bằng..." | HashMap (Complement Lookup) hoặc Two Pointers (nếu đã/có thể sắp xếp) | 3.3.2, 4.3.1 | Tra cứu bù trừ O(1) hoặc khai thác tính đơn điệu sau khi sắp xếp |
| "Subarray/Substring dài nhất/ngắn nhất thỏa điều kiện" | Sliding Window | Chương 5 | Đoạn con liên tục, điều kiện đơn điệu theo kích thước cửa sổ |
| "Tổng của một đoạn (range sum)" | Prefix Sum | 1.6.1, 29.2 | Tiền xử lý O(n), truy vấn O(1) |
| "Cộng dồn giá trị vào nhiều đoạn" | Difference Array | 1.6.2 | Cập nhật O(1) mỗi đoạn, khôi phục O(n) cuối cùng |
| "Phần tử lớn hơn/nhỏ hơn gần nhất tiếp theo" | Monotonic Stack | 7.2.3, 29.3 | Loại bỏ phần tử không còn khả năng là đáp án |
| "Giá trị lớn nhất/nhỏ nhất trong mọi cửa sổ trượt" | Monotonic Deque | 8.3.2 | Kết hợp Sliding Window và Monotonic Stack |
| "Top K", "K phần tử lớn/nhỏ nhất" | Heap kích thước K, hoặc Quick Select | 9.3.1, 29.5 | Tránh sắp xếp toàn bộ O(n log n) |
| "Đường đi ngắn nhất, không trọng số" | BFS | Chương 18 | Thăm theo từng lớp khoảng cách |
| "Đường đi ngắn nhất, có trọng số dương" | Dijkstra | Chương 22 | Greedy + Heap |
| "Tất cả các thành phần liên thông" | DFS/BFS hoặc Union-Find | 19.3.2, Chương 21 | Tùy đồ thị tĩnh hay có cạnh thêm dần |
| "Thứ tự thực hiện có phụ thuộc/tiên quyết" | Topological Sort | Chương 20 | Sắp xếp DAG theo ràng buộc |
| "Liệt kê tất cả / mọi tổ hợp / mọi cách" | Backtracking | Chương 13 | Khám phá toàn bộ cây quyết định |
| "Giá trị nhỏ nhất/lớn nhất sao cho điều kiện X" | Binary Search on Answer | 10.3 | Nếu điều kiện X đơn điệu theo giá trị cần tìm |
| "Mảng/chuỗi đã sắp xếp, tìm kiếm" | Binary Search | Chương 10 | Chia đôi không gian tìm kiếm |
| "Số cách để đạt được...", "giá trị tối ưu với ràng buộc chọn/không chọn" | Dynamic Programming | Chương 24-27 | Overlapping Subproblems + Optimal Substructure |
| "Chọn tối ưu tại mỗi bước mà không cần xem lại" | Greedy (cần chứng minh hoặc thử phản ví dụ trước) | Chương 23 | Chỉ đúng khi có Greedy Choice Property |
| "Tiền tố chung", "autocomplete", "từ điển" | Trie | Chương 16 | Truy vấn tiền tố O(L), không phụ thuộc số lượng từ |
| "Đối xứng", "palindrome" | Two Pointers hoặc DP (nếu là subsequence) | 2.2.2, 27.3 | Two Pointers cho kiểm tra; DP cho tối ưu hóa/đếm |
| "Nhị phân", "bit", "XOR" | Bit Manipulation | Chương 28 | Thao tác trực tiếp trên biểu diễn nhị phân |
| "n ≤ 20" đi kèm yêu cầu duyệt tổ hợp | Bitmask (Backtracking hoặc Bitmask DP) | 28.3.6 | Không gian 2ⁿ đủ nhỏ để duyệt trực tiếp |

---

## 30.3. Nhận diện theo cấu trúc dữ liệu đầu vào

### 30.3.1. Đầu vào là Array/String

Áp dụng thứ tự cân nhắc: (1) có thể giải bằng một lượt duyệt O(n) với HashMap/Two Pointers/Sliding Window hay không (Chương 3-5); (2) nếu có yêu cầu tối ưu hóa với ràng buộc chọn/không chọn, cân nhắc DP 1D (Chương 25); (3) nếu đầu vào có thể sắp xếp mà không ảnh hưởng đến yêu cầu bài toán, cân nhắc Sort trước (Chương 11).

### 30.3.2. Đầu vào là hai String

Gần như luôn dẫn đến **String DP** (Chương 27) nếu bài toán yêu cầu so sánh/biến đổi giữa hai chuỗi (LCS, Edit Distance) — dấu hiệu nhận biết: đề bài đề cập đến **cả hai chuỗi đồng thời** trong yêu cầu kết quả (không phải xử lý độc lập từng chuỗi).

### 30.3.3. Đầu vào là Linked List

Cân nhắc ngay kỹ thuật Fast & Slow Pointer (mục 6.3.2) nếu đề bài liên quan đến chu trình, điểm giữa, hoặc cần duyệt danh sách theo khoảng cách tương đối; cân nhắc Dummy Node (mục 6.3.1) nếu thao tác có thể ảnh hưởng đến chính node `head`.

### 30.3.4. Đầu vào là Binary Tree

Xác định dạng duyệt phù hợp trước (Chương 14): Preorder nếu cần xử lý node cha trước con; Postorder nếu cần thông tin từ con trước khi xử lý cha (chiều cao, đường kính); Level Order nếu cần xử lý theo từng tầng. Nếu là Binary Search Tree, luôn cân nhắc khai thác tính chất Inorder cho ra dãy sắp xếp (mục 15.2.4).

### 30.3.5. Đầu vào là Graph

Xác định trước: đồ thị có trọng số hay không (chọn giữa BFS và Dijkstra), có hướng hay vô hướng (ảnh hưởng cách phát hiện chu trình, mục 19.4.1), cạnh cố định hay thêm dần theo thời gian (ảnh hưởng chọn giữa DFS/BFS và Union-Find, mục 21.3).

### 30.3.6. Đầu vào là Interval (khoảng)

Gần như luôn cần **Sort trước** (mục 11.4) — xác định sắp xếp theo điểm bắt đầu hay điểm kết thúc tùy thuộc bài toán cụ thể (gộp khoảng → điểm bắt đầu; chọn tối đa khoảng không chồng lấn → điểm kết thúc, mục 23.1.3).

### 30.3.7. Đầu vào là Matrix/Grid

Nếu yêu cầu liên quan đến thành phần liên thông (đảo, vùng) → BFS/DFS trên grid (mục 18.3, 19.3.1). Nếu yêu cầu truy vấn tổng vùng nhiều lần → 2D Prefix Sum (mục 29.2). Nếu yêu cầu đường đi tối ưu có ràng buộc chỉ đi xuống/phải → Grid DP (mục 26.2).

---

## 30.4. Nhận diện theo ràng buộc độ phức tạp

Kích thước đầu vào tối đa thường là **gợi ý trực tiếp** cho độ phức tạp kỳ vọng và do đó gợi ý luôn kỹ thuật cần dùng — đây là kỹ năng đã đề cập ở mục 0.2.3 (Clarify) và nên áp dụng triệt để:

| Kích thước đầu vào (n) | Độ phức tạp kỳ vọng | Kỹ thuật thường phù hợp |
|---|---|---|
| n ≤ 10-12 | O(n!) hoặc O(2ⁿ · n) | Backtracking liệt kê hoán vị/tổ hợp đầy đủ |
| n ≤ 20-25 | O(2ⁿ) | Backtracking, Bitmask DP |
| n ≤ 500-1000 | O(n²) hoặc O(n² log n) | DP 2D, Brute force có kiểm soát, Floyd-Warshall |
| n ≤ 10⁵ - 10⁶ | O(n log n) | Sort + kỹ thuật đi kèm, Heap, Binary Search on Answer |
| n ≤ 10⁶ - 10⁸ | O(n) hoặc O(n α(n)) | Two Pointers, Sliding Window, Prefix Sum, Union-Find, HashMap |
| n rất lớn, chỉ cần vài truy vấn | O(log n) mỗi truy vấn | Binary Search, cấu trúc dữ liệu cây |

---

## 30.5. Sơ đồ quyết định tổng hợp

Khi đối diện một bài toán mới, có thể áp dụng trình tự suy luận sau:

```
1. Đọc kỹ đề, xác định cấu trúc dữ liệu đầu vào và loại kết quả yêu cầu
   (mục 30.1.2, kết hợp mục 0.2.3)
        ↓
2. Đối chiếu từ khóa đề bài với Bảng 30.2
        ↓
3. Đối chiếu cấu trúc dữ liệu đầu vào với mục 30.3
        ↓
4. Dùng kích thước đầu vào (mục 30.4) để xác nhận hoặc loại trừ kỹ thuật
   (ví dụ: nếu n ≤ 20, Backtracking thuần túy vẫn khả thi dù có kỹ thuật O(n) tồn tại;
    nếu n ≤ 10^6, Backtracking chắc chắn KHÔNG khả thi, cần tìm kỹ thuật tuyến tính/logarit)
        ↓
5. Nếu vẫn không xác định được: trình bày Brute Force trước (mục 0.2.4),
   quan sát điểm nghẽn hiệu năng để gợi ý hướng tối ưu
```

---

## 30.6. Bài tập tổng hợp — luyện phản xạ nhận diện

*(Không yêu cầu giải chi tiết — mục tiêu là xác định NHANH kỹ thuật phù hợp cho mỗi đề bài trong dưới 30 giây, dựa trên bảng nhận diện ở chương này, trước khi bắt tay vào code.)*

1. "Cho một mảng số nguyên, tìm độ dài dãy con liên tiếp dài nhất có tổng chia hết cho k." → *(Gợi ý: Prefix Sum + HashMap, tương tự mục 3.3.4)*
2. "Cho ma trận nhị phân, đếm số vùng đất liên thông có diện tích lớn nhất." → *(Gợi ý: BFS/DFS trên Grid, mục 18.3/19.3)*
3. "Cho danh sách công việc với thời gian bắt đầu/kết thúc, tìm số công việc tối đa có thể hoàn thành." → *(Gợi ý: Sort + Greedy theo điểm kết thúc, mục 23.1.3)*
4. "Cho hai chuỗi, tìm số ký tự tối thiểu cần thêm vào để một chuỗi trở thành palindrome." → *(Gợi ý: quy về Longest Palindromic Subsequence, mục 27.3)*
5. "Cho danh sách các cặp quan hệ 'bạn bè', xác định hai người có quen nhau qua trung gian hay không, với các cặp quan hệ được thêm dần." → *(Gợi ý: Union-Find, mục 21.1.2)*
6. "Cho một mảng, tìm giá trị nhỏ nhất của phần tử lớn nhất trong K nhóm con liên tiếp chia từ mảng." → *(Gợi ý: Binary Search on Answer, mục 10.3.1)*
7. "Cho một cây nhị phân, tìm tổng đường đi lớn nhất giữa hai node bất kỳ." → *(Gợi ý: Postorder DFS cập nhật kết quả tại mọi node, mục 14.3.5)*

---

*Chương tiếp theo: **Chương 31 — Coding Templates**, tổng hợp toàn bộ khuôn mẫu code sẵn sàng sử dụng cho mỗi kỹ thuật đã học, phục vụ việc ôn tập nhanh trong những ngày cuối trước buổi phỏng vấn.*
