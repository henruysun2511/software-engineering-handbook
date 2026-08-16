# Chương 5: Sliding Window (Cửa sổ trượt)

## 5.1. Khái niệm cốt lõi

### 5.1.1. Định nghĩa

Sliding Window là kỹ thuật duy trì một **đoạn con liên tục (window)** được xác định bởi hai con trỏ `left` và `right`, trong đó `right` mở rộng cửa sổ dần về phía trước và `left` thu hẹp cửa sổ khi cần, nhằm tìm đoạn con thỏa mãn một điều kiện cho trước mà không cần duyệt lại từ đầu cho mỗi vị trí bắt đầu. Đây là một trường hợp chuyên biệt của mô hình Same Direction Two Pointers (mục 4.1.3), áp dụng riêng cho lớp bài toán về **tính chất của đoạn con liên tiếp**.

### 5.1.2. Bản chất — vì sao Sliding Window loại bỏ được một bậc độ phức tạp

Xét bài toán "tìm đoạn con liên tục dài nhất thỏa mãn điều kiện X". Cách brute force duyệt mọi cặp `(left, right)` rồi kiểm tra điều kiện cho từng đoạn tốn O(n²) hoặc O(n³) (nếu kiểm tra điều kiện cũng tốn O(n)).

Sliding Window khai thác một tính chất quan trọng: khi **mở rộng cửa sổ về bên phải** (thêm phần tử `arr[right]`), ta có thể **cập nhật trạng thái của cửa sổ trong O(1)** (ví dụ cộng thêm vào tổng, tăng tần suất một ký tự) thay vì tính lại từ đầu. Tương tự, khi **thu hẹp cửa sổ từ bên trái** (bỏ phần tử `arr[left]`), trạng thái cũng chỉ cần cập nhật O(1). Vì mỗi phần tử chỉ được **thêm vào cửa sổ đúng một lần** và **bị loại khỏi cửa sổ đúng một lần** trong suốt quá trình chạy, tổng số thao tác trên toàn bộ thuật toán là O(n), dù nhìn bề ngoài có hai con trỏ di chuyển "lồng nhau".

**Minh họa trực quan** — cửa sổ trượt trên mảng tìm tổng lớn nhất của đoạn con độ dài cố định `k=3`:

```
arr:     [2, 1, 5, 1, 3, 2]

Bước 1:  [2, 1, 5] 1  3  2     sum = 8
              ↓ trượt sang phải: trừ arr[0]=2, cộng arr[3]=1
Bước 2:   2 [1, 5, 1] 3  2     sum = 8 - 2 + 1 = 7
              ↓
Bước 3:   2  1 [5, 1, 3] 2     sum = 7 - 1 + 3 = 9   ← lớn nhất
```

Mỗi bước trượt chỉ tốn O(1) (một phép trừ, một phép cộng) thay vì phải cộng lại toàn bộ 3 phần tử trong cửa sổ mới — đây chính là bản chất tiết kiệm của kỹ thuật.

### 5.1.3. Fixed Window và Variable Window

**Fixed Window (Cửa sổ kích thước cố định):** độ dài cửa sổ `k` được xác định trước và không đổi trong suốt quá trình duyệt. Cửa sổ luôn trượt đúng một bước mỗi lần: thêm một phần tử mới bên phải, đồng thời loại bỏ một phần tử cũ bên trái.

**Variable Window (Cửa sổ kích thước biến đổi):** độ dài cửa sổ co giãn tùy theo điều kiện bài toán. Khuôn mẫu chung:

```
for right in 0..n-1:
    mở rộng cửa sổ bằng cách thêm arr[right]
    while cửa sổ vi phạm điều kiện:
        thu hẹp cửa sổ bằng cách loại arr[left], tăng left
    cập nhật kết quả dựa trên cửa sổ hiện tại [left, right]
```

Điểm mấu chốt để khuôn mẫu này đạt O(n): con trỏ `left` **chỉ tăng, không bao giờ giảm** trong suốt vòng lặp — tổng số lần `left` tăng trên toàn bộ thuật toán bị chặn bởi `n`, nên dù có vòng `while` lồng bên trong vòng `for`, tổng chi phí vẫn tuyến tính (tương tự kỹ thuật amortized đã phân tích ở mục 0.1.5 và mục 3.3.5).

---

## 5.2. Kiến thức liên quan

### 5.2.1. Window kết hợp với cấu trúc dữ liệu phụ trợ

Bản thân Sliding Window chỉ quản lý **ranh giới** của đoạn con; để kiểm tra điều kiện của đoạn con một cách hiệu quả, cần kết hợp với cấu trúc lưu trạng thái:

- **Window + biến đơn (tổng, đếm):** khi điều kiện chỉ phụ thuộc một giá trị tổng hợp, ví dụ tổng các phần tử.
- **Window + HashMap/Frequency Array:** khi điều kiện phụ thuộc tần suất từng phần tử/ký tự riêng biệt trong cửa sổ (ví dụ: "cửa sổ chứa tối đa k ký tự phân biệt").
- **Window + Monotonic Queue:** khi cần truy vấn giá trị lớn nhất/nhỏ nhất trong cửa sổ hiệu quả hơn O(k) — kỹ thuật này sẽ được trình bày chi tiết ở chương Monotonic Stack/Queue.

### 5.2.2. Nhận diện bài toán Sliding Window

**Dấu hiệu nên nghĩ đến Sliding Window:**
- Đề bài yêu cầu tìm đoạn con liên tục (subarray/substring) **dài nhất**, **ngắn nhất**, hoặc **đếm số lượng** đoạn con thỏa mãn một điều kiện.
- Điều kiện của đoạn con có tính chất: khi cửa sổ càng lớn thì càng dễ vi phạm (hoặc càng dễ thỏa mãn) — tức có tính đơn điệu theo kích thước cửa sổ.
- Dữ liệu đầu vào **không chứa giá trị âm** (đối với bài toán tổng): nếu có số âm, việc mở rộng cửa sổ không đảm bảo tổng tăng đơn điệu, phá vỡ điều kiện tiên quyết của kỹ thuật (khi đó nên cân nhắc Prefix Sum + HashMap như mục 3.3.4).

**Khi nào KHÔNG dùng Sliding Window:** khi bài toán yêu cầu tìm đoạn con (không nhất thiết liên tục — tức subsequence, xem lại phân biệt ở mục 2.2.3), hoặc khi điều kiện đoạn con không đơn điệu theo kích thước cửa sổ.

---

## 5.3. Cài đặt các bài toán kinh điển

### 5.3.1. Maximum Sum Subarray of Size K (Fixed Window)

**Bài toán:** tìm tổng lớn nhất của một đoạn con liên tục có độ dài chính xác bằng `k`.

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int maxSumSubarraySizeK(const vector<int>& arr, int k) {
    int n = arr.size();
    int windowSum = 0;

    // Xây dựng cửa sổ đầu tiên
    for (int i = 0; i < k; i++) {
        windowSum += arr[i];
    }

    int maxSum = windowSum;

    // Trượt cửa sổ: mỗi bước loại phần tử trái, thêm phần tử phải
    for (int right = k; right < n; right++) {
        windowSum += arr[right] - arr[right - k];
        maxSum = max(maxSum, windowSum);
    }

    return maxSum;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — so với brute force O(n·k) tính lại tổng cho mỗi vị trí cửa sổ.

### 5.3.2. Longest Substring Without Repeating Characters (Variable Window)

**Bài toán:** tìm độ dài của substring dài nhất không chứa ký tự lặp lại.

**Bản chất:** mở rộng `right` liên tục; mỗi khi ký tự `s[right]` đã xuất hiện trong cửa sổ hiện tại, thu hẹp `left` cho đến khi loại bỏ được ký tự trùng đó. Dùng HashMap lưu **vị trí xuất hiện gần nhất** của mỗi ký tự để có thể nhảy `left` trực tiếp đến vị trí cần thiết thay vì tăng dần từng bước một, giúp tránh trường hợp xấu O(n²).

**Cài đặt C++:**

```cpp
#include <string>
#include <unordered_map>
#include <algorithm>
using namespace std;

int lengthOfLongestSubstring(const string& s) {
    unordered_map<char, int> lastSeen; // ký tự -> vị trí xuất hiện gần nhất
    int left = 0;
    int maxLength = 0;

    for (int right = 0; right < (int)s.size(); right++) {
        char c = s[right];

        // Nếu ký tự đã xuất hiện trong cửa sổ hiện tại, nhảy left ra khỏi vị trí đó
        if (lastSeen.count(c) && lastSeen[c] >= left) {
            left = lastSeen[c] + 1;
        }

        lastSeen[c] = right;
        maxLength = max(maxLength, right - left + 1);
    }

    return maxLength;
}
```

**Độ phức tạp:** O(n) thời gian (mỗi ký tự được xét O(1) khi cập nhật HashMap), O(min(n, |Σ|)) bộ nhớ với |Σ| là kích thước bảng ký tự.

### 5.3.3. Minimum Window Substring

**Bài toán:** cho hai chuỗi `s` và `t`, tìm substring ngắn nhất của `s` chứa tất cả ký tự của `t` (kể cả trùng lặp về số lượng).

**Bản chất:** đây là bài toán Variable Window điển hình với điều kiện "càng mở rộng càng dễ thỏa mãn, càng thu hẹp càng dễ vi phạm" — tính đơn điệu ngược lại so với bài toán 5.3.2. Dùng một mảng/HashMap đếm tần suất ký tự cần có từ `t`, và một biến đếm số ký tự **đã đủ số lượng yêu cầu** để kiểm tra điều kiện "cửa sổ hiện tại đã hợp lệ" trong O(1) thay vì phải so sánh toàn bộ bảng tần suất mỗi lần.

**Cài đặt C++:**

```cpp
#include <string>
#include <unordered_map>
#include <climits>
using namespace std;

string minWindow(const string& s, const string& t) {
    if (t.empty() || s.empty()) return "";

    unordered_map<char, int> need;   // tần suất ký tự cần có (từ t)
    for (char c : t) need[c]++;

    unordered_map<char, int> window; // tần suất ký tự hiện có trong cửa sổ
    int required = need.size();      // số ký tự PHÂN BIỆT cần đủ số lượng
    int formed = 0;                  // số ký tự phân biệt đã đủ số lượng trong cửa sổ

    int left = 0;
    int bestLen = INT_MAX, bestStart = 0;

    for (int right = 0; right < (int)s.size(); right++) {
        char c = s[right];
        window[c]++;

        if (need.count(c) && window[c] == need[c]) {
            formed++; // ký tự c vừa đạt đủ số lượng yêu cầu
        }

        // Khi cửa sổ đã hợp lệ (chứa đủ mọi ký tự cần), thử thu hẹp từ trái
        while (formed == required) {
            if (right - left + 1 < bestLen) {
                bestLen = right - left + 1;
                bestStart = left;
            }

            char leftChar = s[left];
            window[leftChar]--;
            if (need.count(leftChar) && window[leftChar] < need[leftChar]) {
                formed--; // loại leftChar khiến cửa sổ không còn hợp lệ
            }
            left++;
        }
    }

    return bestLen == INT_MAX ? "" : s.substr(bestStart, bestLen);
}
```

**Độ phức tạp:** O(|s| + |t|) thời gian — mỗi ký tự của `s` chỉ được thêm vào cửa sổ một lần (`right`) và loại khỏi cửa sổ tối đa một lần (`left`); O(|s| + |t|) bộ nhớ phụ cho hai bảng tần suất.

### 5.3.4. Longest Repeating Character Replacement

**Bài toán:** cho chuỗi `s` và số nguyên `k`, tìm độ dài substring dài nhất có thể biến thành chuỗi toàn ký tự giống nhau bằng cách thay đổi tối đa `k` ký tự.

**Bản chất:** một cửa sổ độ dài `windowLength` hợp lệ khi và chỉ khi:

```
windowLength - maxFreqInWindow ≤ k
```

trong đó `maxFreqInWindow` là tần suất của ký tự xuất hiện nhiều nhất trong cửa sổ (số ký tự cần thay = tổng số ký tự trong cửa sổ trừ đi số ký tự đã giống nhau nhiều nhất). Điểm tinh tế trong cài đặt: ta **không cần giảm** `maxFreq` một cách chính xác khi thu hẹp cửa sổ từ bên trái — vì mục tiêu là tìm độ dài **lớn nhất**, một giá trị `maxFreq` bị "trễ" (cũ hơn thực tế) chỉ khiến cửa sổ tạm thời không co lại đúng lúc, nhưng không bao giờ khiến thuật toán báo cáo sai một đáp án lớn hơn giá trị thực.

**Cài đặt C++:**

```cpp
#include <string>
#include <array>
#include <algorithm>
using namespace std;

int characterReplacement(const string& s, int k) {
    array<int, 26> freq{};
    int left = 0;
    int maxFreq = 0;      // tần suất ký tự xuất hiện nhiều nhất từng ghi nhận trong cửa sổ
    int result = 0;

    for (int right = 0; right < (int)s.size(); right++) {
        freq[s[right] - 'A']++;
        maxFreq = max(maxFreq, freq[s[right] - 'A']);

        int windowLength = right - left + 1;

        // Nếu số ký tự cần thay vượt quá k, thu hẹp cửa sổ
        if (windowLength - maxFreq > k) {
            freq[s[left] - 'A']--;
            left++;
        }

        result = max(result, right - left + 1);
    }

    return result;
}
```

**Độ phức tạp:** O(n) thời gian (n = độ dài chuỗi, 26 là hằng số không phụ thuộc n), O(1) bộ nhớ phụ.

### 5.3.5. Permutation in String

**Bài toán:** kiểm tra `s2` có chứa một hoán vị (permutation) nào của `s1` dưới dạng substring hay không.

**Bản chất:** một hoán vị của `s1` chỉ khác nhau về **thứ tự**, nên bài toán quy về: tồn tại một cửa sổ độ dài đúng bằng `|s1|` trong `s2` có **cùng tần suất ký tự** với `s1` hay không — đây là Fixed Window kết hợp so sánh tần suất, tương tự kiểm tra Anagram (mục 2.2.4) nhưng áp dụng liên tục trên từng cửa sổ trượt thay vì một lần so sánh cố định.

**Cài đặt C++:**

```cpp
#include <string>
#include <array>
using namespace std;

bool checkInclusion(const string& s1, const string& s2) {
    int n1 = s1.size(), n2 = s2.size();
    if (n1 > n2) return false;

    array<int, 26> need{}, window{};
    for (char c : s1) need[c - 'a']++;

    for (int i = 0; i < n1; i++) {
        window[s2[i] - 'a']++;
    }
    if (window == need) return true;

    // Trượt cửa sổ độ dài cố định n1 qua s2
    for (int right = n1; right < n2; right++) {
        window[s2[right] - 'a']++;
        window[s2[right - n1] - 'a']--;
        if (window == need) return true;
    }

    return false;
}
```

**Độ phức tạp:** O(n2) thời gian (so sánh hai mảng 26 phần tử là O(1) vì kích thước cố định), O(1) bộ nhớ phụ.

---

## 5.4. Bảng tổng hợp

| Bài toán | Loại Window | Cấu trúc phụ trợ | Độ phức tạp |
|---|---|---|---|
| Maximum Sum Subarray of Size K | Fixed | Biến tổng | O(n) |
| Longest Substring Without Repeating Characters | Variable | HashMap (vị trí gần nhất) | O(n) |
| Minimum Window Substring | Variable | HashMap tần suất kép | O(\|s\| + \|t\|) |
| Longest Repeating Character Replacement | Variable | Mảng tần suất cố định | O(n) |
| Permutation in String | Fixed | Mảng tần suất cố định | O(n) |

---

## 5.5. Danh sách bài tập luyện tập

### Mức Easy
1. Maximum Average Subarray I (Fixed Window)
2. Contains Duplicate II (Fixed Window + HashSet)

### Mức Medium
3. Longest Substring Without Repeating Characters
4. Longest Repeating Character Replacement
5. Permutation in String
6. Find All Anagrams in a String
7. Fruit Into Baskets (Variable Window + HashMap đếm tối đa 2 loại)
8. Max Consecutive Ones III
9. Subarrays with K Different Integers (kỹ thuật "đúng k = tối đa k − tối đa k-1")

### Mức Hard
10. Minimum Window Substring
11. Sliding Window Maximum (kết hợp Monotonic Queue — xem chương Monotonic Stack/Queue)
12. Substring with Concatenation of All Words

---

*Chương tiếp theo: **Chương 6 — Linked List**, chuyển sang cấu trúc dữ liệu dựa trên con trỏ, nơi kỹ thuật Fast & Slow Pointer (một biến thể khác của Two Pointers) được vận dụng để giải quyết các bài toán về chu trình và điểm giữa danh sách.*
