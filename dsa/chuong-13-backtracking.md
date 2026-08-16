# Chương 13: Backtracking (Quay lui)

## 13.1. Khái niệm cốt lõi

### 13.1.1. Định nghĩa

Backtracking là kỹ thuật khám phá **có hệ thống** toàn bộ không gian lời giải của một bài toán bằng cách xây dựng lời giải từng phần một (từng bước lựa chọn), và **quay lui (undo)** ngay khi phát hiện lựa chọn hiện tại không thể dẫn đến lời giải hợp lệ, để thử lựa chọn khác. Về bản chất, Backtracking là **DFS (Depth-First Search) trên cây quyết định (decision tree)**, kết hợp với thao tác hoàn tác trạng thái sau mỗi nhánh đã khám phá xong.

### 13.1.2. Bản chất — Cây quyết định (Decision Tree)

Mỗi bài toán Backtracking có thể hình dung thành một cây, trong đó mỗi node biểu diễn một trạng thái lựa chọn đã thực hiện một phần, và mỗi cạnh biểu diễn **một lựa chọn khả dĩ tiếp theo**. Thuật toán duyệt cây này theo DFS: đi sâu theo một nhánh cho đến khi đạt lời giải hoàn chỉnh hoặc xác định nhánh này không khả thi, sau đó **quay lại node cha** (backtrack) để thử nhánh khác.

**Minh họa Decision Tree** cho bài toán liệt kê tất cả tập con (subsets) của `{1, 2}` — tại mỗi phần tử, có hai lựa chọn: "chọn" hoặc "không chọn":

```
                          {}
                 chọn 1 /    \ không chọn 1
                  {1}              {}
            chọn 2/  \không    chọn 2/  \không
             {1,2}    {1}       {2}      {}

→ Duyệt DFS toàn bộ cây này thu được đúng 4 = 2² tập con: {1,2}, {1}, {2}, {}
```

### 13.1.3. Bản chất "Quay lui" — vì sao cần undo trạng thái

Điểm khác biệt cốt lõi giữa Backtracking và DFS/Recursion thông thường (Chương 12) nằm ở việc **trạng thái (state) được chia sẻ và tái sử dụng** giữa các nhánh thay vì tạo bản sao mới cho mỗi nhánh — vì lý do hiệu năng bộ nhớ. Khi đã khám phá xong một nhánh (một lựa chọn), thuật toán phải **hoàn tác chính xác** thay đổi đã thực hiện trên trạng thái dùng chung, để trạng thái đó "sạch" và đúng đắn khi thử nhánh tiếp theo — nguyên tắc này gọi là **"làm rồi hoàn tác" (do-undo)**.

**Khuôn mẫu chuẩn:**

```
void backtrack(trạng thái hiện tại, lựa chọn còn lại):
    if trạng thái hiện tại là lời giải hoàn chỉnh:
        ghi nhận kết quả
        return

    for mỗi lựa chọn khả dĩ tiếp theo:
        nếu lựa chọn này hợp lệ (thỏa ràng buộc):
            THỰC HIỆN lựa chọn (thay đổi trạng thái)
            backtrack(trạng thái mới, lựa chọn còn lại đã cập nhật)
            HOÀN TÁC lựa chọn (khôi phục trạng thái như trước khi thực hiện)
```

Bước "hoàn tác" chính là dấu hiệu nhận diện đặc trưng nhất của code Backtracking so với đệ quy thông thường.

### 13.1.4. Pruning (Cắt tỉa)

**Bản chất:** vì không gian lời giải của Backtracking thường tăng theo cấp số nhân (Tree Recursion, mục 12.1.3), việc khám phá toàn bộ cây quyết định mà không có chiến lược loại bỏ sớm các nhánh chắc chắn không dẫn đến lời giải hợp lệ sẽ khiến thuật toán chậm không cần thiết. **Pruning** là kỹ thuật kiểm tra ràng buộc **ngay khi thực hiện một lựa chọn**, thay vì đợi đến khi xây dựng xong toàn bộ lời giải mới kiểm tra — nếu ràng buộc đã bị vi phạm giữa chừng, dừng ngay nhánh đó (không tiếp tục đệ quy sâu hơn), tiết kiệm toàn bộ công sức khám phá phần cây quyết định phía dưới nhánh đó.

**Lưu ý quan trọng:** Pruning không thay đổi độ phức tạp **lý thuyết trong trường hợp xấu nhất** (vẫn là hàm mũ), nhưng cải thiện đáng kể hiệu năng **thực tế** bằng cách loại bỏ sớm phần lớn không gian tìm kiếm không khả thi — đây là lý do các bài toán như N-Queens vẫn giải được trong thời gian hợp lý với `n` vừa phải, dù độ phức tạp lý thuyết là hàm mũ.

---

## 13.2. So sánh Backtracking với các kỹ thuật liên quan

| Tiêu chí | Recursion thông thường | Backtracking | DFS trên đồ thị (chương Graph) |
|---|---|---|---|
| Mục tiêu | Tính một giá trị/kết quả | Liệt kê/đếm mọi lời giải khả dĩ | Khám phá/đánh dấu mọi node liên thông |
| Có "hoàn tác" trạng thái | Thường không cần | Bắt buộc | Thường không cần (dùng visited để không quay lại) |
| Độ phức tạp điển hình | Đa thức hoặc log | Hàm mũ (do liệt kê tổ hợp/hoán vị) | Đa thức (mỗi node/cạnh thăm hữu hạn lần) |

---

## 13.3. Cài đặt các bài toán kinh điển

### 13.3.1. Subsets (Tập con)

**Bài toán:** liệt kê toàn bộ tập con của một mảng (không chứa phần tử trùng lặp).

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

void backtrackSubsets(int start, vector<int>& current, const vector<int>& nums,
                       vector<vector<int>>& result) {
    result.push_back(current); // MỖI trạng thái dọc đường đi đều là một tập con hợp lệ

    for (int i = start; i < (int)nums.size(); i++) {
        current.push_back(nums[i]);              // THỰC HIỆN lựa chọn
        backtrackSubsets(i + 1, current, nums, result);
        current.pop_back();                       // HOÀN TÁC lựa chọn
    }
}

vector<vector<int>> subsets(vector<int>& nums) {
    vector<vector<int>> result;
    vector<int> current;
    backtrackSubsets(0, current, nums, result);
    return result;
}
```

**Giải thích tham số `start`:** dùng để đảm bảo mỗi tập con chỉ được sinh ra đúng một lần theo một thứ tự chọn phần tử duy nhất (chỉ chọn các phần tử từ vị trí `start` trở về sau), tránh sinh trùng lặp các tổ hợp giống nhau theo thứ tự khác nhau (ví dụ tránh sinh cả `{1,2}` lẫn `{2,1}` như hai kết quả khác nhau).

**Độ phức tạp:** O(2ⁿ · n) thời gian — có 2ⁿ tập con, mỗi tập con tốn O(n) để sao chép vào `result`; O(n) bộ nhớ phụ cho độ sâu đệ quy (không tính bộ nhớ lưu kết quả).

### 13.3.2. Permutations (Hoán vị)

**Bài toán:** liệt kê toàn bộ hoán vị của một mảng các phần tử phân biệt.

**Bản chất khác biệt so với Subsets:** mỗi lời giải hoàn chỉnh phải sử dụng **toàn bộ** phần tử (không phải một tập con), và thứ tự các phần tử **có ý nghĩa** (khác Subsets, nơi thứ tự không quan trọng) — vì vậy tại mỗi bước, ta được chọn bất kỳ phần tử nào **chưa dùng**, không giới hạn bắt đầu từ một vị trí `start` cố định như Subsets.

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

void backtrackPermute(vector<int>& current, vector<int>& nums, vector<bool>& used,
                       vector<vector<int>>& result) {
    if (current.size() == nums.size()) {
        result.push_back(current); // đã dùng đủ mọi phần tử, đây là một hoán vị hoàn chỉnh
        return;
    }

    for (int i = 0; i < (int)nums.size(); i++) {
        if (used[i]) continue; // Pruning: bỏ qua phần tử đã dùng trong nhánh hiện tại

        used[i] = true;
        current.push_back(nums[i]);            // THỰC HIỆN
        backtrackPermute(current, nums, used, result);
        current.pop_back();                     // HOÀN TÁC
        used[i] = false;                        // HOÀN TÁC trạng thái đã dùng
    }
}

vector<vector<int>> permute(vector<int>& nums) {
    vector<vector<int>> result;
    vector<int> current;
    vector<bool> used(nums.size(), false);
    backtrackPermute(current, nums, used, result);
    return result;
}
```

**Độ phức tạp:** O(n! · n) thời gian — có n! hoán vị, mỗi hoán vị tốn O(n) để sao chép; O(n) bộ nhớ phụ cho độ sâu đệ quy và mảng `used`.

### 13.3.3. Combination Sum

**Bài toán:** cho mảng số nguyên dương (không trùng lặp) và `target`, tìm mọi tổ hợp (có thể dùng lại phần tử không giới hạn số lần) có tổng bằng `target`.

**Bản chất Pruning:** vì các số đều dương, tổng hiện tại chỉ có thể **tăng** khi thêm phần tử — nếu tổng hiện tại đã vượt `target`, dừng nhánh ngay lập tức (không cần đệ quy tiếp) thay vì chờ đến khi xây xong tổ hợp mới kiểm tra.

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

void backtrackCombSum(int start, int remaining, vector<int>& current,
                       const vector<int>& candidates, vector<vector<int>>& result) {
    if (remaining == 0) {
        result.push_back(current);
        return;
    }
    if (remaining < 0) return; // Pruning: tổng đã vượt target, nhánh này vô nghĩa

    for (int i = start; i < (int)candidates.size(); i++) {
        current.push_back(candidates[i]);
        // Truyền lại i (không phải i+1): cho phép DÙNG LẠI candidates[i] không giới hạn
        backtrackCombSum(i, remaining - candidates[i], current, candidates, result);
        current.pop_back();
    }
}

vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
    vector<vector<int>> result;
    vector<int> current;
    backtrackCombSum(0, target, current, candidates, result);
    return result;
}
```

**Độ phức tạp:** phụ thuộc dữ liệu cụ thể, trường hợp xấu nhất theo cấp số mũ O(2^target) trong tình huống có nhiều candidate nhỏ; Pruning giúp giảm đáng kể số nhánh thực tế phải khám phá so với không kiểm tra `remaining < 0`.

### 13.3.4. Generate Parentheses

**Bài toán:** sinh mọi tổ hợp dấu ngoặc hợp lệ với `n` cặp ngoặc.

**Bản chất Pruning:** thay vì sinh mọi chuỗi độ dài `2n` rồi lọc ra chuỗi hợp lệ (rất lãng phí), ta chỉ thêm dấu `'('` khi còn dấu mở chưa dùng, và chỉ thêm dấu `')'` khi số dấu đóng hiện tại **nhỏ hơn** số dấu mở hiện tại — ràng buộc này được kiểm tra ngay tại mỗi bước, loại bỏ hoàn toàn khả năng sinh ra chuỗi không hợp lệ giữa chừng.

**Cài đặt C++:**

```cpp
#include <string>
#include <vector>
using namespace std;

void backtrackParens(string& current, int openCount, int closeCount, int n,
                      vector<string>& result) {
    if ((int)current.size() == 2 * n) {
        result.push_back(current);
        return;
    }

    if (openCount < n) {
        current.push_back('(');
        backtrackParens(current, openCount + 1, closeCount, n, result);
        current.pop_back();
    }

    if (closeCount < openCount) { // Pruning: chỉ đóng khi còn dấu mở chưa được đóng
        current.push_back(')');
        backtrackParens(current, openCount, closeCount + 1, n, result);
        current.pop_back();
    }
}

vector<string> generateParenthesis(int n) {
    vector<string> result;
    string current;
    backtrackParens(current, 0, 0, n, result);
    return result;
}
```

**Độ phức tạp:** O(4ⁿ / √n) thời gian (số Catalan thứ n, cận trên chặt của số lượng chuỗi hợp lệ), nhờ Pruning mà không mất thêm chi phí sinh và loại bỏ chuỗi không hợp lệ.

### 13.3.5. N-Queens

**Bài toán:** đặt `n` quân hậu trên bàn cờ `n×n` sao cho không có hai quân hậu nào tấn công nhau (cùng hàng, cùng cột, hoặc cùng đường chéo).

**Bản chất Pruning:** vì mỗi hàng chỉ được đặt đúng một quân hậu, ta duyệt tuần tự từng hàng, tại mỗi hàng chỉ thử các cột **chưa bị tấn công** bởi quân hậu đã đặt ở các hàng trước — kiểm tra ràng buộc ngay khi thử đặt (không đợi đặt xong toàn bộ `n` quân mới kiểm tra), giúp cắt bỏ sớm phần lớn nhánh không khả thi.

**Cài đặt C++:**

```cpp
#include <vector>
#include <string>
using namespace std;

bool isSafe(int row, int col, vector<int>& queenCols) {
    for (int prevRow = 0; prevRow < row; prevRow++) {
        int prevCol = queenCols[prevRow];
        // Kiểm tra cùng cột, hoặc cùng đường chéo (chênh lệch hàng == chênh lệch cột)
        if (prevCol == col || abs(prevRow - row) == abs(prevCol - col)) {
            return false;
        }
    }
    return true;
}

void backtrackQueens(int row, int n, vector<int>& queenCols, vector<vector<string>>& result) {
    if (row == n) {
        vector<string> board(n, string(n, '.'));
        for (int r = 0; r < n; r++) board[r][queenCols[r]] = 'Q';
        result.push_back(board);
        return;
    }

    for (int col = 0; col < n; col++) {
        if (isSafe(row, col, queenCols)) { // Pruning: chỉ thử vị trí an toàn
            queenCols[row] = col;                              // THỰC HIỆN
            backtrackQueens(row + 1, n, queenCols, result);
            // queenCols[row] sẽ bị ghi đè ở lần thử tiếp theo — không cần hoàn tác tường minh
        }
    }
}

vector<vector<string>> solveNQueens(int n) {
    vector<vector<string>> result;
    vector<int> queenCols(n, -1);
    backtrackQueens(0, n, queenCols, result);
    return result;
}
```

**Độ phức tạp:** O(n!) thời gian trong trường hợp xấu nhất (không có Pruning sẽ là O(nⁿ) — thử mọi cột cho mọi hàng độc lập), Pruning bằng `isSafe` giảm đáng kể số nhánh thực tế phải khám phá.

### 13.3.6. Word Search

**Bài toán:** cho lưới ký tự 2D và một từ, kiểm tra từ đó có thể tạo thành bằng cách nối các ô liền kề (ngang/dọc, không dùng lại một ô hai lần trong cùng một đường đi) hay không.

**Cài đặt C++:**

```cpp
#include <vector>
#include <string>
using namespace std;

bool backtrackSearch(vector<vector<char>>& board, const string& word,
                      int row, int col, int index) {
    if (index == (int)word.size()) return true; // đã khớp toàn bộ từ

    if (row < 0 || row >= (int)board.size() ||
        col < 0 || col >= (int)board[0].size() ||
        board[row][col] != word[index]) {
        return false; // Pruning: ra ngoài lưới hoặc ký tự không khớp
    }

    char temp = board[row][col];
    board[row][col] = '#'; // THỰC HIỆN: đánh dấu ô đã dùng trong đường đi hiện tại

    bool found = backtrackSearch(board, word, row + 1, col, index + 1) ||
                 backtrackSearch(board, word, row - 1, col, index + 1) ||
                 backtrackSearch(board, word, row, col + 1, index + 1) ||
                 backtrackSearch(board, word, row, col - 1, index + 1);

    board[row][col] = temp; // HOÀN TÁC: khôi phục ô để các đường đi khác có thể dùng lại

    return found;
}

bool exist(vector<vector<char>>& board, const string& word) {
    int rows = board.size(), cols = board[0].size();

    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            if (backtrackSearch(board, word, r, c, 0)) return true;
        }
    }
    return false;
}
```

**Độ phức tạp:** O(rows · cols · 4^L) thời gian với `L` là độ dài từ cần tìm (mỗi bước có tối đa 4 hướng đi), O(L) bộ nhớ phụ cho độ sâu đệ quy.

---

## 13.4. Bảng tổng hợp

| Bài toán | Đặc điểm ràng buộc | Pruning áp dụng | Độ phức tạp thời gian |
|---|---|---|---|
| Subsets | Không ràng buộc, mọi tập con hợp lệ | Tham số `start` tránh trùng lặp | O(2ⁿ · n) |
| Permutations | Dùng toàn bộ phần tử, mọi thứ tự | Mảng `used` tránh dùng lại | O(n! · n) |
| Combination Sum | Tổng bằng target | Dừng khi tổng vượt target | Hàm mũ, phụ thuộc dữ liệu |
| Generate Parentheses | Chuỗi ngoặc hợp lệ | Ràng buộc closeCount < openCount | O(4ⁿ / √n) |
| N-Queens | Không tấn công nhau | Hàm `isSafe` kiểm tra trước khi đặt | O(n!) trường hợp xấu nhất |
| Word Search | Khớp ký tự liên tiếp trên lưới | Kiểm tra biên và ký tự trước khi đệ quy | O(rows·cols·4^L) |

---

## 13.5. Khi nào dùng Backtracking

- Đề bài yêu cầu **liệt kê tất cả** lời giải khả dĩ (mọi tổ hợp, hoán vị, tập con) thỏa mãn một ràng buộc — dấu hiệu ngôn ngữ: "tìm tất cả", "liệt kê mọi", "có bao nhiêu cách".
- Bài toán có thể phân rã thành chuỗi các **quyết định tuần tự**, mỗi quyết định có một tập lựa chọn hữu hạn.
- Kích thước đầu vào đủ nhỏ để chấp nhận độ phức tạp hàm mũ (thường n ≤ 20-25 trong giới hạn thời gian phỏng vấn) — nếu `n` lớn, cần xem xét liệu bài toán có bản chất Dynamic Programming (đếm số lượng thay vì liệt kê chi tiết) thay vì Backtracking thuần túy.

---

## 13.6. Danh sách bài tập luyện tập

### Mức Easy
1. Binary Watch
2. Letter Case Permutation

### Mức Medium
3. Subsets
4. Subsets II (chứa phần tử trùng lặp — cần thêm bước Pruning bỏ qua trùng lặp)
5. Permutations
6. Permutations II (chứa phần tử trùng lặp)
7. Combinations
8. Combination Sum
9. Combination Sum II (mỗi phần tử chỉ dùng tối đa một lần)
10. Generate Parentheses
11. Letter Combinations of a Phone Number
12. Palindrome Partitioning
13. Word Search

### Mức Hard
14. N-Queens
15. N-Queens II (chỉ đếm số lượng lời giải, không cần liệt kê — gợi ý tối ưu bộ nhớ)
16. Sudoku Solver
17. Word Search II (kết hợp Trie — xem Chương 16)

---

*Chương tiếp theo: **Chương 14 — Binary Tree**, chuyển sang nhóm cấu trúc dữ liệu dạng cây, nơi Recursion (Chương 12) được vận dụng như phương pháp duyệt tự nhiên nhất.*
