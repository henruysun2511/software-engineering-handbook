# CHƯƠNG 1: TỔNG QUAN VỀ JIRA

---

## 1.1. Jira là gì?

Jira là một nền tảng quản lý dự án và theo dõi công việc được phát triển bởi công ty **Atlassian** (Úc), ra mắt lần đầu vào năm 2002. Ban đầu, Jira được thiết kế chủ yếu phục vụ việc theo dõi lỗi phần mềm (bug tracking), tuy nhiên theo thời gian, nền tảng này đã phát triển thành một công cụ quản lý dự án toàn diện, được sử dụng rộng rãi trong nhiều lĩnh vực khác nhau.

Tên gọi **"Jira"** được cho là bắt nguồn từ tên rút gọn của *Gojira* — tên tiếng Nhật của nhân vật Godzilla — vốn là biệt danh nội bộ mà đội ngũ Atlassian đặt cho công cụ này trong giai đoạn phát triển ban đầu, để phân biệt với một công cụ tương tự là Bugzilla.

Tính đến nay, Jira là một trong những công cụ quản lý dự án phổ biến nhất thế giới, được tin dùng bởi hàng chục nghìn tổ chức — từ các startup nhỏ cho đến các tập đoàn công nghệ lớn như NASA, Spotify, Airbnb và Twitter.

---

## 1.2. Jira được sử dụng để làm gì?

Jira không chỉ là công cụ quản lý lỗi (bug tracker) mà còn đảm nhận nhiều vai trò quan trọng trong vòng đời phát triển phần mềm cũng như quản lý dự án nói chung.

### Các mục đích sử dụng chính:

**a) Quản lý công việc và tiến độ**

Jira cho phép tạo, phân công và theo dõi các đầu công việc (gọi là *Issue*) trong suốt vòng đời của dự án. Mỗi thành viên trong nhóm biết rõ mình cần làm gì, đang ở bước nào và cần hoàn thành khi nào.

**b) Lập kế hoạch Sprint và quản lý Backlog**

Với các nhóm phát triển phần mềm áp dụng mô hình Agile/Scrum, Jira hỗ trợ lập kế hoạch theo từng chu kỳ phát triển ngắn (Sprint), quản lý danh sách công việc chờ xử lý (Backlog) và theo dõi tiến độ theo thời gian thực.

**c) Theo dõi và xử lý lỗi phần mềm**

Jira cung cấp quy trình chuyên biệt để ghi nhận, phân loại, phân công và theo dõi trạng thái xử lý của từng lỗi (Bug), giúp đảm bảo không có lỗi nào bị bỏ sót trong quá trình phát triển.

**d) Phối hợp làm việc nhóm**

Các thành viên có thể trao đổi trực tiếp trong từng đầu công việc thông qua tính năng bình luận (Comment), đề cập đồng nghiệp (Mention), và đính kèm tài liệu liên quan (Attachment).

**e) Báo cáo và đánh giá hiệu suất**

Jira tích hợp các công cụ báo cáo trực quan như Burndown Chart, Velocity Chart, Sprint Report — giúp nhóm và quản lý dự án nắm bắt kịp thời tiến độ thực tế so với kế hoạch.

---

## 1.3. Jira trong quy trình Agile/Scrum

Jira được thiết kế với triết lý phù hợp chặt chẽ với phương pháp phát triển phần mềm **Agile**, đặc biệt là khung làm việc **Scrum** — một trong những phương pháp Agile phổ biến nhất hiện nay.

### Agile là gì?

**Agile** là một triết lý phát triển phần mềm nhấn mạnh vào tính linh hoạt, khả năng thích ứng nhanh với thay đổi và sự phối hợp chặt chẽ giữa các thành viên trong nhóm. Thay vì lập kế hoạch toàn bộ dự án từ đầu đến cuối, Agile chia dự án thành các chu kỳ nhỏ, mỗi chu kỳ tạo ra một phần sản phẩm có thể kiểm tra và phản hồi được.

### Scrum là gì?

**Scrum** là một khung triển khai Agile cụ thể, trong đó công việc được tổ chức thành các **Sprint** — thường kéo dài từ 1 đến 4 tuần. Mỗi Sprint có mục tiêu rõ ràng, danh sách công việc cụ thể và kết thúc bằng một buổi đánh giá (Sprint Review) và cải tiến quy trình (Sprint Retrospective).

### Jira hỗ trợ Scrum như thế nào?

| Hoạt động Scrum | Tính năng tương ứng trong Jira |
|---|---|
| Quản lý Backlog | Backlog view |
| Sprint Planning | Tạo Sprint, kéo Issue vào Sprint |
| Theo dõi tiến độ hàng ngày | Scrum Board, Burndown Chart |
| Sprint Review | Sprint Report |
| Quản lý Issue | Epic, Story, Task, Bug... |

Nhờ sự tương thích này, Jira trở thành công cụ mặc định được hầu hết các nhóm phát triển phần mềm áp dụng Scrum lựa chọn.

---

## 1.4. Các khái niệm cơ bản trong Jira

Trước khi đi vào từng tính năng cụ thể, người dùng cần nắm vững một số khái niệm nền tảng trong Jira. Đây là những "từ khóa" sẽ xuất hiện xuyên suốt trong toàn bộ tài liệu này.

### 1.4.1. Project (Dự án)

**Project** là đơn vị tổ chức lớn nhất trong Jira. Mỗi Project đại diện cho một dự án, một sản phẩm hoặc một nhóm công việc có liên quan đến nhau. Toàn bộ Issue, Board, Sprint và các cấu hình khác đều được tổ chức bên trong một Project nhất định.

Ví dụ: Một công ty có thể tạo các Project riêng biệt như *Website Redesign*, *Mobile App Development*, *Marketing Campaign Q3*...

### 1.4.2. Issue (Đầu công việc)

**Issue** là đơn vị công việc cơ bản nhất trong Jira. Mọi thứ cần được theo dõi — từ một tính năng cần phát triển, một nhiệm vụ cụ thể, cho đến một lỗi phần mềm — đều được biểu diễn dưới dạng Issue.

Issue có nhiều loại khác nhau như Epic, Story, Task, Sub-task và Bug. Mỗi loại phục vụ một mục đích riêng và sẽ được trình bày chi tiết trong Chương 3.

### 1.4.3. Board (Bảng công việc)

**Board** là giao diện trực quan hiển thị toàn bộ trạng thái công việc của nhóm. Các Issue được sắp xếp thành các cột tương ứng với từng giai đoạn trong quy trình làm việc, chẳng hạn: *To Do* → *In Progress* → *Done*.

Board giúp toàn bộ nhóm nhìn thấy được bức tranh tổng thể về tiến độ dự án chỉ trong một màn hình.

### 1.4.4. Backlog (Danh sách công việc chờ)

**Backlog** là tập hợp tất cả các Issue đã được tạo ra nhưng chưa được đưa vào bất kỳ Sprint nào. Đây là nơi lưu trữ, sắp xếp và ưu tiên hóa toàn bộ công việc của dự án.

Trong mô hình Scrum, Backlog được chia thành **Product Backlog** (toàn bộ danh sách công việc của sản phẩm) và **Sprint Backlog** (danh sách công việc được chọn cho một Sprint cụ thể).

### 1.4.5. Sprint (Chu kỳ phát triển)

**Sprint** là một khoảng thời gian cố định — thường từ 1 đến 4 tuần — trong đó nhóm phát triển cam kết hoàn thành một tập hợp công việc đã được xác định trước. Mỗi Sprint bắt đầu bằng buổi lập kế hoạch (Sprint Planning) và kết thúc bằng buổi đánh giá (Sprint Review) cùng buổi cải tiến quy trình (Retrospective).

### 1.4.6. Workflow (Quy trình làm việc)

**Workflow** định nghĩa các trạng thái mà một Issue có thể đi qua trong suốt vòng đời của nó, cùng các điều kiện để chuyển từ trạng thái này sang trạng thái khác. Một Workflow điển hình trong phát triển phần mềm có thể là:

> **To Do** → **In Progress** → **Code Review** → **Testing** → **Done**

Workflow đảm bảo mọi thành viên trong nhóm tuân theo cùng một quy trình làm việc nhất quán.

### 1.4.7. Dashboard (Bảng điều khiển)

**Dashboard** là trang tổng hợp cho phép người dùng theo dõi nhiều thông tin quan trọng cùng lúc thông qua các widget (gadget) trực quan. Dashboard có thể được tùy chỉnh theo nhu cầu của từng cá nhân hoặc nhóm, hiển thị các thông tin như danh sách Issue đang xử lý, biểu đồ tiến độ Sprint, thống kê hiệu suất nhóm...

---

## 1.5. Tại sao nên sử dụng Jira?

Có nhiều công cụ quản lý dự án trên thị trường, tuy nhiên Jira nổi bật nhờ một số ưu điểm đặc thù:

**Thiết kế hướng đến quy trình phát triển phần mềm**

Không giống các công cụ quản lý dự án đa năng thông thường, Jira được xây dựng với hiểu biết sâu sắc về vòng đời phát triển phần mềm. Các khái niệm như Epic, Story, Sprint hay Bug Workflow đều phản ánh thực tiễn của ngành.

**Tích hợp hệ sinh thái phong phú**

Jira tích hợp liền mạch với hàng trăm công cụ khác trong quy trình phát triển phần mềm như GitHub, GitLab, Bitbucket (quản lý mã nguồn), Confluence (tài liệu), Slack (giao tiếp nhóm), Jenkins, và nhiều công cụ CI/CD khác.

**Khả năng tùy chỉnh cao**

Workflow, trường thông tin (field), quyền truy cập và giao diện Board đều có thể được cấu hình linh hoạt để phù hợp với quy trình làm việc đặc thù của từng nhóm.

**Báo cáo và minh bạch thông tin**

Jira cung cấp các công cụ báo cáo tích hợp giúp nhóm theo dõi tiến độ, phát hiện sớm điểm tắc nghẽn và đưa ra quyết định dựa trên dữ liệu thực tế.

---

## 1.6. Hạn chế cần lưu ý

Mặc dù là công cụ mạnh mẽ, Jira cũng tồn tại một số điểm cần lưu ý đối với người dùng mới:

- **Đường cong học tập tương đối dốc:** Do có nhiều tính năng và khái niệm chuyên biệt, người mới cần thời gian để làm quen với giao diện và quy trình làm việc trong Jira.
- **Dễ bị lạm dụng cấu hình:** Nếu không có quy ước thống nhất trong nhóm, Jira có thể trở nên phức tạp và khó quản lý theo thời gian.
- **Chi phí:** Jira là phần mềm thương mại. Phiên bản miễn phí có giới hạn về số lượng thành viên và tính năng; các tính năng nâng cao đòi hỏi đăng ký gói trả phí.

---

## 1.7. Hai loại Project trong Jira — Điều cần biết trước khi bắt đầu

Khi tạo Project mới trong Jira, người dùng phải chọn một trong hai loại: **Company-managed** (Classic) hoặc **Team-managed** (Next-gen). Đây là lựa chọn quan trọng vì sau khi tạo, không thể chuyển đổi giữa hai loại.

**Toàn bộ tài liệu này được viết theo góc nhìn của người dùng Team-managed** — loại phổ biến hơn với các nhóm nhỏ và startup vì dễ cài đặt, không cần Jira Admin.

| | **Team-managed** | **Company-managed** |
|---|---|---|
| **Tên cũ** | Next-gen | Classic |
| **Quản lý bởi** | Bất kỳ thành viên nào | Jira Administrator |
| **Cấu hình** | Đơn giản, trực tiếp trong Project | Phức tạp, qua Schemes & Screens |
| **Workflow** | Chỉnh trực tiếp trên Board | Tùy chỉnh sâu qua Admin panel |
| **Phù hợp** | Team nhỏ, startup, dự án đơn lẻ | Tổ chức lớn, nhiều team, cần chuẩn hóa |
| **Time Tracking** | Không hỗ trợ | Có hỗ trợ |
| **Báo cáo nâng cao** | Giới hạn | Đầy đủ hơn |

> **Ghi chú cho người dùng Company-managed:** Các khái niệm cơ bản (Issue, Sprint, Board, Backlog...) hoàn toàn giống nhau. Sự khác biệt chủ yếu nằm ở **cách thao tác cài đặt và cấu hình**, không phải ở quy trình làm việc hàng ngày. Các chương sau sẽ ghi chú rõ tại những điểm có sự khác biệt đáng kể.

---

## Tóm tắt chương

Chương 1 đã giới thiệu những nền tảng cơ bản nhất về Jira — từ nguồn gốc, mục đích sử dụng, sự liên hệ với phương pháp Agile/Scrum, cho đến các khái niệm cốt lõi mà người dùng sẽ gặp xuyên suốt quá trình làm việc với nền tảng này. Việc nắm vững các khái niệm trong chương này là điều kiện tiên quyết để tiếp cận hiệu quả các nội dung chuyên sâu hơn ở các chương tiếp theo.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Project | Không gian tổ chức toàn bộ dự án |
| Issue | Đơn vị công việc cơ bản |
| Board | Bảng trực quan theo dõi trạng thái công việc |
| Backlog | Danh sách công việc chờ được xử lý |
| Sprint | Chu kỳ làm việc có thời hạn cố định |
| Workflow | Quy trình chuyển trạng thái của công việc |
| Dashboard | Bảng tổng hợp thông tin dự án |

---

*Chương tiếp theo: **Chương 2 — Jira Project: Tổ chức và Quản lý Dự án***
