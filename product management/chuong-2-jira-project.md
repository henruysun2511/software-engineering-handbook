# CHƯƠNG 2: JIRA PROJECT — TỔ CHỨC VÀ QUẢN LÝ DỰ ÁN

---

## 2.1. Project là gì?

Trong Jira, **Project** (Dự án) là đơn vị tổ chức lớn nhất và cao nhất. Mọi thành phần khác trong Jira — Issue, Board, Sprint, Workflow, thành viên — đều tồn tại bên trong phạm vi của một Project cụ thể.

Có thể hình dung Project như một "thư mục lớn" chứa toàn bộ công việc, quy trình và thông tin liên quan đến một dự án hoặc sản phẩm nhất định. Mỗi Project có không gian làm việc riêng biệt, hoàn toàn độc lập với các Project khác trong cùng một tổ chức.

Một tổ chức có thể tạo nhiều Project song song trên Jira. Ví dụ:

- `WEB` — Dự án phát triển website
- `APP` — Dự án phát triển ứng dụng di động
- `MKT` — Dự án chiến dịch marketing

Mỗi Project được định danh bằng một **Project Key** — chuỗi ký tự viết tắt do người dùng đặt (thường từ 2 đến 5 ký tự). Project Key sẽ được dùng để đặt mã định danh cho từng Issue bên trong Project đó. Ví dụ: Issue trong Project có Key là `WEB` sẽ được đánh số lần lượt là `WEB-1`, `WEB-2`, `WEB-3`...

---

## 2.2. Các loại Project phổ biến

Khi tạo một Project mới trong Jira, người dùng cần chọn **loại Project** phù hợp với phương pháp quản lý công việc của nhóm. Hai loại Project phổ biến nhất là **Scrum** và **Kanban**.

### 2.2.1. Scrum Project

**Scrum Project** được thiết kế cho các nhóm làm việc theo chu kỳ Sprint có thời hạn cố định. Đây là lựa chọn phù hợp khi nhóm có kế hoạch phát triển rõ ràng theo từng giai đoạn, cần kiểm soát chặt chẽ tiến độ và thường xuyên đánh giá kết quả sau mỗi Sprint.

Scrum Project cung cấp đầy đủ các công cụ đặc thù của Scrum, bao gồm: Backlog, Sprint, Scrum Board và các báo cáo như Burndown Chart, Velocity Chart.

**Phù hợp với:** Nhóm phát triển phần mềm, nhóm có lịch phát hành định kỳ, dự án cần quản lý chặt chẽ tiến độ từng giai đoạn.

### 2.2.2. Kanban Project

**Kanban Project** phù hợp với các nhóm xử lý công việc theo luồng liên tục, không chia theo Sprint. Công việc được đưa vào và hoàn thành theo thứ tự ưu tiên mà không cần cam kết theo chu kỳ cố định.

Kanban Project tập trung vào việc trực quan hóa luồng công việc và tối thiểu hóa lượng công việc đang xử lý cùng lúc (Work In Progress — WIP).

**Phù hợp với:** Nhóm vận hành, nhóm hỗ trợ kỹ thuật (IT Support), nhóm xử lý yêu cầu liên tục không theo kế hoạch cố định.

### So sánh Scrum và Kanban Project

| Tiêu chí | Scrum Project | Kanban Project |
|---|---|---|
| Chu kỳ làm việc | Theo Sprint (1–4 tuần) | Liên tục, không có Sprint |
| Backlog | Có | Có |
| Sprint | Có | Không |
| Giới hạn WIP | Không bắt buộc | Khuyến nghị |
| Báo cáo | Burndown, Velocity... | Cumulative Flow |
| Phù hợp | Phát triển sản phẩm | Vận hành, hỗ trợ |

---

## 2.3. Cấu trúc của một Project

Một Jira Project được tổ chức theo cấu trúc phân cấp rõ ràng, từ tổng thể đến chi tiết:

```
Project
├── Board (Scrum Board / Kanban Board)
├── Backlog
├── Sprint (nếu là Scrum Project)
├── Issues
│   ├── Epic
│   │   ├── Story
│   │   │   ├── Task
│   │   │   └── Sub-task
│   │   └── Bug
└── Reports & Dashboard
```

Mỗi thành phần trong cấu trúc này đảm nhận một vai trò riêng và sẽ được trình bày chi tiết trong các chương tương ứng.

---

## 2.4. Project Settings (Cài đặt dự án)

**Project Settings** là nơi quản trị viên dự án (Project Administrator) cấu hình toàn bộ các thông số kỹ thuật và hành vi của Project. Đây là khu vực quan trọng, ảnh hưởng trực tiếp đến cách nhóm làm việc với Jira hàng ngày.

### Các nhóm cài đặt chính:

**a) Thông tin chung (Details)**

Cho phép chỉnh sửa tên Project, mô tả, Project Key, loại Project và ảnh đại diện.

**b) Thành viên và quyền truy cập (People & Access)**

Quản lý danh sách thành viên trong Project, phân quyền vai trò (Role) cho từng người: Administrator, Developer, hay Viewer.

**c) Issue Types (Loại Issue)**

Cấu hình các loại Issue được sử dụng trong Project. Tùy theo nhu cầu, nhóm có thể bật hoặc tắt một số loại Issue nhất định.

**d) Workflow**

Cấu hình quy trình chuyển trạng thái (Workflow) cho từng loại Issue. Mỗi loại Issue có thể sử dụng một Workflow riêng.

**e) Components và Labels**

Tạo và quản lý các nhãn phân loại Issue theo chức năng hoặc module trong dự án.

**f) Versions**

Quản lý các phiên bản phát hành (Release) của dự án. Tính năng này đặc biệt hữu ích khi dự án có nhiều đợt phát hành theo lịch trình.

> **Lưu ý:** Chỉ thành viên có vai trò **Project Administrator** mới có quyền truy cập và thay đổi Project Settings. Thành viên thông thường chỉ có quyền xem và làm việc với nội dung trong Project.

> 📌 **Khác biệt với Team-managed:** Project Settings trong Team-managed đơn giản hơn đáng kể — bạn có thể tự cấu hình trực tiếp mà **không cần nhờ Jira Admin**. Các mục như Workflow được chỉnh ngay trên Board (kéo thêm cột, đổi tên cột) thay vì vào menu cấu hình riêng. Một số tính năng của Company-managed như **Versions**, **Components** và **Screens** không có hoặc bị giới hạn trong Team-managed.

---

## 2.5. Thành viên và quyền truy cập Project

Jira cho phép kiểm soát chặt chẽ việc ai có thể xem và làm gì trong một Project thông qua hệ thống **vai trò (Role)** và **quyền hạn (Permission)**.

### Các vai trò phổ biến trong một Project:

| Vai trò | Quyền hạn tiêu biểu |
|---|---|
| **Project Administrator** | Toàn quyền: cấu hình Project, quản lý thành viên, chỉnh sửa Workflow |
| **Developer** | Tạo, chỉnh sửa, chuyển trạng thái Issue; log công việc |
| **Viewer / Reporter** | Chỉ xem Issue và tạo Issue mới, không thể chỉnh sửa |

> 📌 **Khác biệt với Team-managed:** Hệ thống phân quyền đơn giản hơn, chỉ có hai mức: **Administrator** và **Member**. Không có khái niệm "Roles" phức tạp như Company-managed. Ai được thêm vào Project đều có thể tạo và chỉnh sửa Issue mặc định.

### Quyền truy cập Project (Project Access):

Jira hỗ trợ ba mức độ truy cập cho một Project:

- **Private:** Chỉ những thành viên được thêm vào mới có thể xem và làm việc trong Project.
- **Public (trong tổ chức):** Tất cả thành viên trong tổ chức đều có thể xem Project, nhưng chỉ những người được phân quyền mới có thể chỉnh sửa.
- **Open:** Bất kỳ ai có đường link đều có thể xem Project (thường dùng cho các dự án mã nguồn mở).

Trong thực tế, hầu hết các nhóm phát triển phần mềm sử dụng chế độ **Private** để bảo mật thông tin dự án.

---

## 2.6. Thực hành: Tạo một Project mới

Để tạo một Project mới trong Jira, thực hiện các bước sau:

**Bước 1:** Từ thanh điều hướng trên cùng, chọn **Projects** → **Create project**.

**Bước 2:** Jira hiển thị danh sách template. Chọn **Scrum** hoặc **Kanban** tùy theo phương thức làm việc của nhóm.

> 📌 **Với Team-managed:** Ở bước này, sau khi chọn template (Scrum/Kanban), Jira sẽ hỏi bạn chọn **"Team-managed project"** hay **"Company-managed project"**. Hãy chọn **Team-managed** nếu bạn muốn tự quản lý mà không cần Jira Admin.

**Bước 3:** Đặt tên Project (Project Name) và Project Key. Jira sẽ tự động gợi ý Key dựa trên tên, người dùng có thể chỉnh sửa lại cho ngắn gọn và dễ nhận biết.

**Bước 4:** Chọn chế độ truy cập (Access) cho Project — Private hoặc Public trong tổ chức.

**Bước 5:** Nhấn **Create project** để hoàn tất. Jira sẽ tự động tạo Project và chuyển hướng người dùng vào Board của Project vừa tạo.

---

## Tóm tắt chương

Chương 2 đã trình bày tổng quan về Project — nền tảng tổ chức công việc trong Jira. Người dùng cần nắm vững sự khác biệt giữa Scrum Project và Kanban Project để lựa chọn loại phù hợp với phương thức làm việc của nhóm.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Project | Không gian làm việc độc lập cho một dự án |
| Project Key | Mã định danh ngắn gọn của Project (VD: WEB, APP) |
| Scrum Project | Tổ chức công việc theo Sprint có thời hạn |
| Kanban Project | Tổ chức công việc theo luồng liên tục |
| Project Settings | Khu vực cấu hình toàn bộ thông số kỹ thuật của Project |
| Project Role | Vai trò xác định quyền hạn của từng thành viên |

---

*Chương tiếp theo: **Chương 3 — Issue: Đơn vị Công việc Cơ bản trong Jira***
