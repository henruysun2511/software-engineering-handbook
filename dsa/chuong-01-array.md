# Chương 1: Array (Mảng)

## 1.1. Khái niệm cốt lõi

### 1.1.1. Định nghĩa

Array (mảng) là một cấu trúc dữ liệu lưu trữ một tập hợp các phần tử **cùng kiểu dữ liệu**, được sắp xếp liên tiếp nhau trong bộ nhớ và được truy cập thông qua một chỉ số (index) duy nhất. Đây là cấu trúc dữ liệu nền tảng nhất trong khoa học máy tính, và phần lớn các cấu trúc dữ liệu phức tạp hơn (stack, queue, hash table, heap...) đều được xây dựng dựa trên array.

### 1.1.2. Bản chất bộ nhớ — vì sao truy cập là O(1)

Điều làm nên sức mạnh của array không nằm ở cú pháp `arr[i]`, mà nằm ở **cách bộ nhớ được cấp phát**. Khi một array được khai báo, hệ thống cấp một vùng nhớ **liên tục** (contiguous memory block) đủ chứa toàn bộ phần tử. Giả sử array bắt đầu tại địa chỉ `base`, mỗi phần tử chiếm `size` byte, thì địa chỉ của phần tử tại chỉ số `i` được tính trực tiếp bằng công thức:

```
address(arr[i]) = base + i * size
```

Đây là một phép tính số học đơn giản (một phép nhân, một phép cộng), không phụ thuộc vào giá trị của `i` hay kích thước mảng — vì vậy truy cập phần tử bất kỳ luôn mất **thời gian hằng số O(1)**, bất kể mảng có 10 hay 10 triệu phần tử.

**Minh họa bộ nhớ** với mảng `int arr[5] = {10, 20, 30, 40, 50}`, giả sử `base = 1000` và `sizeof(int) = 4` byte:

```
Địa chỉ:   1000   1004   1008   1012   1016
           +----+ +----+ +----+ +----+ +----+
Giá trị:   | 10 | | 20 | | 30 | | 40 | | 50 |
           +----+ +----+ +----+ +----+ +----+
Chỉ số:     [0]    [1]    [2]    [3]    [4]
```

Muốn truy cập `arr[3]`: `address = 1000 + 3*4 = 1012` → đọc trực tiếp giá trị `40`. Không cần duyệt qua các phần tử trước đó — đây chính là điểm khác biệt cốt lõi so với Linked List (sẽ so sánh ở mục 1.5).

### 1.1.3. Static Array và Dynamic Array

**Static Array** có kích thước cố định, được xác định tại thời điểm biên dịch hoặc khởi tạo, và không thể thay đổi trong suốt vòng đời của nó (ví dụ: `int arr[100]`, hoặc `std::array` trong C++).

**Dynamic Array** (ví dụ `std::vector` trong C++, `ArrayList` trong Java) cho phép thay đổi kích thước trong lúc chạy chương trình. Bản chất bên trong, dynamic array vẫn dùng một vùng nhớ liên tục có **capacity** (dung lượng thực tế đã cấp phát) lớn hơn hoặc bằng **size** (số phần tử hiện có). Khi thêm phần tử vượt quá capacity, dynamic array thực hiện quá trình **resize**:

1. Cấp phát một vùng nhớ mới lớn hơn (thường gấp đôi capacity cũ).
2. Sao chép toàn bộ phần tử cũ sang vùng nhớ mới.
3. Giải phóng vùng nhớ cũ.
4. Thêm phần tử mới vào vùng nhớ mới.

**Minh họa quá trình tăng trưởng (doubling strategy):**

```
size=1, cap=1   [10]
size=2, cap=2   [10, 20]
size=3, cap=4   [10, 20, 30, _]        ← resize từ cap=2 lên cap=4
size=4, cap=4   [10, 20, 30, 40]
size=5, cap=8   [10, 20, 30, 40, 50, _, _, _]   ← resize từ cap=4 lên cap=8
```

### 1.1.4. Amortized Analysis — vì sao push_back vẫn là O(1)

Thoạt nhìn, thao tác resize có chi phí O(n) vì phải sao chép toàn bộ phần tử, khiến ta lầm tưởng `push_back` có độ phức tạp O(n). Tuy nhiên, nếu áp dụng chiến lược **tăng gấp đôi (doubling)**, số lần resize trong n thao tác chèn liên tiếp chỉ là O(log n) lần, và tổng chi phí sao chép qua tất cả các lần resize là:

```
1 + 2 + 4 + 8 + ... + n ≈ 2n
```

Chia đều tổng chi phí 2n cho n thao tác `push_back`, ta được chi phí trung bình là O(1) cho mỗi lần gọi — đây gọi là **độ phức tạp khấu hao (amortized complexity)**. Đây là lý do vì sao trong tài liệu và phỏng vấn, người ta vẫn nói `vector::push_back` là O(1) amortized, dù thỉnh thoảng có một lần gọi tốn O(n).

---

## 1.2. Các thao tác cơ bản và độ phức tạp

| Thao tác | Mô tả | Độ phức tạp |
|---|---|---|
| Truy cập (Access) | `arr[i]` | O(1) |
| Duyệt (Traversal) | Đi qua toàn bộ phần tử | O(n) |
| Tìm kiếm (Search, chưa sắp xếp) | Tìm giá trị bất kỳ | O(n) |
| Cập nhật (Update) | `arr[i] = x` | O(1) |
| Chèn cuối (Insert at end) | `push_back` | O(1) amortized |
| Chèn giữa/đầu (Insert at index) | Phải dịch chuyển các phần tử phía sau | O(n) |
| Xóa cuối (Delete at end) | `pop_back` | O(1) |
| Xóa giữa/đầu (Delete at index) | Phải dịch chuyển các phần tử phía sau | O(n) |

**Giải thích bản chất chèn/xóa giữa mảng:** vì các phần tử phải nằm liên tục trong bộ nhớ, khi chèn một phần tử vào giữa mảng, tất cả phần tử phía sau vị trí chèn buộc phải dịch chuyển sang phải một ô để nhường chỗ — đây là nguồn gốc của độ phức tạp O(n), khác hẳn với Linked List (chỉ cần đổi con trỏ, O(1) nếu đã có vị trí).

---

## 1.3. Kiến thức liên quan

### 1.3.1. Tính cục bộ bộ nhớ (Cache Locality)

Vì các phần tử của array nằm liên tiếp nhau, khi CPU đọc `arr[i]`, nó thường nạp luôn cả một khối bộ nhớ lân cận vào cache (cache line, thường 64 byte). Điều này khiến việc duyệt tuần tự một array trên thực tế **nhanh hơn đáng kể** so với duyệt Linked List có cùng độ phức tạp lý thuyết O(n), do Linked List phân mảnh bộ nhớ khiến CPU phải cache-miss liên tục. Đây là lý do vì sao array (và vector) thường được ưu tiên trong lập trình hiệu năng cao.

### 1.3.2. Mảng đa chiều (Multi-dimensional Array)

Mảng 2D (ma trận) trong C++ thực chất vẫn được lưu trên một vùng nhớ liên tục theo thứ tự **row-major** (các phần tử cùng hàng nằm liền kề nhau). Công thức tính địa chỉ phần tử `arr[i][j]` trong ma trận kích thước `rows x cols`:

```
address(arr[i][j]) = base + (i * cols + j) * size
```

Hiểu điều này giải thích vì sao duyệt ma trận theo hàng (row-first) luôn nhanh hơn duyệt theo cột (column-first) do tận dụng cache locality tốt hơn.

---

## 1.4. So sánh Static Array và Dynamic Array

| Tiêu chí | Static Array | Dynamic Array |
|---|---|---|
| Kích thước | Cố định | Thay đổi được |
| Vị trí cấp phát | Thường trên stack | Thường trên heap |
| Chi phí truy cập | O(1) | O(1) |
| Chi phí thêm phần tử cuối | Không áp dụng | O(1) amortized |
| Overhead bộ nhớ | Không có | Có (capacity thường > size) |
| Ví dụ trong C++ | `int arr[100]`, `std::array` | `std::vector` |

---

## 1.5. So sánh Array và Linked List — khi nào dùng gì

| Tiêu chí | Array | Linked List |
|---|---|---|
| Truy cập theo chỉ số | O(1) | O(n) |
| Chèn/xóa đầu mảng | O(n) | O(1) |
| Chèn/xóa cuối mảng | O(1) amortized | O(1) nếu có tail pointer |
| Chèn/xóa giữa (đã biết vị trí) | O(n) | O(1) |
| Cache locality | Tốt | Kém |
| Overhead bộ nhớ mỗi phần tử | Thấp | Cao (lưu thêm con trỏ) |

**Khi nào dùng Array:** khi cần truy cập ngẫu nhiên nhiều (random access), khi kích thước dữ liệu ổn định hoặc chỉ thêm/xóa ở cuối, khi cần tối ưu hiệu năng nhờ cache locality.

**Khi nào dùng Linked List:** khi cần chèn/xóa thường xuyên ở đầu hoặc giữa danh sách mà không muốn trả giá O(n) dịch chuyển phần tử, khi kích thước dữ liệu biến động liên tục và khó dự đoán trước.

---

## 1.6. Các pattern quan trọng trên Array

### 1.6.1. Prefix Sum (Tổng tiền tố)

**Bản chất:** Prefix Sum là kỹ thuật tiền xử lý (pre-processing) giúp trả lời truy vấn "tổng các phần tử từ chỉ số `l` đến `r`" trong O(1) thay vì O(n), bằng cách đánh đổi một lần xử lý trước tốn O(n) và O(n) bộ nhớ phụ.

Gọi `prefix[i]` là tổng các phần tử từ `arr[0]` đến `arr[i-1]` (quy ước `prefix[0] = 0`). Khi đó:

```
prefix[i] = prefix[i-1] + arr[i-1]
```

Tổng đoạn `[l, r]` (chỉ số 0-based, bao gồm cả hai đầu) được tính bằng:

```
sum(l, r) = prefix[r+1] - prefix[l]
```

**Minh họa** với `arr = [3, 1, 4, 1, 5]`:

```
Chỉ số:      0   1   2   3   4
arr:         3   1   4   1   5
prefix:  0   3   4   8   9   14
         ↑ prefix[0]=0 (không có phần tử nào)
```

Muốn tính tổng đoạn `[1, 3]` (tức `1+4+1=6`): `sum = prefix[4] - prefix[1] = 9 - 3 = 6` ✓

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

class PrefixSum {
private:
    vector<long long> prefix;

public:
    // Tiền xử lý: O(n) thời gian, O(n) bộ nhớ
    explicit PrefixSum(const vector<int>& arr) {
        int n = arr.size();
        prefix.assign(n + 1, 0);
        for (int i = 0; i < n; i++) {
            prefix[i + 1] = prefix[i] + arr[i];
        }
    }

    // Truy vấn tổng đoạn [l, r] (0-based, bao gồm cả hai đầu): O(1)
    long long rangeSum(int l, int r) const {
        return prefix[r + 1] - prefix[l];
    }
};
```

**Độ phức tạp:** Tiền xử lý O(n) thời gian và O(n) bộ nhớ; mỗi truy vấn sau đó O(1).

**Khi nào dùng:** khi có nhiều truy vấn tổng đoạn trên một mảng **không thay đổi** (immutable). Nếu mảng bị cập nhật thường xuyên xen kẽ với truy vấn, cần cấu trúc khác (Fenwick Tree / Segment Tree — xem chương Advanced Trees).

### 1.6.2. Difference Array (Mảng sai phân)

**Bản chất:** Difference Array giải quyết bài toán ngược lại với Prefix Sum — khi cần thực hiện nhiều thao tác **cộng dồn một giá trị vào toàn bộ một đoạn** `[l, r]`, thay vì cập nhật từng phần tử O(n) mỗi lần, ta chỉ cập nhật 2 vị trí O(1), rồi khôi phục mảng gốc bằng một lượt Prefix Sum ở cuối.

Gọi `diff[i] = arr[i] - arr[i-1]` (với `arr[-1] = 0`). Muốn cộng giá trị `val` vào mọi phần tử trong đoạn `[l, r]`:

```
diff[l]   += val
diff[r+1] -= val   (nếu r+1 còn trong phạm vi mảng)
```

Sau khi thực hiện xong tất cả thao tác, lấy prefix sum của mảng `diff` sẽ ra mảng `arr` đã được cập nhật.

**Minh họa:** `arr` ban đầu toàn số 0, độ dài 6. Thực hiện `add(1, 3, +5)` rồi `add(2, 4, +2)`:

```
diff sau add(1,3,+5):   [0, +5, 0, 0, -5, 0]
diff sau add(2,4,+2):   [0, +5, +2, 0, -5, -2]
prefix sum của diff:    [0, 5, 7, 7, 2, 0]
```

→ `arr = [0, 5, 7, 7, 2, 0]`, đúng bằng: đoạn [1,3] được +5, đoạn [2,4] được +2 (chồng lấn tại [2,3] cộng dồn cả hai).

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

class DifferenceArray {
private:
    vector<long long> diff;
    int n;

public:
    explicit DifferenceArray(int size) : n(size), diff(size + 1, 0) {}

    // Cộng val vào mọi phần tử trong đoạn [l, r]: O(1)
    void addRange(int l, int r, long long val) {
        diff[l] += val;
        if (r + 1 <= n) diff[r + 1] -= val;
    }

    // Khôi phục mảng gốc sau khi thực hiện xong mọi thao tác: O(n)
    vector<long long> build() const {
        vector<long long> arr(n);
        long long running = 0;
        for (int i = 0; i < n; i++) {
            running += diff[i];
            arr[i] = running;
        }
        return arr;
    }
};
```

**Độ phức tạp:** Mỗi thao tác cập nhật đoạn O(1); khôi phục mảng cuối cùng O(n). So với cách cập nhật trực tiếp từng đoạn (O(n) mỗi lần), Difference Array vượt trội khi có nhiều thao tác cập nhật đoạn.

**Khi nào dùng:** khi có nhiều thao tác "cộng vào một đoạn" và chỉ cần đọc kết quả cuối cùng (không cần đọc giá trị tức thời xen kẽ giữa các lần cập nhật).

### 1.6.3. Kadane's Algorithm (Maximum Subarray)

**Bài toán:** Tìm tổng lớn nhất của một dãy con liên tiếp (subarray) trong mảng số nguyên (có thể chứa số âm).

**Bản chất:** Đây thực chất là một dạng Dynamic Programming đơn giản. Gọi `curSum` là tổng lớn nhất của dãy con liên tiếp **kết thúc tại vị trí hiện tại**. Tại mỗi phần tử, ta đứng trước lựa chọn nhị phân:

- Hoặc **nối tiếp** dãy con đang có: `curSum + arr[i]`
- Hoặc **bắt đầu dãy con mới** từ chính `arr[i]`: `arr[i]`

Chọn phương án nào cho tổng lớn hơn — vì nếu `curSum` đang âm, việc "mang" nó theo chỉ làm giảm tổng của dãy con tiếp theo, nên tốt hơn là bỏ đi và bắt đầu lại.

```
curSum[i] = max(arr[i], curSum[i-1] + arr[i])
answer = max(curSum[i]) với mọi i
```

**Minh họa** với `arr = [-2, 1, -3, 4, -1, 2, 1, -5, 4]`:

```
i:        0    1    2    3    4    5    6    7    8
arr:     -2    1   -3    4   -1    2    1   -5    4
curSum:  -2    1   -2    4    3    5    6    1    5
maxSoFar:-2    1    1    4    4    5    6    6    6
```

→ Kết quả: `6`, ứng với dãy con `[4, -1, 2, 1]`.

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
#include <climits>
using namespace std;

int maxSubArray(const vector<int>& arr) {
    int curSum = arr[0];
    int maxSum = arr[0];

    for (int i = 1; i < (int)arr.size(); i++) {
        // Nối tiếp dãy con cũ, hoặc bắt đầu dãy con mới từ arr[i]
        curSum = max(arr[i], curSum + arr[i]);
        maxSum = max(maxSum, curSum);
    }

    return maxSum;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — cải tiến đáng kể so với cách brute force O(n²) hoặc O(n³) duyệt toàn bộ cặp `(l, r)`.

### 1.6.4. Product Except Self

**Bài toán:** Cho mảng `arr`, trả về mảng `result` sao cho `result[i]` bằng tích của toàn bộ phần tử trong `arr` **trừ** `arr[i]`, không được dùng phép chia, độ phức tạp O(n).

**Bản chất:** `result[i]` có thể phân tách thành tích của hai phần độc lập:

```
result[i] = (tích các phần tử bên trái i) × (tích các phần tử bên phải i)
```

Ta duyệt mảng hai lượt: lượt đầu tính tích tiền tố (từ trái sang phải), lượt sau tính tích hậu tố (từ phải sang trái) và nhân dồn trực tiếp vào `result`, tránh phải cấp thêm mảng phụ.

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

vector<long long> productExceptSelf(const vector<int>& arr) {
    int n = arr.size();
    vector<long long> result(n, 1);

    // Lượt 1: result[i] = tích các phần tử bên trái i
    long long leftProduct = 1;
    for (int i = 0; i < n; i++) {
        result[i] = leftProduct;
        leftProduct *= arr[i];
    }

    // Lượt 2: nhân thêm tích các phần tử bên phải i
    long long rightProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= arr[i];
    }

    return result;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ (không tính mảng kết quả).

### 1.6.5. Rotate Array (Xoay mảng)

**Bài toán:** Xoay mảng sang phải `k` bước tại chỗ (in-place), ví dụ `[1,2,3,4,5,6,7]` xoay `k=3` thành `[5,6,7,1,2,3,4]`.

**Bản chất:** Cách trực quan nhất là xoay từng bước một (O(n·k)), nhưng có một thủ thuật tối ưu dựa trên tính chất của phép **đảo ngược (reverse)**: xoay phải `k` bước tương đương với việc đảo ngược toàn bộ mảng, rồi đảo ngược riêng từng phần `k` phần tử đầu và `n-k` phần tử còn lại.

```
Gốc:                [1,2,3,4,5,6,7], k=3
Đảo ngược toàn bộ:  [7,6,5,4,3,2,1]
Đảo ngược k=3 đầu:  [5,6,7,4,3,2,1]
Đảo ngược phần còn: [5,6,7,1,2,3,4]   ← kết quả
```

**Cài đặt C++:**

```cpp
#include <vector>
#include <algorithm>
using namespace std;

void rotateRight(vector<int>& arr, int k) {
    int n = arr.size();
    k %= n; // xử lý trường hợp k >= n
    if (k == 0) return;

    reverse(arr.begin(), arr.end());
    reverse(arr.begin(), arr.begin() + k);
    reverse(arr.begin() + k, arr.end());
}
```

**Độ phức tạp:** O(n) thời gian (mỗi phần tử được đảo tối đa hằng số lần), O(1) bộ nhớ phụ — tối ưu hơn hẳn cách dùng mảng tạm (O(n) bộ nhớ) hay xoay từng bước (O(n·k) thời gian).

### 1.6.6. Move Zeroes (In-place Modification với Two Pointers)

**Bài toán:** Di chuyển toàn bộ số 0 trong mảng về cuối, giữ nguyên thứ tự tương đối các phần tử khác 0, thực hiện tại chỗ.

**Bản chất:** Dùng kỹ thuật hai con trỏ — một con trỏ `writePos` đánh dấu vị trí tiếp theo cần ghi một phần tử khác 0, một con trỏ duyệt `i` đi qua toàn bộ mảng. Mỗi khi gặp phần tử khác 0, ta hoán đổi nó về vị trí `writePos` rồi tăng `writePos` lên 1.

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

void moveZeroes(vector<int>& arr) {
    int writePos = 0;

    for (int i = 0; i < (int)arr.size(); i++) {
        if (arr[i] != 0) {
            swap(arr[writePos], arr[i]);
            writePos++;
        }
    }
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ.

### 1.6.7. Merge Sorted Arrays (Gộp hai mảng đã sắp xếp)

**Bài toán:** Gộp hai mảng đã sắp xếp thành một mảng sắp xếp duy nhất.

**Bản chất:** Vì cả hai mảng đã sắp xếp, ta chỉ cần duyệt song song bằng hai con trỏ, mỗi bước so sánh phần tử đang xét của hai mảng và chọn phần tử nhỏ hơn để đưa vào mảng kết quả — đây chính là bước "merge" trong thuật toán Merge Sort.

**Cài đặt C++:**

```cpp
#include <vector>
using namespace std;

vector<int> mergeSortedArrays(const vector<int>& a, const vector<int>& b) {
    vector<int> result;
    result.reserve(a.size() + b.size());

    int i = 0, j = 0;
    while (i < (int)a.size() && j < (int)b.size()) {
        if (a[i] <= b[j]) result.push_back(a[i++]);
        else result.push_back(b[j++]);
    }
    // Chép nốt phần còn lại của mảng chưa duyệt hết
    while (i < (int)a.size()) result.push_back(a[i++]);
    while (j < (int)b.size()) result.push_back(b[j++]);

    return result;
}
```

**Độ phức tạp:** O(n + m) thời gian với `n, m` là độ dài hai mảng, O(n + m) bộ nhớ cho mảng kết quả.

---

## 1.7. Bảng tổng hợp pattern và khi nào dùng

| Pattern | Bài toán giải quyết | Độ phức tạp | Khi nào dùng |
|---|---|---|---|
| Prefix Sum | Tổng đoạn nhiều truy vấn | O(n) build, O(1) query | Mảng tĩnh, nhiều truy vấn tổng đoạn |
| Difference Array | Cộng dồn giá trị vào nhiều đoạn | O(1) update, O(n) build cuối | Nhiều thao tác cập nhật đoạn, đọc kết quả sau cùng |
| Kadane's Algorithm | Tổng dãy con liên tiếp lớn nhất | O(n) | Bài toán tối ưu trên dãy con liên tiếp |
| Two Pointers (in-place) | Sắp xếp lại phần tử tại chỗ | O(n) | Cần tiết kiệm bộ nhớ, không cấp mảng phụ |
| Reverse trick | Xoay mảng tại chỗ | O(n) | Cần xoay mảng mà không dùng thêm O(n) bộ nhớ |

---

## 1.8. Danh sách bài tập luyện tập

### Mức Easy
1. Two Sum (kết hợp HashMap — xem Chương 3)
2. Contains Duplicate
3. Remove Duplicates from Sorted Array
4. Move Zeroes
5. Best Time to Buy and Sell Stock
6. Merge Sorted Array (LeetCode 88)
7. Plus One
8. Find Pivot Index (ứng dụng Prefix Sum)

### Mức Medium
9. Maximum Subarray (Kadane's Algorithm)
10. Product of Array Except Self
11. Rotate Array
12. 3Sum (kết hợp Two Pointers — xem Chương 4)
13. Range Sum Query — Immutable (ứng dụng Prefix Sum trực tiếp)
14. Subarray Sum Equals K (Prefix Sum + HashMap — xem Chương 3)
15. Range Addition (ứng dụng Difference Array trực tiếp)
16. Next Permutation
17. Set Matrix Zeroes

### Mức Hard
18. First Missing Positive
19. Trapping Rain Water (Two Pointers — xem Chương 4)
20. Maximum Subarray Sum Circular
21. Median of Two Sorted Arrays

---

*Chương tiếp theo: **Chương 2 — String**, tiếp nối bằng các kỹ thuật xử lý ký tự, hashing chuỗi, và các bài toán palindrome/anagram/substring.*
