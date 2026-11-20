# BÁO CÁO KỸ THUẬT: XÂY DỰNG PIPELINE CI/CD KẾT NỐI POSTGRESQL (SESSION 9 - BÀI 3)

## 1. Bối cảnh & Mục tiêu nghiệp vụ
- **Dự án:** Quản lý nhân sự (`employee-service` / QuickBite HR Platform).
- **Vấn đề thực tế:** Các lập trình viên thường xuyên đẩy code gây lỗi hệ thống do không kiểm thử kỹ tầng truy vấn cơ sở dữ liệu (Database query & ORM/JPA mapping).
- **Giải pháp:** Thiết lập pipeline CI/CD tự động gồm 2 giai đoạn tuần tự:
  1. **Stage `test`:** Khởi chạy một container PostgreSQL tạm thời qua từ khóa `services`, nạp cấu hình tài khoản/CSDL qua `variables`, và chạy Integration Test bằng `./gradlew test`.
  2. **Stage `build`:** Chỉ khi toàn bộ bài test vượt qua, stage này mới được kích hoạt để đóng gói ứng dụng thành file JAR (`./gradlew build -x test`) và lưu trữ kết quả qua `artifacts` với hạn sử dụng `1 day`.

---

## 2. Toàn văn cấu hình `.gitlab-ci.yml` chuẩn xác

```yaml
# ==============================================================================
# GitLab CI/CD Pipeline - Quản lý nhân sự (Employee Service)
# Bài tập 3: Xây dựng Pipeline CI/CD kết nối PostgreSQL (Vận dụng chuyên sâu)
# ==============================================================================

# 1. Định nghĩa 2 stages tuần tự trong quy trình CI/CD
stages:
  - test
  - build

# 2. Cấu hình biến môi trường toàn cục cho Database PostgreSQL
variables:
  POSTGRES_DB: hr_db
  POSTGRES_USER: hr_user
  POSTGRES_PASSWORD: hr_password
  POSTGRES_HOST_AUTH_METHOD: trust

# ------------------------------------------------------------------------------
# Stage 1: Kiểm thử tích hợp tự động với PostgreSQL Database
# ------------------------------------------------------------------------------
test_job:
  stage: test
  image: gradle:7.6-jdk17
  # Khai báo service PostgreSQL chạy song song với môi trường kiểm thử
  services:
    - postgres:14-alpine
  variables:
    # URL kết nối từ container test tới service container PostgreSQL qua hostname 'postgres'
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/hr_db
    SPRING_DATASOURCE_USERNAME: hr_user
    SPRING_DATASOURCE_PASSWORD: hr_password
    SPRING_DATASOURCE_DRIVER: org.postgresql.Driver
  before_script:
    - chmod +x ./gradlew
  script:
    - echo "================================================================"
    - echo "===> Stage 1: Chạy Unit & Integration Tests với PostgreSQL..."
    - echo "================================================================"
    - ./gradlew test

# ------------------------------------------------------------------------------
# Stage 2: Đóng gói ứng dụng Spring Boot (Chỉ thực thi khi Stage test PASS)
# ------------------------------------------------------------------------------
build_job:
  stage: build
  image: gradle:7.6-jdk17
  before_script:
    - chmod +x ./gradlew
  script:
    - echo "================================================================"
    - echo "===> Stage 2: Đóng gói ứng dụng thành file JAR..."
    - echo "================================================================"
    - ./gradlew build -x test
  # Cấu hình lưu trữ kết quả build dưới dạng artifacts trong 1 ngày
  artifacts:
    name: "employee-service-build-$CI_COMMIT_SHORT_SHA"
    paths:
      - build/libs/*.jar
    expire_in: 1 day
```

---

## 3. Phân tích chuyên sâu các thành phần kỹ thuật

### 3.1. Cơ chế hoạt động của `services` trong GitLab CI
- **Định nghĩa:** Từ khóa `services` cho phép GitLab Runner Docker executor khởi tạo một hoặc nhiều container phụ trợ (auxiliary/sidecar containers) song song với container thực thi job chính.
- **Phân giải địa chỉ mạng (Networking & DNS):**
  - GitLab Runner tự động tạo một Docker Bridge network riêng biệt cho từng job.
  - Tên hostname nội bộ của service được tự động phân giải dựa trên tên image sau khi loại bỏ tag:
    - Khai báo image: `postgres:14-alpine` $\rightarrow$ Tên hostname truy cập trong mạng: `postgres`.
    - Do đó, chuỗi kết nối JDBC từ container `gradle:7.6-jdk17` gọi sang Database là: `jdbc:postgresql://postgres:5432/hr_db`.
- **Vòng đời (Lifecycle):** Container PostgreSQL được tạo ra khi job bắt đầu và bị xóa bỏ hoàn toàn ngay khi job kết thúc, đảm bảo tính sạch sẽ (stateless) và không lưu trữ rác dữ liệu giữa các lần chạy CI.

### 3.2. Cấu hình các biến môi trường (`variables`) cho PostgreSQL
- Container `postgres:14-alpine` khi khởi động sẽ đọc các biến môi trường chuẩn của PostgreSQL Docker Official Image:
  - `POSTGRES_DB: hr_db`: Tự động khởi tạo schema database có tên `hr_db`.
  - `POSTGRES_USER: hr_user`: Tạo tài khoản database người dùng `hr_user`.
  - `POSTGRES_PASSWORD: hr_password`: Thiết lập mật khẩu cho người dùng.
- Đồng thời, Spring Boot hỗ trợ nạp cấu hình thông qua biến môi trường (`SPRING_DATASOURCE_*`), giúp ứng dụng tự động nhận diện và trỏ trực tiếp vào service database trong môi trường CI mà không cần sửa code.

### 3.3. Tính tuần tự của các Stage (`stages: - test - build`)
- Thứ tự các phần tử trong block `stages` quyết định thứ tự thực thi của pipeline.
- Mọi job thuộc stage `build` chỉ được lập lịch và chạy sau khi tất cả các job thuộc stage `test` thành công (exit code 0).
- Nếu bất kỳ bài kiểm thử nào trong `./gradlew test` thất bại, pipeline sẽ dừng ngay lập tức tại stage 1 và đánh dấu trạng thái **FAILED**. Nhờ đó, mã nguồn lỗi không bao giờ được đóng gói hoặc triển khai lên môi trường tiếp theo.

### 3.4. Quản lý kết quả đầu ra với `artifacts` và `expire_in`
- **`paths: - build/libs/*.jar`**: Chỉ định cho GitLab Runner thu thập file thực thi `.jar` sinh ra sau lệnh `./gradlew build -x test`.
- **`expire_in: 1 day`**: Quy định thời gian lưu trữ artifact trên máy chủ là 1 ngày. Sau 24 giờ, hệ thống sẽ tự động dọn dẹp file này nhằm tiết kiệm không gian lưu trữ đĩa cứng cho hệ thống CI/CD.

---

## 4. Hướng dẫn nộp bài lên Portal
1. **Link GitHub nộp bài:**
   `https://github.com/NguyenDuong1509/Session9-BTVN-Devops-B3`
2. **Nội dung file `.gitlab-ci.yml`:** Copy trực tiếp nội dung file `.gitlab-ci.yml` vào form nộp bài theo yêu cầu.
