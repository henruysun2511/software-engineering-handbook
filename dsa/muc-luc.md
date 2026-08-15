# Mục lục DSA — Ôn tập Live Coding Interview

**Chú thích tier:**
- 🔴 **Core** — bắt buộc thành thạo, code trơn tru không cần suy nghĩ nhiều
- 🟡 **Should-know** — nắm chắc ý tưởng và code được khi cần, không cần thuộc lòng từng dòng

---

## PART 0 — Nền tảng & Kỹ năng phỏng vấn

### 0.1 Độ phức tạp thuật toán 🔴
- Big O, Big Θ, Big Ω
- Phân tích time/space complexity
- Amortized analysis (VD: dynamic array resize)

### 0.2 Tư duy giải bài trong phỏng vấn 🔴
- Clarify đề bài: edge case, input constraints, input size (để đoán độ phức tạp yêu cầu)
- Nêu brute force trước, sau đó tối ưu dần
- Think aloud — nói to suy nghĩ khi code
- Tự tạo test case trước khi chạy/submit
- Debug trực tiếp khi bị bug, không hoảng
- Phân tích lại complexity sau khi code xong
- Quản lý thời gian (thường 30–45 phút/bài)
- Mock interview — luyện với người khác hoặc tự quay lại xem

---

## PART I — Array, String & Hash Table

### 1. Array 🔴
- Static vs Dynamic Array, indexing, traversal, insert/delete/update
- Patterns: Prefix Sum, Suffix Sum, Difference Array, Frequency Array, In-place Modification
- Bài kinh điển: Kadane's Algorithm (Maximum Subarray), Product Except Self, Rotate Array, Move Zeroes, Merge Arrays

### 2. String 🔴
- Character frequency, string hashing, palindrome, substring vs subsequence, anagram
- Bài kinh điển: Valid Anagram, Valid Palindrome, Longest Common Prefix, Group Anagrams, String Compression

### 3. Hash Table (HashMap / HashSet) 🔴
- Hash function, collision (khái niệm, không cần cài lại)
- Patterns: Frequency Counting, Complement Lookup, Deduplication, Prefix Sum + HashMap
- Bài kinh điển: Two Sum, Contains Duplicate, Longest Consecutive Sequence, Subarray Sum Equals K, Group Anagrams

---

## PART II — Two Pointers & Sliding Window 🔴

### 4. Two Pointers
- Opposite direction (left/right) vs same direction (slow/fast)
- Ứng dụng: sorted array, pair sum, remove duplicates, partition, palindrome
- Bài kinh điển: Two Sum II, 3Sum, Container With Most Water, Trapping Rain Water

### 5. Sliding Window
- Fixed window vs Variable window
- Window + HashMap / Frequency / Monotonic Queue
- Bài kinh điển: Longest Substring Without Repeating Characters, Minimum Window Substring, Longest Repeating Character Replacement, Permutation in String
- **Nhận diện bài:** khi nào dùng sliding window, khi nào không — luyện phản xạ nhận keyword "longest/shortest subarray/substring thỏa điều kiện"

---

## PART III — Linked List, Stack, Queue & Heap

### 6. Linked List 🔴
- Singly vs Doubly Linked List
- Two Pointers: Fast & Slow Pointer, Dummy Node
- Reverse Linked List (iterative & recursive)
- Bài kinh điển: Reverse Linked List, Detect Cycle, Find Middle Node, Merge Two Sorted Lists, Remove N-th Node From End

### 7. Stack 🔴
- LIFO, push/pop/peek
- Ứng dụng: Parentheses Matching, Expression Evaluation, Monotonic Stack
- Bài kinh điển: Valid Parentheses, Min Stack, Next Greater Element, Daily Temperatures

### 8. Queue & Deque 🟡
- FIFO, Circular Queue, Deque
- Ứng dụng: BFS, Sliding Window
- Bài kinh điển: Sliding Window Maximum

### 9. Heap / Priority Queue 🟡
- Min-heap, Max-heap, heapify, complexity
- Ứng dụng: Top K, K-th Largest/Smallest, Merge K Sorted Lists, Median Maintenance
- Bài kinh điển: Kth Largest Element, Top K Frequent Elements, Task Scheduler, Find Median from Data Stream

---

## PART IV — Binary Search & Sorting

### 10. Binary Search 🔴
- Search boundaries: first/last position, lower/upper bound
- Binary Search on Answer (tìm kiếm trên không gian đáp án)
- Search in Rotated Sorted Array
- Bài kinh điển: Search Insert Position, Search Rotated Sorted Array, Find Peak Element, Koko Eating Bananas
- **Template:** Standard, Lower Bound, Binary Search on Answer

### 11. Sorting (áp dụng, không cần tự cài) 🟡
- Dùng hàm sort() built-in là chính; hiểu Stable vs Unstable để biết khi nào ảnh hưởng kết quả
- Interview pattern: Sort + Two Pointers, Sort + Greedy, Custom Comparator

---

## PART V — Recursion & Backtracking

### 12. Recursion 🔴
- Base case, recursive case, call stack, complexity
- Linear recursion, tree recursion, divide and conquer

### 13. Backtracking 🔴
- Decision tree, template chung, pruning
- Pattern: Subsets, Permutations, Combinations, Combination Sum, Parentheses Generation
- Bài kinh điển: N-Queens, Word Search, Letter Combinations

---

## PART VI — Trees

### 14. Binary Tree 🔴
- Traversal: Preorder, Inorder, Postorder, Level Order (recursive & iterative)
- Height, depth, subtree
- Bài kinh điển: Maximum Depth, Diameter, Same Tree, Invert Binary Tree, Path Sum, Lowest Common Ancestor

### 15. Binary Search Tree 🔴
- Properties, search, insert, delete, inorder traversal
- Bài kinh điển: Validate BST, Kth Smallest Element, Lowest Common Ancestor in BST

### 16. Trie 🟡
- Insert, search, prefix search — hay gặp trong bài autocomplete/word

---

## PART VII — Graph

### 17. Graph Fundamentals 🔴
- Terminology: vertex, edge, degree, path, cycle, connected component
- Directed/undirected, weighted/unweighted, cyclic/acyclic
- Biểu diễn: Adjacency List (ưu tiên dùng), Adjacency Matrix, Edge List

### 18. BFS 🔴
- Queue-based BFS trên graph và grid
- Multi-source BFS
- Bài kinh điển: Number of Islands, Rotting Oranges, Word Ladder, Flood Fill, Shortest Path in Unweighted Graph

### 19. DFS 🔴
- Recursive & Iterative DFS trên graph và grid
- Connected Components, Cycle Detection

### 20. Topological Sort 🟡
- Kahn's Algorithm, DFS-based Topological Sort
- Bài kinh điển: Course Schedule, Course Schedule II

### 21. Union-Find (DSU) 🟡
- Parent array, Find, Union, Path Compression, Union by Rank/Size
- Bài kinh điển: Number of Connected Components, Redundant Connection

### 22. Dijkstra (Shortest Path, positive weights) 🟡
- Ý tưởng, khi nào dùng thay vì BFS thường (đồ thị có trọng số dương)
- Bảng chọn nhanh:

| Bài toán | Thuật toán |
|---|---|
| Unweighted Graph | BFS |
| Positive Weights | Dijkstra |

---

## PART VIII — Greedy 🟡

### 23. Greedy Algorithm
- Local optimal → global optimal, cách nhận diện bài Greedy
- Sorting + Greedy, Interval Greedy
- Bài kinh điển: Jump Game, Gas Station, Merge Intervals, Non-overlapping Intervals, Meeting Rooms
- **Greedy vs DP:** cách phân biệt khi nào dùng cái nào, cách tìm counterexample để chứng minh Greedy sai

---

## PART IX — Dynamic Programming

### 24. DP Fundamentals 🔴
- Overlapping subproblems, optimal substructure
- State, transition, base case
- Top-down (memoization) vs Bottom-up (tabulation)

### 25. 1D DP 🔴
- Climbing Stairs, House Robber, Maximum Subarray, Coin Change, Decode Ways

### 26. 2D DP 🔴
- Grid DP: Unique Paths, Minimum Path Sum
- Knapsack: 0/1 Knapsack, Unbounded Knapsack, Subset Sum

### 27. String DP 🟡
- Longest Common Subsequence, Longest Palindromic Subsequence, Edit Distance, Word Break

---

## PART X — Bit Manipulation 🟡

### 28. Bitwise Fundamentals & Patterns
- AND, OR, XOR, NOT, Shift
- Check odd/even, check power of 2, XOR trick, set/clear/toggle bit, count set bits, bitmask
- Bài kinh điển: Single Number, Missing Number, Counting Bits, Subsets using Bitmask

---

## PART XI — Advanced Patterns (tổng hợp bổ sung)

### 29. Prefix Sum / Difference Array 🔴
- 1D/2D Prefix Sum, Range Sum Query, Range Update

### 30. Monotonic Stack & Monotonic Queue 🟡
- Next/Previous Greater/Smaller Element, Sliding Window Maximum/Minimum

### 31. Interval Problems 🟡
- Merge Intervals, Insert Interval, Meeting Rooms I & II, Interval Scheduling

### 32. Top K Problems 🟡
- Heap, Quick Select, Bucket Sort

---

## PART XII — Problem Recognition (Pattern Matching theo đề bài) 🔴

Đây là phần quan trọng nhất để phản xạ nhanh trong 45 phút:

| Thấy keyword trong đề | → Nghĩ đến |
|---|---|
| "Pair / Two Numbers" | HashMap / Two Pointers |
| "Longest / Shortest Subarray/Substring" | Sliding Window / Prefix Sum |
| "Range Sum" | Prefix Sum |
| "Next Greater/Smaller" | Monotonic Stack |
| "Top K" | Heap / Quick Select |
| "Shortest Path" | BFS / Dijkstra |
| "Connected Components" | DFS / BFS / DSU |
| "Dependencies / Order" | Topological Sort |
| "All Possibilities / Combinations" | Backtracking |
| "Optimal value + Choices" | DP / Greedy |
| "Sorted array + Search" | Binary Search |
| "Prefix matching" | Trie / Prefix Sum |

---

## PART XIII — Coding Templates (cheat sheet, dùng lúc ôn nhanh trước buổi phỏng vấn)

1. Binary Search Template (standard + lower bound + on-answer)
2. Two Pointers Template
3. Sliding Window Template
4. BFS Template
5. DFS Template
6. Backtracking Template
7. Heap Template
8. Topological Sort Template
9. Union-Find Template
10. Dijkstra Template
11. 1D DP Template
12. 2D DP Template

---

## Gợi ý dùng tài liệu này

- Nếu chỉ còn **1–2 tuần**: tập trung 100% vào 🔴 Core (Part 0, I, II, III mục 6–7, IV mục 10, V, VI mục 14–15, VII mục 17–19, IX mục 24–26, XI mục 29, XII)
- Nếu còn **3–4 tuần**: thêm 🟡 Should-know
