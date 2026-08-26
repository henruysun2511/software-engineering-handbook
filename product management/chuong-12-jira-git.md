# CHƯƠNG 12: JIRA & GIT — TÍCH HỢP VỚI HỆ THỐNG QUẢN LÝ MÃ NGUỒN

---

## 12.1. Tại sao tích hợp Jira với Git?

Trong quy trình phát triển phần mềm hiện đại, hai hệ thống được sử dụng song song và liên tục: **Jira** quản lý công việc và **Git** quản lý mã nguồn. Khi hai hệ thống này hoạt động độc lập, thông tin bị phân mảnh — Developer phải tự nhớ commit nào liên quan đến Issue nào, Project Manager phải hỏi thủ công tiến độ từng tính năng.

Tích hợp Jira với Git (GitHub, GitLab, Bitbucket...) giải quyết vấn đề này bằng cách tạo **liên kết tự động** giữa Issue trong Jira và các hoạt động trong Git (branch, commit, pull request). Kết quả là mọi người đều thấy được bức tranh đầy đủ từ cả hai phía mà không cần chuyển đổi qua lại giữa các công cụ.

---

## 12.2. Jira Issue Key — Cầu nối giữa hai hệ thống

**Jira Issue Key** (ví dụ: `WEB-42`, `APP-15`) là định danh duy nhất của mỗi Issue, đồng thời là chìa khóa tạo ra sự liên kết giữa Jira và Git.

Nguyên lý hoạt động đơn giản: khi Developer đề cập đến một Issue Key trong **tên branch**, **nội dung commit** hay **tiêu đề Pull Request**, Jira tự động nhận diện và liên kết thông tin tương ứng vào trang chi tiết của Issue đó.

Nhờ vậy, chỉ cần mở một Issue trong Jira, người dùng có thể thấy ngay toàn bộ hoạt động phát triển liên quan: branch nào đang làm, commit nào đã được thực hiện, Pull Request nào đang chờ review — tất cả hiển thị trong tab **Development** của Issue.

---

## 12.3. Liên kết Jira với GitHub/GitLab

### Điều kiện tiên quyết:

Để tích hợp hoạt động, Jira Administrator cần cấu hình kết nối giữa Jira và nền tảng Git thông qua **Jira Software → Apps → GitHub/GitLab Integration** (hoặc cài đặt app tích hợp từ Atlassian Marketplace). Đây là cài đặt cấp tổ chức, thực hiện một lần duy nhất.

Sau khi cấu hình xong, tất cả thành viên trong tổ chức đều hưởng lợi từ tích hợp này mà không cần thiết lập thêm gì.

### Cơ chế hoạt động:

Jira theo dõi các sự kiện từ Git (push, commit, pull request, merge) thông qua **webhook** — một cơ chế thông báo tự động. Mỗi khi có sự kiện Git chứa Jira Issue Key, webhook gửi thông tin đến Jira và Jira cập nhật trang Issue tương ứng.

---

## 12.4. Branch theo Jira Issue

Quy ước đặt tên branch theo Jira Issue Key là một trong những thực hành quan trọng nhất khi tích hợp hai hệ thống.

### Cấu trúc tên branch chuẩn:

```
[loại-công-việc]/[ISSUE-KEY]-[mô-tả-ngắn]
```

**Các tiền tố loại công việc phổ biến:**

| Tiền tố | Loại công việc |
|---|---|
| `feature/` | Phát triển tính năng mới |
| `bugfix/` | Sửa lỗi thông thường |
| `hotfix/` | Sửa lỗi khẩn cấp trên Production |
| `refactor/` | Tái cấu trúc code, không thay đổi chức năng |
| `chore/` | Công việc kỹ thuật nội bộ (cập nhật dependency, cấu hình...) |

**Ví dụ thực tế:**

```bash
feature/WEB-42-user-login
feature/APP-15-payment-gateway-integration
bugfix/WEB-67-fix-email-validation
hotfix/APP-23-critical-payment-crash
refactor/WEB-89-optimize-database-queries
```

### Cách tạo branch từ Jira:

Jira hỗ trợ tạo branch trực tiếp từ trang Issue mà không cần vào GitHub/GitLab:

**Bước 1:** Mở Issue cần tạo branch.

**Bước 2:** Trong tab **Development** (hoặc cột bên phải) → nhấn **Create branch**.

**Bước 3:** Chọn repository, branch gốc (thường là `main` hoặc `develop`) và Jira tự động gợi ý tên branch theo Issue Key.

**Bước 4:** Điều chỉnh tên branch nếu cần → nhấn **Create branch**.

Branch mới sẽ được tạo trực tiếp trên GitHub/GitLab và liên kết tự động với Issue.

---

## 12.5. Commit liên kết với Jira

Khi Developer thực hiện commit, việc đề cập Jira Issue Key trong **commit message** giúp Jira tự động liên kết commit đó với Issue tương ứng.

### Cú pháp commit message chuẩn:

```
[ISSUE-KEY]: [Mô tả ngắn gọn về thay đổi]

[Mô tả chi tiết hơn nếu cần - không bắt buộc]
```

**Ví dụ:**

```bash
git commit -m "WEB-42: Implement login form validation"
git commit -m "APP-15: Add Stripe payment gateway integration"
git commit -m "WEB-67: Fix email regex to accept plus sign in username"
git commit -m "APP-23: Hotfix critical null pointer in payment processor"
```

### Commit message tốt cần đáp ứng:

- **Bắt đầu bằng Issue Key:** Giúp Jira nhận diện và liên kết tự động
- **Dùng động từ chỉ hành động:** *Implement, Add, Fix, Update, Remove, Refactor...*
- **Ngắn gọn nhưng đủ nghĩa:** Không cần giải thích "tại sao" trong dòng đầu — chỉ cần "làm gì"
- **Viết hoa chữ cái đầu tiên sau Issue Key**
- **Không kết thúc bằng dấu chấm**

### Kết quả trong Jira:

Sau khi commit được push lên remote repository, tab **Development** trong Issue sẽ hiển thị:
- Số lượng commit liên quan
- Tên branch chứa commit
- Trạng thái build (nếu có tích hợp CI/CD)

---

## 12.6. Pull Request và Jira

**Pull Request (PR)** — hay **Merge Request (MR)** trong GitLab — là bước quan trọng trong quy trình Code Review. Khi PR được tạo với tiêu đề chứa Issue Key, Jira tự động liên kết và hiển thị trạng thái PR trong Issue.

### Cấu trúc tiêu đề Pull Request:

```
[ISSUE-KEY]: [Mô tả tính năng hoặc fix]
```

**Ví dụ:**

```
WEB-42: Add user authentication module
APP-15: Integrate Stripe payment gateway
WEB-67: Fix email validation regex
```

### Thông tin PR hiển thị trong Jira:

| Thông tin | Mô tả |
|---|---|
| **PR Status** | Open, Merged, Declined |
| **Review Status** | Chưa review, Approved, Changes Requested |
| **Build Status** | Passed, Failed, In Progress (nếu tích hợp CI/CD) |
| **Số lượng reviewer** | Bao nhiêu người đã approve |

### Quy trình Code Review tích hợp Jira-GitHub:

```
1. Developer hoàn thành code trên branch feature/WEB-42-...
2. Developer tạo Pull Request với tiêu đề "WEB-42: ..."
3. Jira tự động cập nhật Issue WEB-42: PR đang Open
4. Reviewer nhận thông báo, kiểm tra code
5. Nếu approve: PR được merge → Jira cập nhật: PR Merged
6. Nếu yêu cầu sửa: Developer sửa và push thêm commit
7. Sau khi merge, Developer chuyển Issue sang trạng thái Testing/Done
```

---

## 12.7. Theo dõi Development trong Jira

Tab **Development** trong mỗi Issue là nơi tổng hợp toàn bộ thông tin phát triển liên quan đến Issue đó.

### Thông tin trong tab Development:

**Branches:** Danh sách các branch đang liên kết với Issue. Hiển thị tên branch, repository và số commit chưa merge.

**Commits:** Danh sách tất cả commit đề cập đến Issue Key này, bao gồm: nội dung commit message, tên tác giả, thời gian và link đến commit trên GitHub/GitLab.

**Pull Requests:** Danh sách tất cả PR liên quan, kèm trạng thái review và build.

**Builds:** Nếu có tích hợp CI/CD (Jenkins, GitHub Actions, CircleCI...), kết quả build tự động sẽ hiển thị ở đây — giúp biết ngay code mới có pass test tự động hay không.

**Deployments:** Nếu có tích hợp với hệ thống deployment, thông tin về Issue được deploy đến môi trường nào (Staging, Production) sẽ xuất hiện ở đây.

---

## 12.8. Lợi ích của tích hợp Jira-Git

### Với Developer:

- Không cần nhớ tay xem commit nào liên quan đến Issue nào
- Có thể tạo branch trực tiếp từ Jira mà không cần vào GitHub
- Lịch sử phát triển của mỗi tính năng được lưu tập trung

### Với QA:

- Biết được chính xác commit nào cần test cho từng Bug fix
- Theo dõi được trạng thái PR — Bug đã được merge chưa hay chưa?
- Có thể yêu cầu test trên đúng branch mà Developer đang làm

### Với Scrum Master / Project Manager:

- Theo dõi tiến độ thực tế dựa trên hoạt động Git, không chỉ dựa trên báo cáo miệng
- Phát hiện Issue bị "tắc" không có commit mới trong nhiều ngày
- Đánh giá tốc độ phát triển dựa trên dữ liệu thực

---

## 12.9. Quy ước làm việc (Git Conventions) khi dùng Jira

Để tích hợp hoạt động hiệu quả, nhóm cần thống nhất và tuân thủ một số quy ước chung:

**Quy ước 1 — Luôn có Issue trước khi code:**
Không bắt đầu viết code cho một tính năng hay sửa Bug khi chưa có Issue trong Jira. Issue là "hợp đồng" định nghĩa công việc cần làm.

**Quy ước 2 — Đặt tên branch theo Issue Key:**
Mọi branch đều phải chứa Issue Key. Không chấp nhận branch có tên như `feature/new-feature` hay `fix/bug` mà không có Issue Key.

**Quy ước 3 — Commit message luôn bắt đầu bằng Issue Key:**
Giúp Jira tự động liên kết và dễ tra cứu lịch sử phát triển.

**Quy ước 4 — Một branch cho một Issue:**
Tránh làm nhiều Issue không liên quan trên cùng một branch — gây khó khăn khi review và merge.

**Quy ước 5 — Cập nhật trạng thái Issue đồng bộ với hoạt động Git:**
Khi tạo PR → chuyển Issue sang "In Review". Khi PR được merge → chuyển sang "Testing". Không để trạng thái Issue lạc hậu so với thực tế code.

---

## Tóm tắt chương

Tích hợp Jira với Git tạo ra một luồng thông tin liên tục và tự động giữa quản lý công việc và phát triển mã nguồn. Nhờ đó, mọi thành viên — dù là Developer, QA hay Project Manager — đều có cái nhìn đầy đủ và chính xác về tiến độ mà không cần hỏi han thủ công.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Issue Key | Định danh Issue, cầu nối giữa Jira và Git |
| Branch convention | Quy ước đặt tên branch theo Issue Key |
| Commit message | Nội dung commit chứa Issue Key để liên kết tự động |
| Pull Request | Yêu cầu review và merge code, liên kết với Jira Issue |
| Tab Development | Nơi hiển thị toàn bộ hoạt động Git của một Issue |
| Webhook | Cơ chế tự động thông báo sự kiện Git sang Jira |
| CI/CD Integration | Tích hợp kết quả build/deploy vào Jira Issue |

---

*Chương tiếp theo: **Chương 13 — Quản lý Công việc trong Team***
