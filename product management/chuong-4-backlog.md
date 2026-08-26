# CHƯƠNG 4: BACKLOG — QUẢN LÝ DANH SÁCH CÔNG VIỆC

---

## 4.1. Backlog là gì?

**Backlog** trong Jira là danh sách tổng hợp tất cả các Issue đã được tạo ra trong một Project nhưng chưa được đưa vào bất kỳ Sprint nào. Đây là nơi tập trung toàn bộ công việc "đang chờ" của dự án — từ các tính năng lớn cần phát triển, các cải tiến nhỏ, cho đến các lỗi chưa được xử lý.

Nếu Board là nơi thể hiện công việc **đang được thực hiện**, thì Backlog là nơi thể hiện công việc **sẽ được thực hiện**. Đây là hai góc nhìn bổ trợ lẫn nhau, giúp nhóm vừa kiểm soát được hiện tại vừa lên kế hoạch cho tương lai.

Backlog thường được ví như một "hàng đợi thông minh" — không phải mọi thứ đều quan trọng như nhau, và thứ tự trong Backlog phản ánh mức độ ưu tiên mà nhóm đã thống nhất.

---

## 4.2. Product Backlog trong Scrum

Trong phương pháp Scrum, Backlog có vai trò trung tâm và được phân thành hai cấp độ:

### Product Backlog

**Product Backlog** là danh sách đầy đủ, có thứ tự ưu tiên, của tất cả công việc cần thực hiện để xây dựng và hoàn thiện sản phẩm. Product Backlog thuộc trách nhiệm quản lý của **Product Owner** — người chịu trách nhiệm định hướng sản phẩm và xác định đâu là những thứ quan trọng nhất cần làm trước.

Product Backlog không bao giờ "hoàn chỉnh" — nó liên tục được bổ sung, điều chỉnh và sắp xếp lại theo phản hồi từ người dùng, yêu cầu từ stakeholder và kết quả từ các Sprint đã hoàn thành.

### Sprint Backlog

**Sprint Backlog** là tập hợp con của Product Backlog — bao gồm các Issue được nhóm chọn để thực hiện trong một Sprint cụ thể. Sprint Backlog được xác định trong buổi Sprint Planning và được cố định trong suốt thời gian của Sprint (ngoại trừ một số điều chỉnh nhỏ).

| | Product Backlog | Sprint Backlog |
|---|---|---|
| **Phạm vi** | Toàn bộ sản phẩm | Một Sprint cụ thể |
| **Người quản lý** | Product Owner | Development Team |
| **Thay đổi** | Liên tục | Hạn chế trong Sprint |
| **Thời gian** | Dài hạn | Ngắn hạn (1–4 tuần) |

---

## 4.3. Quản lý Backlog trên Jira

Trong Jira, Backlog được truy cập qua menu điều hướng bên trái của Project, mục **Backlog**. Giao diện Backlog hiển thị toàn bộ Issue chưa được đưa vào Sprint, được sắp xếp theo thứ tự từ trên xuống dưới — Issue ở trên cùng có mức độ ưu tiên cao hơn.

### Giao diện Backlog gồm các thành phần chính:

**Phần trên — Các Sprint đang mở:** Hiển thị các Sprint đang được lập kế hoạch hoặc đang chạy, cùng danh sách Issue đã được kéo vào Sprint đó.

**Phần dưới — Backlog:** Danh sách tất cả Issue chưa thuộc Sprint nào, sắp xếp theo thứ tự ưu tiên.

Người dùng có thể tương tác trực tiếp với Backlog theo nhiều cách: tạo Issue mới, kéo thả để sắp xếp thứ tự, lọc theo nhiều tiêu chí khác nhau, hoặc kéo Issue vào Sprint.

---

## 4.4. Sắp xếp thứ tự ưu tiên (Prioritization)

Sắp xếp thứ tự ưu tiên trong Backlog là một trong những hoạt động quan trọng và đòi hỏi nhiều suy nghĩ nhất trong quản lý dự án. Thứ tự trong Backlog phản ánh câu trả lời cho câu hỏi: *"Nếu chỉ có thể làm một việc tiếp theo, nhóm sẽ làm gì?"*

### Các yếu tố ảnh hưởng đến mức độ ưu tiên:

- **Giá trị kinh doanh (Business Value):** Issue mang lại lợi ích gì cho người dùng hoặc tổ chức?
- **Rủi ro và sự phụ thuộc:** Issue này có bị chặn bởi Issue khác không? Có rủi ro kỹ thuật nào cần giải quyết sớm không?
- **Deadline và cam kết:** Có thời hạn cố định nào cần tuân thủ không?
- **Công sức thực hiện (Effort):** Issue quá lớn so với giá trị mang lại thì nên xem xét lại hoặc chia nhỏ.

### Cách sắp xếp trong Jira:

Trong giao diện Backlog, giữ và kéo (drag & drop) Issue lên hoặc xuống để thay đổi thứ tự ưu tiên. Issue ở vị trí cao hơn sẽ được xem xét đưa vào Sprint trước.

---

## 4.5. Estimation (Ước lượng công sức)

**Estimation** là quá trình nhóm phát triển đánh giá mức độ phức tạp hoặc công sức cần thiết để hoàn thành một Issue. Đây là hoạt động tập thể — cả nhóm cùng thảo luận và thống nhất, không phải quyết định đơn phương của một cá nhân.

Estimation không phải là dự đoán chính xác thời gian thực hiện — mà là một ước lượng tương đối dựa trên sự hiểu biết chung của nhóm về độ phức tạp của công việc.

---

## 4.6. Story Point

**Story Point** là đơn vị đo lường phổ biến nhất được dùng trong Estimation. Không giống như giờ làm việc (hour), Story Point không đo thời gian tuyệt đối mà đo **độ phức tạp tương đối** của một Issue so với các Issue khác.

### Tại sao dùng Story Point thay vì giờ?

- **Tránh áp lực thời gian:** Ước lượng theo giờ dễ bị ảnh hưởng bởi cảm xúc và áp lực hoàn thành đúng hạn. Story Point tập trung vào bản chất công việc, không phải thời gian.
- **Phản ánh tốc độ nhóm (Velocity):** Tổng số Story Point hoàn thành trong mỗi Sprint cho thấy năng lực thực sự của nhóm — được gọi là Velocity.
- **Dễ thảo luận hơn:** Thay vì tranh luận "việc này mất 2 hay 3 giờ?", nhóm thảo luận "việc này phức tạp hơn hay tương đương với việc kia?".

### Thang điểm Story Point phổ biến:

Nhiều nhóm sử dụng dãy số **Fibonacci** để đánh giá Story Point: `1, 2, 3, 5, 8, 13, 21...`

Lý do sử dụng Fibonacci là vì khoảng cách giữa các con số tăng dần, phản ánh sự thật rằng càng ước lượng công việc lớn thì càng có nhiều sai số — và nên phân loại lớn hơn là cố đoán chính xác.

| Story Point | Mức độ |
|---|---|
| 1–2 | Rất nhỏ, đơn giản |
| 3–5 | Trung bình |
| 8 | Lớn, phức tạp |
| 13+ | Rất lớn, nên xem xét chia nhỏ |

> **Nguyên tắc thực hành:** Issue có Story Point từ 13 trở lên thường quá lớn để hoàn thành trong một Sprint — nhóm nên xem xét chia nhỏ thành nhiều Issue.

### Cách ghi Story Point trong Jira:

Mở Issue → tìm trường **Story Points** (hoặc **Story point estimate**) → nhập con số ước lượng. Trường này chỉ hiển thị với các loại Issue phù hợp như Story và Task.

---

## 4.7. Backlog Refinement (Làm mịn Backlog)

**Backlog Refinement** (còn gọi là *Backlog Grooming*) là buổi họp định kỳ của nhóm nhằm xem xét, làm rõ và chuẩn bị các Issue trong Backlog trước khi chúng được đưa vào Sprint.

### Mục tiêu của Backlog Refinement:

- Đảm bảo các Issue có mô tả đầy đủ, rõ ràng và sẵn sàng để thực hiện
- Loại bỏ hoặc gộp các Issue không còn phù hợp
- Chia nhỏ các Issue quá lớn thành các phần có thể thực hiện được trong một Sprint
- Ước lượng Story Point cho các Issue chưa được estimate
- Sắp xếp lại thứ tự ưu tiên nếu cần

### Tần suất:

Thông thường, Backlog Refinement được tổ chức một đến hai lần mỗi Sprint, không kéo dài quá 1–2 giờ. Đây không phải là buổi họp quyết định ai làm gì — mà là buổi chuẩn bị để Sprint Planning diễn ra suôn sẻ hơn.

---

## 4.8. Epic và cách tổ chức Backlog

Với các dự án lớn, Backlog có thể chứa hàng trăm Issue và trở nên khó quản lý nếu không có cách tổ chức hợp lý. Jira cho phép sử dụng **Epic** như một công cụ nhóm và lọc Issue trong Backlog.

### Epic Panel trong Backlog:

Ở phía bên trái của giao diện Backlog, Jira hiển thị **Epic Panel** — danh sách các Epic đang có trong Project. Khi chọn một Epic, Backlog sẽ tự động lọc và chỉ hiển thị các Issue thuộc Epic đó, giúp nhóm tập trung vào một phần công việc cụ thể.

### Màu sắc của Epic:

Mỗi Epic trong Jira được gán một màu sắc riêng. Màu này xuất hiện dưới dạng nhãn màu bên cạnh tên Issue trong Backlog và Board, giúp nhận biết nhanh một Issue thuộc Epic nào chỉ qua màu sắc mà không cần đọc tên.

---

## 4.9. Đưa Issue vào Sprint

Sau khi Backlog đã được sắp xếp và các Issue đã được estimate, bước tiếp theo là chọn Issue để đưa vào Sprint sắp tới trong buổi **Sprint Planning**.

### Cách đưa Issue vào Sprint trong Jira:

**Cách 1 — Kéo thả:** Trong giao diện Backlog, giữ và kéo Issue từ vùng Backlog lên vùng Sprint muốn đưa vào.

**Cách 2 — Chuột phải:** Nhấp chuột phải vào Issue → chọn **Send to Sprint** → chọn Sprint muốn đưa vào.

**Cách 3 — Chọn nhiều Issue cùng lúc:** Giữ phím `Shift` hoặc `Ctrl` và nhấp chọn nhiều Issue → nhấp chuột phải → **Send to Sprint**.

### Lựa chọn Issue nào để đưa vào Sprint?

Nhóm nên căn cứ vào ba yếu tố chính:

1. **Thứ tự ưu tiên:** Issue ở trên cùng Backlog là ưu tiên cao nhất.
2. **Capacity của nhóm:** Tổng Story Point của các Issue được chọn không nên vượt quá Velocity trung bình của nhóm trong các Sprint trước.
3. **Sprint Goal:** Các Issue được chọn nên hướng đến một mục tiêu chung, rõ ràng cho Sprint.

---

## Tóm tắt chương

Backlog là trái tim của quá trình lập kế hoạch trong Jira. Một Backlog được quản lý tốt — có thứ tự ưu tiên rõ ràng, Issue được mô tả đầy đủ và estimate hợp lý — là điều kiện tiên quyết để các Sprint diễn ra suôn sẻ và hiệu quả.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Backlog | Danh sách tất cả Issue chưa vào Sprint |
| Product Backlog | Toàn bộ công việc cần làm cho sản phẩm |
| Sprint Backlog | Công việc được chọn cho một Sprint cụ thể |
| Prioritization | Sắp xếp Issue theo mức độ quan trọng |
| Estimation | Ước lượng độ phức tạp của Issue |
| Story Point | Đơn vị đo độ phức tạp tương đối |
| Backlog Refinement | Buổi họp làm rõ và chuẩn bị Issue trước Sprint |
| Velocity | Tổng Story Point nhóm hoàn thành trong một Sprint |

---

*Chương tiếp theo: **Chương 5 — Sprint: Quản lý Chu kỳ Phát triển***
