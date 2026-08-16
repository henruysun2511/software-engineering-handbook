# Chương 7: Stack (Ngăn xếp)

## 7.1. Khái niệm cốt lõi

### 7.1.1. Định nghĩa

Stack là cấu trúc dữ liệu tuyến tính hoạt động theo nguyên tắc **LIFO (Last In, First Out)** — phần tử được thêm vào sau cùng sẽ là phần tử được lấy ra đầu tiên. Ba thao tác cơ bản: `push` (thêm phần tử lên đỉnh), `pop` (lấy và loại bỏ phần tử đỉnh), `peek`/`top` (xem phần tử đỉnh mà không loại bỏ).

### 7.1.2. Bản chất — vì sao Stack luôn đạt O(1)

Stack có thể cài đặt trên nền Array (thêm/bớt tại cuối mảng) hoặc trên nền Linked List (thêm/bớt tại đầu danh sách). Điểm chung: **mọi thao tác chỉ xảy ra tại một đầu duy nhất** (đỉnh stack) — đây chính là lý do Stack luôn đạt O(1) cho `push`/`pop`/`peek`, vì nó chỉ tận dụng đúng những thao tác vốn đã O(1) của Array (chèn/xóa cuối, mục 1.2) hoặc Linked List (chèn/xóa đầu, mục 6.2), không bao giờ cần thao tác ở giữa cấu trúc.

**Minh họa** với chuỗi thao tác `push(1), push(2), push(3), pop()`:

```
push(1):  [1]
push(2):  [1, 2]
push(3):  [1, 2, 3]
                  ↑ đỉnh (top)
pop():    [1, 2]     → trả về giá trị 3 (phần tử vào sau cùng, ra trước tiên)
```

### 7.1.3. Call Stack — mối liên hệ với đệ quy

Bản chất của việc gọi hàm đệ quy (đã đề cập ở mục 6.4.1 khi so sánh Reverse Linked List iterative và recursive) chính là thao tác trên một **Stack ẩn** do hệ thống quản lý (Call Stack): mỗi lần gọi hàm, một "khung" (frame) chứa biến cục bộ và địa chỉ trở về được `push` vào Call Stack; khi hàm kết thúc, khung đó được `pop` ra và luồng thực thi quay lại đúng vị trí đã gọi. Đây là lý do đệ quy có độ sâu lớn có thể gây tràn stack (stack overflow) — cũng là căn cứ giải thích độ phức tạp không gian O(n) của lời giải đệ quy đã nêu ở bài tập mục 0.3.

---

## 7.2. Ứng dụng và kỹ thuật quan trọng

### 7.2.1. Parentheses Matching (Kiểm tra dấu ngoặc hợp lệ)

**Bản chất:** một chuỗi dấu ngoặc hợp lệ có tính chất **lồng nhau đúng thứ tự** — dấu ngoặc mở gần nhất phải được đóng trước các dấu ngoặc mở trước đó, đúng bản chất LIFO của Stack. Mỗi khi gặp dấu mở, `push` vào stack; mỗi khi gặp dấu đóng, so sánh với đỉnh stack — nếu khớp cặp thì `pop`, nếu không khớp thì chuỗi không hợp lệ.

### 7.2.2. Expression Evaluation (Tính giá trị biểu thức)

**Bản chất:** Stack được dùng để chuyển đổi và tính giá trị biểu thức toán học, đặc biệt hữu ích khi biểu thức có nhiều toán tử với độ ưu tiên khác nhau hoặc chứa dấu ngoặc — dùng một Stack lưu toán hạng (operand) và một Stack lưu toán tử (operator), xử lý theo độ ưu tiên khi gặp toán tử mới.

### 7.2.3. Monotonic Stack (Ngăn xếp đơn điệu)

**Bản chất:** Monotonic Stack là một Stack luôn duy trì tính chất **đơn điệu tăng hoặc giảm** từ đáy lên đỉnh — mỗi khi một phần tử mới sắp được `push` vào mà phá vỡ tính đơn điệu, các phần tử ở đỉnh vi phạm sẽ bị `pop` ra trước. Kỹ thuật này giải quyết hiệu quả lớp bài toán "tìm phần tử lớn hơn/nhỏ hơn gần nhất" mà brute force cần O(n²) (so sánh mọi cặp), đưa về O(n) nhờ quan sát quan trọng: **một khi phần tử A đã bị loại vì có phần tử B lớn hơn đứng sau nó, A sẽ không bao giờ là câu trả lời cho bất kỳ phần tử nào xét sau B nữa** — vì B luôn là lựa chọn tốt hơn A. Điều này đảm bảo mỗi phần tử chỉ bị `push` một lần và `pop` tối đa một lần trong suốt thuật toán, cho tổng độ phức tạp O(n) (một dạng phân tích amortized khác đã gặp ở mục 0.1.5 và mục 3.3.5).

**Minh họa** tìm "Next Greater Element" cho `arr = [2, 1, 2, 4, 3]` bằng Monotonic Stack giảm dần (lưu chỉ số):

```
i=0, arr[0]=2: stack rỗng → push(0)           stack (chỉ số): [0]
i=1, arr[1]=1: 1 < arr[top]=2 → push(1)        stack: [0, 1]
i=2, arr[2]=2: 2 > arr[1]=1 → pop, result[1]=2
               2 == arr[0]=2 → không pop nữa (chỉ pop khi NHỎ HƠN), push(2)
                                                stack: [0, 2]
i=3, arr[3]=4: 4 > arr[2]=2 → pop, result[2]=4
               4 > arr[0]=2 → pop, result[0]=4
               stack rỗng → push(3)            stack: [3]
i=4, arr[4]=3: 3 < arr[3]=4 → push(4)          stack: [3, 4]

Kết thúc: các chỉ số còn lại trong stack (3, 4) không có Next Greater → result = -1
```

---

## 7.3. Cài đặt các bài toán kinh điển

### 7.3.1. Valid Parentheses

**Bài toán:** kiểm tra chuỗi chỉ chứa `()[]{}`  có hợp lệ (mở đóng đúng cặp, đúng thứ tự lồng nhau) hay không.

**Cài đặt C++:**

```cpp
#include <string>
#include <stack>
#include <unordered_map>
using namespace std;

bool isValid(const string& s) {
    stack<char> st;
    unordered_map<char, char> matchPair = {{')', '('}, {']', '['}, {'}', '{'}};

    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            st.push(c);
        } else {
            // c là dấu đóng: phải khớp với đỉnh stack hiện tại
            if (st.empty() || st.top() != matchPair[c]) return false;
            st.pop();
        }
    }

    return st.empty(); // stack rỗng nghĩa là mọi dấu mở đều đã được đóng đúng cặp
}
```

**Độ phức tạp:** O(n) thời gian, O(n) bộ nhớ phụ trong trường hợp xấu nhất (toàn bộ ký tự là dấu mở).

### 7.3.2. Min Stack

**Bài toán:** thiết kế Stack hỗ trợ thêm thao tác `getMin()` trả về giá trị nhỏ nhất hiện có trong stack, tất cả thao tác đều O(1).

**Bản chất:** dùng thêm một stack phụ `minStack` lưu song song giá trị nhỏ nhất **tính đến thời điểm mỗi lần push** — không phải chỉ giá trị nhỏ nhất toàn cục, mà là một "lịch sử" giá trị nhỏ nhất tương ứng với từng trạng thái của stack chính, để khi `pop` khỏi stack chính, ta cũng biết chính xác giá trị nhỏ nhất của trạng thái stack *trước đó*.

**Cài đặt C++:**

```cpp
#include <stack>
using namespace std;

class MinStack {
private:
    stack<int> mainStack;
    stack<int> minStack; // minStack.top() luôn là giá trị nhỏ nhất hiện tại

public:
    void push(int val) {
        mainStack.push(val);
        if (minStack.empty() || val <= minStack.top()) {
            minStack.push(val);
        } else {
            minStack.push(minStack.top()); // lặp lại min hiện tại, giữ đồng bộ độ sâu
        }
    }

    void pop() {
        mainStack.pop();
        minStack.pop();
    }

    int top() {
        return mainStack.top();
    }

    int getMin() {
        return minStack.top();
    }
};
```

**Độ phức tạp:** O(1) cho mọi thao tác, đánh đổi bằng O(n) bộ nhớ phụ cho stack thứ hai.

### 7.3.3. Next Greater Element (Monotonic Stack)

**Bài toán:** với mỗi phần tử trong mảng, tìm phần tử lớn hơn gần nhất phía bên phải nó; nếu không có, kết quả là -1.

**Cài đặt C++:**

```cpp
#include <vector>
#include <stack>
using namespace std;

vector<int> nextGreaterElement(const vector<int>& arr) {
    int n = arr.size();
    vector<int> result(n, -1);
    stack<int> st; // lưu CHỈ SỐ, duy trì tính đơn điệu giảm dần theo giá trị

    for (int i = 0; i < n; i++) {
        // Khi arr[i] lớn hơn phần tử tại đỉnh stack, đó chính là Next Greater của nó
        while (!st.empty() && arr[st.top()] < arr[i]) {
            result[st.top()] = arr[i];
            st.pop();
        }
        st.push(i);
    }

    return result; // các chỉ số còn lại trong stack giữ nguyên giá trị -1
}
```

**Độ phức tạp:** O(n) thời gian — mỗi chỉ số được `push` đúng một lần và `pop` tối đa một lần; O(n) bộ nhớ phụ. So với brute force O(n²) (với mỗi phần tử, quét toàn bộ phần tử phía sau).

### 7.3.4. Daily Temperatures

**Bài toán:** với mỗi ngày, tìm số ngày phải chờ đến khi có một ngày nhiệt độ cao hơn; nếu không có ngày nào, kết quả là 0.

**Bản chất:** cùng khuôn mẫu Monotonic Stack như Next Greater Element, chỉ khác kết quả trả về là **khoảng cách chỉ số** thay vì giá trị.

**Cài đặt C++:**

```cpp
#include <vector>
#include <stack>
using namespace std;

vector<int> dailyTemperatures(const vector<int>& temperatures) {
    int n = temperatures.size();
    vector<int> result(n, 0);
    stack<int> st; // lưu chỉ số, đơn điệu giảm dần theo nhiệt độ

    for (int i = 0; i < n; i++) {
        while (!st.empty() && temperatures[st.top()] < temperatures[i]) {
            int prevIndex = st.top();
            st.pop();
            result[prevIndex] = i - prevIndex; // khoảng cách ngày chờ
        }
        st.push(i);
    }

    return result;
}
```

**Độ phức tạp:** O(n) thời gian, O(n) bộ nhớ phụ.

### 7.3.5. Largest Rectangle in Histogram

**Bài toán:** cho mảng chiều cao các cột liền kề độ rộng 1, tìm diện tích hình chữ nhật lớn nhất có thể tạo thành.

**Bản chất:** với mỗi cột `i`, hình chữ nhật có chiều cao `height[i]` sẽ mở rộng được sang trái và phải cho đến khi gặp cột **thấp hơn** nó — tức cần tìm "Previous Smaller Element" và "Next Smaller Element" cho từng cột, đây chính là Monotonic Stack tăng dần. Cài đặt tối ưu tính toán cả hai chiều trong một lượt duyệt duy nhất bằng cách xử lý diện tích ngay khi một phần tử bị `pop` khỏi stack (tại thời điểm đó, ta đã biết cả biên trái lẫn biên phải của nó).

**Cài đặt C++:**

```cpp
#include <vector>
#include <stack>
#include <algorithm>
using namespace std;

int largestRectangleArea(const vector<int>& heights) {
    stack<int> st; // lưu chỉ số, đơn điệu tăng dần theo chiều cao
    int maxArea = 0;
    int n = heights.size();

    for (int i = 0; i <= n; i++) {
        // Dùng chiều cao 0 khi i == n để đảm bảo mọi phần tử còn lại đều được pop và xử lý
        int currentHeight = (i == n) ? 0 : heights[i];

        while (!st.empty() && heights[st.top()] > currentHeight) {
            int height = heights[st.top()];
            st.pop();
            // Biên trái là chỉ số ngay dưới đỉnh stack sau khi pop, biên phải là i
            int width = st.empty() ? i : i - st.top() - 1;
            maxArea = max(maxArea, height * width);
        }
        st.push(i);
    }

    return maxArea;
}
```

**Độ phức tạp:** O(n) thời gian, O(n) bộ nhớ phụ — so với brute force O(n²) (mở rộng thủ công từ mỗi cột).

---

## 7.4. Bảng tổng hợp

| Bài toán | Kỹ thuật | Độ phức tạp |
|---|---|---|
| Valid Parentheses | Stack cơ bản | O(n) |
| Min Stack | Stack song song | O(1) mỗi thao tác |
| Next Greater Element | Monotonic Stack giảm dần | O(n) |
| Daily Temperatures | Monotonic Stack giảm dần | O(n) |
| Largest Rectangle in Histogram | Monotonic Stack tăng dần | O(n) |

---

## 7.5. Danh sách bài tập luyện tập

### Mức Easy
1. Valid Parentheses
2. Baseball Game
3. Implement Stack using Queues (so sánh đánh đổi khi cài đặt cấu trúc này bằng cấu trúc khác)
4. Remove All Adjacent Duplicates In String

### Mức Medium
5. Min Stack
6. Evaluate Reverse Polish Notation
7. Next Greater Element II (mảng vòng — circular array)
8. Daily Temperatures
9. Decode String
10. Asteroid Collision

### Mức Hard
11. Largest Rectangle in Histogram
12. Basic Calculator
13. Maximal Rectangle (mở rộng 2D của Largest Rectangle in Histogram)
14. Trapping Rain Water (thử giải lại bằng Monotonic Stack, so sánh với cách Two Pointers ở mục 4.3.4)

---

*Chương tiếp theo: **Chương 8 — Queue & Deque**, giới thiệu nguyên tắc FIFO đối lập với LIFO của Stack, nền tảng cho thuật toán BFS ở các chương Graph.*
