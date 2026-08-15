# Chương 0: Nền tảng & Kỹ năng phỏng vấn

## 0.1. Độ phức tạp thuật toán

### 0.1.1. Vì sao cần phân tích độ phức tạp

Trước khi đi vào bất kỳ cấu trúc dữ liệu hay thuật toán cụ thể nào, cần một công cụ chung để **đo lường và so sánh** hiệu năng giữa các lời giải khác nhau cho cùng một bài toán, mà không phụ thuộc vào tốc độ phần cứng hay ngôn ngữ lập trình cụ thể. Công cụ đó là **ký hiệu tiệm cận (asymptotic notation)** — mô tả tốc độ tăng trưởng của thời gian chạy hoặc bộ nhớ sử dụng khi kích thước đầu vào `n` tiến tới vô cùng, bỏ qua các hằng số và các số hạng bậc thấp không đáng kể.

### 0.1.2. Big O, Big Θ, Big Ω — bản chất và phân biệt

Ba ký hiệu này mô tả ba khía cạnh khác nhau của cùng một hàm độ phức tạp:

- **Big O (O)** — **cận trên (upper bound):** mô tả tốc độ tăng trưởng **tối đa** mà thuật toán có thể đạt tới, tức là trường hợp xấu nhất (worst case). Đây là ký hiệu được dùng phổ biến nhất trong giao tiếp thực tế vì nó đảm bảo an toàn: "thuật toán này sẽ không chậm hơn thế".
- **Big Ω (Omega)** — **cận dưới (lower bound):** mô tả tốc độ tăng trưởng **tối thiểu**, tức trường hợp tốt nhất (best case) mà thuật toán phải mất ít nhất bấy nhiêu thời gian.
- **Big Θ (Theta)** — **cận chặt (tight bound):** áp dụng khi cận trên và cận dưới trùng nhau, mô tả chính xác tốc độ tăng trưởng của thuật toán trong mọi trường hợp.

**Minh họa bằng ví dụ Linear Search** (tìm một phần tử trong mảng chưa sắp xếp):

```
Trường hợp tốt nhất:  phần tử cần tìm nằm ngay đầu mảng   → Ω(1)
Trường hợp xấu nhất:  phần tử cần tìm nằm cuối mảng, 
                       hoặc không tồn tại                  → O(n)
```

Vì cận trên O(n) và cận dưới Ω(1) không trùng nhau, Linear Search **không có** một Θ(n) chung cho mọi trường hợp — ta chỉ có thể nói "Linear Search chạy trong O(n) ở trường hợp xấu nhất". Ngược lại, việc duyệt toàn bộ mảng để tính tổng luôn mất đúng `n` bước bất kể dữ liệu, nên có thể khẳng định chặt: Θ(n).

**Trong thực hành phỏng vấn**, do quy ước chung, người ta thường dùng Big O để chỉ độ phức tạp trường hợp xấu nhất (đôi khi ngầm hiểu là cả trường hợp trung bình), và đây cũng là ký hiệu được dùng xuyên suốt tài liệu này.

### 0.1.3. Các lớp độ phức tạp thường gặp

| Ký hiệu | Tên gọi | Ví dụ thuật toán | n=10 | n=1.000 | n=1.000.000 |
|---|---|---|---|---|---|
| O(1) | Hằng số | Truy cập mảng theo chỉ số | 1 | 1 | 1 |
| O(log n) | Logarit | Binary Search | ~3 | ~10 | ~20 |
| O(n) | Tuyến tính | Duyệt mảng một lượt | 10 | 1.000 | 1.000.000 |
| O(n log n) | Tuyến tính-logarit | Merge Sort, Quick Sort | ~33 | ~10.000 | ~20.000.000 |
| O(n²) | Bậc hai | Duyệt lồng hai vòng lặp | 100 | 1.000.000 | 10¹² |
| O(2ⁿ) | Mũ | Duyệt vét cạn tập con (subset) | 1.024 | ~10³⁰¹ | không khả thi |

Bảng trên minh họa vì sao việc phân tích độ phức tạp **không phải là lý thuyết suông**: với `n = 1.000.000`, một thuật toán O(n²) có thể mất hàng giờ trong khi thuật toán O(n log n) chỉ mất chưa đến một giây — đây chính là căn cứ để đánh giá một lời giải "chấp nhận được" hay "cần tối ưu thêm" trong phỏng vấn.

### 0.1.4. Nhận diện độ phức tạp từ cấu trúc code

Trong một buổi live coding, khả năng đọc nhanh độ phức tạp từ chính đoạn code (của mình hoặc gợi ý từ người phỏng vấn) là kỹ năng bắt buộc. Một số quy tắc nhận diện phổ biến, minh họa bằng C++:

**O(1) — không phụ thuộc kích thước đầu vào:**
```cpp
int getFirst(const vector<int>& arr) {
    return arr[0]; // luôn một bước, bất kể arr lớn hay nhỏ
}
```

**O(n) — một vòng lặp duyệt qua n phần tử:**
```cpp
int sumAll(const vector<int>& arr) {
    int sum = 0;
    for (int x : arr) sum += x; // n bước
    return sum;
}
```

**O(log n) — kích thước bài toán giảm theo cấp số nhân (thường chia đôi) sau mỗi bước:**
```cpp
int binarySearch(const vector<int>& arr, int target) {
    int lo = 0, hi = (int)arr.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1; // mỗi bước loại bỏ một nửa không gian tìm kiếm
    }
    return -1;
}
```

**O(n²) — hai vòng lặp lồng nhau, mỗi vòng chạy tỉ lệ với n:**
```cpp
bool hasDuplicatePair(const vector<int>& arr) {
    for (int i = 0; i < (int)arr.size(); i++) {
        for (int j = i + 1; j < (int)arr.size(); j++) { // vòng trong cũng ~n bước
            if (arr[i] == arr[j]) return true;
        }
    }
    return false;
}
```

**O(n log n) — vòng lặp n bước, mỗi bước gọi một thao tác O(log n) (hoặc thuật toán chia-để-trị như Merge Sort):**
```cpp
// std::sort trong C++ dùng Introsort — trung bình O(n log n)
sort(arr.begin(), arr.end());
```

**Quy tắc thực hành nhanh:** đếm số vòng lặp lồng nhau phụ thuộc trực tiếp vào `n` (mỗi tầng lồng thường cộng thêm một bậc n); nếu bài toán liên tục **chia đôi** không gian tìm kiếm, nghĩ ngay đến O(log n); nếu có đệ quy phân rã bài toán thành nhiều bài toán con **chồng lấn (overlapping)**, cần cân nhắc Dynamic Programming để tránh độ phức tạp mũ (xem Chương 9).

### 0.1.5. Amortized Analysis (Độ phức tạp khấu hao)

Không phải lúc nào độ phức tạp "trên giấy" của một thao tác đơn lẻ cũng phản ánh đúng chi phí thực tế khi thao tác đó được gọi lặp lại nhiều lần. **Amortized Analysis** phân tích tổng chi phí của một **dãy** thao tác, chia đều cho số lần gọi, thay vì chỉ nhìn vào thao tác tốn kém nhất.

Ví dụ kinh điển đã trình bày ở mục 1.1.4: `vector::push_back` có một số lần gọi tốn O(n) (khi resize), nhưng nếu áp dụng chiến lược tăng gấp đôi dung lượng, tổng chi phí của n lần gọi liên tiếp chỉ là O(n), do đó chi phí **khấu hao trung bình** mỗi lần gọi là O(1). Kỹ thuật phân tích này còn xuất hiện trở lại ở mục 3.3.5 (Longest Consecutive Sequence), nơi một vòng lặp `while` lồng bên trong vòng lặp ngoài **trông có vẻ** O(n²) nhưng thực chất chỉ chạy tổng cộng O(n) qua toàn bộ thuật toán.

**Nguyên tắc chung để nhận diện amortized O(1) trong phỏng vấn:** nếu một thao tác tốn kém (như resize, hoặc mở rộng một dãy) chỉ xảy ra **hiếm dần theo cấp số nhân** khi kích thước bài toán tăng, trong khi phần lớn các lần gọi còn lại chỉ tốn O(1), thì trung bình toàn cục vẫn là O(1).

---

## 0.2. Tư duy giải bài trong phỏng vấn

### 0.2.1. Vì sao kỹ năng này quan trọng ngang với kiến thức thuật toán

Một hiểu lầm phổ biến là live coding interview chỉ đánh giá "có biết thuật toán đúng hay không". Trên thực tế, hầu hết quy trình đánh giá của các công ty đặt trọng số đáng kể vào **cách người ứng viên tiếp cận vấn đề**: khả năng làm rõ đề bài mơ hồ, giao tiếp trong lúc suy nghĩ, xử lý khi bị bí hoặc bị bug, và quản lý thời gian hợp lý. Một lời giải đúng nhưng đạt được nhờ đoán mò trong im lặng thường được đánh giá thấp hơn một lời giải gần đúng nhưng có quá trình tư duy rõ ràng, có khả năng tự phát hiện lỗi.

### 0.2.2. Khung quy trình giải bài — UMPIRE

Một khung tư duy giúp cấu trúc hóa toàn bộ quá trình giải bài trong phỏng vấn:

| Bước | Nội dung | Mục tiêu |
|---|---|---|
| **U**nderstand | Đọc kỹ đề, xác định input/output, hỏi lại nếu mơ hồ | Tránh giải sai đề |
| **M**atch | Liên hệ đề bài với pattern/cấu trúc dữ liệu đã học | Định hướng lời giải |
| **P**lan | Trình bày ý tưởng bằng lời hoặc pseudocode trước khi code | Người phỏng vấn xác nhận hướng đi đúng trước khi tốn thời gian code sai |
| **I**mplement | Viết code, vừa code vừa giải thích | Thể hiện tư duy, dễ được gợi ý nếu lạc hướng |
| **R**eview | Rà lại code, chạy thử test case | Phát hiện lỗi trước khi người phỏng vấn chỉ ra |
| **E**valuate | Phân tích độ phức tạp thời gian/không gian của lời giải cuối | Thể hiện khả năng đánh giá đánh đổi |

### 0.2.3. Làm rõ đề bài (Clarify)

**Bản chất:** đề bài phỏng vấn thường được cố tình để mơ hồ ở một số điểm — đây không phải sơ suất của người ra đề mà là **phép thử** xem ứng viên có thói quen kiểm tra giả định trước khi lao vào code hay không. Các câu hỏi nên đặt ra trước khi viết dòng code đầu tiên:

- Input có thể rỗng, chứa giá trị âm, chứa phần tử trùng lặp không?
- Kích thước đầu vào tối đa là bao nhiêu (giúp suy ra độ phức tạp mục tiêu — n ≤ 20 gợi ý có thể chấp nhận O(2ⁿ), n ≤ 10⁶ đòi hỏi O(n) hoặc O(n log n))?
- Nếu có nhiều đáp án hợp lệ, trả về đáp án nào? Nếu không tìm được đáp án, trả về gì?
- Có được phép sửa đổi input tại chỗ (in-place) hay không?

### 0.2.4. Brute Force trước, tối ưu sau

**Bản chất:** trình bày một lời giải đúng nhưng chưa tối ưu (thường là duyệt vét cạn) trước khi cố nghĩ ra lời giải tối ưu ngay từ đầu mang lại ba lợi ích: (1) đảm bảo có ít nhất một lời giải chạy đúng nếu hết thời gian, (2) giúp người phỏng vấn xác nhận cách hiểu đề bài đúng trước khi đầu tư thời gian vào hướng tối ưu, (3) chính lời giải brute force thường **gợi ý trực tiếp** điểm nghẽn cần cải thiện (ví dụ: brute force dùng vòng lặp lồng để tìm cặp phần tử → điểm nghẽn là tra cứu lặp lại → gợi ý dùng HashMap như mục 3.3.2).

### 0.2.5. Giao tiếp khi code (Think Aloud)

**Bản chất:** người phỏng vấn không thể đọc được suy nghĩ của ứng viên. Nói to những gì đang cân nhắc — kể cả khi đang phân vân giữa hai hướng tiếp cận — biến quá trình tư duy vốn vô hình thành thứ có thể quan sát và đánh giá được. Im lặng kéo dài trong lúc code thường khiến người phỏng vấn không biết ứng viên đang tự tin triển khai hay đang bế tắc.

**Cấu trúc gợi ý khi trình bày:**
1. Nêu ý tưởng tổng quát bằng một câu ("Mình sẽ dùng hai con trỏ, một xuất phát từ đầu, một từ cuối...")
2. Giải thích invariant (bất biến) mà thuật toán duy trì trong suốt quá trình chạy
3. Trước khi code một đoạn phức tạp, nói ngắn gọn logic sắp viết

### 0.2.6. Tự tạo test case trước khi chạy

**Bản chất:** viết code xong không có nghĩa là đã xong — cần tự kiểm chứng bằng ít nhất ba loại test case:

- **Test case điển hình:** input thông thường, dùng để kiểm tra logic chính.
- **Test case biên (edge case):** mảng rỗng, một phần tử, giá trị trùng lặp toàn bộ, giá trị âm/0.
- **Test case lớn (nếu cần):** để tự đánh giá độ phức tạp có đạt yêu cầu hay không.

Việc chủ động tự kiểm tra trước khi người phỏng vấn hỏi "bạn có chắc code này đúng không?" thể hiện tính cẩn trọng — một tín hiệu tích cực quan trọng không kém độ chính xác của thuật toán.

### 0.2.7. Debug trực tiếp khi bị lỗi

**Bản chất:** bị bug giữa buổi phỏng vấn là chuyện bình thường và **không phải là điểm trừ tự động** — cách xử lý khi bị bug mới là điều được đánh giá. Quy trình debug có hệ thống thay vì sửa ngẫu nhiên:

1. Xác định chính xác test case nào đang sai, giá trị mong đợi và giá trị thực tế.
2. Dò lại từng bước bằng cách đọc lại code hoặc "chạy tay" (trace) trên test case nhỏ.
3. Đặt giả thuyết cụ thể về nguyên nhân trước khi sửa, tránh sửa ngẫu nhiên hàng loạt dòng code cùng lúc.

### 0.2.8. Phân tích độ phức tạp sau khi code xong

**Bản chất:** sau khi có lời giải chạy đúng, việc chủ động phân tích lại độ phức tạp thời gian và không gian (áp dụng trực tiếp các kỹ thuật ở mục 0.1) thể hiện khả năng tự đánh giá — đồng thời mở ra cơ hội thảo luận về hướng tối ưu tiếp theo nếu người phỏng vấn hỏi "có cách nào nhanh hơn không?".

### 0.2.9. Quản lý thời gian

**Bản chất:** một buổi live coding interview thường kéo dài 30-45 phút cho một hoặc hai bài. Phân bổ thời gian hợp lý là yếu tố quyết định có hoàn thành được bài hay không:

| Giai đoạn | Tỉ lệ thời gian gợi ý (bài 45 phút) |
|---|---|
| Làm rõ đề bài + nêu hướng tiếp cận | ~5 phút |
| Code lời giải | ~25 phút |
| Test + debug | ~10 phút |
| Phân tích độ phức tạp + thảo luận mở rộng | ~5 phút |

Nếu sau 10-15 phút vẫn chưa tìm ra hướng tối ưu, nên chủ động trình bày brute force (mục 0.2.4) để đảm bảo có kết quả, thay vì im lặng cố nghĩ đến hết giờ.

### 0.2.10. Mock Interview (Luyện tập mô phỏng)

**Bản chất:** các kỹ năng ở mục 0.2.3 đến 0.2.9 là kỹ năng **giao tiếp**, không thể luyện thành thạo chỉ bằng cách tự giải bài trong im lặng — cần luyện tập trong điều kiện gần giống thật nhất:

- Luyện với một người khác đóng vai người phỏng vấn (bạn bè, đồng nghiệp, hoặc nền tảng mock interview trực tuyến).
- Nếu không có người luyện cùng, tự quay lại video khi giải bài và nói to suy nghĩ, sau đó xem lại để đánh giá mức độ rõ ràng trong giao tiếp.
- Sau mỗi buổi mock, ghi chú lại các điểm cần cải thiện (thường gặp: nói quá ít, bỏ qua bước clarify, quên test case biên).

---

## 0.3. Bài tập luyện tập

### Phân tích độ phức tạp (đọc code, xác định Big O)

1. Xác định độ phức tạp thời gian của đoạn code sau và giải thích lý do:
```cpp
void printPairs(const vector<int>& arr) {
    for (int i = 0; i < arr.size(); i++) {
        for (int j = 0; j < arr.size(); j++) {
            cout << arr[i] << " " << arr[j] << endl;
        }
    }
}
```

2. Xác định độ phức tạp của hàm đệ quy tính Fibonacci sau, và giải thích vì sao nó kém hiệu quả hơn cách dùng vòng lặp:
```cpp
int fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
}
```

3. Xác định độ phức tạp của đoạn code sau (gợi ý: chú ý vòng lặp `while` bên trong không reset `j` mỗi lần lặp `i`):
```cpp
void twoPointerExample(const vector<int>& arr) {
    int j = 0;
    for (int i = 0; i < arr.size(); i++) {
        while (j < arr.size() && arr[j] < arr[i]) {
            j++;
        }
    }
}
```

4. So sánh độ phức tạp không gian (space complexity) giữa cách giải đệ quy và cách giải lặp (iterative) cho cùng một bài toán duyệt cây — vì sao đệ quy tốn thêm O(h) bộ nhớ với `h` là chiều cao cây?

### Thực hành kỹ năng phỏng vấn

5. Chọn một bài Easy bất kỳ ở Chương 1-3, tự đặt ra ít nhất 4 câu hỏi clarify trước khi giải, dựa theo khung ở mục 0.2.3.

6. Chọn một bài Medium, trình bày (viết ra hoặc nói thành tiếng) lời giải brute force trước, sau đó chỉ rõ điểm nghẽn hiệu năng gợi ý hướng tối ưu là gì.

7. Tự quay video giải một bài bất kỳ trong 20 phút, áp dụng đầy đủ khung UMPIRE ở mục 0.2.2, sau đó xem lại và tự đánh giá: đã nói đủ rõ ý tưởng chưa, có bỏ sót bước test case biên không.

---

*Chương tiếp theo: **Chương 1 — Array**, bắt đầu đi vào từng cấu trúc dữ liệu cụ thể, áp dụng trực tiếp các nguyên tắc phân tích độ phức tạp và tư duy giải bài đã trình bày ở chương này.*
