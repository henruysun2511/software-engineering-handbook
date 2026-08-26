# CHƯƠNG 9: JQL & SEARCH — TÌM KIẾM VÀ LỌC ISSUE NÂNG CAO

---

## 9.1. Tổng quan về tìm kiếm trong Jira

Khi dự án phát triển theo thời gian, số lượng Issue tích lũy trong Jira có thể lên đến hàng trăm, thậm chí hàng nghìn. Trong bối cảnh đó, khả năng tìm kiếm và lọc Issue chính xác trở thành kỹ năng không thể thiếu của bất kỳ ai làm việc thường xuyên với Jira.

Jira cung cấp hai chế độ tìm kiếm chính:

- **Basic Search (Tìm kiếm cơ bản):** Giao diện trực quan với các bộ lọc dạng nút bấm và menu thả xuống, phù hợp với người mới hoặc khi cần lọc theo các tiêu chí đơn giản.
- **Advanced Search với JQL (Tìm kiếm nâng cao):** Ngôn ngữ truy vấn dạng văn bản, cho phép tạo các điều kiện lọc phức tạp, linh hoạt và chính xác hơn nhiều so với Basic Search.

---

## 9.2. Basic Search (Tìm kiếm cơ bản)

**Basic Search** là điểm khởi đầu lý tưởng cho người mới làm quen với tìm kiếm trong Jira. Giao diện cung cấp sẵn các bộ lọc phổ biến nhất dưới dạng menu thả xuống, không yêu cầu người dùng biết cú pháp hay ngôn ngữ truy vấn.

### Cách truy cập:

Trên thanh điều hướng → nhấn **Issues** → chọn **Search for Issues**. Trang tìm kiếm mở ra với chế độ Basic Search mặc định.

### Các bộ lọc có sẵn trong Basic Search:

| Bộ lọc | Mô tả |
|---|---|
| **Project** | Lọc Issue thuộc một hoặc nhiều Project cụ thể |
| **Issue Type** | Lọc theo loại Issue (Epic, Story, Task, Bug...) |
| **Status** | Lọc theo trạng thái hiện tại |
| **Assignee** | Lọc theo người được giao |
| **Reporter** | Lọc theo người tạo Issue |
| **Priority** | Lọc theo mức độ ưu tiên |
| **Label** | Lọc theo nhãn |
| **Sprint** | Lọc theo Sprint cụ thể |

Người dùng chọn giá trị từ các menu, Jira tự động cập nhật danh sách kết quả theo thời gian thực. Có thể kết hợp nhiều bộ lọc cùng lúc.

### Hạn chế của Basic Search:

Basic Search chỉ đáp ứng được các truy vấn đơn giản. Khi cần các điều kiện phức tạp hơn — ví dụ: "Issue được cập nhật trong 7 ngày qua, thuộc Sprint hiện tại, chưa có Assignee và có Priority là High" — Basic Search không đủ khả năng. Đây là lúc cần đến JQL.

---

## 9.3. JQL là gì?

**JQL** (viết tắt của *Jira Query Language*) là ngôn ngữ truy vấn chuyên dụng của Jira, cho phép người dùng xây dựng các câu lệnh tìm kiếm với độ chính xác và linh hoạt cao. JQL có cú pháp tương tự SQL — ngôn ngữ truy vấn cơ sở dữ liệu — nhưng được thiết kế riêng cho dữ liệu Issue trong Jira.

Một câu lệnh JQL cơ bản có cấu trúc:

```
[Trường] [Toán tử] [Giá trị]
```

Ví dụ:

```
assignee = "nguyen.van.a"
status = "In Progress"
priority = High
```

Nhiều điều kiện có thể kết hợp bằng các toán tử logic `AND`, `OR`, `NOT`.

### Tại sao nên học JQL?

JQL là công cụ mạnh mẽ nhất để khai thác dữ liệu từ Jira. Thành thạo JQL giúp người dùng:
- Tạo các báo cáo tùy chỉnh chính xác theo nhu cầu
- Lưu các bộ lọc thường dùng để tái sử dụng
- Tạo Dashboard thông minh từ các Filter JQL
- Theo dõi công việc của toàn nhóm một cách hệ thống

---

## 9.4. Cú pháp JQL cơ bản

### Cấu trúc câu lệnh đơn:

```
field operator value
```

Trong đó:
- **field:** Tên trường thông tin của Issue (project, status, assignee, priority...)
- **operator:** Toán tử so sánh (=, !=, >, <, IN, NOT IN, IS, IS NOT...)
- **value:** Giá trị cần so sánh (có thể là text, số, ngày tháng hoặc hàm đặc biệt)

### Các câu lệnh JQL thường gặp:

**Lọc theo Project:**
```jql
project = "WEB"
project IN ("WEB", "APP", "MKT")
```

**Lọc theo Assignee:**
```jql
assignee = "nguyen.van.a"
assignee = currentUser()
assignee is EMPTY
```

**Lọc theo Status:**
```jql
status = "In Progress"
status IN ("In Progress", "Code Review", "Testing")
status != Done
```

**Lọc theo Priority:**
```jql
priority = High
priority IN (Highest, High)
```

**Lọc theo Issue Type:**
```jql
issuetype = Bug
issuetype IN (Story, Task)
```

**Lọc theo Sprint:**
```jql
sprint = "Sprint 5"
sprint in openSprints()
sprint in closedSprints()
```

**Lọc theo thời gian:**
```jql
created >= "2024-01-01"
updated >= -7d
due <= "2024-12-31"
```

**Kết hợp nhiều điều kiện:**
```jql
project = "WEB" AND status = "In Progress" AND assignee = currentUser()
project = "APP" AND priority IN (Highest, High) AND status != Done
```

---

## 9.5. Các toán tử JQL

### Toán tử so sánh:

| Toán tử | Ý nghĩa | Ví dụ |
|---|---|---|
| `=` | Bằng | `status = Done` |
| `!=` | Khác | `status != Done` |
| `>` | Lớn hơn | `priority > Medium` |
| `<` | Nhỏ hơn | `priority < High` |
| `>=` | Lớn hơn hoặc bằng | `created >= "2024-01-01"` |
| `<=` | Nhỏ hơn hoặc bằng | `due <= "2024-06-30"` |

### Toán tử tập hợp:

| Toán tử | Ý nghĩa | Ví dụ |
|---|---|---|
| `IN` | Thuộc tập hợp | `status IN ("To Do", "In Progress")` |
| `NOT IN` | Không thuộc tập hợp | `assignee NOT IN ("user1", "user2")` |

### Toán tử kiểm tra rỗng:

| Toán tử | Ý nghĩa | Ví dụ |
|---|---|---|
| `IS EMPTY` | Trường không có giá trị | `assignee IS EMPTY` |
| `IS NOT EMPTY` | Trường có giá trị | `due IS NOT EMPTY` |

### Toán tử lịch sử (Historical Operators):

| Toán tử | Ý nghĩa | Ví dụ |
|---|---|---|
| `WAS` | Từng có giá trị | `status WAS "In Progress"` |
| `WAS IN` | Từng thuộc tập hợp | `status WAS IN ("Testing", "Review")` |
| `CHANGED` | Đã thay đổi | `status CHANGED` |

Toán tử lịch sử đặc biệt hữu ích khi cần tìm Issue đã từng ở một trạng thái nào đó — dù hiện tại đã chuyển sang trạng thái khác.

### Toán tử logic:

| Toán tử | Ý nghĩa | Ví dụ |
|---|---|---|
| `AND` | Tất cả điều kiện đều đúng | `project = WEB AND status = Done` |
| `OR` | Ít nhất một điều kiện đúng | `priority = High OR priority = Highest` |
| `NOT` | Phủ định điều kiện | `NOT status = Done` |

---

## 9.6. Hàm đặc biệt trong JQL

JQL cung cấp một số hàm dựng sẵn giúp tạo các điều kiện động — tức là điều kiện tự động thay đổi theo ngữ cảnh mà không cần cập nhật thủ công.

| Hàm | Ý nghĩa |
|---|---|
| `currentUser()` | Người dùng đang đăng nhập |
| `now()` | Thời điểm hiện tại |
| `startOfDay()` | Đầu ngày hôm nay |
| `endOfDay()` | Cuối ngày hôm nay |
| `startOfWeek()` | Đầu tuần này |
| `endOfWeek()` | Cuối tuần này |
| `startOfMonth()` | Đầu tháng này |
| `openSprints()` | Các Sprint đang Active |
| `closedSprints()` | Các Sprint đã đóng |
| `membersOf("group")` | Thành viên của một nhóm |

### Ví dụ sử dụng hàm:

```jql
-- Issue của tôi đang xử lý trong Sprint hiện tại
assignee = currentUser() AND sprint in openSprints() AND status != Done

-- Issue được tạo trong tuần này và chưa có người nhận
created >= startOfWeek() AND assignee IS EMPTY

-- Bug chưa xong, được cập nhật trong 3 ngày gần nhất
issuetype = Bug AND status != Done AND updated >= -3d

-- Issue quá hạn chưa hoàn thành
due < now() AND status != Done
```

---

## 9.7. ORDER BY — Sắp xếp kết quả

Mặc định, kết quả tìm kiếm JQL được sắp xếp theo mức độ liên quan. Người dùng có thể chỉ định thứ tự sắp xếp bằng mệnh đề `ORDER BY`:

```jql
project = WEB ORDER BY priority DESC
project = APP AND status = "In Progress" ORDER BY updated ASC
project = MKT ORDER BY created DESC, priority ASC
```

Các trường thường dùng để sắp xếp: `priority`, `created`, `updated`, `due`, `assignee`, `status`.

---

## 9.8. Lưu Search và Filter

Sau khi tạo một câu lệnh JQL hữu ích, người dùng có thể **lưu lại thành Filter** để tái sử dụng mà không cần gõ lại từ đầu mỗi lần.

### Cách lưu Filter:

**Bước 1:** Tạo câu lệnh JQL trong ô tìm kiếm Advanced Search.

**Bước 2:** Nhấn **Save as** (hoặc **Save filter**) ở phía trên kết quả tìm kiếm.

**Bước 3:** Đặt tên Filter ngắn gọn, dễ nhận biết. Ví dụ: *"My open issues this sprint"*, *"Critical bugs unassigned"*.

**Bước 4:** Nhấn **Submit** để lưu.

### Chia sẻ Filter:

Filter có thể được chia sẻ với:
- **Chỉ mình tôi (Private):** Mặc định khi tạo mới
- **Nhóm cụ thể:** Chia sẻ với một nhóm thành viên
- **Toàn bộ tổ chức:** Mọi người trong Jira đều thấy và dùng được

### Sử dụng Filter đã lưu:

Filter đã lưu xuất hiện trong menu **Issues → My Filters** hoặc **Issues → All Filters**. Filter cũng có thể được dùng làm nguồn dữ liệu cho Dashboard Gadget — cho phép hiển thị danh sách Issue tùy chỉnh trực tiếp trên Dashboard.

---

## 9.9. Một số câu JQL thực tế hay dùng

Dưới đây là tập hợp các câu JQL phổ biến trong thực tế làm việc hàng ngày:

```jql
-- Tất cả Issue đang được giao cho tôi, chưa xong
assignee = currentUser() AND status != Done ORDER BY priority DESC

-- Bug chưa xử lý, ưu tiên cao
issuetype = Bug AND status = "To Do" AND priority IN (Highest, High)

-- Issue trong Sprint hiện tại bị quá hạn
sprint in openSprints() AND due < now() AND status != Done

-- Issue chưa có người nhận trong Sprint hiện tại
sprint in openSprints() AND assignee IS EMPTY

-- Tất cả Epic đang mở
issuetype = Epic AND status != Done

-- Issue được tạo trong tháng này
created >= startOfMonth()

-- Issue không có Story Point (chưa estimate)
issuetype in (Story, Task) AND "Story Points" is EMPTY

-- Issue tôi đã báo cáo và chưa được xử lý
reporter = currentUser() AND status = "To Do"
```

---

## Tóm tắt chương

JQL là kỹ năng quan trọng giúp khai thác tối đa dữ liệu trong Jira. Bắt đầu từ Basic Search cho các truy vấn đơn giản, sau đó dần chuyển sang JQL khi cần độ chính xác và linh hoạt cao hơn. Lưu các Filter thường dùng để tiết kiệm thời gian và tái sử dụng cho Dashboard.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Basic Search | Tìm kiếm đơn giản qua giao diện nút bấm |
| JQL | Ngôn ngữ truy vấn nâng cao của Jira |
| Field | Trường thông tin của Issue dùng trong JQL |
| Operator | Toán tử so sánh hoặc logic trong JQL |
| `currentUser()` | Hàm trả về người dùng đang đăng nhập |
| `openSprints()` | Hàm trả về các Sprint đang Active |
| Filter | Câu lệnh JQL được lưu lại để tái sử dụng |
| ORDER BY | Mệnh đề sắp xếp kết quả tìm kiếm |

---

*Chương tiếp theo: **Chương 10 — Bug Management: Quản lý Lỗi Phần mềm***
