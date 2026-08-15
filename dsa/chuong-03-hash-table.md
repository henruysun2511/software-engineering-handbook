# Chương 3: Hash Table (Bảng băm)

## 3.1. Khái niệm cốt lõi

### 3.1.1. Định nghĩa

Hash Table là cấu trúc dữ liệu lưu trữ các cặp `(khóa, giá trị)` (key-value) và cho phép **tra cứu, chèn, xóa** với độ phức tạp trung bình **O(1)** — không phụ thuộc vào số lượng phần tử đang lưu trữ. Đây là bước nhảy vọt so với Array: nếu Array cho O(1) khi biết chính xác **chỉ số**, thì Hash Table cho O(1) khi biết **khóa** bất kỳ (số nguyên, chuỗi, hay bất kỳ kiểu dữ liệu nào có thể băm được).

### 3.1.2. Bản chất — vì sao tra cứu là O(1) trung bình

Cốt lõi của Hash Table là một mảng cố định gồm nhiều **bucket** (ngăn chứa), kết hợp với một **hàm băm (hash function)** ánh xạ mỗi khóa thành một chỉ số bucket:

```
bucket_index = hashFunction(key) mod capacity
```

Vì hàm băm tính toán trực tiếp từ khóa (không cần duyệt qua các phần tử khác), việc xác định "khóa này nên nằm ở bucket nào" luôn mất O(1) — độ phức tạp của toàn bộ thao tác tra cứu do đó cũng xấp xỉ O(1), miễn là mỗi bucket chỉ chứa rất ít phần tử.

**Minh họa** với hàm băm đơn giản `hash(key) = key mod 7` trên bảng có 7 bucket, chèn các khóa `{10, 22, 31, 4}`:

```
10 mod 7 = 3        22 mod 7 = 1        31 mod 7 = 3        4 mod 7 = 4

Bucket:  0     1      2     3           4     5     6
        [ ]  [22]   [ ]  [10]→[31]    [4]   [ ]   [ ]
                          ↑
                    va chạm (collision):
                    10 và 31 cùng rơi vào bucket 3
```

### 3.1.3. Va chạm (Collision) và cách xử lý

Vì số lượng khóa có thể là vô hạn nhưng số bucket là hữu hạn, va chạm (hai khóa khác nhau cho cùng chỉ số bucket) là điều **không thể tránh khỏi** — đây là hệ quả trực tiếp của nguyên lý chuồng bồ câu (pigeonhole principle). Hai chiến lược xử lý va chạm phổ biến:

**Separate Chaining (Nối dây chuyền):** mỗi bucket chứa một danh sách liên kết (hoặc cây cân bằng) lưu tất cả phần tử va chạm vào cùng bucket đó. Khi tra cứu, hệ thống nhảy đến đúng bucket O(1), rồi duyệt tuyến tính trong danh sách nhỏ đó để tìm khóa khớp.

**Open Addressing (Định vị mở):** khi bucket bị chiếm, hệ thống tìm một bucket trống khác theo một quy luật xác định (linear probing, quadratic probing, double hashing) và lưu phần tử tại đó thay vì mở rộng bucket ra thành danh sách.

Trong thực tế lập trình thi đấu và phỏng vấn, ta hầu như không cần tự cài đặt cơ chế này — `unordered_map`/`unordered_set` trong C++ (dùng Separate Chaining) đã xử lý sẵn. Điều quan trọng là **hiểu bản chất** để giải thích được vì sao độ phức tạp trung bình là O(1) nhưng **trường hợp xấu nhất là O(n)** (khi toàn bộ khóa va chạm vào cùng một bucket, bảng băm suy biến thành một danh sách liên kết đơn).

### 3.1.4. Load Factor và Resize

**Load Factor** đo mức độ "chật" của bảng băm:

```
load_factor = số phần tử đang lưu / số bucket
```

Khi load factor vượt một ngưỡng nhất định (thường 0.75), bảng băm tự động **resize**: cấp một mảng bucket lớn hơn (thường gấp đôi) và băm lại (rehash) toàn bộ phần tử cũ sang bảng mới. Cơ chế này hoàn toàn tương tự **doubling strategy** của Dynamic Array đã trình bày ở mục 1.1.4, và cũng áp dụng **amortized analysis** để chứng minh chi phí trung bình mỗi lần chèn vẫn là O(1) dù thỉnh thoảng có một lần resize tốn O(n).

---

## 3.2. HashMap và HashSet

| Cấu trúc | Lưu trữ | Ứng dụng chính | Trong C++ |
|---|---|---|---|
| HashMap | Cặp `(khóa, giá trị)` | Tra cứu giá trị theo khóa, đếm tần suất | `unordered_map<K, V>` |
| HashSet | Chỉ khóa (không giá trị) | Kiểm tra tồn tại, loại trùng lặp | `unordered_set<K>` |

Về bản chất cài đặt, HashSet chỉ là một HashMap mà giá trị bị bỏ qua (hoặc gán giá trị "rỗng" cố định) — mọi tính chất về độ phức tạp và va chạm của hai cấu trúc là như nhau.

---

## 3.3. Các pattern quan trọng

### 3.3.1. Frequency Counting (Đếm tần suất)

**Bản chất:** Khi cần đếm số lần xuất hiện của các phần tử mà **không biết trước phạm vi giá trị** (khác với mục 2.2.1 dùng mảng cố định cho 26 chữ cái), HashMap là lựa chọn tự nhiên vì khóa có thể là bất kỳ giá trị nào.

```cpp
#include <unordered_map>
#include <vector>
using namespace std;

unordered_map<int, int> countFrequency(const vector<int>& arr) {
    unordered_map<int, int> freq;
    for (int x : arr) {
        freq[x]++; // nếu x chưa tồn tại, unordered_map tự khởi tạo giá trị mặc định = 0
    }
    return freq;
}
```

### 3.3.2. Complement Lookup (Tra cứu phần tử bù)

**Bản chất:** Đây là kỹ thuật cốt lõi cho lớp bài toán dạng "tìm cặp phần tử thỏa mãn điều kiện tổng/hiệu". Thay vì duyệt lồng hai vòng lặp O(n²) để thử mọi cặp, ta duyệt một lượt O(n), và tại mỗi phần tử, tra cứu ngay xem "phần tử bù" cần tìm đã xuất hiện trước đó hay chưa — biến bài toán tìm kiếm cặp thành bài toán tra cứu O(1).

**Bài toán Two Sum:** cho mảng `arr` và số `target`, tìm chỉ số hai phần tử có tổng bằng `target`.

**Minh họa** với `arr = [2, 7, 11, 15]`, `target = 9`:

```
i=0: arr[0]=2, cần tìm bù = 9-2=7  → chưa có trong map → lưu {2: 0}
i=1: arr[1]=7, cần tìm bù = 9-7=2  → đã có trong map tại chỉ số 0! → trả về [0, 1]
```

**Cài đặt C++:**

```cpp
#include <vector>
#include <unordered_map>
using namespace std;

vector<int> twoSum(const vector<int>& arr, int target) {
    unordered_map<int, int> seen; // value -> index

    for (int i = 0; i < (int)arr.size(); i++) {
        int complement = target - arr[i];
        auto it = seen.find(complement);
        if (it != seen.end()) {
            return {it->second, i};
        }
        seen[arr[i]] = i;
    }

    return {}; // không tìm thấy
}
```

**Độ phức tạp:** O(n) thời gian trung bình (thay vì O(n²) brute force), O(n) bộ nhớ phụ.

### 3.3.3. Deduplication (Loại bỏ trùng lặp)

**Bản chất:** HashSet lưu trữ các phần tử đã gặp; mỗi phần tử mới chỉ cần tra cứu O(1) xem đã tồn tại trong set hay chưa.

```cpp
#include <vector>
#include <unordered_set>
using namespace std;

bool containsDuplicate(const vector<int>& arr) {
    unordered_set<int> seen;
    for (int x : arr) {
        if (seen.count(x)) return true;
        seen.insert(x);
    }
    return false;
}
```

**Độ phức tạp:** O(n) thời gian trung bình, O(n) bộ nhớ phụ.

### 3.3.4. Prefix Sum + HashMap

**Bản chất:** Đây là sự kết hợp trực tiếp giữa kỹ thuật Prefix Sum (mục 1.6.1) và Complement Lookup. Bài toán "đếm số dãy con liên tiếp có tổng bằng k" (Subarray Sum Equals K) không thể giải bằng hai con trỏ thông thường vì mảng có thể chứa số âm (điều kiện đơn điệu bị phá vỡ). Ta khai thác đẳng thức:

```
sum(arr[l..r]) = prefix[r+1] - prefix[l] = k
⟺  prefix[l] = prefix[r+1] - k
```

Tức là tại mỗi vị trí `r`, ta cần biết đã có **bao nhiêu vị trí `l` trước đó** có `prefix[l] = prefix[r+1] - k`. Dùng HashMap lưu tần suất các giá trị prefix sum đã gặp, tra cứu tức thời thay vì duyệt lại từ đầu.

**Cài đặt C++:**

```cpp
#include <vector>
#include <unordered_map>
using namespace std;

int subarraySumEqualsK(const vector<int>& arr, int k) {
    unordered_map<int, int> prefixCount;
    prefixCount[0] = 1; // prefix sum = 0 xuất hiện 1 lần (trước khi bắt đầu mảng)

    int runningSum = 0;
    int result = 0;

    for (int x : arr) {
        runningSum += x;
        // Tìm xem đã có bao nhiêu prefix trước đó bằng (runningSum - k)
        auto it = prefixCount.find(runningSum - k);
        if (it != prefixCount.end()) {
            result += it->second;
        }
        prefixCount[runningSum]++;
    }

    return result;
}
```

**Độ phức tạp:** O(n) thời gian trung bình, O(n) bộ nhớ phụ — so với brute force O(n²) duyệt mọi cặp `(l, r)`.

### 3.3.5. Longest Consecutive Sequence

**Bài toán:** Cho mảng số nguyên không sắp xếp, tìm độ dài dãy số nguyên liên tiếp dài nhất (ví dụ `[100, 4, 200, 1, 3, 2]` → dãy liên tiếp dài nhất là `[1,2,3,4]`, độ dài 4).

**Bản chất:** Đưa toàn bộ phần tử vào HashSet để tra cứu tồn tại O(1). Điểm mấu chốt để đạt O(n) thay vì O(n²): với mỗi số `x`, ta chỉ bắt đầu đếm dãy liên tiếp nếu `x - 1` **không tồn tại** trong tập — điều này đảm bảo mỗi dãy liên tiếp chỉ được đếm đúng một lần duy nhất, bắt đầu từ phần tử nhỏ nhất của nó.

**Cài đặt C++:**

```cpp
#include <vector>
#include <unordered_set>
#include <algorithm>
using namespace std;

int longestConsecutive(const vector<int>& arr) {
    unordered_set<int> numSet(arr.begin(), arr.end());
    int longest = 0;

    for (int x : numSet) {
        // Chỉ bắt đầu đếm nếu x là điểm khởi đầu của một dãy (x-1 không tồn tại)
        if (numSet.count(x - 1) == 0) {
            int length = 1;
            int current = x;
            while (numSet.count(current + 1)) {
                current++;
                length++;
            }
            longest = max(longest, length);
        }
    }

    return longest;
}
```

**Độ phức tạp:** O(n) thời gian trung bình — dù có vòng lặp `while` lồng bên trong, mỗi phần tử chỉ được mở rộng (extend) đúng một lần trên toàn bộ thuật toán, nên tổng công việc vẫn tuyến tính (kỹ thuật phân tích này gọi là **amortized qua toàn cục**, khác với amortized qua từng lần gọi ở mục 1.1.4).

---

## 3.4. So sánh Hash Table với các cấu trúc tra cứu khác

| Cấu trúc | Tra cứu | Chèn/Xóa | Duy trì thứ tự | Khi nào dùng |
|---|---|---|---|---|
| Array (biết chỉ số) | O(1) | O(1) cuối / O(n) giữa | Có (theo chỉ số) | Biết trước phạm vi chỉ số liên tục |
| Hash Table | O(1) trung bình, O(n) xấu nhất | O(1) trung bình | Không | Khóa bất kỳ, không cần thứ tự |
| TreeMap/Ordered Map (cây cân bằng) | O(log n) | O(log n) | Có (theo khóa) | Cần duyệt khóa theo thứ tự, hoặc cần thao tác range |
| Sorted Array + Binary Search | O(log n) | O(n) | Có | Dữ liệu tĩnh, ít thay đổi |

**Khi nào dùng Hash Table:** khi cần tra cứu/đếm/kiểm tra tồn tại nhanh, khóa không nằm trong phạm vi liên tục nhỏ (nếu phạm vi nhỏ và biết trước, dùng mảng tần suất như mục 2.2.1 sẽ nhanh hơn do tránh chi phí tính hash), và không quan tâm đến thứ tự phần tử.

**Khi nào KHÔNG nên dùng Hash Table:** khi cần duyệt phần tử theo thứ tự tăng/giảm dần (nên dùng cây cân bằng như `std::map`), khi cần tìm phần tử nhỏ nhất lớn hơn một giá trị cho trước (lower bound — Hash Table không hỗ trợ hiệu quả).

---

## 3.5. Bảng tổng hợp độ phức tạp

| Thao tác | Trung bình | Xấu nhất |
|---|---|---|
| Tra cứu (lookup) | O(1) | O(n) |
| Chèn (insert) | O(1) amortized | O(n) |
| Xóa (delete) | O(1) | O(n) |
| Duyệt toàn bộ | O(n + capacity) | O(n + capacity) |

*Trường hợp xấu nhất O(n) xảy ra khi toàn bộ khóa va chạm vào cùng một bucket — trên thực tế gần như không xảy ra với hàm băm tốt và dữ liệu ngẫu nhiên, nên trong phân tích độ phức tạp bài phỏng vấn, ta mặc định coi các thao tác Hash Table là O(1).*

---

## 3.6. Danh sách bài tập luyện tập

### Mức Easy
1. Two Sum
2. Contains Duplicate
3. Valid Anagram (giải lại bằng HashMap thay vì mảng 26 phần tử, so sánh khi nào cách nào tốt hơn)
4. Ransom Note
5. Jewels and Stones
6. Intersection of Two Arrays

### Mức Medium
7. Group Anagrams
8. Subarray Sum Equals K
9. Longest Consecutive Sequence
10. Top K Frequent Elements (kết hợp Heap — xem Chương 9)
11. 4Sum II
12. Insert Delete GetRandom O(1)
13. Copy List with Random Pointer (kết hợp Linked List — xem Chương 6)

### Mức Hard
14. LRU Cache (kết hợp HashMap + Doubly Linked List — xem Chương 6)
15. Substring with Concatenation of All Words
16. First Missing Positive (so sánh cách giải bằng HashSet vs cách in-place O(1) bộ nhớ)

---

*Chương tiếp theo: **Chương 4 — Linked List**, chuyển sang cấu trúc dữ liệu dựa trên con trỏ, nơi các đánh đổi (trade-off) so với Array được vận dụng triệt để.*
