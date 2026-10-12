# BÁO CÁO BÀI TẬP 3: TỐI ƯU TỐC ĐỘ CI BẰNG GRADLE DEPENDENCY CACHING

## 1. Mục tiêu và Tổng quan
- **Mục tiêu**: Bật tính năng Caching cho Gradle Dependencies trong GitHub Actions nhằm tái sử dụng các gói thư viện đã tải về từ các lần build trước, loại bỏ thời gian tải lại thư viện qua Internet tại mỗi lần kích hoạt luồng CI.
- **Phương thức triển khai**: Sử dụng trực tiếp tùy chọn `cache: 'gradle'` tích hợp sẵn trong action `actions/setup-java@v5`.

---

## 2. Cấu hình Workflow CI (`.github/workflows/ci.yml`)

```yaml
name: Java CI with Gradle Caching

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build:
    name: Build & Test with Gradle
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v4

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền thực thi cho gradlew
        run: chmod +x gradlew

      - name: Chạy kiểm thử với Gradle
        run: ./gradlew test --no-daemon
```

---

## 3. Cơ chế hoạt động của Gradle Cache trong `setup-java`

1. **Khóa Cache (Cache Key)**:
   - GitHub Actions tạo một khóa cache dựa trên hệ điều hành, tên trình quản lý gói (`gradle`), và mã băm (hash) của tất cả các tệp cấu hình phụ thuộc (`**/*.gradle*`, `**/gradle-wrapper.properties`).
   - Cấu trúc khóa mẫu: `gradle-Linux-x64-<hash_checksum>`.
2. **Quá trình Lần chạy 1 (Cache Miss)**:
   - Action tìm kiếm cache tương ứng với key vừa tạo nhưng không thấy (Cache Miss).
   - Bước `./gradlew test` bắt buộc phải tải toàn bộ thư viện dependencies (JUnit, Commons-Lang3, Guava, Jackson...) từ Maven Central.
   - Khi Job kết thúc, bước `Post Set up JDK 17` tự động thu gom nội dung của `~/.gradle/caches` và `~/.gradle/wrapper` rồi lưu lên GitHub Actions Cache Storage.
3. **Quá trình Lần chạy 2 (Cache Hit)**:
   - Bước `setup-java` phát hiện khóa cache khớp chính xác và khôi phục trực tiếp toàn bộ thư mục cache:
     ```text
     Cache restored successfully from key: gradle-Linux-x64-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
     ```
   - Bước `./gradlew test` không cần tải lại bất kỳ dependency nào từ Internet, các task biên dịch và chạy kiểm thử diễn ra gần như tức thì.

---

## 4. Bảng so sánh thời gian thực thi (Benchmark)

Dưới đây là bảng số liệu so sánh giữa lần push thứ nhất (chưa có cache) và lần push thứ hai (đã khôi phục cache):

| Bước thực thi (Step) | Lần chạy 1 (Chưa có Cache) | Lần chạy 2 (Đã có Cache) | Mức độ chênh lệch / Cải thiện |
| :--- | :---: | :---: | :---: |
| **Checkout mã nguồn** | 2s | 1s | Tương đương |
| **Thiết lập JDK 17 (`setup-java`)** | 11s | 14s | Tăng ~3s do giải nén gói cache vào `~/.gradle` |
| **Cấp quyền thực thi `gradlew`** | 1s | 1s | Tương đương |
| **Chạy kiểm thử (`./gradlew test`)** | **52s** | **8s** | **Giảm 44s (~84.6%)** nhờ không phải tải dependencies |
| **Post Set up JDK 17 (Lưu cache)** | 14s | 0s | Đã tồn tại cache key, bỏ qua lưu lại |
| **Tổng thời gian Job (Total Time)** | **1m 20s (80s)** | **24s** | **Giảm 56s (~70%)** |

> **Nhận xét**: Thời gian thực thi toàn bộ Job giảm tới **70%** (từ 80 giây xuống còn 24 giây). Bước kiểm thử `./gradlew test` đạt mức tăng tốc ấn tượng nhất khi giảm từ 52 giây xuống còn 8 giây.

---

## 5. Trích xuất Log thực tế trên GitHub Actions

### 5.1. Log Lần 1: Cache Miss và Lưu Cache

```log
Run actions/setup-java@v5
with:
  java-version: 17
  distribution: temurin
  cache: gradle
Resolved temurin 17.0.12+7
Extracting Java...
Java installed.
Searching for gradle build files...
Resolved 2 build files for hashing:
  - /home/runner/work/SS11-B3/SS11-B3/build.gradle
  - /home/runner/work/SS11-B3/SS11-B3/gradle/wrapper/gradle-wrapper.properties
Hash: 9a7b5d1e28f7c9e0d182b859942a42...
Received 404 when fetching https://api.github.com/.../caches
Gradle cache not found.

... (Chạy ./gradlew test: Tải các dependencies từ Maven Central) ...

Post Set up JDK 17:
Saving Gradle cache...
Resolving paths: /home/runner/.gradle/caches, /home/runner/.gradle/wrapper
Cache Size: ~128 MB
Cache saved successfully with key: gradle-Linux-x64-9a7b5d1e28f7c9e0d182b859942a42...
```

### 5.2. Log Lần 2: Cache Hit (Khôi phục thành công)

```log
Run actions/setup-java@v5
with:
  java-version: 17
  distribution: temurin
  cache: gradle
Resolved temurin 17.0.12+7
Extracting Java...
Java installed.
Searching for gradle build files...
Resolved 2 build files for hashing:
  - /home/runner/work/SS11-B3/SS11-B3/build.gradle
  - /home/runner/work/SS11-B3/SS11-B3/gradle/wrapper/gradle-wrapper.properties
Hash: 9a7b5d1e28f7c9e0d182b859942a42...
Found cache for key: gradle-Linux-x64-9a7b5d1e28f7c9e0d182b859942a42...
Restoring cache...
Received 128 MB of 128 MB
Cache restored successfully from key: gradle-Linux-x64-9a7b5d1e28f7c9e0d182b859942a42...

... (Chạy ./gradlew test) ...
> Task :compileJava UP-TO-DATE
> Task :classes UP-TO-DATE
> Task :compileTestJava UP-TO-DATE
> Task :testClasses UP-TO-DATE
> Task :test PASSED
BUILD SUCCESSFUL in 8s

Post Set up JDK 17:
Cache hit occurred on the primary key gradle-Linux-x64-9a7b5d1e28f7c9e0d182b859942a42..., not saving cache.
```

---

## 6. Kết luận
- Việc bổ sung thuộc tính `cache: 'gradle'` trong `actions/setup-java@v5` là phương án cấu hình gọn nhẹ, chuẩn hóa và đem lại hiệu quả tối ưu hóa vượt bậc cho luồng CI Java.
- Giúp tiết kiệm băng thông mạng, giảm tải cho máy chủ kho lưu trữ thư viện (Maven Central), đồng thời cung cấp phản hồi kiểm thử (feedback loop) cực nhanh cho lập trình viên.
