# Hướng dẫn code dự án <project_name>

Tài liệu hướng dẫn cách làm việc với dự án Spring Boot `<project_name>`.

## Danh sách tài liệu

| # | File | Nội dung |
|---|------|----------|
| 1 | [01-SETUP_GUIDE.md](./01-SETUP_GUIDE.md) | Tạo dự án — môi trường, chạy, build, test, quy trình hằng ngày |
| 2 | [02-PROJECT_STRUCTURE.md](./02-PROJECT_STRUCTURE.md) | Cấu trúc dự án — thư mục, package, module, quy tắc phân lớp |
| 3 | [03-CONFIGURATION_GUIDE.md](./03-CONFIGURATION_GUIDE.md) | Cấu hình môi trường — `application.properties`, profile, `.env` |
| 4 | [04-ARCHITECTURE_GUIDE.md](./04-ARCHITECTURE_GUIDE.md) | Kiến trúc code — luồng request, trách nhiệm từng tầng, viết code mới |
| 5 | [05-CRUD_TUTORIAL.md](./05-CRUD_TUTORIAL.md) | API CRUD mẫu — dựng từng bước module `employee` |
| 6 | [06-AUTH_SECURITY_GUIDE.md](./06-AUTH_SECURITY_GUIDE.md) | Xác thực & phân quyền — JWT, OAuth2, RBAC, rate limit |
| 7 | [07-DATABASE_JPA_GUIDE.md](./07-DATABASE_JPA_GUIDE.md) | Database & JPA — Entity, soft delete, mã hóa, quan hệ, query |
| 8 | [08-RESPONSE_ERROR_GUIDE.md](./08-RESPONSE_ERROR_GUIDE.md) | Response chuẩn & xử lý lỗi — `ApiResponse`, `GlobalExceptionHandler` |
| 9 | [09-CACHE_AUDIT_GUIDE.md](./09-CACHE_AUDIT_GUIDE.md) | Cache Redis (fallback) & audit log — `@Cacheable`, `@AuditAction`, `AuditContext` |
| 10 | [10-TESTING_GUIDE.md](./10-TESTING_GUIDE.md) | Viết test — JUnit 5, Mockito, `BaseUnitTest`, cách chạy |
| 11 | [11-DEPLOYMENT_GUIDE.md](./11-DEPLOYMENT_GUIDE.md) | Deploy — Dockerfile, docker-compose, production |
| 12 | [12-NOTIFICATION_SCHEDULER_GUIDE.md](./12-NOTIFICATION_SCHEDULER_GUIDE.md) | Notification & scheduler — SSE realtime, `@Scheduled`, nhắc ca làm |
| B | [BUSINESS_LOGIC_GUIDE.md](./BUSINESS_LOGIC_GUIDE.md) | Logic nghiệp vụ các module — chấm công, nghỉ phép, hợp đồng, tính lương |
| S | [SHARED_MODULE_GUIDE.md](./SHARED_MODULE_GUIDE.md) | Chi tiết module `shared` — từng package, từng file, cách chúng hoạt động |

## Gợi ý thứ tự đọc

1. **01-SETUP_GUIDE.md** → cài môi trường, chạy được project.
2. **02-PROJECT_STRUCTURE.md** → nắm cấu trúc thư mục.
3. **03-CONFIGURATION_GUIDE.md** → hiểu các file cấu hình.
4. **04-ARCHITECTURE_GUIDE.md** → hiểu luồng code, cách viết code mới.
5. **05-CRUD_TUTORIAL.md** → dựng 1 API CRUD mẫu theo từng bước.
6. **06-AUTH_SECURITY_GUIDE.md** → hiểu cơ chế JWT, phân quyền khi code endpoint.
7. **07-DATABASE_JPA_GUIDE.md** → hiểu Entity, soft delete, mã hóa, query.
8. **08-RESPONSE_ERROR_GUIDE.md** → format response & cách ném lỗi chuẩn.
9. **09-CACHE_AUDIT_GUIDE.md** → bật cache & ghi audit khi viết service.
10. **10-TESTING_GUIDE.md** → viết unit test chuẩn cho service.
11. **11-DEPLOYMENT_GUIDE.md** → đóng gói & triển khai bằng Docker.
12. **12-NOTIFICATION_SCHEDULER_GUIDE.md** → thông báo realtime (SSE) & công việc nền (`@Scheduled`).
B. **BUSINESS_LOGIC_GUIDE.md** → hiểu logic nghiệp vụ khi sửa/xây dựng tính năng.
S. **SHARED_MODULE_GUIDE.md** → đào sâu từng file trong module `shared` khi cần.

## Tài liệu liên quan

- `CODE_FLOW_GUIDE.md` — thứ tự luồng code của các module (shared, auth, employee).
