# Software Engineering Handbook

Tài liệu học tập lộ trình **Software Engineer / Fullstack Developer** — viết bằng tiếng Việt, theo phong cách thực chiến. Nội dung được thiết kế cho người đã có nền tảng lập trình, học nhanh theo hướng áp dụng vào dự án thực tế, đi từ nền tảng cơ bản tới các chủ đề nâng cao (concurrency, security, deployment).

## Mục lục

| Thư mục | Nội dung |
|---|---|
| [`backend/`](backend/) | Backend cơ bản: Networking, Software Engineering, Kiến trúc Backend, Database, NestJS, Xử lý bất đồng bộ, Operations, Security |
| [`java-spring-boot/`](java-spring-boot/) | Lộ trình Java & Spring Boot: Java Core → Java Nâng cao → Spring Framework → Spring Boot → JPA → Security → Testing → Advanced |
| [`frontend/`](frontend/) | Frontend: JavaScript, TypeScript, Browser, React, Next.js, CSS, State Management, Form, Auth, Realtime, Design Patterns |
| [`mobile/`](mobile/) | Mobile: Flutter, React Native |
| [`devops/`](devops/) | DevOps: Linux, Docker, CI/CD, Reverse Proxy, Deployment, Cloud, Monitoring |
| [`testing/`](testing/) | Kiểm thử: Backend Testing, Frontend Testing |
| [`dsa/`](dsa/) | Cấu trúc dữ liệu & Giải thuật |

## Lộ trình đề xuất

1. **Nền tảng chung** — [`backend/`](backend/chuong-01-networking.md) → [`frontend/`](frontend/chuong-01-javascript.md)
2. **Backend** — chọn 1 trong 2 hướng:
   - [`java-spring-boot/`](java-spring-boot/chuong-00-java-initerary.md) — Java & Spring Boot
   - [`backend/`](backend/chuong-05-nestjs-core.md) — NestJS
3. **Frontend** — [`frontend/`](frontend/chuong-04-react.md) → [`frontend/`](frontend/chuong-05-nextjs.md)
4. **Kiểm thử** — [`testing/`](testing/)
5. **DevOps** — [`devops/`](devops/)
6. **Học thêm về thuật toán** — [`dsa/`](dsa/)

## Chi tiết các chương

### Backend

| Chương | Nội dung |
|---|---|
| [Chương 1](backend/chuong-01-networking.md) | Networking: TCP/IP, HTTP, DNS |
| [Chương 2](backend/chuong-02-software-engineering.md) | Software Engineering |
| [Chương 3](backend/chuong-03-backend-architecture.md) | Kiến trúc Backend |
| [Chương 4](backend/chuong-04-database.md) | Database |
| [Chương 5](backend/chuong-05-nestjs-core.md) | NestJS Core |
| [Chương 6](backend/chuong-06-processing-techniques.md) | Kỹ thuật xử lý bất đồng bộ |
| [Chương 7](backend/chuong-07-operations.md) | Operations |
| [Chương 8](backend/chuong-08-security.md) | Security |
| [Chương 9](backend/chuong-09-backend-common-features.md) | Các tính năng Backend phổ biến |

### Java & Spring Boot

| Chương | Nội dung |
|---|---|
| [Chương 0](java-spring-boot/chuong-00-java-initerary.md) | Lộ trình học Java & Spring Boot |
| [Chương 1](java-spring-boot/chuong-01-java-core.md) | Java Core |
| [Chương 2](java-spring-boot/chuong-02-java-advanced.md) | Java Nâng cao: Design Pattern, JVM, Reflection |
| [Chương 3](java-spring-boot/chuong-03-spring-framework.md) | Spring Framework: IoC, DI, Bean |
| [Chương 4](java-spring-boot/chuong-04-spring-boot.md) | Làm quen với Spring Boot |
| [Chương 5](java-spring-boot/chuong-05-database-jpa.md) | Làm việc với cơ sở dữ liệu (JPA) |
| [Chương 6](java-spring-boot/chuong-06-security.md) | Bảo mật và xác thực |
| [Chương 7](java-spring-boot/chuong-07-testing.md) | Testing |
| [Chương 8](java-spring-boot/chuong-08-advanced-common-features.md) | Các chủ đề nâng cao |

### Frontend

| Chương | Nội dung |
|---|---|
| [Chương 1](frontend/chuong-01-javascript.md) | JavaScript |
| [Chương 2](frontend/chuong-02-typescript.md) | TypeScript |
| [Chương 3](frontend/chuong-03-browser.md) | Browser |
| [Chương 4](frontend/chuong-04-react.md) | React |
| [Chương 5](frontend/chuong-05-nextjs.md) | Next.js |
| [Chương 6](frontend/chuong-06-css.md) | CSS |
| [Chương 7](frontend/chuong-07-state-management.md) | State Management |
| [Chương 8](frontend/chuong-08-server-state.md) | Server State |
| [Chương 9](frontend/chuong-09-form-handling.md) | Xử lý Form |
| [Chương 10](frontend/chuong-10-networking.md) | Networking |
| [Chương 11](frontend/chuong-11-auth.md) | Authentication |
| [Chương 12](frontend/chuong-12-realtime.md) | Realtime |
| [Chương 13](frontend/chuong-13-design-patterns.md) | Design Patterns |

### DevOps

| Chương | Nội dung |
|---|---|
| [Chương 1](devops/chuong-01-linux.md) | Linux |
| [Chương 2](devops/chapter-02-docker.md) | Docker |
| [Chương 3](devops/chapter-03-cicd.md) | CI/CD |
| [Chương 4](devops/chapter-04-reverse-proxy.md) | Reverse Proxy |
| [Chương 5](devops/chapter-05-deployment.md) | Deployment |
| [Chương 6](devops/chapter-06-cloud.md) | Cloud |
| [Chương 7](devops/chapter-07-monitoring.md) | Monitoring |

### Mobile & Testing

- **Mobile:** [`flutter.md`](mobile/flutter.md) · [`react-native.md`](mobile/react-native.md)
- **Testing:** [`be-testing.md`](testing/be-testing.md) · [`fe-testing.md`](testing/fe-testing.md)
- **DSA:** [`CTDL-On-Nhanh.md`](dsa/CTDL-On-Nhanh.md)

## Định dạng tài liệu

- Mỗi chương là một file Markdown (`.md`).
- Sơ đồ **mermaid** được render trên GitHub và hầu hết editor Markdown.
- Có **mục lục** đầu mỗi chương để dễ điều hướng.

## Cách đóng góp

- Bổ sung hoặc chỉnh sửa nội dung trong thư mục chủ đề tương ứng.
- Giữ nguyên cấu trúc: tiêu đề chương, mục lục, nội dung theo mục.
- Commit với message mô tả rõ nội dung thay đổi.
