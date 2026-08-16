# Chương 6: Linked List (Danh sách liên kết)

## 6.1. Khái niệm cốt lõi

### 6.1.1. Định nghĩa

Linked List là cấu trúc dữ liệu tuyến tính gồm các **node** (nút), mỗi node lưu một giá trị dữ liệu và một (hoặc nhiều) con trỏ trỏ đến node kế tiếp. Khác với Array, các node của Linked List **không nhất thiết nằm liên tục trong bộ nhớ** — chúng được liên kết với nhau thông qua con trỏ, cho phép cấp phát và giải phóng từng node độc lập tại thời điểm chạy.

### 6.1.2. Bản chất — đánh đổi giữa truy cập và chỉnh sửa

Đây là hệ quả trực tiếp của việc từ bỏ tính liên tục bộ nhớ đã tận dụng ở Array (mục 1.1.2). Vì các node phân tán trong bộ nhớ, không có công thức tính địa chỉ trực tiếp như `base + i*size` — muốn đến node thứ `i`, bắt buộc phải **đi qua tuần tự** từ node đầu, khiến truy cập theo chỉ số tốn O(n). Đổi lại, việc chèn hoặc xóa một node **khi đã có sẵn con trỏ đến vị trí đó** chỉ cần thay đổi một vài con trỏ, không cần dịch chuyển phần tử như Array, nên tốn O(1).

**Minh họa cấu trúc Singly Linked List** với giá trị `10 → 20 → 30 → nullptr`:

```
head
 ↓
[10 | next] → [20 | next] → [30 | next] → nullptr
```

**Minh họa thao tác chèn node giá trị 15 vào giữa 10 và 20** (đã có con trỏ `prev` trỏ đến node chứa 10):

```
Trước:  [10] → [20] → [30] → nullptr

Bước 1: tạo node mới [15], cho newNode.next = prev.next (tức trỏ đến [20])
Bước 2: cho prev.next = newNode

Sau:    [10] → [15] → [20] → [30] → nullptr
```

Toàn bộ thao tác chỉ gồm hai phép gán con trỏ — O(1), không phụ thuộc số lượng node phía sau.

### 6.1.3. Singly Linked List và Doubly Linked List

**Singly Linked List:** mỗi node chỉ lưu con trỏ `next` trỏ tới node kế tiếp, chỉ có thể duyệt theo một chiều (từ đầu đến cuối).

**Doubly Linked List:** mỗi node lưu thêm con trỏ `prev` trỏ ngược lại node liền trước, cho phép duyệt hai chiều và **xóa một node bất kỳ trong O(1)** nếu đã có con trỏ đến chính node đó (không cần con trỏ đến node liền trước như Singly Linked List, vì đã có sẵn `prev`).

```
Doubly:  nullptr ← [10] ⇄ [20] ⇄ [30] → nullptr
                    prev,next    prev,next
```

**Cài đặt cấu trúc node cơ bản trong C++:**

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};

struct DoublyListNode {
    int val;
    DoublyListNode* prev;
    DoublyListNode* next;
    DoublyListNode(int x) : val(x), prev(nullptr), next(nullptr) {}
};
```

---

## 6.2. Các thao tác cơ bản và độ phức tạp

| Thao tác | Array (Dynamic) | Singly Linked List | Doubly Linked List |
|---|---|---|---|
| Truy cập theo chỉ số | O(1) | O(n) | O(n) |
| Chèn/xóa đầu | O(n) | O(1) | O(1) |
| Chèn/xóa cuối | O(1) amortized | O(n)* | O(1)** |
| Chèn/xóa giữa (đã có con trỏ vị trí) | O(n) | O(n)*** | O(1) |
| Tìm kiếm giá trị | O(n) | O(n) | O(n) |

*\* O(n) nếu không giữ con trỏ `tail`, vì phải duyệt từ đầu để tìm node cuối.*
*\*\* O(1) nếu giữ sẵn con trỏ `tail`.*
*\*\*\* Dù đã có con trỏ đến node cần xóa, Singly Linked List vẫn cần O(n) để tìm node **liền trước** nó (do không có `prev`), trừ trường hợp chèn/xóa ngay sau một node đã biết.*

---

## 6.3. Kỹ thuật quan trọng

### 6.3.1. Dummy Node (Node giả)

**Bản chất:** khi thao tác chèn/xóa có thể xảy ra ngay tại vị trí `head` (ví dụ xóa chính node đầu tiên), code xử lý thường phải viết riêng một nhánh đặc biệt cho trường hợp này, làm phức tạp logic. Kỹ thuật Dummy Node tạo một node giả đặt **trước** `head` thực sự, biến `head` thành "node thứ hai" — nhờ đó mọi thao tác chèn/xóa, kể cả tại vị trí đầu danh sách, đều có thể xử lý bằng **cùng một đoạn code** không cần rẽ nhánh đặc biệt.

```cpp
ListNode dummy(0);
dummy.next = head;
ListNode* prev = &dummy;
// ... xử lý thống nhất, không cần if riêng cho trường hợp xóa head
return dummy.next; // trả về head thực sự (có thể đã thay đổi)
```

### 6.3.2. Fast & Slow Pointer

**Bản chất:** dùng hai con trỏ duyệt danh sách với **tốc độ khác nhau** — thường `slow` đi một bước, `fast` đi hai bước mỗi lần lặp. Kỹ thuật này khai thác một tính chất hình học đơn giản: nếu danh sách có chu trình (cycle), `fast` chắc chắn sẽ "vòng lại" và gặp `slow` sau một số hữu hạn bước (giống hai người chạy trên đường đua vòng tròn với tốc độ khác nhau, người nhanh chắc chắn sẽ đuổi kịp và vượt qua người chậm); nếu danh sách không có chu trình, `fast` sẽ chạm `nullptr` trước.

### 6.3.3. Reverse Linked List (Đảo ngược danh sách)

**Bản chất:** đảo ngược từng liên kết `next` một, cần lưu tạm ba con trỏ tại mỗi bước (node trước, node hiện tại, node sau) để không làm mất dấu vết danh sách khi đã đảo chiều con trỏ.

---

## 6.4. Cài đặt các bài toán kinh điển

### 6.4.1. Reverse Linked List

**Cài đặt C++ (Iterative):**

```cpp
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    ListNode* curr = head;

    while (curr != nullptr) {
        ListNode* nextTemp = curr->next; // lưu tạm trước khi ghi đè
        curr->next = prev;               // đảo chiều liên kết
        prev = curr;                     // dịch chuyển prev
        curr = nextTemp;                 // dịch chuyển curr
    }

    return prev; // prev là head mới sau khi đảo ngược
}
```

**Cài đặt C++ (Recursive):**

```cpp
ListNode* reverseListRecursive(ListNode* head) {
    if (head == nullptr || head->next == nullptr) return head;

    ListNode* newHead = reverseListRecursive(head->next);
    head->next->next = head; // node kế tiếp trỏ ngược lại head
    head->next = nullptr;    // head trở thành node cuối, next = null

    return newHead;
}
```

**Độ phức tạp:** cả hai cách đều O(n) thời gian. Cách iterative dùng O(1) bộ nhớ phụ; cách recursive dùng O(n) bộ nhớ do ngăn xếp lời gọi đệ quy (call stack) — minh chứng cụ thể cho đánh đổi độ phức tạp không gian đã nêu ở bài tập mục 0.3.

### 6.4.2. Detect Cycle (Floyd's Cycle Detection)

**Bài toán:** kiểm tra một Linked List có chứa chu trình hay không.

**Cài đặt C++:**

```cpp
bool hasCycle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;

    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true; // hai con trỏ gặp nhau → có chu trình
    }

    return false; // fast chạm nullptr → không có chu trình
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — vượt trội so với cách dùng HashSet lưu các node đã thăm (O(n) bộ nhớ phụ).

**Mở rộng — tìm điểm bắt đầu chu trình:** sau khi `slow` và `fast` gặp nhau, đặt một con trỏ thứ ba tại `head`, cho nó và `slow` cùng di chuyển mỗi lần một bước — điểm chúng gặp nhau chính là node bắt đầu chu trình. Tính chất này xuất phát từ chứng minh toán học về khoảng cách giữa điểm gặp nhau và điểm bắt đầu chu trình bằng khoảng cách từ `head` đến điểm bắt đầu chu trình.

### 6.4.3. Find Middle Node

**Bài toán:** tìm node ở giữa Linked List trong một lượt duyệt.

**Cài đặt C++:**

```cpp
ListNode* middleNode(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;

    // Khi fast đến cuối (đi 2 bước), slow đang ở đúng giữa (đi 1 bước)
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
    }

    return slow;
}
```

**Độ phức tạp:** O(n) thời gian, O(1) bộ nhớ phụ — so với cách đếm số node trước rồi duyệt lại lần hai (vẫn O(n) nhưng duyệt hai lượt).

### 6.4.4. Merge Two Sorted Lists

**Bài toán:** gộp hai Linked List đã sắp xếp thành một danh sách sắp xếp duy nhất — bản chất tương tự mục 1.6.7 (Merge Sorted Arrays) nhưng thao tác trên con trỏ thay vì chỉ số mảng.

**Cài đặt C++:**

```cpp
ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
    ListNode dummy(0);       // Dummy Node để thống nhất logic
    ListNode* tail = &dummy;

    while (list1 != nullptr && list2 != nullptr) {
        if (list1->val <= list2->val) {
            tail->next = list1;
            list1 = list1->next;
        } else {
            tail->next = list2;
            list2 = list2->next;
        }
        tail = tail->next;
    }

    // Nối phần còn lại của danh sách chưa duyệt hết
    tail->next = (list1 != nullptr) ? list1 : list2;

    return dummy.next;
}
```

**Độ phức tạp:** O(n + m) thời gian với n, m là độ dài hai danh sách, O(1) bộ nhớ phụ (chỉ tái sử dụng node có sẵn, không tạo node mới).

### 6.4.5. Remove N-th Node From End

**Bài toán:** xóa node thứ `n` tính từ cuối danh sách, chỉ trong một lượt duyệt.

**Bản chất:** dùng hai con trỏ với khoảng cách cố định `n` node giữa chúng. Cho con trỏ `fast` đi trước `n` bước, sau đó cả `fast` và `slow` cùng di chuyển — khi `fast` chạm cuối danh sách, `slow` đang đứng ngay trước node cần xóa (nhờ khoảng cách cố định đã thiết lập).

**Cài đặt C++:**

```cpp
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode dummy(0);
    dummy.next = head;
    ListNode* slow = &dummy;
    ListNode* fast = &dummy;

    // Đưa fast đi trước n+1 bước (tính cả dummy) để slow dừng đúng trước node cần xóa
    for (int i = 0; i <= n; i++) {
        fast = fast->next;
    }

    while (fast != nullptr) {
        slow = slow->next;
        fast = fast->next;
    }

    ListNode* toDelete = slow->next;
    slow->next = slow->next->next;
    delete toDelete;

    return dummy.next;
}
```

**Độ phức tạp:** O(n) thời gian (một lượt duyệt), O(1) bộ nhớ phụ — so với cách đếm độ dài danh sách trước rồi duyệt lại lần hai.

---

## 6.5. So sánh Linked List với Array

*(Bảng chi tiết đã trình bày ở mục 1.5; dưới đây bổ sung góc nhìn từ phía Linked List)*

| Tiêu chí | Ưu thế thuộc về |
|---|---|
| Truy cập ngẫu nhiên theo chỉ số | Array |
| Cache locality (hiệu năng thực tế khi duyệt) | Array |
| Chèn/xóa tại vị trí đã biết con trỏ | Linked List |
| Không cần biết trước kích thước tối đa | Linked List (không cần lo tràn capacity/resize) |
| Tiết kiệm bộ nhớ overhead mỗi phần tử | Array (không cần lưu con trỏ) |

**Khi nào ưu tiên Linked List:** cài đặt các cấu trúc dữ liệu khác cần chèn/xóa O(1) tại đầu/cuối (Stack, Queue — xem Chương 7, 8), cài đặt LRU Cache (kết hợp HashMap + Doubly Linked List), hoặc khi kích thước dữ liệu biến động liên tục mà không muốn trả giá resize của Dynamic Array.

---

## 6.6. Danh sách bài tập luyện tập

### Mức Easy
1. Reverse Linked List
2. Merge Two Sorted Lists
3. Linked List Cycle
4. Middle of the Linked List
5. Remove Duplicates from Sorted List
6. Palindrome Linked List (kết hợp Fast & Slow Pointer + Reverse)

### Mức Medium
7. Remove N-th Node From End of List
8. Add Two Numbers (biểu diễn số nguyên dạng Linked List)
9. Reorder List
10. Odd Even Linked List
11. Copy List with Random Pointer (kết hợp HashMap — xem lại Chương 3)
12. Rotate List

### Mức Hard
13. Merge K Sorted Lists (kết hợp Heap — xem Chương 9)
14. Reverse Nodes in k-Group
15. LRU Cache (kết hợp HashMap + Doubly Linked List)

---

*Chương tiếp theo: **Chương 7 — Stack**, giới thiệu cấu trúc LIFO và các ứng dụng nền tảng như kiểm tra tính hợp lệ của biểu thức và kỹ thuật Monotonic Stack.*
