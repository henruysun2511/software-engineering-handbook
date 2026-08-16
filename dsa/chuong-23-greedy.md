# Chương 23: Greedy Algorithm (Thuật toán tham lam)

## 23.1. Khái niệm cốt lõi

### 23.1.1. Định nghĩa

Greedy Algorithm là chiến lược giải quyết bài toán tối ưu bằng cách tại **mỗi bước**, luôn chọn lựa chọn **tốt nhất theo tiêu chí cục bộ (locally optimal)** ngay tại thời điểm đó, mà không xem xét lại hay thay đổi các lựa chọn đã đưa ra trước đó, với kỳ vọng rằng chuỗi lựa chọn cục bộ tối ưu này sẽ dẫn đến lời giải **tối ưu toàn cục (globally optimal)**.

### 23.1.2. Bản chất — hai điều kiện để Greedy đúng đắn

Điểm mấu chốt và cũng là điểm dễ sai lầm nhất khi áp dụng Greedy: **chiến lược tham lam không phải lúc nào cũng cho ra lời giải tối ưu** — nó chỉ đúng khi bài toán thỏa mãn hai tính chất sau:

**Greedy Choice Property (Tính chất lựa chọn tham lam):** tồn tại một lời giải tối ưu toàn cục có thể đạt được bằng cách thực hiện lựa chọn tham lam tại bước đầu tiên, sau đó chỉ cần giải bài toán con còn lại một cách tối ưu. Nói cách khác, lựa chọn cục bộ tốt nhất **không bao giờ khiến ta bỏ lỡ** lời giải tối ưu toàn cục.

**Optimal Substructure (Cấu trúc con tối ưu):** lời giải tối ưu của bài toán gốc chứa trong nó lời giải tối ưu của các bài toán con — tính chất này cũng là nền tảng của Dynamic Programming (Chương 25), điểm khác biệt nằm ở chỗ Greedy chỉ cần xét **một** bài toán con (do đã "chốt" lựa chọn tham lam), trong khi DP cần xét xử lý **nhiều khả năng** bài toán con và chọn ra tốt nhất.

### 23.1.3. Exchange Argument — kỹ thuật chứng minh Greedy đúng đắn

**Bản chất:** để chứng minh một chiến lược Greedy cho ra lời giải tối ưu, kỹ thuật phổ biến nhất là **exchange argument** (lập luận hoán đổi): giả sử tồn tại một lời giải tối ưu **khác** với lời giải mà Greedy tạo ra, chứng minh rằng ta luôn có thể "hoán đổi" (điều chỉnh) lời giải tối ưu đó để nó **giống với lựa chọn của Greedy** tại bước đang xét, mà **không làm lời giải trở nên tệ hơn**. Nếu chứng minh được điều này cho mọi bước, suy ra lời giải của Greedy cũng tốt bằng lời giải tối ưu bất kỳ — tức bản thân nó cũng là tối ưu.

**Minh họa cho bài toán Activity Selection** (đã áp dụng ở mục 11.4.2 — Non-overlapping Intervals, chọn tối đa số khoảng không chồng lấn): chiến lược Greedy luôn chọn hoạt động có **thời điểm kết thúc sớm nhất** trước. Giả sử có lời giải tối ưu khác không chọn hoạt động này đầu tiên — ta luôn có thể thay hoạt động đầu tiên của lời giải đó bằng hoạt động kết thúc sớm nhất (vì nó kết thúc không muộn hơn, nên chắc chắn không xung đột với các hoạt động còn lại của lời giải tối ưu), cho ra một lời giải **tốt bằng hoặc tốt hơn** — điều này chứng minh lựa chọn Greedy luôn an toàn.

---

## 23.2. Sort + Greedy — mẫu hình phổ biến nhất

### 23.2.1. Bản chất

Đã được giới thiệu chi tiết ở mục 11.1.2 và 11.4: phần lớn chiến lược Greedy đòi hỏi xử lý các phần tử theo một **thứ tự ưu tiên nhất định** (ví dụ theo thời điểm kết thúc sớm nhất, theo trọng số nhẹ nhất) để đảm bảo Greedy Choice Property được thỏa mãn tại mỗi bước — đạt được bằng cách **sắp xếp trước**. Đây là lý do Sort thường xuất hiện như bước đầu tiên trong lời giải Greedy.

### 23.2.2. Các bài toán đã trình bày ở chương trước

Để tránh trùng lặp, các bài toán Sort + Greedy kinh điển đã được trình bày chi tiết ở những chương trước, được liệt kê lại đây như một phần của bức tranh tổng thể về Greedy:

- **Merge Intervals** (mục 11.4.1): sắp xếp theo điểm bắt đầu, gộp các khoảng chồng lấn liên tiếp.
- **Non-overlapping Intervals** (mục 11.4.2): sắp xếp theo điểm kết thúc, luôn giữ lại khoảng kết thúc sớm nhất (ứng dụng trực tiếp Exchange Argument ở mục 23.1.3).
- **Kruskal's Algorithm** (mục 21.4.3): sắp xếp cạnh theo trọng số tăng dần, kết hợp Union-Find để xây dựng Minimum Spanning Tree.
- **Dijkstra's Algorithm** (mục 22.1.3): tuy không sắp xếp trước, nhưng bản chất Greedy "luôn xử lý đỉnh có khoảng cách tạm thời nhỏ nhất" đạt được nhờ Min-Heap — một dạng "sắp xếp động" liên tục cập nhật thứ tự ưu tiên.

---

## 23.3. Cài đặt các bài toán kinh điển mới

### 23.3.1. Jump Game

**Bài toán:** cho mảng `nums`, mỗi phần tử `nums[i]` là số bước nhảy xa nhất có thể thực hiện từ vị trí `i`, xác định có thể đến được vị trí cuối cùng của mảng hay không, xuất phát từ vị trí 0.

**Bản chất Greedy:** thay vì thử mọi cách nhảy có thể (Backtracking, độ phức tạp hàm mũ), ta chỉ cần theo dõi **vị trí xa nhất có thể đến được** tính đến thời điểm hiện tại (`maxReach`), duyệt qua từng vị trí và cập nhật `maxReach = max(maxReach, i + nums[i])`. Nếu tại bất kỳ thời điểm nào, vị trí đang xét `i` vượt quá `maxReach` hiện có, nghĩa là không thể đến được vị trí đó — thất bại.

```cpp
#include <vector>
#include <algorithm>
using namespace std;

bool canJump(const vector<int>& nums) {
    int maxReach = 0;

    for (int i = 0; i < (int)nums.size(); i++) {
        if (i > maxReach) return false; // vị trí i không thể đến được từ các bước trước

        maxReach = max(maxReach, i + nums[i]);

        if (maxReach >= (int)nums.size() - 1) return true; // đã đủ xa để tới đích
    }

    return true;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — so với Backtracking thuần túy O(2ⁿ) thử mọi khả năng nhảy.

### 23.3.2. Gas Station

**Bài toán:** cho hai mảng `gas` (lượng xăng tại mỗi trạm) và `cost` (chi phí xăng để đi từ trạm đó đến trạm kế tiếp), tìm trạm xuất phát để có thể đi hết một vòng tròn, hoặc -1 nếu không thể.

**Bản chất Greedy:** hai quan sát quan trọng dẫn đến lời giải O(n):

1. Nếu tổng `gas` nhỏ hơn tổng `cost`, không thể hoàn thành vòng tròn từ bất kỳ điểm nào — trả về -1 ngay.
2. Nếu tổng `gas ≥ tổng cost` (đảm bảo tồn tại lời giải), điểm xuất phát hợp lệ chính là điểm **ngay sau** vị trí mà lượng xăng tích lũy (`tank`) chạm mức âm **thấp nhất** — vì bắt đầu từ đó, lượng xăng sẽ không bao giờ âm nữa trong suốt hành trình còn lại (đây là điểm "khó khăn nhất" đã bị vượt qua).

```cpp
#include <vector>
using namespace std;

int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int totalTank = 0, currentTank = 0;
    int startIndex = 0;

    for (int i = 0; i < (int)gas.size(); i++) {
        int diff = gas[i] - cost[i];
        totalTank += diff;
        currentTank += diff;

        if (currentTank < 0) {
            // Không thể xuất phát từ bất kỳ điểm nào trong đoạn [startIndex, i]
            startIndex = i + 1; // thử điểm xuất phát mới ngay sau vị trí thất bại
            currentTank = 0;    // reset lại lượng xăng tích lũy
        }
    }

    return totalTank >= 0 ? startIndex : -1;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — so với brute force O(n²) thử từng điểm xuất phát và mô phỏng toàn bộ vòng tròn.

### 23.3.3. Assign Cookies

**Bài toán:** cho mảng `g` (độ "thèm ăn" tối thiểu của mỗi trẻ em) và mảng `s` (kích thước mỗi cái bánh quy), mỗi trẻ chỉ hài lòng nếu được phát bánh có kích thước ≥ độ thèm ăn của mình, mỗi trẻ nhận tối đa 1 bánh. Tìm số trẻ tối đa có thể làm hài lòng.

**Bản chất Greedy (ứng dụng Sort + Greedy điển hình):** sắp xếp cả hai mảng tăng dần, dùng Two Pointers (Chương 4): luôn thử ghép trẻ có độ thèm ăn **nhỏ nhất chưa được phục vụ** với bánh **nhỏ nhất chưa dùng** — nếu bánh đủ lớn, ghép cặp thành công (dùng "vừa đủ", để dành bánh lớn hơn cho trẻ khó tính hơn); nếu không đủ, bỏ qua bánh đó (nó quá nhỏ để làm hài lòng bất kỳ trẻ nào còn lại, vì trẻ đang xét đã là trẻ dễ tính nhất).

```cpp
#include <vector>
#include <algorithm>
using namespace std;

int findContentChildren(vector<int>& g, vector<int>& s) {
    sort(g.begin(), g.end());
    sort(s.begin(), s.end());

    int childIdx = 0, cookieIdx = 0;
    int contentCount = 0;

    while (childIdx < (int)g.size() && cookieIdx < (int)s.size()) {
        if (s[cookieIdx] >= g[childIdx]) {
            contentCount++;
            childIdx++; // trẻ này đã được phục vụ, xét trẻ tiếp theo
        }
        cookieIdx++; // dù ghép được hay không, bánh này đã được xét xong
    }

    return contentCount;
}
```

**Độ phức tạp:** O(n log n + m log m) thời gian (chi phí sắp xếp hai mảng chiếm ưu thế), O(1) bộ nhớ phụ ngoài chi phí sắp xếp.

---

## 23.4. Greedy vs Dynamic Programming

### 23.4.1. Khi nào dùng Greedy, khi nào dùng DP

**Bản chất khác biệt:** cả hai kỹ thuật đều dựa trên Optimal Substructure (mục 23.1.2), nhưng khác nhau ở cách xử lý bài toán con:

- **Greedy** chỉ xét **một** lựa chọn tại mỗi bước (lựa chọn "tham lam" tốt nhất cục bộ) và không bao giờ xem xét lại — chỉ đúng khi Greedy Choice Property được thỏa mãn.
- **Dynamic Programming** (Chương 25) xét **tất cả các khả năng lựa chọn** tại mỗi bước, lưu lại kết quả tốt nhất cho từng bài toán con để tránh tính lại (memoization) — luôn cho ra lời giải đúng nếu bài toán có Optimal Substructure, kể cả khi Greedy Choice Property không thỏa mãn, nhưng đổi lại độ phức tạp thường cao hơn Greedy.

### 23.4.2. Counterexample — minh họa Greedy có thể sai

**Bài toán Coin Change** (đổi tiền với số lượng đồng xu tối thiểu) là ví dụ kinh điển cho thấy Greedy **không phải lúc nào cũng đúng**: với hệ đồng xu tiêu chuẩn `{1, 5, 10, 25}` (đơn vị cent của Mỹ), chiến lược Greedy "luôn chọn đồng xu lớn nhất có thể" cho ra lời giải tối ưu. Nhưng với hệ đồng xu `{1, 3, 4}`, để đổi giá trị `6`:

```
Greedy (luôn chọn đồng lớn nhất trước):
  6 → chọn 4 → còn 2 → chọn 1 → còn 1 → chọn 1 → còn 0
  Tổng: 4 + 1 + 1 = 3 đồng xu

Lời giải tối ưu thực sự:
  6 = 3 + 3
  Tổng: 2 đồng xu   ← ÍT HƠN, Greedy đã sai
```

**Giải thích vì sao Greedy thất bại ở đây:** với hệ đồng xu `{1, 3, 4}`, Greedy Choice Property **không được thỏa mãn** — việc chọn đồng lớn nhất (4) tại bước đầu **không dẫn đến** lời giải tối ưu toàn cục. Đây chính là dấu hiệu cho thấy bài toán này cần Dynamic Programming (sẽ giải chi tiết ở mục 26.5) để xét đầy đủ mọi khả năng thay vì tham lam chọn một hướng duy nhất.

### 23.4.3. Bảng so sánh tổng hợp

| Tiêu chí | Greedy | Dynamic Programming |
|---|---|---|
| Số lựa chọn xét tại mỗi bước | 1 (lựa chọn tham lam) | Nhiều (mọi khả năng) |
| Yêu cầu chứng minh đúng đắn | Bắt buộc (Exchange Argument hoặc tương đương) | Không cần — đúng đắn tự nhiên nếu có Optimal Substructure |
| Độ phức tạp điển hình | Thường O(n) hoặc O(n log n) | Thường O(n²) hoặc cao hơn, tùy số chiều trạng thái |
| Rủi ro | Có thể cho lời giải sai nếu áp dụng nhầm | Luôn đúng nếu mô hình hóa bài toán chính xác |
| Cách nhận diện | Có thể chứng minh lựa chọn cục bộ không bao giờ "thiệt" về sau | Bài toán con chồng lấn (overlapping subproblems), không có thứ tự tham lam rõ ràng |

---

## 23.5. Khi nào dùng Greedy

- Bài toán có thể chứng minh (hoặc trực giác mạnh mẽ gợi ý) rằng lựa chọn tối ưu cục bộ không bao giờ ảnh hưởng xấu đến khả năng đạt tối ưu toàn cục.
- Bài toán về khoảng/interval (scheduling, phân bổ tài nguyên theo thời gian).
- Bài toán đồ thị: Minimum Spanning Tree (Kruskal, Prim), Shortest Path với trọng số không âm (Dijkstra).
- Khi Brute Force hoặc DP cho độ phức tạp quá cao, và có thể tìm được tiêu chí sắp xếp/lựa chọn hợp lý để thử nghiệm chiến lược Greedy trước, sau đó kiểm chứng bằng test case hoặc phản ví dụ.

**Lời khuyên thực hành trong phỏng vấn:** khi nghi ngờ một bài toán có thể giải bằng Greedy, nên **thử tìm phản ví dụ** trước khi trình bày lời giải — nếu không tìm được phản ví dụ sau một vài thử nghiệm hợp lý, đó là tín hiệu tốt (dù chưa phải chứng minh chặt chẽ) để tự tin trình bày hướng Greedy với người phỏng vấn, đồng thời có thể đề cập ngắn gọn ý tưởng Exchange Argument nếu được hỏi "vì sao cách này đúng?".

---

## 23.6. Danh sách bài tập luyện tập

### Mức Easy
1. Assign Cookies
2. Lemonade Change
3. Best Time to Buy and Sell Stock II (Greedy đơn giản, cộng dồn mọi khoảng tăng giá)

### Mức Medium
4. Jump Game
5. Jump Game II (biến thể tối thiểu hóa số bước nhảy, không chỉ kiểm tra khả thi)
6. Gas Station
7. Merge Intervals (ôn lại từ mục 11.4.1)
8. Non-overlapping Intervals (ôn lại từ mục 11.4.2)
9. Partition Labels
10. Task Scheduler (ôn lại từ mục 9.4.3, kết hợp Greedy + Heap)

### Mức Hard
11. Candy (Greedy hai chiều — duyệt trái sang phải rồi phải sang trái)
12. Minimum Number of Taps to Open to Water a Garden

---

*Chương tiếp theo: **Chương 24 — DP Fundamentals**, chính thức hệ thống hóa Dynamic Programming — kỹ thuật đã được nhắc đến nhiều lần từ Chương 12 (Memoized Fibonacci) đến Chương 23 (giải quyết trường hợp Greedy thất bại) — thành một khung tư duy đầy đủ và có hệ thống.*
