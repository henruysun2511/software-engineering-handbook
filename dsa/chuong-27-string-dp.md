# Chương 27: String DP

## 27.1. Khái niệm cốt lõi

### 27.1.1. Đặc điểm nhận diện

String DP là lớp bài toán 2D DP (Chương 26) chuyên biệt, trong đó **hai chiều của State** tương ứng với **hai chỉ số trên một hoặc hai chuỗi** — khác với Grid DP (hai chiều là tọa độ không gian) hay Knapsack (một chiều chỉ số vật phẩm, một chiều sức chứa). Đây là lớp bài toán cực kỳ phổ biến trong phỏng vấn vì kết hợp trực tiếp hai chủ đề đã học: String (Chương 2, đặc biệt là phân biệt Substring/Subsequence ở mục 2.2.3) và Dynamic Programming.

---

## 27.2. Longest Common Subsequence (LCS)

### 27.2.1. Bài toán

Cho hai chuỗi `text1` và `text2`, tìm độ dài dãy con chung dài nhất (Subsequence — không cần liên tiếp, theo đúng định nghĩa ở mục 2.2.3) giữa hai chuỗi.

### 27.2.2. Áp dụng khung 5 bước

**Bước 1 — Nhận diện DP:** brute force thử mọi cặp subsequence của hai chuỗi tốn O(2^m · 2^n) — hàm mũ kép, rõ ràng cần DP. Bài toán có Optimal Substructure: LCS của hai chuỗi đầy đủ được xây dựng từ LCS của các tiền tố ngắn hơn.

**Bước 2 — State:** `dp[i][j]` = độ dài LCS giữa `text1[0..i-1]` (i ký tự đầu) và `text2[0..j-1]` (j ký tự đầu). Quy ước dùng độ dài tiền tố (thay vì chỉ số trực tiếp) giúp tránh xử lý chỉ số âm ở Base Case.

**Bước 3 — Transition:** so sánh ký tự cuối cùng của hai tiền tố đang xét:
- **Nếu `text1[i-1] == text2[j-1]`:** ký tự này chắc chắn thuộc LCS (thuộc lời giải tối ưu — có thể chứng minh bằng exchange argument tương tự mục 23.1.3), nên `dp[i][j] = dp[i-1][j-1] + 1`.
- **Nếu khác nhau:** LCS không thể chứa **đồng thời cả hai** ký tự cuối này, nên phải bỏ qua ký tự cuối của **một trong hai** chuỗi — lấy giá trị tốt hơn giữa hai lựa chọn:
```
dp[i][j] = dp[i-1][j-1] + 1                      nếu text1[i-1] == text2[j-1]
dp[i][j] = max(dp[i-1][j], dp[i][j-1])            nếu text1[i-1] != text2[j-1]
```

**Bước 4 — Base Case:** `dp[0][j] = 0` và `dp[i][0] = 0` với mọi `i, j` — LCS với một chuỗi rỗng luôn bằng 0.

**Bước 5 — Answer:** `dp[m][n]` với `m, n` là độ dài hai chuỗi gốc.

**Minh họa bảng DP** với `text1 = "abcde"`, `text2 = "ace"`:

```
        ""  a   c   e
    ""   0  0   0   0
    a    0  1   1   1
    b    0  1   1   1
    c    0  1   2   2
    d    0  1   2   2
    e    0  1   2   3

dp[1][1]: text1[0]='a', text2[0]='a' → khớp → dp[0][0]+1 = 1
dp[3][2]: text1[2]='c', text2[1]='c' → khớp → dp[2][1]+1 = 1+1 = 2
dp[5][3]: text1[4]='e', text2[2]='e' → khớp → dp[4][2]+1 = 2+1 = 3
```

→ Kết quả: `3` (LCS = "ace").

### 27.2.3. Cài đặt C++

```cpp
#include <string>
#include <vector>
#include <algorithm>
using namespace std;

int longestCommonSubsequence(const string& text1, const string& text2) {
    int m = text1.size(), n = text2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1[i - 1] == text2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }

    return dp[m][n];
}
```

**Độ phức tạp:** O(m·n) thời gian, O(m·n) bộ nhớ phụ (tối ưu được xuống O(min(m,n)) bằng Rolling Array, mục 24.4, vì Transition chỉ phụ thuộc hàng liền trước) — so với brute force O(2^(m+n)).

---

## 27.3. Longest Palindromic Subsequence

### 27.3.1. Bài toán

Cho một chuỗi, tìm độ dài dãy con đối xứng (palindromic subsequence) dài nhất.

### 27.3.2. Bản chất — quy về LCS

**Quan sát quan trọng:** dãy con đối xứng dài nhất của chuỗi `s` chính là **LCS giữa `s` và chuỗi đảo ngược của `s`** — vì một dãy con vừa xuất hiện trong `s` theo đúng thứ tự, vừa xuất hiện trong `reverse(s)` theo đúng thứ tự, thì đó chính là một dãy con đọc xuôi và đọc ngược giống nhau, tức là palindrome.

```cpp
#include <string>
#include <algorithm>
using namespace std;

int longestPalindromeSubseq(const string& s) {
    string reversed = s;
    reverse(reversed.begin(), reversed.end());

    return longestCommonSubsequence(s, reversed); // tái sử dụng hàm ở mục 27.2.3
}
```

**Độ phức tạp:** O(n²) thời gian, O(n²) bộ nhớ phụ (áp dụng trực tiếp độ phức tạp LCS với `m = n` là độ dài chuỗi gốc) — minh chứng cụ thể cho giá trị của việc **nhận diện bài toán mới thực chất là bài toán cũ dưới một góc nhìn khác**, một kỹ năng quan trọng trong phỏng vấn.

---

## 27.4. Edit Distance (Khoảng cách chỉnh sửa)

### 27.4.1. Bài toán

Cho hai chuỗi `word1` và `word2`, tìm số thao tác **tối thiểu** (chèn một ký tự, xóa một ký tự, hoặc thay thế một ký tự) để biến `word1` thành `word2`.

### 27.4.2. Áp dụng khung 5 bước

**Bước 2 — State:** `dp[i][j]` = số thao tác tối thiểu để biến `word1[0..i-1]` thành `word2[0..j-1]`.

**Bước 3 — Transition:** xét ký tự cuối cùng của hai tiền tố đang xét:

- **Nếu `word1[i-1] == word2[j-1]`:** không cần thao tác gì cho cặp ký tự cuối này, kế thừa trực tiếp: `dp[i][j] = dp[i-1][j-1]`.
- **Nếu khác nhau:** phải thực hiện đúng một trong ba thao tác, chọn phương án tốn ít nhất:
  - **Thay thế** ký tự cuối của `word1` thành ký tự cuối của `word2`: `1 + dp[i-1][j-1]` (giải quyết xong phần còn lại của cả hai tiền tố ngắn hơn).
  - **Xóa** ký tự cuối của `word1`: `1 + dp[i-1][j]` (giờ chỉ cần biến `word1[0..i-2]` thành `word2[0..j-1]`).
  - **Chèn** thêm một ký tự vào `word1` để khớp ký tự cuối của `word2`: `1 + dp[i][j-1]` (giờ chỉ cần biến `word1[0..i-1]` thành `word2[0..j-2]`).

```
dp[i][j] = dp[i-1][j-1]                                              nếu word1[i-1] == word2[j-1]
dp[i][j] = 1 + min(dp[i-1][j-1], dp[i-1][j], dp[i][j-1])             nếu khác nhau
           (thay thế)           (xóa)        (chèn)
```

**Bước 4 — Base Case:** `dp[0][j] = j` (biến chuỗi rỗng thành `j` ký tự cần đúng `j` phép chèn); `dp[i][0] = i` (biến `i` ký tự thành chuỗi rỗng cần đúng `i` phép xóa).

**Bước 5 — Answer:** `dp[m][n]`.

**Minh họa bảng DP** với `word1 = "horse"`, `word2 = "ros"`:

```
         ""  r   o   s
    ""    0  1   2   3
    h     1  1   2   3
    o     2  2   1   2
    r     3  2   2   2
    s     4  3   3   2
    e     5  4   4   3

dp[2][2]: word1[1]='o', word2[1]='o' → khớp → dp[1][1] = 1
dp[3][1]: word1[2]='r', word2[0]='r' → khớp → dp[2][0] = 2
dp[5][3]: word1[4]='e', word2[2]='s' → khác → 1+min(dp[4][2]=3, dp[4][3]=2, dp[5][2]=4) = 1+2 = 3
```

→ Kết quả: `3` (horse → rorse (thay h→r) → rose (xóa r) → ros (xóa e), tổng 3 thao tác).

### 27.4.3. Cài đặt C++

```cpp
#include <string>
#include <vector>
#include <algorithm>
using namespace std;

int minDistance(const string& word1, const string& word2) {
    int m = word1.size(), n = word2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1));

    for (int i = 0; i <= m; i++) dp[i][0] = i; // Base Case: xóa hết i ký tự
    for (int j = 0; j <= n; j++) dp[0][j] = j; // Base Case: chèn đủ j ký tự

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1[i - 1] == word2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + min({dp[i - 1][j - 1], dp[i - 1][j], dp[i][j - 1]});
            }
        }
    }

    return dp[m][n];
}
```

**Độ phức tạp:** O(m·n) thời gian, O(m·n) bộ nhớ phụ (tối ưu được xuống O(min(m,n)) bằng Rolling Array).

---

## 27.5. Word Break

### 27.5.1. Bài toán

Cho chuỗi `s` và một từ điển `wordDict`, xác định `s` có thể được phân tách thành một dãy các từ liên tiếp đều thuộc từ điển hay không.

### 27.5.2. Bản chất — 1D DP trên chuỗi (không phải 2D)

**Lưu ý quan trọng:** dù thuộc phạm vi "String DP", bài toán này thực chất chỉ cần **State một chiều** (không phải hai chuỗi so sánh với nhau như LCS/Edit Distance) — đặt ở đây vì bản chất thao tác trên chuỗi và độ phức tạp liên quan đến độ dài từ điển, gần với tinh thần chương này hơn Chương 25.

- **State:** `dp[i]` = `s[0..i-1]` (i ký tự đầu) có thể phân tách hợp lệ hay không (kiểu boolean).
- **Transition:** `dp[i] = true` nếu tồn tại một điểm cắt `j < i` sao cho `dp[j] = true` **và** `s[j..i-1]` (đoạn còn lại) thuộc từ điển.
```
dp[i] = OR (dp[j] AND s[j..i-1] ∈ wordDict) với mọi j từ 0 đến i-1
```
- **Base Case:** `dp[0] = true` (chuỗi rỗng luôn "phân tách hợp lệ" — không cần phân tách gì).
- **Answer:** `dp[n]`.

### 27.5.3. Cài đặt C++

```cpp
#include <string>
#include <vector>
#include <unordered_set>
using namespace std;

bool wordBreak(const string& s, vector<string>& wordDict) {
    unordered_set<string> dict(wordDict.begin(), wordDict.end()); // tra cứu O(1), xem lại Chương 3
    int n = s.size();
    vector<bool> dp(n + 1, false);
    dp[0] = true;

    for (int i = 1; i <= n; i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && dict.count(s.substr(j, i - j))) {
                dp[i] = true;
                break; // đã tìm được một cách phân tách hợp lệ, không cần thử thêm j khác
            }
        }
    }

    return dp[n];
}
```

**Độ phức tạp:** O(n²) thời gian cho vòng lặp lồng (chưa tính chi phí `substr` và tra cứu HashSet, mỗi lần tối đa O(n) cho việc tạo chuỗi con — tổng O(n³) nếu tính đầy đủ, có thể tối ưu bằng String Hashing từ mục 2.2.5 để giảm chi phí so sánh xuống O(1) mỗi lần); O(n) bộ nhớ phụ cho `dp`.

*Ghi chú kết hợp Trie:* nếu từ điển rất lớn, có thể dùng Trie (Chương 16) để tối ưu việc kiểm tra `s[j..i-1]` có thuộc từ điển hay không, tránh chi phí hash lại nhiều lần cho các tiền tố chung.

---

## 27.6. Bảng tổng hợp

| Bài toán | State | Transition đặc trưng | Độ phức tạp |
|---|---|---|---|
| Longest Common Subsequence | dp[i][j]: LCS của 2 tiền tố | Khớp: +1 từ đường chéo; khác: max hai hướng | O(m·n) |
| Longest Palindromic Subsequence | Quy về LCS(s, reverse(s)) | — | O(n²) |
| Edit Distance | dp[i][j]: chi phí biến đổi 2 tiền tố | Khớp: kế thừa đường chéo; khác: 1 + min 3 hướng | O(m·n) |
| Word Break | dp[i]: i ký tự đầu phân tách được không | OR qua mọi điểm cắt hợp lệ | O(n²) đến O(n³) |

---

## 27.7. Danh sách bài tập luyện tập

### Mức Medium
1. Longest Common Subsequence
2. Longest Palindromic Subsequence
3. Edit Distance
4. Word Break
5. Delete Operation for Two Strings (biến thể Edit Distance chỉ cho phép xóa, không cho thay thế/chèn)
6. Distinct Subsequences (đếm SỐ CÁCH thay vì kiểm tra tồn tại)
7. Interleaving String

### Mức Hard
8. Longest Palindromic Substring (lưu ý: Substring — liên tiếp, khác bản chất với Subsequence ở mục 27.3, cần Transition khác biệt dựa trên tính đối xứng cục bộ)
9. Regular Expression Matching (String DP kết hợp xử lý ký tự đại diện `.` và `*`)
10. Wildcard Matching
11. Shortest Common Supersequence (mở rộng của LCS — dựng chuỗi ngắn nhất chứa cả hai chuỗi làm subsequence)

---

*Đến đây, tài liệu đã hoàn thành Part IX — Dynamic Programming (Chương 24-27), bao phủ khung tư duy nền tảng, các dạng State phổ biến nhất (1D, 2D Grid/Knapsack, String). Chương tiếp theo: **Chương 28 — Bit Manipulation**, chuyển sang nhóm kỹ thuật thao tác trực tiếp trên biểu diễn nhị phân của số nguyên.*
