# QuickBite Microservices - Employee Service (Quản lý nhân sự)

> **Session 9 - Bài tập 3:** Xây dựng Pipeline CI/CD kết nối PostgreSQL (Vận dụng chuyên sâu)

## 📌 Tổng quan dự án
Dự án `employee-service` thuộc nền tảng Microservices của QuickBite, quản lý thông tin nhân viên, phòng ban và dữ liệu tiền lương. Dự án sử dụng **Spring Boot 3.2.3**, **Java 17**, **Spring Data JPA**, và cơ sở dữ liệu **PostgreSQL 14**.

---

## 🚀 Cấu trúc Pipeline CI/CD (`.gitlab-ci.yml`)

Pipeline được thiết kế gồm 2 giai đoạn tuần tự:
1. **Stage 1 (`test`):**
   - Sử dụng Docker image `gradle:7.6-jdk17`.
   - Sử dụng từ khóa `services` để dựng container `postgres:14-alpine`.
   - Cấu hình các biến môi trường: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`.
   - Thực thi `./gradlew test` kiểm thử toàn bộ logic nghiệp vụ và truy vấn CSDL.
2. **Stage 2 (`build`):**
   - Chỉ được kích hoạt khi Stage 1 hoàn thành thành công.
   - Thực thi `./gradlew build -x test` để đóng gói file JAR.
   - Khai báo `artifacts` lưu trữ `build/libs/*.jar` với hạn sử dụng `expire_in: 1 day`.

---

## 🛠 Hướng dẫn chạy kiểm thử tại máy Local

### 1. Chạy Unit/Integration Test:
```bash
./gradlew test
```

### 2. Đóng gói ứng dụng:
```bash
./gradlew build -x test
```
File JAR sau khi build sẽ nằm tại: `build/libs/employee-service-1.0.0.jar`.

---

## 📸 Minh chứng hoàn thành
- [Báo cáo kỹ thuật chi tiết](file:///EXPLANATION.md)
- [Ảnh kiến trúc Pipeline](file:///homework/session_09/exercise_03/gitlab_ci_pipeline_architecture.png)
- [Ảnh minh chứng thực thi thành công](file:///homework/session_09/exercise_03/gitlab_ci_execution_proof.png)
