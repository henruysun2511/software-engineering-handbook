# CHƯƠNG 3: ISSUE — ĐƠN VỊ CÔNG VIỆC CƠ BẢN TRONG JIRA

---

## 3.1. Issue là gì?

**Issue** là đơn vị công việc nhỏ nhất và cũng là thành phần trung tâm trong toàn bộ hệ thống Jira. Mọi thứ cần được theo dõi, quản lý hay xử lý trong một dự án — dù là một tính năng cần phát triển, một nhiệm vụ cụ thể, một lỗi phần mềm hay một yêu cầu cải tiến — đều được ghi nhận dưới dạng Issue.

Mỗi Issue mang một mã định danh duy nhất (Issue Key) theo cấu trúc `[PROJECT-KEY]-[Số thứ tự]`. Ví dụ: `WEB-42` là Issue thứ 42 trong Project có Key là `WEB`. Mã này được gán tự động và không thay đổi trong suốt vòng đời của Issue, giúp dễ dàng tham chiếu và tra cứu.

---

## 3.2. Các loại Issue

Jira phân chia Issue thành nhiều loại (Issue Type) khác nhau, mỗi loại phục vụ một mục đích cụ thể trong quy trình phát triển. Việc sử dụng đúng loại Issue giúp nhóm tổ chức công việc rõ ràng và theo dõi tiến độ hiệu quả hơn.

### 3.2.1. Epic

**Epic** là đơn vị công việc lớn nhất trong hệ thống phân cấp Issue. Một Epic thường đại diện cho một tính năng lớn, một module hoặc một mục tiêu kinh doanh cấp cao, có thể kéo dài nhiều Sprint và được chia nhỏ thành nhiều Story hoặc Task bên dưới.

> **Ví dụ:** Epic "Hệ thống xác thực người dùng" có thể bao gồm các Story như: Đăng ký tài khoản, Đăng nhập, Quên mật khẩu, Xác thực hai yếu tố...

Epic thường được dùng để nhóm các công việc liên quan lại với nhau, giúp quản lý dự án nắm bắt được tiến độ tổng thể của từng tính năng lớn.

### 3.2.2. Story (User Story)

**Story** hay **User Story** là mô tả một tính năng hoặc chức năng từ góc nhìn của người dùng cuối. Story thường được viết theo cấu trúc:

> *"Với tư cách là [vai trò người dùng], tôi muốn [hành động] để [mục đích/lợi ích]."*

Ví dụ: *"Với tư cách là người dùng, tôi muốn đặt lại mật khẩu qua email để có thể truy cập lại tài khoản khi quên mật khẩu."*

Story là đơn vị công việc phổ biến nhất trong Scrum, thường được hoàn thành trong một Sprint và có độ phức tạp vừa phải — không quá lớn cũng không quá nhỏ.

### 3.2.3. Task

**Task** là loại Issue đại diện cho một công việc kỹ thuật hoặc nhiệm vụ cụ thể, không nhất thiết phải gắn liền với yêu cầu từ người dùng. Task thường được dùng cho các công việc nội bộ như: cập nhật thư viện, viết tài liệu kỹ thuật, cấu hình môi trường, hay các công việc hỗ trợ khác.

Task có thể đứng độc lập hoặc được gắn vào một Story/Epic cụ thể.

### 3.2.4. Sub-task (Công việc con)

**Sub-task** là công việc con được tách ra từ một Issue cha (Story hoặc Task). Sub-task được sử dụng khi một Story hoặc Task quá lớn cần được chia nhỏ thành các phần có thể phân công cho nhiều người khác nhau hoặc hoàn thành theo thứ tự.

> **Ví dụ:** Story "Trang đăng nhập" có thể được chia thành các Sub-task: Thiết kế UI, Viết API xác thực, Tích hợp Frontend-Backend, Viết Unit Test...

Sub-task không thể đứng độc lập — chúng luôn phải thuộc về một Issue cha.

### 3.2.5. Bug

**Bug** là loại Issue đặc biệt dùng để ghi nhận các lỗi, sự cố hoặc hành vi không đúng mong đợi trong phần mềm. Bug có cấu trúc thông tin riêng biệt so với các loại Issue khác, bao gồm: mô tả lỗi, các bước tái hiện, kết quả mong đợi và kết quả thực tế.

Bug sẽ được trình bày chi tiết hơn trong Chương 10 — Quản lý Bug.

### Tóm tắt phân cấp Issue

```
Epic  (Tính năng lớn, nhiều Sprint)
└── Story  (Yêu cầu từ người dùng, 1 Sprint)
    ├── Sub-task  (Công việc con)
    └── Sub-task

Task  (Công việc kỹ thuật/nội bộ)
└── Sub-task

Bug  (Lỗi phần mềm)
└── Sub-task
```

> 📌 **Khác biệt với Team-managed:** Trong Team-managed, các loại Issue mặc định là **Epic**, **Story**, **Task** và **Bug** — không có **Sub-task** riêng biệt. Thay vào đó, bạn tạo **"Child issue"** trực tiếp từ Issue cha. Ngoài ra, Team-managed cho phép **thêm hoặc đổi tên loại Issue** trực tiếp trong Project Settings mà không cần Jira Admin.

---

## 3.3. Cấu trúc của một Issue

Mỗi Issue trong Jira bao gồm nhiều trường thông tin (field) khác nhau. Nắm rõ vai trò của từng trường giúp người dùng tạo Issue đầy đủ và chuẩn xác ngay từ đầu.

### 3.3.1. Summary (Tiêu đề)

**Summary** là tiêu đề ngắn gọn mô tả nội dung của Issue. Đây là trường thông tin bắt buộc và là thứ đầu tiên mọi người nhìn thấy khi duyệt danh sách Issue trên Board hoặc Backlog.

Một Summary tốt cần ngắn gọn, cụ thể và đủ nghĩa khi đứng một mình. Tránh dùng các tiêu đề mơ hồ như *"Fix bug"* hay *"Update page"* — nên thay bằng *"Sửa lỗi không gửi được email xác nhận đăng ký"* hay *"Cập nhật giao diện trang thanh toán theo thiết kế mới"*.

### 3.3.2. Description (Mô tả)

**Description** là phần mô tả chi tiết nội dung, yêu cầu, bối cảnh và bất kỳ thông tin nào cần thiết để người được giao công việc có thể hiểu và thực hiện. Description hỗ trợ định dạng văn bản phong phú (rich text), cho phép chèn hình ảnh, bảng biểu, đoạn code và liên kết.

### 3.3.3. Assignee (Người được giao)

**Assignee** là thành viên chịu trách nhiệm thực hiện Issue. Mỗi Issue thường chỉ có một Assignee duy nhất. Việc phân công rõ ràng giúp tránh tình trạng "không ai làm" hoặc nhiều người cùng làm một việc.

### 3.3.4. Reporter (Người tạo)

**Reporter** là người đã tạo Issue. Trường này thường được điền tự động theo tài khoản đang đăng nhập. Reporter có trách nhiệm cung cấp thông tin đầy đủ và theo dõi tiến độ xử lý Issue mình đã tạo.

### 3.3.5. Priority (Mức độ ưu tiên)

**Priority** phản ánh mức độ quan trọng và cấp thiết của Issue. Jira mặc định cung cấp năm mức độ ưu tiên:

| Mức độ | Ý nghĩa |
|---|---|
| **Highest / Critical** | Phải xử lý ngay, ảnh hưởng nghiêm trọng đến hệ thống |
| **High** | Quan trọng, cần xử lý trong Sprint hiện tại |
| **Medium** | Bình thường, ưu tiên sau High |
| **Low** | Ít quan trọng, có thể xử lý khi rảnh |
| **Lowest** | Không cấp thiết, có thể để sang Sprint sau |

### 3.3.6. Labels (Nhãn)

**Labels** là các thẻ tự do dùng để phân loại Issue theo chủ đề, tính năng hoặc bất kỳ tiêu chí nào do nhóm tự định nghĩa. Labels rất hữu ích khi cần lọc hoặc tìm kiếm Issue theo nhóm đặc thù. Ví dụ: `frontend`, `backend`, `performance`, `security`...

### 3.3.7. Component (Thành phần)

**Component** là đơn vị phân loại Issue theo module hoặc chức năng trong dự án. Khác với Labels (tự do), Component được định nghĩa trước trong Project Settings và thường phản ánh cấu trúc kỹ thuật của sản phẩm. Ví dụ: `Authentication`, `Payment Gateway`, `Notification Service`...

### 3.3.8. Due Date (Ngày hoàn thành)

**Due Date** xác định thời hạn cần hoàn thành Issue. Trường này giúp nhóm nhận biết các Issue có deadline sắp đến và ưu tiên xử lý kịp thời.

### 3.3.9. Attachment (Tệp đính kèm)

**Attachment** cho phép đính kèm các tệp liên quan vào Issue như: ảnh chụp màn hình lỗi, file thiết kế, tài liệu yêu cầu, video minh họa... Tính năng này đặc biệt quan trọng khi tạo Bug Report.

### 3.3.10. Comment (Bình luận)

**Comment** là kênh trao đổi trực tiếp trong một Issue. Thành viên có thể đặt câu hỏi, cập nhật tiến độ, làm rõ yêu cầu hoặc chia sẻ kết quả công việc thông qua phần bình luận. Comment giúp toàn bộ lịch sử trao đổi liên quan đến một công việc được lưu giữ tại một chỗ.

---

## 3.4. Tạo và chỉnh sửa Issue

### Tạo Issue mới

**Bước 1:** Nhấn nút **Create** (thường ở góc trên bên trái hoặc thanh điều hướng) hoặc nhấn phím tắt **C** khi đang ở trong Project.

**Bước 2:** Chọn **Project** và **Issue Type** phù hợp (Epic, Story, Task, Bug...).

**Bước 3:** Điền **Summary** — tiêu đề ngắn gọn, rõ ràng.

**Bước 4:** Bổ sung các thông tin cần thiết: Description, Assignee, Priority, Labels, Due Date...

**Bước 5:** Nhấn **Create** để hoàn tất. Issue mới sẽ xuất hiện trong Backlog hoặc trực tiếp trên Board tùy theo cấu hình.

### Chỉnh sửa Issue

Để chỉnh sửa, nhấn vào Issue Key hoặc tiêu đề để mở trang chi tiết. Hầu hết các trường thông tin đều có thể chỉnh sửa trực tiếp bằng cách nhấn vào giá trị cần thay đổi — không cần vào chế độ chỉnh sửa riêng.

---

## 3.5. Assign Issue (Phân công công việc)

Phân công Issue cho đúng người là bước quan trọng để đảm bảo công việc được thực hiện rõ ràng, tránh mơ hồ về trách nhiệm.

Để gán Assignee, mở Issue cần phân công → nhấn vào trường **Assignee** → chọn thành viên từ danh sách. Người được giao sẽ nhận thông báo qua email hoặc trong ứng dụng.

Jira cũng hỗ trợ tính năng **Assign to me** — cho phép người dùng tự nhận công việc về mình chỉ với một cú nhấp.

---

## 3.6. Comment và Mention

### Comment

Để thêm bình luận, mở Issue → kéo xuống phần **Activity** → nhập nội dung vào ô comment → nhấn **Save**. Comment hỗ trợ định dạng văn bản, chèn ảnh và đoạn code.

### Mention (Đề cập)

Khi cần thu hút sự chú ý của một thành viên cụ thể, sử dụng ký tự **@** theo sau là tên hoặc username của người đó ngay trong phần Comment hay Description. Người được đề cập sẽ nhận thông báo tức thì.

> **Ví dụ:** `@Nguyen xem giúp mình phần logic xác thực nhé.`

---

## 3.7. Link các Issue

Jira cho phép thiết lập mối quan hệ giữa các Issue thông qua tính năng **Link**. Điều này giúp theo dõi sự phụ thuộc và liên kết giữa các đầu công việc khác nhau.

Các kiểu liên kết phổ biến:

| Kiểu liên kết | Ý nghĩa |
|---|---|
| **blocks / is blocked by** | Issue này chặn (hoặc bị chặn bởi) Issue kia |
| **relates to** | Hai Issue có liên quan đến nhau |
| **duplicates / is duplicated by** | Issue này là bản sao của Issue kia |
| **clones / is cloned by** | Issue này được sao chép từ Issue kia |

Để thêm liên kết, mở Issue → nhấn **Link** (hoặc **More → Link**) → chọn kiểu liên kết và nhập Issue Key cần liên kết.

> 📌 **Khác biệt với Team-managed:** Tính năng Link Issue có thể bị giới hạn hoặc giao diện khác so với Company-managed. Trong một số phiên bản Team-managed, bạn tìm thấy Link bằng cách nhấn vào **"..."** (menu ba chấm) trong trang chi tiết Issue.

---

## 3.8. Clone Issue

Tính năng **Clone** cho phép tạo ra một bản sao của Issue hiện tại, giữ nguyên toàn bộ hoặc một phần thông tin gốc. Clone thường được dùng khi cần tạo nhiều Issue tương tự nhau, giúp tiết kiệm thời gian nhập liệu.

Để clone, mở Issue → nhấn **"..."** (menu ba chấm, thường ở góc trên bên phải trang Issue) → chọn **Copy issue** → điều chỉnh thông tin nếu cần → nhấn **Create**.

> 📌 **Khác biệt với Team-managed:** Tùy chọn Clone/Copy có thể có tên là **"Copy issue"** thay vì "Clone" và nằm trong menu **"..."** thay vì menu "More".

---

## 3.9. Quan hệ Parent – Child (Cha – Con)

Jira tổ chức Issue theo cấu trúc phân cấp cha – con (Parent – Child), phản ánh mức độ chi tiết của công việc:

- **Epic** là cha của **Story** và **Task**
- **Story** hoặc **Task** là cha của **Sub-task**

Mối quan hệ này giúp nhóm dễ dàng theo dõi tiến độ từng tầng: khi toàn bộ Sub-task của một Story hoàn thành, Story đó mới được coi là hoàn thành; khi tất cả Story trong một Epic xong, Epic mới có thể đóng lại.

Trong giao diện Issue, phần **Child Issues** hiển thị danh sách các Issue con cùng trạng thái hiện tại của chúng, cho phép theo dõi tổng thể mà không cần mở từng Issue riêng lẻ.

---

## Tóm tắt chương

Chương 3 đã trình bày toàn diện về Issue — trái tim của mọi hoạt động trong Jira. Hiểu rõ các loại Issue và cách sử dụng từng trường thông tin là nền tảng để làm việc hiệu quả với toàn bộ các tính năng còn lại của Jira.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Issue | Đơn vị công việc cơ bản trong Jira |
| Epic | Tính năng lớn, bao gồm nhiều Story/Task |
| Story | Yêu cầu từ góc nhìn người dùng cuối |
| Task | Công việc kỹ thuật hoặc nội bộ |
| Sub-task | Công việc con, thuộc về một Issue cha |
| Bug | Lỗi phần mềm cần được xử lý |
| Assignee | Người chịu trách nhiệm thực hiện Issue |
| Priority | Mức độ ưu tiên xử lý Issue |
| Comment | Kênh trao đổi trực tiếp trong Issue |

---

*Chương tiếp theo: **Chương 4 — Backlog: Quản lý Danh sách Công việc***
