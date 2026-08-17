# Chương 28: Bit Manipulation (Thao tác bit)

## 28.1. Khái niệm cốt lõi

### 28.1.1. Định nghĩa

Bit Manipulation là nhóm kỹ thuật thao tác trực tiếp trên **biểu diễn nhị phân** của số nguyên, khai thác các phép toán bitwise (AND, OR, XOR, NOT, dịch bit) để giải quyết bài toán với độ phức tạp thời gian và không gian tối ưu hơn đáng kể so với cách tiếp cận thông thường — vì các phép toán này được CPU thực thi trực tiếp ở cấp độ phần cứng, tốc độ gần như tức thời.

### 28.1.2. Bản chất — biểu diễn nhị phân

Mọi số nguyên trong máy tính được lưu trữ dưới dạng chuỗi bit (0 và 1). Với số nguyên không dấu 8-bit, giá trị `13` được biểu diễn:

```
Vị trí bit:  7  6  5  4  3  2  1  0
Giá trị:     0  0  0  0  1  1  0  1     = 8 + 4 + 1 = 13
```

Số âm trong hầu hết hệ thống hiện đại (bao gồm C++) được biểu diễn theo **bù hai (two's complement)**: lấy biểu diễn nhị phân của số dương tương ứng, đảo toàn bộ bit (0 thành 1, 1 thành 0), rồi cộng thêm 1. Cách biểu diễn này giúp phép cộng/trừ số âm hoạt động nhất quán với số dương mà không cần logic xử lý riêng — không đi sâu vào phần cứng vì không phải trọng tâm phỏng vấn, nhưng cần nắm để hiểu vì sao một số thao tác bit (như dịch phải trên số âm) có thể cho kết quả khác trực giác.

---

## 28.2. Các phép toán bitwise cơ bản

| Phép toán | Ký hiệu C++ | Bản chất |
|---|---|---|
| AND | `&` | Bit kết quả là 1 chỉ khi **cả hai** bit tương ứng đều là 1 |
| OR | `\|` | Bit kết quả là 1 khi **ít nhất một** trong hai bit là 1 |
| XOR | `^` | Bit kết quả là 1 khi **hai bit khác nhau** (một 0 một 1) |
| NOT | `~` | Đảo ngược toàn bộ bit (0 thành 1, 1 thành 0) |
| Left Shift | `<<` | Dịch toàn bộ bit sang trái, tương đương nhân với `2^k` |
| Right Shift | `>>` | Dịch toàn bộ bit sang phải, tương đương chia nguyên cho `2^k` |

**Minh họa trực quan** với `a = 12 (1100)`, `b = 10 (1010)`:

```
a & b:  1100
        1010
      = 1000  = 8

a | b:  1100
        1010
      = 1110  = 14

a ^ b:  1100
        1010
      = 0110  = 6
```

---

## 28.3. Các pattern quan trọng

### 28.3.1. Kiểm tra chẵn/lẻ

**Bản chất:** bit cuối cùng (bit thứ 0) của một số quyết định tính chẵn lẻ — số lẻ luôn có bit cuối là 1, số chẵn luôn có bit cuối là 0. Phép `AND` với `1` sẽ "cô lập" đúng bit này.

```cpp
bool isOdd(int n) {
    return (n & 1) == 1; // nhanh hơn n % 2 == 1 về mặt lý thuyết phần cứng, dù trình biên dịch hiện đại thường tự tối ưu cả hai như nhau
}
```

### 28.3.2. Kiểm tra lũy thừa của 2

**Bản chất:** một số là lũy thừa của 2 khi và chỉ khi biểu diễn nhị phân của nó có **đúng một bit 1** (ví dụ `8 = 1000`, `16 = 10000`). Quan sát quan trọng: `n & (n-1)` luôn **xóa đi bit 1 thấp nhất** của `n` — nếu `n` là lũy thừa của 2 (chỉ có một bit 1 duy nhất), phép toán này sẽ cho kết quả `0`.

**Minh họa** với `n = 8 (1000)`:

```
n:      1000
n-1:    0111
n&(n-1):0000   = 0 → là lũy thừa của 2
```

Với `n = 12 (1100)` (không phải lũy thừa của 2):

```
n:      1100
n-1:    1011
n&(n-1):1000   = 8 ≠ 0 → không phải lũy thừa của 2
```

```cpp
bool isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}
```

### 28.3.3. XOR Trick — tìm phần tử không trùng lặp

**Bản chất:** phép XOR có hai tính chất đại số quan trọng: `x ^ x = 0` (một số XOR với chính nó luôn bằng 0) và `x ^ 0 = x` (XOR với 0 giữ nguyên giá trị), đồng thời XOR có tính **giao hoán và kết hợp** (thứ tự thực hiện không ảnh hưởng kết quả). Với bài toán "tìm phần tử xuất hiện đúng một lần trong mảng mà mọi phần tử khác đều xuất hiện đúng hai lần", XOR toàn bộ mảng sẽ khiến **mọi cặp trùng lặp tự triệt tiêu** (`x ^ x = 0`), chỉ còn lại đúng phần tử duy nhất.

```cpp
#include <vector>
using namespace std;

int singleNumber(const vector<int>& nums) {
    int result = 0;
    for (int num : nums) {
        result ^= num; // các cặp trùng lặp tự triệt tiêu lẫn nhau
    }
    return result;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — so với cách dùng HashSet (Chương 3) cũng đạt O(n) thời gian nhưng tốn thêm O(n) bộ nhớ phụ, XOR Trick vượt trội về không gian.

### 28.3.4. Set / Clear / Toggle Bit

**Bản chất:** ba thao tác cơ bản để chỉnh sửa một bit cụ thể tại vị trí `i`, đều dựa trên việc tạo một "mặt nạ" (`mask = 1 << i`) chỉ có đúng bit thứ `i` là 1:

```cpp
int setBit(int n, int i) {
    return n | (1 << i); // OR với mask: buộc bit i thành 1, giữ nguyên các bit khác
}

int clearBit(int n, int i) {
    return n & ~(1 << i); // AND với mask đã đảo ngược: buộc bit i thành 0, giữ nguyên các bit khác
}

int toggleBit(int n, int i) {
    return n ^ (1 << i); // XOR với mask: đảo ngược đúng bit i, giữ nguyên các bit khác
}

bool checkBit(int n, int i) {
    return (n >> i) & 1; // dịch bit i về vị trí cuối, rồi cô lập bằng AND với 1
}
```

### 28.3.5. Đếm số bit 1 (Count Set Bits)

**Bản chất:** áp dụng lặp lại quan sát ở mục 28.3.2 — `n & (n-1)` luôn xóa đi **đúng một bit 1** (bit 1 thấp nhất), bất kể `n` có bao nhiêu bit 1. Lặp lại thao tác này cho đến khi `n` về 0, số lần lặp chính là số bit 1 ban đầu.

```cpp
int countSetBits(int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1); // xóa bit 1 thấp nhất mỗi lần lặp
        count++;
    }
    return count;
}
```

**Độ phức tạp:** O(số bit 1 trong n) thời gian — nhanh hơn cách kiểm tra tuần tự từng bit một (O(32) hoặc O(64) cố định bất kể giá trị `n`), vì vòng lặp chỉ chạy đúng bằng số bit 1 thực sự tồn tại.

### 28.3.6. Bitmask — biểu diễn tập hợp con bằng số nguyên

**Bản chất:** một số nguyên `n`-bit có thể biểu diễn một **tập con** của tập `n` phần tử — bit thứ `i` bằng 1 nghĩa là phần tử thứ `i` **có mặt** trong tập con, bằng 0 nghĩa là **không có mặt**. Đây là kỹ thuật nền tảng để duyệt **toàn bộ 2ⁿ tập con** của một tập hợp bằng vòng lặp đơn giản thay vì Backtracking (Chương 13) khi `n` đủ nhỏ (thường n ≤ 20).

**Minh họa** với tập `{A, B, C}` (n=3), số nguyên `5 = 101` (nhị phân) biểu diễn tập con `{A, C}` (bit 0 và bit 2 được bật):

```
Bitmask:     101
Vị trí bit:  210
             ↑ ↑
           bit2(C) bit0(A) đều bật → tập con {A, C}
```

```cpp
#include <vector>
using namespace std;

vector<vector<int>> subsetsUsingBitmask(const vector<int>& nums) {
    int n = nums.size();
    vector<vector<int>> result;

    // Duyệt qua mọi giá trị từ 0 đến 2^n - 1, mỗi giá trị là một bitmask biểu diễn một tập con
    for (int mask = 0; mask < (1 << n); mask++) {
        vector<int> subset;
        for (int i = 0; i < n; i++) {
            if (mask & (1 << i)) { // kiểm tra bit thứ i có được bật hay không
                subset.push_back(nums[i]);
            }
        }
        result.push_back(subset);
    }

    return result;
}
```

**Độ phức tạp:** O(2ⁿ · n) thời gian — cùng bậc độ phức tạp với cách Backtracking ở mục 13.3.1, nhưng cách Bitmask thường có **hằng số nhỏ hơn** trong thực hành (không có overhead của lời gọi hàm đệ quy), và đặc biệt hữu ích khi cần biểu diễn trạng thái tập con một cách **gọn nhẹ** để làm State cho Dynamic Programming (Bitmask DP — kỹ thuật nâng cao nằm ngoài phạm vi trọng tâm của tài liệu này, nhưng đáng biết đến khi gặp bài toán với `n` rất nhỏ và cần theo dõi "tập con nào đã được xử lý").

---

## 28.4. Bảng tổng hợp

| Pattern | Công thức cốt lõi | Ứng dụng |
|---|---|---|
| Kiểm tra chẵn/lẻ | `n & 1` | Phân loại nhanh |
| Kiểm tra lũy thừa của 2 | `n & (n-1) == 0` | Kiểm tra cấu trúc dữ liệu dạng cây nhị phân đầy |
| XOR Trick | `x ^ x = 0` | Tìm phần tử duy nhất trong mảng trùng lặp theo cặp |
| Set/Clear/Toggle Bit | `n \| (1<<i)`, `n & ~(1<<i)`, `n ^ (1<<i)` | Quản lý trạng thái nhị phân |
| Đếm bit 1 | Lặp `n &= (n-1)` | Đếm nhanh số phần tử trong tập con biểu diễn bởi bitmask |
| Bitmask duyệt tập con | `mask` từ `0` đến `2^n - 1` | Thay thế Backtracking khi n nhỏ, làm State cho Bitmask DP |

---

## 28.5. Khi nào dùng Bit Manipulation

- Bài toán liên quan trực tiếp đến biểu diễn nhị phân (đếm bit, kiểm tra lũy thừa của 2).
- Cần tối ưu bộ nhớ tối đa — biểu diễn trạng thái/tập hợp bằng một số nguyên duy nhất thay vì mảng hoặc HashSet (Chương 3).
- Bài toán tìm phần tử duy nhất/thiếu trong mảng có tính chất cặp đôi (ứng dụng XOR Trick).
- Kích thước tập hợp nhỏ (n ≤ 20) và cần duyệt toàn bộ tập con — Bitmask thay thế Backtracking (Chương 13) hoặc làm nền tảng State cho DP nâng cao.
- Nhận diện qua từ khóa: "nhị phân", "bit", "XOR", hoặc ràng buộc "n ≤ 20" đi kèm yêu cầu duyệt mọi tập con/trạng thái.

---

## 28.6. Danh sách bài tập luyện tập

### Mức Easy
1. Single Number
2. Number of 1 Bits
3. Counting Bits
4. Power of Two
5. Missing Number (XOR Trick biến thể — XOR cả chỉ số lẫn giá trị)
6. Reverse Bits

### Mức Medium
7. Single Number II (mỗi phần tử khác xuất hiện 3 lần thay vì 2 — cần kỹ thuật đếm bit theo modulo 3)
8. Single Number III (hai phần tử duy nhất thay vì một — cần tách nhóm bằng bit phân biệt)
9. Subsets (thử giải lại bằng Bitmask, so sánh với Backtracking ở mục 13.3.1)
10. Sum of Two Integers (cộng hai số chỉ dùng phép bit, không dùng toán tử `+`)

### Mức Hard
11. Maximum XOR of Two Numbers in an Array (kết hợp Trie nhị phân — mở rộng của Chương 16)
12. Minimum XOR Sum of Two Arrays (Bitmask DP)

---

*Đến đây, tài liệu đã hoàn thành các chương thuộc nhóm kỹ thuật giải thuật cốt lõi (Part I-X theo mục lục gốc). Các chương tiếp theo trong lộ trình đầy đủ — Advanced Patterns (Monotonic Stack/Queue, Interval, Prefix Sum nâng cao), Problem Recognition, và Coding Templates tổng hợp — đóng vai trò hệ thống hóa và ôn tập nhanh trước ngày phỏng vấn, tổng hợp lại các kỹ thuật đã học xuyên suốt từ Chương 0 đến Chương 28.*
