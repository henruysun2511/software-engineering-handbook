# Phần 1: Tạo dự án

## 1. Kiểm tra Java

```bash
java -version
javac -version
```

## 2. Kiểm tra Maven Wrapper

```bash
.\mvnw.cmd -version
```

## 3. Cài dependency (lần đầu hoặc sau khi sửa pom.xml)

```bash
.\mvnw.cmd clean install
```

Hoặc chỉ tải dependency mà không build:

```bash
.\mvnw.cmd dependency:resolve
```

## 4. Chạy Spring Boot

```bash
.\mvnw.cmd spring-boot:run
```

## 5. Dừng server

Trong Terminal:

```
Ctrl + C
```

## 6. Clean project

```bash
.\mvnw.cmd clean
```

## 7. Build project

```bash
.\mvnw.cmd package
```

Sau khi build sẽ có:

```
target/
└── <project_name>-0.0.1-SNAPSHOT.jar
```

## 8. Chạy file JAR

Sau khi build:

```bash
java -jar target\<project_name>-0.0.1-SNAPSHOT.jar
```

## 9. Chạy test

```bash
.\mvnw.cmd test
```

## 10. Cập nhật dependency sau khi sửa pom.xml

```bash
.\mvnw.cmd clean install
```

Hoặc trong VS Code:

```
Ctrl + Shift + P
↓
Java: Clean Java Language Server Workspace
```

---

## Quy trình làm việc hằng ngày

### Lần đầu clone project

```bash
git clone <repo>

cd <project_name>

.\mvnw.cmd clean install

.\mvnw.cmd spring-boot:run
```

### Mỗi ngày code

```bash
git pull

.\mvnw.cmd spring-boot:run
```

### Khi thêm dependency vào pom.xml

```bash
.\mvnw.cmd clean install
```

### Khi project lỗi build

```bash
.\mvnw.cmd clean

.\mvnw.cmd clean install
```

### Khi chuẩn bị deploy

```bash
.\mvnw.cmd clean package
```

---

## Tương đương với Node.js

| Node.js                | Spring Boot (Maven)          |
| ---------------------- | ---------------------------- |
| `npm install`          | `.\mvnw.cmd clean install`   |
| `npm run dev`          | `.\mvnw.cmd spring-boot:run` |
| `npm test`             | `.\mvnw.cmd test`            |
| `npm run build`        | `.\mvnw.cmd package`         |
| `node dist/main.js`    | `java -jar target/*.jar`     |
| `rm -rf node_modules`  | `.\mvnw.cmd clean`           |

---

## Các lệnh bạn sẽ dùng nhiều nhất

```bash
# Kiểm tra Java
java -version
javac -version

# Kiểm tra Maven
.\mvnw.cmd -version

# Cài/cập nhật dependency
.\mvnw.cmd clean install

# Chạy project
.\mvnw.cmd spring-boot:run

# Build
.\mvnw.cmd package

# Chạy test
.\mvnw.cmd test

# Chạy file JAR sau khi build
java -jar target\<project_name>-0.0.1-SNAPSHOT.jar
```

Đối với quá trình học Spring Boot, 95% thời gian bạn chỉ cần sử dụng ba lệnh sau:

```bash
.\mvnw.cmd clean install      # Sau khi thêm hoặc cập nhật dependency
.\mvnw.cmd spring-boot:run    # Chạy ứng dụng
.\mvnw.cmd test               # Chạy test (khi bắt đầu viết test)
```
