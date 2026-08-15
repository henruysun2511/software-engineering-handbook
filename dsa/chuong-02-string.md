# Chương 2: String (Chuỗi ký tự)

## 2.1. Khái niệm cốt lõi

### 2.1.1. Định nghĩa

String (chuỗi) là một dãy các ký tự (character) được sắp xếp liên tiếp nhau, về bản chất là một **trường hợp đặc biệt của Array** với kiểu phần tử là `char`. Vì vậy, mọi tính chất bộ nhớ và độ phức tạp đã trình bày ở Chương 1 (truy cập O(1), duyệt O(n), chèn/xóa giữa O(n)...) đều áp dụng trực tiếp cho string. Điều khiến String trở thành một chương riêng biệt là tập hợp các **bài toán và kỹ thuật đặc thù** phát sinh khi làm việc với ký tự và ngữ nghĩa văn bản.

### 2.1.2. Biểu diễn ký tự — ASCII và Unicode

Mỗi ký tự trong máy tính thực chất là một con số nguyên được diễn giải theo một bảng mã. **ASCII** (American Standard Code for Information Interchange) dùng 7-8 bit để biểu diễn 128-256 ký tự, đủ cho chữ cái Latin, chữ số và ký tự đặc biệt thông dụng (ví dụ `'a'` = 97, `'A'` = 65, `'0'` = 48). Chính vì ký tự là số nguyên, các phép toán số học trên `char` là hợp lệ và thường được tận dụng, ví dụ:

```cpp
char c = 'd';
int index = c - 'a';   // = 3, ánh xạ ký tự sang chỉ số 0-25 trong bảng chữ cái
```

**Unicode** mở rộng phạm vi biểu diễn lên hàng triệu ký tự để hỗ trợ đa ngôn ngữ (bao gồm tiếng Việt có dấu), thường được mã hóa bằng UTF-8 (độ dài biến đổi 1-4 byte mỗi ký tự). Trong phạm vi luyện tập thuật toán, hầu hết bài tập chỉ thao tác trên tập ký tự ASCII (chữ cái thường/hoa, chữ số), nhưng cần lưu ý rằng khi xử lý chuỗi có dấu tiếng Việt, một "ký tự" hiển thị có thể chiếm nhiều byte — duyệt theo byte (`char`) sẽ sai khác so với duyệt theo ký tự thực (code point).

### 2.1.3. Mutable và Immutable String

Đây là một khác biệt quan trọng giữa các ngôn ngữ lập trình, ảnh hưởng trực tiếp đến độ phức tạp của các thao tác:

- Trong **C++**, `std::string` là **mutable** (có thể thay đổi tại chỗ): `s[i] = 'x'` hợp lệ, và các thao tác như `s += "abc"` có thể tận dụng cơ chế cấp phát động tương tự `std::vector` (amortized O(1) khi nối thêm ở cuối, xem lại mục 1.1.4).
- Trong **Java** và **Python**, `String`/`str` là **immutable**: mỗi lần "sửa đổi" chuỗi thực chất tạo ra một chuỗi mới, khiến việc nối chuỗi trong vòng lặp có thể tốn O(n²) nếu không dùng cấu trúc trung gian (`StringBuilder` trong Java, `list` + `join` trong Python).

Vì tài liệu này dùng C++, ta có lợi thế thao tác trực tiếp trên `std::string` như một mảng ký tự mà không lo ngại chi phí ẩn của tính bất biến.

**Ghi chú về Small String Optimization (SSO):** cài đặt `std::string` hiện đại (libstdc++, libc++) áp dụng tối ưu hóa cho chuỗi ngắn (thường dưới 15-22 ký tự tùy trình biên dịch) bằng cách lưu trực tiếp trên stack thay vì cấp phát heap, giúp tránh chi phí cấp phát động cho phần lớn chuỗi nhỏ thường gặp trong bài tập.

---

## 2.2. Kỹ thuật xử lý chuỗi

### 2.2.1. Character Frequency (Tần suất ký tự)

**Bản chất:** Vì tập ký tự thường bị giới hạn (26 chữ cái thường, hoặc 256 ký tự ASCII), ta có thể đếm tần suất xuất hiện của từng ký tự bằng một **mảng cố định** thay vì HashMap, giúp giảm hằng số chi phí (không cần tính hash, không có overhead của bảng băm).

```cpp
array<int, 26> freq{};
for (char c : s) {
    freq[c - 'a']++;
}
```

Đây là nền tảng cho hàng loạt bài toán: kiểm tra Anagram, tìm ký tự xuất hiện nhiều nhất, Sliding Window đếm ký tự (xem Chương 5).

### 2.2.2. Palindrome (Chuỗi đối xứng)

**Bản chất:** Một chuỗi là palindrome nếu đọc từ trái sang phải giống hệt đọc từ phải sang trái. Cách kiểm tra tối ưu nhất là dùng **Two Pointers** xuất phát từ hai đầu chuỗi, tiến dần vào giữa, so sánh từng cặp ký tự đối xứng — không cần tạo chuỗi đảo ngược (tránh tốn thêm O(n) bộ nhớ).

**Minh họa** với `s = "racecar"`:

```
r  a  c  e  c  a  r
↑                 ↑     so sánh s[0] và s[6]: 'r' == 'r' ✓
   ↑           ↑        so sánh s[1] và s[5]: 'a' == 'a' ✓
      ↑     ↑           so sánh s[2] và s[4]: 'c' == 'c' ✓
         ↑              left == right, dừng lại → palindrome
```

**Cài đặt C++:**

```cpp
bool isPalindrome(const string& s) {
    int left = 0, right = (int)s.size() - 1;
    while (left < right) {
        if (s[left] != s[right]) return false;
        left++;
        right--;
    }
    return true;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ.

### 2.2.3. Substring và Subsequence — phân biệt bản chất

Đây là hai khái niệm thường bị nhầm lẫn nhưng có bản chất khác nhau hoàn toàn:

- **Substring (chuỗi con liên tiếp):** một đoạn ký tự **liên tục** trích từ chuỗi gốc, giữ nguyên thứ tự và vị trí liền kề.
- **Subsequence (dãy con):** một tập ký tự được **chọn lọc** từ chuỗi gốc theo đúng thứ tự xuất hiện, nhưng **không nhất thiết liên tiếp** — có thể bỏ qua một số ký tự ở giữa.

**Minh họa** với chuỗi gốc `"abcde"`:

| Loại | Ví dụ hợp lệ | Ví dụ không hợp lệ | Lý do không hợp lệ |
|---|---|---|---|
| Substring | `"bcd"` | `"bd"` | Bỏ qua ký tự `'c'` ở giữa → không liên tiếp |
| Subsequence | `"bd"` | `"db"` | Sai thứ tự xuất hiện gốc |

**Số lượng:** một chuỗi độ dài `n` có tối đa `n(n+1)/2` substring khác vị trí, nhưng có tới `2^n` subsequence (mỗi ký tự có 2 lựa chọn: chọn hoặc không chọn) — đây là lý do các bài toán subsequence thường liên quan đến Dynamic Programming (xem Chương 9) để tránh duyệt vét cạn theo cấp số mũ.

### 2.2.4. Anagram

**Bản chất:** Hai chuỗi là anagram của nhau nếu chúng chứa **cùng một tập ký tự với cùng tần suất**, chỉ khác thứ tự sắp xếp. Có hai cách kiểm tra:

1. **Sắp xếp cả hai chuỗi** rồi so sánh — O(n log n).
2. **Đếm tần suất ký tự** bằng mảng cố định rồi so sánh hai mảng tần suất — O(n), tối ưu hơn cách 1.

**Cài đặt C++ (cách đếm tần suất, tối ưu):**

```cpp
#include <string>
#include <array>
using namespace std;

bool isAnagram(const string& s, const string& t) {
    if (s.size() != t.size()) return false;

    array<int, 26> freq{};
    for (char c : s) freq[c - 'a']++;
    for (char c : t) freq[c - 'a']--;

    for (int count : freq) {
        if (count != 0) return false;
    }
    return true;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ (mảng 26 phần tử là hằng số, không phụ thuộc độ dài chuỗi).

### 2.2.5. String Hashing (Băm chuỗi)

**Bản chất:** String Hashing ánh xạ một chuỗi thành một số nguyên (hash value) sao cho hai chuỗi giống nhau luôn cho cùng một hash, và (với xác suất rất cao) hai chuỗi khác nhau cho hash khác nhau. Mục tiêu là biến việc **so sánh hai chuỗi O(n)** thành việc **so sánh hai số nguyên O(1)**.

Kỹ thuật phổ biến nhất là **Polynomial Rolling Hash**, coi chuỗi như một số viết trong hệ cơ số `p` (base), lấy phần dư theo một số nguyên tố lớn `m` để tránh tràn số và giảm va chạm (collision):

```
hash(s) = (s[0]·p^(n-1) + s[1]·p^(n-2) + ... + s[n-1]·p^0) mod m
```

**Điểm mạnh cốt lõi — tính "rolling":** khi cần tính hash của mọi substring độ dài `k` trong một chuỗi (sliding window trên chuỗi), ta không cần tính lại từ đầu mỗi lần (sẽ tốn O(n·k)), mà có thể tính **hash tiền tố (prefix hash)** một lần O(n), sau đó suy ra hash của bất kỳ substring nào trong O(1), tương tự bản chất của Prefix Sum ở mục 1.6.1 nhưng áp dụng cho phép nhân thay vì phép cộng.

**Cài đặt C++ (Prefix Hash với một base và một modulo):**

```cpp
#include <string>
#include <vector>
using namespace std;

class StringHasher {
private:
    vector<long long> prefixHash;
    vector<long long> powP;
    static const long long BASE = 131;
    static const long long MOD = 1'000'000'007;

public:
    explicit StringHasher(const string& s) {
        int n = s.size();
        prefixHash.assign(n + 1, 0);
        powP.assign(n + 1, 1);

        for (int i = 0; i < n; i++) {
            prefixHash[i + 1] = (prefixHash[i] * BASE + s[i]) % MOD;
            powP[i + 1] = (powP[i] * BASE) % MOD;
        }
    }

    // Trả về hash của substring s[l..r] (0-based, bao gồm cả hai đầu): O(1)
    long long getHash(int l, int r) const {
        long long result = prefixHash[r + 1]
                          - (prefixHash[l] * powP[r - l + 1]) % MOD;
        result = (result % MOD + MOD) % MOD; // đảm bảo không âm
        return result;
    }
};
```

**Độ phức tạp:** Xây dựng O(n), mỗi truy vấn hash một substring O(1).

**Khi nào dùng:** so sánh nhanh nhiều substring, kiểm tra chuỗi lặp lại (repeated substring), bài toán Rabin-Karp (tìm chuỗi con trong văn bản), hoặc khi cần dùng chuỗi làm khóa trong HashMap mà không muốn chi phí so sánh trực tiếp từng ký tự. **Lưu ý:** hash luôn có xác suất va chạm (dù rất nhỏ) — trong bài toán yêu cầu độ chính xác tuyệt đối, có thể dùng đồng thời hai cặp `(base, mod)` khác nhau (double hashing) để giảm xác suất va chạm gần như về 0.

---

## 2.3. So sánh các khái niệm

### 2.3.1. Substring vs Subsequence vs Subarray

| Khái niệm | Áp dụng cho | Tính liên tục | Số lượng tối đa (chuỗi độ dài n) |
|---|---|---|---|
| Substring | String | Bắt buộc liên tiếp | n(n+1)/2 |
| Subarray | Array | Bắt buộc liên tiếp | n(n+1)/2 |
| Subsequence | String hoặc Array | Không cần liên tiếp | 2^n |

*(Subarray và Substring là cùng một khái niệm, chỉ khác tên gọi tùy vào kiểu dữ liệu chứa nó là Array hay String.)*

### 2.3.2. So sánh chuỗi: theo từng ký tự vs theo hash

| Tiêu chí | So sánh trực tiếp (`==`) | So sánh bằng Hash |
|---|---|---|
| Độ phức tạp một lần so sánh | O(n) | O(1) sau khi đã tính hash |
| Độ chính xác | Tuyệt đối | Có xác suất va chạm (rất nhỏ) |
| Chi phí tiền xử lý | Không cần | O(n) xây dựng prefix hash |
| Khi nào dùng | So sánh ít lần, chuỗi ngắn | So sánh nhiều lần, chuỗi dài, cần tốc độ cao |

---

## 2.4. Cài đặt các bài toán kinh điển

### 2.4.1. Valid Palindrome (bỏ qua ký tự không phải chữ/số)

**Bài toán:** Kiểm tra một chuỗi có phải palindrome hay không, chỉ xét chữ cái và chữ số, bỏ qua khoảng trắng/ký tự đặc biệt, không phân biệt hoa thường.

```cpp
#include <string>
#include <cctype>
using namespace std;

bool isValidPalindrome(const string& s) {
    int left = 0, right = (int)s.size() - 1;

    while (left < right) {
        // Bỏ qua ký tự không phải chữ/số
        while (left < right && !isalnum(s[left])) left++;
        while (left < right && !isalnum(s[right])) right--;

        if (tolower(s[left]) != tolower(s[right])) return false;
        left++;
        right--;
    }
    return true;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ.

### 2.4.2. Longest Common Prefix (Tiền tố chung dài nhất)

**Bài toán:** Cho một mảng chuỗi, tìm tiền tố chung dài nhất của tất cả chuỗi.

**Bản chất:** Lấy chuỗi đầu tiên làm chuẩn so sánh, sau đó rút ngắn dần tiền tố mỗi khi phát hiện nó không còn là tiền tố của chuỗi tiếp theo — cách này tránh phải so sánh ký tự-đối-ký tự trên toàn bộ mảng nhiều lần không cần thiết.

```cpp
#include <vector>
#include <string>
using namespace std;

string longestCommonPrefix(const vector<string>& strs) {
    if (strs.empty()) return "";

    string prefix = strs[0];

    for (int i = 1; i < (int)strs.size(); i++) {
        // Rút ngắn prefix cho đến khi nó thực sự là tiền tố của strs[i]
        while (strs[i].find(prefix) != 0) {
            prefix.pop_back();
            if (prefix.empty()) return "";
        }
    }

    return prefix;
}
```

**Độ phức tạp:** O(S) với S là tổng độ dài tất cả chuỗi trong trường hợp xấu nhất, O(1) bộ nhớ phụ ngoài kết quả.

### 2.4.3. Group Anagrams

**Bài toán:** Cho một mảng chuỗi, nhóm các chuỗi là anagram của nhau vào cùng một nhóm.

**Bản chất:** Hai chuỗi là anagram khi và chỉ khi chúng có cùng dạng chuẩn hóa sau khi sắp xếp ký tự. Dùng chuỗi đã sắp xếp làm **khóa (key)** trong HashMap, gom các chuỗi cùng khóa vào chung một nhóm.

```cpp
#include <vector>
#include <string>
#include <unordered_map>
#include <algorithm>
using namespace std;

vector<vector<string>> groupAnagrams(vector<string>& strs) {
    unordered_map<string, vector<string>> groups;

    for (const string& s : strs) {
        string key = s;
        sort(key.begin(), key.end()); // chuẩn hóa: "eat" -> "aet"
        groups[key].push_back(s);
    }

    vector<vector<string>> result;
    result.reserve(groups.size());
    for (auto& [key, group] : groups) {
        result.push_back(move(group));
    }
    return result;
}
```

**Độ phức tạp:** O(n · k log k) với `n` là số chuỗi, `k` là độ dài trung bình mỗi chuỗi (do sắp xếp từng chuỗi); O(n · k) bộ nhớ.

### 2.4.4. String Compression (Nén chuỗi tại chỗ)

**Bài toán:** Nén một chuỗi bằng cách thay thế các ký tự lặp liên tiếp bằng ký tự và số lần lặp (ví dụ `"aabcccccaaa"` → `"a2b1c5a3"`), thực hiện tại chỗ (in-place) trên mảng ký tự, trả về độ dài mới.

**Bản chất:** Dùng hai con trỏ — một con trỏ `read` duyệt qua chuỗi gốc để đếm số lần lặp của từng ký tự, một con trỏ `write` ghi kết quả nén trực tiếp đè lên mảng gốc (vì kết quả nén luôn ngắn hơn hoặc bằng độ dài gốc khi có ký tự lặp).

```cpp
#include <vector>
#include <string>
using namespace std;

int compress(vector<char>& chars) {
    int write = 0;
    int read = 0;
    int n = chars.size();

    while (read < n) {
        char currentChar = chars[read];
        int count = 0;

        // Đếm số lần lặp liên tiếp của currentChar
        while (read < n && chars[read] == currentChar) {
            read++;
            count++;
        }

        chars[write++] = currentChar;
        if (count > 1) {
            string countStr = to_string(count);
            for (char digit : countStr) {
                chars[write++] = digit;
            }
        }
    }

    return write; // độ dài mới của chuỗi đã nén
}
```

**Độ phức tạp:** O(n) thời gian (mỗi ký tự chỉ được `read` một lần), O(1) bộ nhớ phụ ngoài kết quả ghi đè tại chỗ.

---

## 2.5. Bảng tổng hợp độ phức tạp

| Kỹ thuật / Bài toán | Độ phức tạp thời gian | Bộ nhớ phụ |
|---|---|---|
| Character Frequency | O(n) | O(1) — bảng 26/256 phần tử |
| Palindrome Check (Two Pointers) | O(n) | O(1) |
| Anagram Check | O(n) | O(1) |
| String Hashing — xây dựng | O(n) | O(n) |
| String Hashing — mỗi truy vấn | O(1) | — |
| Longest Common Prefix | O(S), S = tổng độ dài | O(1) |
| Group Anagrams | O(n · k log k) | O(n · k) |
| String Compression | O(n) | O(1) |

---

## 2.6. Danh sách bài tập luyện tập

### Mức Easy
1. Valid Anagram
2. Valid Palindrome
3. Reverse String
4. Longest Common Prefix
5. Implement strStr() (tìm vị trí xuất hiện substring)
6. Ransom Note (đếm tần suất ký tự)
7. First Unique Character in a String

### Mức Medium
8. Group Anagrams
9. Longest Substring Without Repeating Characters (Sliding Window — xem Chương 5)
10. String Compression
11. Longest Palindromic Substring
12. Encode and Decode Strings
13. Zigzag Conversion
14. Repeated Substring Pattern (ứng dụng String Hashing)

### Mức Hard
15. Minimum Window Substring (Sliding Window — xem Chương 5)
16. Longest Palindromic Subsequence (Dynamic Programming — xem Chương 9)
17. Regular Expression Matching
18. Text Justification

---

*Chương tiếp theo: **Chương 3 — Hash Table**, đi sâu vào cơ chế băm, cách xử lý va chạm, và các pattern tra cứu O(1) áp dụng cho Array/String đã học ở hai chương trước.*
