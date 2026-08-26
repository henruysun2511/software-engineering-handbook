# CHƯƠNG 13: QUẢN LÝ CÔNG VIỆC TRONG TEAM

---

## 13.1. Tổng quan

Jira không chỉ là công cụ cá nhân để theo dõi công việc của bản thân — đây còn là nền tảng phối hợp nhóm, nơi mọi thành viên làm việc cùng nhau trong một không gian chung, minh bạch và có tổ chức.

Chương này tổng hợp các tính năng và thực hành liên quan đến việc **làm việc nhóm hiệu quả trên Jira** — từ phân công công việc, theo dõi tiến độ, trao đổi trong Issue, cho đến các công cụ nhỏ nhưng quan trọng như Label, Component và Checklist.

---

## 13.2. Phân công công việc (Task Assignment)

Phân công rõ ràng là nền tảng của mọi nhóm làm việc hiệu quả. Trong Jira, trách nhiệm cá nhân được thể hiện thông qua trường **Assignee** của mỗi Issue.

### Nguyên tắc phân công:

**Mỗi Issue chỉ có một Assignee duy nhất.** Khi một công việc có nhiều người cùng chịu trách nhiệm, thực tế là không ai chịu trách nhiệm. Nếu công việc cần nhiều người tham gia, chia thành Sub-task và phân công từng Sub-task cho từng người.

**Phân công dựa trên chuyên môn và tải công việc.** Không phân công cơ học theo kiểu "xoay vòng" — mà xem xét ai phù hợp nhất về kỹ năng và ai đang có dư capacity.

**Người nhận biết mình được giao việc.** Jira tự động gửi thông báo khi Assignee thay đổi, nhưng với công việc quan trọng, nên Mention trực tiếp trong Comment để đảm bảo người nhận chú ý.

### Các hình thức phân công:

**a) Phân công bởi Scrum Master / Tech Lead trong Sprint Planning:**
Đây là hình thức phổ biến nhất. Trong buổi Sprint Planning, nhóm thảo luận và quyết định ai làm Issue nào dựa trên chuyên môn và tải công việc.

**b) Tự nhận (Pull System):**
Thay vì đẩy công việc từ trên xuống, mỗi Developer tự chọn Issue từ cột "To Do" trên Board khi họ rảnh. Hệ thống này khuyến khích sự chủ động nhưng đòi hỏi nhóm có kỷ luật cao và hiểu rõ thứ tự ưu tiên.

**c) Phân công trong quá trình Sprint:**
Khi có Bug phát sinh hoặc Issue đột xuất được thêm vào giữa Sprint, Scrum Master hoặc Tech Lead phân công ngay dựa trên ai đang có capacity.

---

## 13.3. Theo dõi tiến độ cá nhân

Mỗi thành viên nên có thói quen theo dõi danh sách công việc của mình thường xuyên — không phải để bị kiểm soát mà để chủ động quản lý thời gian và ưu tiên đúng.

### Dashboard cá nhân:

Tạo một Dashboard riêng với Gadget **Assigned to Me** lọc tất cả Issue đang được giao, sắp xếp theo Priority hoặc Due Date. Đây là trang đầu tiên nên mở khi bắt đầu ngày làm việc.

### JQL theo dõi cá nhân:

```jql
-- Tất cả việc của tôi đang xử lý
assignee = currentUser() AND status != Done ORDER BY priority DESC

-- Việc của tôi trong Sprint hiện tại
assignee = currentUser() AND sprint in openSprints()

-- Việc của tôi sắp đến hạn
assignee = currentUser() AND due <= endOfWeek() AND status != Done

-- Việc của tôi bị quá hạn
assignee = currentUser() AND due < now() AND status != Done
```

### Cập nhật trạng thái Issue đúng lúc:

Một thành viên chuyên nghiệp không chỉ làm việc mà còn phản ánh đúng trạng thái thực tế lên Jira. Khi bắt đầu làm một Issue → chuyển sang "In Progress" ngay. Khi hoàn thành → chuyển sang trạng thái kế tiếp ngay. Không để Board hiển thị thông tin lạc hậu so với thực tế.

---

## 13.4. Theo dõi tiến độ nhóm

Scrum Master và Tech Lead cần góc nhìn tổng thể về toàn bộ nhóm — ai đang làm gì, ai đang bị tắc, phần nào của Sprint đang chậm.

### Các góc nhìn theo dõi nhóm:

**Scrum Board theo Swimlane Assignee:**
Cấu hình Board với Swimlane theo Assignee — mỗi hàng là một thành viên. Chỉ cần nhìn Board là thấy ngay khối lượng công việc và tiến độ của từng người trong Sprint.

**JQL theo dõi nhóm:**

```jql
-- Issue trong Sprint hiện tại chưa bắt đầu
sprint in openSprints() AND status = "To Do"

-- Issue đang bị tắc (In Progress quá lâu)
sprint in openSprints() AND status = "In Progress" AND updated <= -3d

-- Issue chưa có Assignee trong Sprint hiện tại
sprint in openSprints() AND assignee IS EMPTY

-- Tổng quan tiến độ theo thành viên
sprint in openSprints() ORDER BY assignee ASC, status ASC
```

**Burndown Chart hàng ngày:**
Xem Burndown Chart vào cuối mỗi ngày để đánh giá nhóm có đang đi đúng tiến độ hay không. Nếu đường thực tế liên tục cao hơn đường lý tưởng từ sớm, cần can thiệp ngay thay vì chờ đến cuối Sprint.

---

## 13.5. Comment và trao đổi trong Issue

Toàn bộ thông tin và trao đổi liên quan đến một công việc nên được ghi lại trong phần **Comment** của Issue — không phải trong email riêng, không phải trong chat ngoài luồng. Điều này đảm bảo mọi người đều có thể xem lại lịch sử trao đổi đầy đủ, kể cả thành viên mới tham gia sau.

### Những gì nên viết vào Comment:

- Cập nhật tiến độ: *"Đã hoàn thành phần backend, đang làm UI — dự kiến xong hôm nay."*
- Phát sinh vấn đề: *"Phát hiện API của bên thứ ba trả về format khác tài liệu, cần xác nhận lại."*
- Quyết định kỹ thuật: *"Sau khi thảo luận, quyết định dùng Redis để cache thay vì Memcached vì..."*
- Kết quả test: *"Đã test trên Chrome, Firefox, Safari — pass. Edge chưa test được."*
- Yêu cầu hỗ trợ: *"@NguyenVanA bạn xem giúp mình phần này với nhé, mình không chắc cách xử lý edge case này."*

### Định dạng Comment:

Jira hỗ trợ **rich text** trong Comment: in đậm, in nghiêng, danh sách có dấu đầu dòng, bảng, đoạn code (với syntax highlighting), và chèn ảnh trực tiếp. Sử dụng định dạng phù hợp giúp Comment dễ đọc hơn, đặc biệt khi đề cập đến đoạn code hay liệt kê nhiều điểm.

---

## 13.6. Mention thành viên

**Mention** (đề cập) là cách nhanh nhất để thông báo cho một thành viên cụ thể về một điều cần chú ý trong Issue.

### Cách dùng Mention:

Trong Comment hoặc Description, gõ ký tự **@** theo sau là tên hoặc username của người cần đề cập. Jira sẽ gợi ý danh sách người dùng phù hợp. Chọn đúng người → người đó sẽ nhận thông báo tức thì.

```
@TranThiB bạn có thể review PR này trước 3 giờ chiều không?
@LeVanC mình cần thông tin về API spec của module Payment nhé.
@NguyenVanA Issue này liên quan đến Bug WEB-45 mà bạn đang fix đó.
```

### Nguyên tắc dùng Mention hiệu quả:

- Mention khi cần **hành động cụ thể** từ người đó — không Mention để "cho biết"
- Không Mention cả nhóm (@all) cho mọi thứ — gây nhiễu thông báo và làm giảm hiệu quả
- Khi Mention, nêu rõ **muốn người đó làm gì** thay vì chỉ tag tên

---

## 13.7. Attachment (Tệp đính kèm)

**Attachment** cho phép đính kèm bất kỳ tệp nào vào Issue: ảnh chụp màn hình, bản thiết kế (Figma export), file Excel, tài liệu PDF, video demo, log lỗi...

### Cách đính kèm tệp:

- **Kéo thả:** Kéo file từ máy tính vào vùng Description hoặc Comment
- **Nút Attach:** Nhấn biểu tượng kẹp ghim (📎) trong thanh công cụ của Comment
- **Dán ảnh:** Nhấn `Ctrl+V` / `Cmd+V` để dán ảnh trực tiếp từ clipboard — cực kỳ tiện khi chụp màn hình

### Tệp nên đính kèm:

- **Bug Report:** Screenshot lỗi, video tái hiện, console log
- **UI Story:** File thiết kế, mockup, prototype link
- **Technical Task:** Diagram kiến trúc, API spec, tài liệu tham khảo
- **Meeting note:** Kết quả thảo luận liên quan đến Issue

---

## 13.8. Sub-task và Checklist

### Sub-task

**Sub-task** (đã giới thiệu trong Chương 3) là công cụ chia nhỏ một Issue lớn thành các phần có thể phân công và theo dõi riêng lẻ.

**Khi nào nên dùng Sub-task:**
- Story cần nhiều người với chuyên môn khác nhau (Frontend, Backend, QA) cùng tham gia
- Công việc có các bước rõ ràng, mỗi bước có thể hoàn thành và kiểm tra độc lập
- Cần theo dõi tiến độ chi tiết hơn mức Issue cha

**Tạo Sub-task:**
Mở Issue cha → cuộn xuống phần **Child Issues** → nhấn **Create child issue** → chọn Issue Type là **Sub-task** → điền thông tin.

### Checklist

Jira không có tính năng Checklist tích hợp sẵn trong phiên bản cơ bản, nhưng có hai cách phổ biến để tạo checklist:

**Cách 1 — Dùng Markdown trong Description:**

```
**Checklist hoàn thành:**
- [x] Viết unit test
- [x] Code review pass
- [ ] QA sign-off
- [ ] Update documentation
```

**Cách 2 — Dùng Sub-task:**
Tạo mỗi mục checklist thành một Sub-task riêng. Cách này cho phép phân công từng mục cho người khác nhau và theo dõi trạng thái từng mục trên Board.

---

## 13.9. Labels và Components

### Labels (Nhãn)

**Labels** là các thẻ tự do được gắn vào Issue để phân loại theo tiêu chí tùy ý. Khác với các trường cố định như Priority hay Issue Type, Labels hoàn toàn linh hoạt — nhóm tự định nghĩa và sử dụng theo nhu cầu.

**Ví dụ Labels phổ biến:**

| Label | Mục đích |
|---|---|
| `frontend` / `backend` / `fullstack` | Phân loại theo stack kỹ thuật |
| `performance` | Issue liên quan đến hiệu năng |
| `security` | Issue liên quan đến bảo mật |
| `tech-debt` | Nợ kỹ thuật cần giải quyết |
| `blocked` | Issue đang bị chặn bởi yếu tố bên ngoài |
| `quick-win` | Issue đơn giản, có thể làm nhanh |
| `needs-design` | Issue đang chờ bản thiết kế |

**Nguyên tắc dùng Labels:**
- Thống nhất trong nhóm về tập Labels được dùng — tránh mỗi người tự tạo Labels theo ý riêng
- Không tạo quá nhiều Labels — giảm hiệu quả lọc và khó nhớ
- Labels nên bổ sung thông tin mà các trường khác không có

### Components (Thành phần)

**Components** là đơn vị phân loại Issue theo module hoặc chức năng kỹ thuật trong dự án. Khác với Labels (tự do), Components được định nghĩa trước trong **Project Settings** và thường phản ánh kiến trúc của hệ thống.

**Ví dụ Components:**

```
Hệ thống thương mại điện tử:
├── Authentication
├── Product Catalog
├── Shopping Cart
├── Payment Gateway
├── Order Management
├── Notification Service
└── Admin Dashboard
```

**Lợi ích của Components:**
- Lọc nhanh toàn bộ Issue thuộc một module cụ thể
- Phân công Bug đến đúng người phụ trách module đó
- Thống kê số lượng Bug theo module — phát hiện khu vực code cần cải thiện

---

## 13.10. Deadline và Due Date

**Due Date** là ngày cần hoàn thành Issue. Không phải Issue nào cũng cần Due Date — nhưng với các Issue có cam kết thời gian cụ thể (deadline từ khách hàng, ngày phát hành, sự kiện cố định), Due Date là thông tin bắt buộc.

### Sử dụng Due Date hiệu quả:

**Đặt Due Date thực tế:** Due Date không phải để gây áp lực mà để lập kế hoạch. Đặt Due Date không thực tế (quá sát so với công sức thực tế) gây stress và mất ý nghĩa của trường này.

**Theo dõi Issue sắp đến hạn:**

```jql
due <= endOfWeek() AND status != Done
due < now() AND status != Done
```

**Thông báo tự động:** Jira có thể cấu hình gửi thông báo khi Due Date đến gần (qua Jira Automation), giúp Assignee và Reporter không bỏ lỡ deadline.

---

## 13.11. Notifications (Thông báo)

Jira tự động gửi thông báo khi có sự kiện liên quan đến Issue mà người dùng quan tâm. Hiểu cơ chế thông báo giúp tránh bị ngập lụt email và không bỏ sót thông tin quan trọng.

### Các sự kiện kích hoạt thông báo:

- Issue được tạo mới trong Project
- Issue được phân công cho tôi (tôi là Assignee)
- Comment được thêm vào Issue tôi đang theo dõi
- Tôi được Mention trong Comment hoặc Description
- Trạng thái Issue thay đổi
- Issue sắp đến Due Date

### Watch Issue (Theo dõi Issue):

Người dùng có thể tự thêm mình vào danh sách theo dõi của bất kỳ Issue nào — dù không phải Assignee hay Reporter — bằng cách nhấn nút **Watch** (biểu tượng mắt) trong Issue. Khi có bất kỳ cập nhật nào, người theo dõi sẽ nhận thông báo.

### Quản lý thông báo:

Vào **Profile → Notification preferences** để tùy chỉnh loại thông báo nào nhận qua email, loại nào nhận trong ứng dụng. Không tắt hết thông báo — nhưng cũng không nên nhận mọi thông báo trong Project lớn.

---

## 13.12. Văn hóa làm việc tốt trên Jira

Jira chỉ hoạt động hiệu quả khi toàn bộ nhóm sử dụng nhất quán. Dưới đây là một số thực hành văn hóa giúp Jira trở thành "nguồn sự thật duy nhất" (single source of truth) của nhóm:

**Cập nhật Jira là một phần của công việc, không phải việc phụ:**
Thông tin trong Jira phải luôn phản ánh thực tế. Cập nhật trạng thái, log thời gian, thêm comment không phải "overhead" — đây là trách nhiệm của mỗi thành viên với cả nhóm.

**Không trao đổi công việc hoàn toàn bên ngoài Jira:**
Các quyết định kỹ thuật quan trọng, thông tin về yêu cầu, kết quả kiểm tra — những gì có giá trị lâu dài nên được ghi lại trong Issue liên quan, dù ban đầu được thảo luận qua Slack hay cuộc họp.

**Issue rõ ràng từ đầu tiết kiệm thời gian về sau:**
Đầu tư thêm 5 phút để viết mô tả Issue rõ ràng có thể tiết kiệm hàng giờ giải thích qua lại sau đó.

**Giải quyết vấn đề công khai trong Issue:**
Khi gặp khó khăn hoặc cần hỗ trợ, đặt câu hỏi trong Comment của Issue — không nhắn tin riêng. Câu trả lời trong Comment có ích cho cả nhóm và được lưu lại cho người sau.

---

## Tóm tắt chương

Jira phát huy tối đa giá trị khi cả nhóm sử dụng nhất quán và có kỷ luật. Phân công rõ ràng, cập nhật trạng thái kịp thời, trao đổi minh bạch trong Issue — đây là những thực hành đơn giản nhưng tạo nên sự khác biệt lớn giữa một nhóm dùng Jira như công cụ thực sự và một nhóm chỉ dùng Jira như hình thức.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Assignee | Người chịu trách nhiệm thực hiện Issue |
| Pull System | Thành viên tự chọn và nhận công việc từ Board |
| Comment | Kênh trao đổi chính thức và lưu trữ trong Issue |
| Mention (@) | Thông báo đến một thành viên cụ thể |
| Attachment | Tệp đính kèm minh họa hoặc bổ sung thông tin |
| Sub-task | Chia nhỏ Issue lớn thành các phần có thể phân công riêng |
| Label | Nhãn tự do phân loại Issue theo tiêu chí tùy ý |
| Component | Phân loại Issue theo module hoặc chức năng kỹ thuật |
| Due Date | Ngày cần hoàn thành Issue |
| Watch | Theo dõi Issue để nhận thông báo cập nhật |

---

*Chương tiếp theo: **Chương 14 — Quy trình Scrum Thực tế trên Jira***
