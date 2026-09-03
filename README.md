# Software Engineering Handbook

Bộ tài liệu tự học **Software Engineer / Fullstack Developer** bằng tiếng Việt, ưu tiên kiến thức thực chiến và khả năng áp dụng vào dự án. Nội dung trải từ nền tảng lập trình, frontend/backend, testing và DevOps đến cấu trúc dữ liệu, mobile và tài liệu theo dự án.

## Mục lục tổng quan

| Nhóm tài liệu | Nội dung chính |
| --- | --- |
| [Backend](backend/) | Nền tảng backend, JavaScript/TypeScript, Node.js, Express, NestJS, database, security, system design, tools |
| [Java & Spring Boot](backend/java-spring-boot/) | Java Core → Spring Framework → Spring Boot → JPA → Security → Testing |
| [Frontend](frontend/) | JavaScript, TypeScript, Browser, React, Next.js, CSS, state, form, auth, realtime, testing |
| [Angular](frontend/angular/) | Angular từ nền tảng đến SSR/deployment |
| [Testing](testing/) | Lộ trình kiểm thử tổng quát; frontend/backend testing chuyên biệt |
| [DevOps](devops/) | Linux, Docker, CI/CD, reverse proxy, deployment, cloud, monitoring |
| [DSA](dsa/) | Cấu trúc dữ liệu, thuật toán, dynamic programming và luyện phỏng vấn |
| [Mobile](mobile/) | Flutter và React Native |
| [Product Management](product%20management/) | Jira, Scrum, workflow, backlog, sprint và quản lý team |
| [Project](project/) | Tech stack, luồng nghiệp vụ, code example và câu hỏi phỏng vấn theo dự án |

## Lộ trình đề xuất

### Hướng Fullstack JavaScript/TypeScript

1. [JavaScript chuyên sâu](backend/languages%20%26%20frameworks/javascript-chuyen-sau.md) và [các kiểu TypeScript](backend/languages%20%26%20frameworks/typescript-cac-kieu-thuong-dung.md).
2. [JavaScript runtime & Node.js](backend/languages%20%26%20frameworks/javascript-runtime-va-nodejs-chuyen-sau.md) → [ExpressJS](backend/languages%20%26%20frameworks/expressjs-chuyen-sau.md) hoặc [NestJS](backend/languages%20%26%20frameworks/nestjs-chuyen-sau.md).
3. Học [Frontend](#frontend), sau đó [Testing](#testing) và [DevOps](#devops).
4. Bổ sung [DSA](#dsa) và tài liệu [Project](project/) để luyện phỏng vấn/thực chiến.

### Hướng Java & Spring Boot

1. Bắt đầu từ [lộ trình Java & Spring Boot](backend/java-spring-boot/chuong-00-java-initerary.md).
2. Học tuần tự Java Core → Java nâng cao → Spring Framework → Spring Boot → JPA → Security → Testing.
3. Tham khảo [code guide](backend/java-spring-boot/code-guide/00-GUIDE_INDEX.md) khi cần triển khai dự án.

## Backend

### Nền tảng và kiến trúc

| Tài liệu | Nội dung |
| --- | --- |
| [10 khái niệm backend cơ bản](backend/10-khai-niem-backend-co-ban.md) | Tổng quan các khái niệm backend |
| [Nền tảng lập trình](backend/base/nen-tang-lap-trinh-chuyen-sau.md) | Kiến thức lập trình cốt lõi |
| [OOP và 4 tính chất](backend/base/oop-va-4-tinh-chat-chuyen-sau.md) | Lập trình hướng đối tượng |
| [Networking](backend/networking/networking.md) | Networking cho backend |
| [ORM: Prisma và TypeORM](backend/database/orm-prisma-vs-typeorm-chuyen-sau.md) | Lựa chọn và sử dụng ORM |
| [Git](backend/git/git.md) · [Git Flow](backend/git/git-flow.md) | Quy trình làm việc với Git |

### Ngôn ngữ và framework

| Tài liệu | Nội dung |
| --- | --- |
| [JavaScript chuyên sâu](backend/languages%20%26%20frameworks/javascript-chuyen-sau.md) | Hoisting, scope, TDZ, closure, `this`, callback, Promise, async/await |
| [JavaScript runtime & Node.js](backend/languages%20%26%20frameworks/javascript-runtime-va-nodejs-chuyen-sau.md) | Event loop, runtime và Node.js |
| [TypeScript chuyên sâu](backend/languages%20%26%20frameworks/typescript-chuyen-sau.md) | Cơ chế hoạt động của TypeScript |
| [Các kiểu TypeScript thường dùng](backend/languages%20%26%20frameworks/typescript-cac-kieu-thuong-dung.md) | Primitive, object, union, generic, utility type và nhiều hơn |
| [ExpressJS chuyên sâu](backend/languages%20%26%20frameworks/expressjs-chuyen-sau.md) | Xây dựng API với Express |
| [NestJS Core](backend/languages%20%26%20frameworks/nestjs-core.md) · [NestJS chuyên sâu](backend/languages%20%26%20frameworks/nestjs-chuyen-sau.md) | Framework NestJS |

### System design, bảo mật và công cụ

- [System design](backend/system%20design/): API, caching Redis, design patterns, upload file, message queue, SOLID, scheduling, webhook, websocket và hiệu năng.
- [Security](backend/security/): bảo mật backend và resilience patterns.
- [Tools](backend/tools/): BullMQ, Jest, MongoDB, OAuth2 Google, Prisma, Redis, Swagger, TypeORM, Winston và Socket.
- [Backend testing](backend/testing/be-testing.md): kiểm thử backend.

### Java & Spring Boot

| Chương | Nội dung |
| --- | --- |
| [Chương 0](backend/java-spring-boot/chuong-00-java-initerary.md) | Lộ trình Java & Spring Boot |
| [Chương 1](backend/java-spring-boot/chuong-01-java-core.md) | Java Core |
| [Chương 2](backend/java-spring-boot/chuong-02-java-advanced.md) | Java nâng cao |
| [Chương 3](backend/java-spring-boot/chuong-03-spring-framework.md) | Spring Framework |
| [Chương 4](backend/java-spring-boot/chuong-04-spring-boot.md) | Spring Boot |
| [Chương 5](backend/java-spring-boot/chuong-05-database-jpa.md) | Database & JPA |
| [Chương 6](backend/java-spring-boot/chuong-06-security.md) | Security |
| [Chương 7](backend/java-spring-boot/chuong-07-testing.md) | Testing |
| [Chương 8](backend/java-spring-boot/chuong-08-advanced-common-features.md) | Chủ đề nâng cao |

## Frontend

| Chương | Nội dung |
| --- | --- |
| [Chương 1](frontend/chuong-01-javascript.md) | JavaScript |
| [Chương 2](frontend/chuong-02-typescript.md) | TypeScript |
| [Chương 3](frontend/chuong-03-browser.md) | Browser |
| [Chương 4](frontend/chuong-04-react.md) | React |
| [Chương 5](frontend/chuong-05-nextjs.md) | Next.js |
| [Chương 6](frontend/chuong-06-css.md) | CSS |
| [Chương 7](frontend/chuong-07-state-management.md) | State management |
| [Chương 8](frontend/chuong-08-server-state.md) | Server state |
| [Chương 9](frontend/chuong-09-form-handling.md) | Xử lý form |
| [Chương 10](frontend/chuong-10-networking.md) | Networking |
| [Chương 11](frontend/chuong-11-auth.md) | Authentication |
| [Chương 12](frontend/chuong-12-realtime.md) | Realtime |
| [Chương 13](frontend/chuong-13-design-patterns.md) | Design patterns |
| [Chương 14](frontend/chuong-14-testing.md) | Testing |

- [Frontend tools](frontend/tools/): Axios, React Hooks, Redux, SEO, TanStack Query, Zod và Zustand.
- [Lộ trình Angular](frontend/angular/): 12 chương từ nền tảng Angular đến SSR/deployment.

## Testing

- [Lộ trình testing](testing/): 15 chương về kiểm thử.
- [Frontend testing](testing/fe-testing.md) và [backend testing](backend/testing/be-testing.md).

## DevOps

[Linux](devops/chuong-01-linux.md) · [Docker](devops/chapter-02-docker.md) · [CI/CD](devops/chapter-03-cicd.md) · [Reverse Proxy](devops/chapter-04-reverse-proxy.md) · [Deployment](devops/chapter-05-deployment.md) · [Cloud](devops/chapter-06-cloud.md) · [Monitoring](devops/chapter-07-monitoring.md)

## DSA

Lộ trình gồm [nền tảng và kỹ năng phỏng vấn](dsa/chuong-00-nen-tang-va-ky-nang-phong-van.md), các cấu trúc dữ liệu/thuật toán phổ biến, dynamic programming, bit manipulation, nhận diện dạng bài và template giải bài.

- [Mục lục DSA đầy đủ](dsa/muc-luc.md)
- [Ôn nhanh](dsa/on-nhanh.md)
- [Lời giải contest](dsa/contest_solutions.md)

## Mobile

- [Flutter](mobile/flutter/): Dart, widget, navigation, state management, API, testing, performance và deployment.
- [React Native](mobile/react-native/): components, navigation, state/data fetching, native capabilities, animation, performance, testing và publish.

## Đóng góp

- Đặt tài liệu vào thư mục chủ đề phù hợp và dùng tên file rõ nghĩa, viết thường, phân cách bằng dấu gạch nối.
- Cập nhật README khi thêm một tài liệu/lộ trình quan trọng.
- Dùng Markdown, tiêu đề rõ ràng và ví dụ code có thể đọc độc lập.
- Kiểm tra các liên kết nội bộ trước khi commit.
